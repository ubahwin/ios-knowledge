## Содержание

- [Combine](#combine)
  - [Publisher'ы](#publisherы)
    - [Just](#just)
    - [Optional](#optional)
    - [Result](#result)
    - [Future](#future)
    - [Subject](#subject)
    - [Empty](#empty)
    - [@Published](#published)
    - [NotificationCenter, URLSession, Timer](#notificationcenter-urlsession-timer)
    - [Чистка типов паблишеров](#чистка-типов-паблишеров)
  - [Subscriber'ы](#subscriberы)
    - [Методы для подписок](#методы-для-подписок)
    - [AnyCancellable](#anycancellable)
  - [Operator](#operator)

# Combine

`Combine` – инструмент от Apple высокого уровня абстракции для асинхронной работы с данными, основан на *реактивном* подходе.

В основе *реактивного программирования* лежит паттерн проектирования **Observer**.

**Data Stream** – поток данных `Publisher -> Operator -> Subscriber`.

## Publisher'ы

**Publisher** – издатель, уведомляет об изменениях

> Publisher может отправить только 3 вида данных:
> - Сами данные
> - Ошибка
> - Сигнал о завершении работы

**По выдаче событий:**
- *One-shot* – выдают данные один раз.
- *Continuous broadcasting* – выдают данные несколько раз.

**По реакции на подписчиков:**
- *Hot* – отправляют данные, даже если некому слушать.
- *Cold* – отправляют данные, когда есть подписчик.

**По типу данных:**

|          | One-shot | Continuous broadcasting |
|----------|----------  |----------|
| **Hot**  | `Future`           | `Subject`, `Timer` `NotificationCenter`   |
| **Cold** | `Just`, `Empty`, `Optional`, `Result`, `URLSession`  | `Sequence`, `Timer`   |

### Just

Генерирует событие один раз, никогда не возвращает ошибку (`Never`)

```swift
let publisher: Just<String> = Just("Event!")

publisher
    .sink(
        receiveCompletion: { completion in
            print("completion status: \(completion)")
        }, receiveValue: { value in
            print("value: \(value)")
        }
    )
```
Вывод:
```
value: Event!
completion status: finished
```

### Optional

Как и Just, только при значении `nil` ничего не генерирует

### Result

Генерирует событие один раз, может вернуть ошибку.

```swift
let value = 10

var result: Result<Int, Error> = .success(value)

let publisher: Result<Int, Error>.Publisher = result.publisher

publisher
    .sink(
        receiveCompletion: { completion in
            switch completion {
                case .finished:
                    print("completion status: \(completion)")
                case .failure(let error):
                    print("error: \(error)")
            }
        }, receiveValue: { value in
            print("value: \(value)")
        }
    )
```

`Optional` и `Result` уже есть и без Combine, Combine расширяет эти типы, делая их паблишерами через `.Publisher` (например, `Optional<Int>.Publisher`)

### Future

Как `Result`, только заворачивается под closure типа

```swift
func fetchData(completion: @escaping (Result<Int, Error>) -> Void) { ... }
```

Таким образом:

```swift
func fetchDataAsPublisher() -> Future<Int, Error> {
    Future { promise in 
        fetchData { completion in 
            promise(completion)
        }
    }
}
```

### Subject

Как `Future`, только оборачивается в свойства, а не методы. А ещё он `Continuous broadcasting`.
Имеет две имплементации:

**1. PassthroughSubject**

```swift
let subject = PassthroughSubject<Int, Never>()

subject.send(1)

subject
    .sink( 
        receiveCompletion: { completion in
            print(completion)
        },
        receiveValue: { value in
            print(value)
        }
     )

subject.send(2)

subject.send(completion: .finished)
```

Будет вывод:

```
4
finished
```

Данные генерируются только после того, как появился подписчик.

**2. CurrentValueSubject**

Как и прошлый, только хранит ещё и **последнее** отправленное значение **до** подписки, и **все** значения **после** подписки.

### Empty

Никогда не генерирует события.

### @Published

Чаще используется в [SwiftUI](/swiftui/README.md)

### NotificationCenter, URLSession, Timer

`NotificationCenter`, `Timer`, `URLSession` существуют в Foundation, Combine их расширяет, добавляя возможность быть паблишером.

Примеры реализации:

**NotificationCenter**

```swift
func subscribeForNotification() {
    NotificationCenter.default
        .publisher(for: UIApplication.didEnterBackgroundNotification)
        .sink { notification in
            print(notification.name.rawValue)
        }
        .store(in: &bag)
}
```

**Timer**

```swift
let timer = Timer.publish(every: 1, on: .main, in: .default)
timer
    .autoconnect() // автозапуск при подписке
    .sink { secondsLeft in
        print(secondsLeft)
    }
    .store(in: &bag)
```

**URLSession**

```swift
func loadData() -> AnyPublisher<[String], Never> {
    URLSession.shared.dataTaskPublisher(for: url)
        .map { $0.data }
        .decode(type: [String].self, decoder: JSONDecoder())
        .replaceError(with: [])
        .subscribe(on: DispatchQueue.main)
        .receive(on: DispatchQueue.main)
        .eraseToAnyPublisher()
}
```

### Чистка типов паблишеров

У паблишеров свои типы данных и с учетом *строгой типизации* с ними не удобно работать.

Можно стереть тип методом `.eraseToAnyPublisher()`

```swift
let publisher = Future<Int, Never> { _ in }
    .eraseToAnyPublisher()
```

После этого, типом паблишера становится `AnyPublisher<Int, Never>`

## Subscriber'ы

**Subscriber** – подписчик, связывается с publisher и следит за изменениями.

### Методы для подписок

`.sink()` из примера выше и есть подписчик

```swift
func sink(receiveValue: @escaping ((Self.Output) -> Void)) -> AnyCancellable
```

Конкретно эта реализация `.sink()` работает только с паблишерами, которые никогда не вернут ошибку (`Failure == Never`), однако есть `.sink()` который работает со всеми паблишерами:

```swift
func sink(receiveCompletion: @escaping ((Subscribers.Completion<Self.Failure>) -> Void), receiveValue: @escaping ((Self.Output) -> Void)) -> AnyCancellable
```

Блок `receiveCompletion` работает при `one-shot`, либо `.send(completion:)`, либо если заранее известно количество элементов.

`.assign()` присваивает значение паблишера в свойство другого объекта:

```swift
class Person {
    var disease: String?
}

class Hospital {
    let publisher = PassthroughSubject<String?, Never>()
    var bag = Set<AnyCancellable>()
    let person = Person()
    
    func makeDiagnosisWithAssing() {
        publisher
            .assign(to: \.disease, on: person)
            .store(in: &bag)
    }
    func makeDiagnosisWithSink() {
        publisher
            .sink { [weak self] value in
                self?.person.disease = value
            }
            .store(in: &bag)
    }
}
```

Выше представлена, для примера, реализация присваивания диагноза человеку в больнице. Методы делают одно и то же, только один через `sink`, а другой `assign`. 

`assign` работает только с паблишерами `Failure == Never`. Работает только ссылочными данными, если бы `Person` был `struct` была бы ошибка.

Есть ещё `.assing(to:)`, она работает только с паблишерами `@Published`:

```swift
class ViewModel: ObservableObject {
    @Published var lastUpdated: Date = Date()

    init() {
         Timer.publish(every: 1.0, on: .main, in: .common)
             .autoconnect()
             .assign(to: &$lastUpdated)
    }
}
```

### AnyCancellable
Паблишер всегда возвращает объект, соответствующий протоколу `Cancellable`.
`AnyCancellable` позволяет сохранять подписку в коллекцию этого типа, чтобы избежать утечки памяти, например, при закрытии экрана, и для жизни подписчика за пределами scope.
Сохранять в `AnyCancellable` можно через `.store(in:)`.

```swift
var cancellable: [AnyCancellable] = []

publisher
    .sink { receivedValue in
        print(receivedValue)
    }
    .store(in: &cancellable)
```

У `AnyCancellable` есть ещё метод `.cancel()`, он отменят подписку. По дефолту `AnyCancellable` делает это сам.

## Operator

**Operator** – промежуток между паблишером и подписчиком, обрабатывают данные.

[Все операторы](https://developer.apple.com/documentation/combine/publishers-merge-publisher-operators)

Например, `.map()`:

```swift
let arrayPublisher = [1, 2, 3].publisher
arrayPublisher
    .map { initialValue in
        String(initialValue)
    }
    .sink { ... }
    .store(in: ...)
```
Оператор переводит `[Int]` в `[String]`

Оператор `.handleEvents()` предоставляет более тонкую обработку событий: 

```swift
.handleEvents(
    receiveSubscription: { subscription in },
    receiveOutput: { value in },
    receiveCompletion: { completion in },
    receiveCancel: { },
    receiveRequest: { demand in }
)
```
