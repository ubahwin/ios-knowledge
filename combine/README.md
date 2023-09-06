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

**`Just`**

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

**`Result`**

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

## Subscriber'ы

**Subscriber** – подписчик, связывается с publisher и следит за изменениями.

`.sink()` из примера выше и есть подписчик

```swift
func sink(receiveValue: @escaping ((Self.Output) -> Void)) -> AnyCancellable
```

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

## Operator

**Operator** – промежуток между паблишером и подписчиком, обрабатывают данные.

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
