# HW_Coroutines-Scopes-Cancellation-Supervision

1. Вопросы: Cancellation

    Ответ 1.1
Строка `println("ok")` не отработает.

    Ответ 1.2
Строка `println("ok")` первой вложенной корутины (`child`) не будет выполнена из-за явной отмены.

2. Вопросы: Exception Handling

    Ответ 2.1
Строка `e.printStackTrace()` не отработает.

    Ответ 2.2
Да, строка `e.printStackTrace()` в этом коде отработает

    Ответ 2.3
Строка `e.printStackTrace()` будет выполнена

    Ответ 2.4
Cтрока `throw Exception("something bad happened")` внутри первого `launch` не будет достигнута, потому что корутина будет отменена в момент `delay` из-за выброшенного исключения во втором `launch`.

    Ответ 2.5
обе строки `throw Exception("something bad happened")` будут исполнены

    Ответ 2.6
Нет, не отработает.

    Ответ 2.7
Cтрока `println("ok")` не отработает, так как произойдет завершение JVM-процесса из-за не отработанного исключения.

### Пояснения к ответу 1.1
Поскольку `job` была отменена через 100 миллисекунд, а вложенные корутины должны были ждать 500 миллисекунд до выполнения `println("ok")`, они также будут отменены вместе с родительской `job`, не успев достигнуть строки `println("ok")`.
Таким образом, ни одно “ok” не будет напечатано.

### Пояснения к ответу 1.2
Однако, вторая вложенная корутина `launch { delay(500); println("ok") }` продолжит свое выполнение и, поскольку ее не отменяют, она напечатает “ok”.
Итого, будет напечатано одно “ok” (от второй вложенной корутины).

### Пояснения к ответу 2.1
Блок `try-catch` расположен вокруг вызова `launch`, а не внутри корутины. Это означает, что он пытается перехватить исключения, которые происходят непосредственно при вызове `launch`, а не исключения, которые генерируются внутри корутины позже.
Чтобы перехватить исключение из корутины, нужно использовать механизмы, предназначенные для обработки исключений в корутинах, такие как `CoroutineExceptionHandler` или поместить `try-catch` непосредственно внутри блока `launch`.

### Пояснения к ответу 2.2

`coroutineScope { throw Exception("something bad happened") }:` Внутри блока try вы вызываете функцию `coroutineScope`.

`coroutineScope` — это так называемый `building-block coroutine builder`, который создает новый дочерний скоуп. Важно, что он перебрасывает исключения вверх по иерархии корутинов, а не перехватывает их.

`throw Exception("something bad happened"):` Внутри `coroutineScope` выбрасывается исключение.
Так как `coroutineScope` не обрабатывает исключение, оно пробрасывается “наружу” в родительский `launch` корутин.

Родительский `launch` корутин содержит блок `try-catch`, который перехватывает это исключение.
`catch (e: Exception) { e.printStackTrace() }`: Когда исключение перехвачено, выполняется код внутри блока `catch`, а именно `e.printStackTrace()`

### Пояснения к ответу 2.3
Особенность `supervisorScope` в том, что он создает `Supervisors Job`, который не отменяет свои дочерние элементы при возникновении исключения в одном из них. Однако, исключение, брошенное внутри `supervisorScope`, все равно будет распространяться вверх по иерархии корутин, если оно не будет перехвачено ниже.
В вашем случае, исключение `Exception("something bad happened")`, брошенное внутри `supervisorScope`, распространится до ближайшего `try-catch` блока. Этим блоком является `try-catch` в родительской корутине.

Поэтому, `catch (e: Exception)` перехватит это исключение, и строка `e.printStackTrace()` будет выполнена.

### Пояснения к ответу 2.4
Первое исключение: Корутина во втором `launch` выбросит исключение практически сразу же. Как только это произойдет, `coroutineScope` это обнаружит.
`coroutineScope` немедленно отменит первый `launch` (который в этот момент находится в состоянии `delay`).
Затем `coroutineScop`e сам выбросит это исключение.
`try-catch`: Внешний блок `try-catch` перехватит это исключение, которое распространилось от `coroutineScope`.

### Пояснения к ответу 2.5
Когда используется `supervisorScope`, ошибки в дочерних корутинах не приводят к отмене всех остальных дочерних корутин внутри этого `supervisorScope`. Каждая дочерняя корутина обрабатывает свои исключения независимо.

### Пояснения к ответу 2.6
Исключение выбрасывается в родительской корутине, которая запускает две дочерние корутины. Когда исключение выбрасывается, оно “проходит наверх” по иерархии корутин. Это означает, что родительская корутина завершает свою работу из-за исключения, и, как следствие, отменяет свои дочерние корутины.


### Пояснения к ответу 2.7
Ключевое отличие `SupervisorJob()` от обычного `Job()` заключается в том, что он не отменяет дочерние корутины, если одна из дочерних корутин завершается с ошибкой. То есть, если в одном из внутренних `launch` произойдет исключение, это не повлияет на другие дочерние корутины внутри того же `SupervisorJob()`,
но в то-же время так как исключение произошло в Корутине 2, которая является launch корутиной (не `async` с `await`), и у нее нет своего `CoroutineExceptionHandler`, это исключение становится неперехваченным.
`SupervisorJob` предотвращает отмену Корутины 3 (она продолжит `delay(1000)`).
Однако, Корутина 2 сама по себе отменяется. Исключение распространяется вверх от Корутины 2 к ее родителю, которым является Корутина 1.
Корутина 1 (запущенная `CoroutineScope(EmptyCoroutineContext).launch`) также не имеет `SupervisorJob` или `CoroutineExceptionHandler`. Поэтому, когда исключение всплывает до нее, Корутина 1 отменяется.
Поскольку Корутина 1 является корневой частью инициированной корутины, и исключение не перехвачено нигде по пути, оно в конце концов достигает потока, в котором выполняется main (`main thread`).
JVM завершает работу.


# Домашнее задание к занятию «10. Coroutines: Scopes, Cancellation, Supervision»

Выполненное задание прикрепите ссылкой на ваши GitHub-проекты в личном кабинете студента на сайте [netology.ru](https://netology.ru).

## Вопросы: Cancellation

### Вопрос №1

Отработает ли в этом коде строка `<--`? Поясните, почему да или нет.

```kotlin
fun main() = runBlocking {
    val job = CoroutineScope(EmptyCoroutineContext).launch {
        launch {
            delay(500)
            println("ok") // <--
        }
        launch {
            delay(500)
            println("ok")
        }
    }
    delay(100)
    job.cancelAndJoin()
}
```

### Вопрос №2

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() = runBlocking {
    val job = CoroutineScope(EmptyCoroutineContext).launch {
        val child = launch {
            delay(500)
            println("ok") // <--
        }
        launch {
            delay(500)
            println("ok")
        }
        delay(100)
        child.cancel()
    }
    delay(100)
    job.join()
}
```

## Вопросы: Exception Handling

### Вопрос №1

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    with(CoroutineScope(EmptyCoroutineContext)) {
        try {
            launch {
                throw Exception("something bad happened")
            }
        } catch (e: Exception) {
            e.printStackTrace() // <--
        }
    }
    Thread.sleep(1000)
}
```

### Вопрос №2

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        try {
            coroutineScope {
                throw Exception("something bad happened")
            }
        } catch (e: Exception) {
            e.printStackTrace() // <--
        }
    }
    Thread.sleep(1000)
}
```

### Вопрос №3

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        try {
            supervisorScope {
                throw Exception("something bad happened")
            }
        } catch (e: Exception) {
            e.printStackTrace() // <--
        }
    }
    Thread.sleep(1000)
}
```

### Вопрос №4

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        try {
            coroutineScope {
                launch {
                    delay(500)
                    throw Exception("something bad happened") // <--
                }
                launch {
                    throw Exception("something bad happened")
                }
            }
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }
    Thread.sleep(1000)
}
```

### Вопрос №5

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        try {
            supervisorScope {
                launch {
                    delay(500)
                    throw Exception("something bad happened") // <--
                }
                launch {
                    throw Exception("something bad happened")
                }
            }
        } catch (e: Exception) {
            e.printStackTrace() // <--
        }
    }
    Thread.sleep(1000)
}
```

### Вопрос №6

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        CoroutineScope(EmptyCoroutineContext).launch {
            launch {
                delay(1000)
                println("ok") // <--
            }
            launch {
                delay(500)
                println("ok")
            }
            throw Exception("something bad happened")
        }
    }
    Thread.sleep(1000)
}
```

### Вопрос №7

Отработает ли в этом коде строка `<--`. Поясните, почему да или нет.

```kotlin
fun main() {
    CoroutineScope(EmptyCoroutineContext).launch {
        CoroutineScope(EmptyCoroutineContext + SupervisorJob()).launch {
            launch {
                delay(1000)
                println("ok") // <--
            }
            launch {
                delay(500)
                println("ok")
            }
            throw Exception("something bad happened")
        }
    }
    Thread.sleep(1000)
}
```

