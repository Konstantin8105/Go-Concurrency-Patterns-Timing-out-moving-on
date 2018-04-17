# Паттерны параллельного(concurrency) программирования в Go: таймаут и переходы
23 Sep 2010

Andrew Gerrand

Tags: concurrency, technical

## Источник

Перевод документа https://blog.golang.org/go-concurrency-patterns-timing-out-and

Версия Go:
```
go version go1.8.3 linux/amd64
```

## Introduction

Параллельное программирование имеет свои идиомы. Хорошим примером является время ожидания(timeout). Хотя каналы в Go не поддерживают их напрямую, но их легко реализовать. Скажем, мы хотим получить из канала `ch`, но хотим ждать не более одной секунды для получения значения. Мы начнем с создания канала и запуска горутины, который будет спать перед отправкой в канал:

```golang
	    timeout := make(chan bool, 1) // Создание буферизованного канала с размером 1
	    go func() {
	        time.Sleep(1 * time.Second) // Горутина спит перед отправкой
	        timeout <- true
	    }()
```

Затем мы можем использовать оператор `select` для получения из каналов `ch` или `timeout`. Если ничего не приходит в канал `ch` через одну секунду, выбирается тайм-аут, и попытка чтения из канала ch прекращается.

```golang
	    select {
	    case <-ch:
			// чтение из канала ch
	    case <-timeout:
			// чтение из канала ch не происходит, т.к. происходит таймаут
	    }
```
Канал `timeout` буферизуется для хранение 1 значения, позволяя `timeout` горутине отправлять на канал и затем выходить. Горотина не знает (или не заботится) о том, получено ли значение. Это означает, что горутина не будет висеть всегда, если «ch» получит событие до истечения таймаута. Канал «тайм-аут» в конечном итоге будет освобожден сборщиком мусора.

(В данном примере мы использовали `time.Sleep` для демонстрации работы горутин и каналов. В реальных программах вы должны использовать (`time.After`)[http://golang.org/pkg/time/#After], функция, которая возвращает канал и отправляет по этому каналу после указанной продолжительности.)

Let's look at another variation of this pattern. In this example we have a program that reads from multiple replicated databases simultaneously. The program needs only one of the answers, and it should accept the answer that arrives first.

The function `Query` takes a slice of database connections and a `query` string. It queries each of the databases in parallel and returns the first response it receives:
```golang
	func Query(conns []Conn, query string) Result {
	    ch := make(chan Result)
	    for _, conn := range conns {
	        go func(c Conn) {
	            select {
	            case ch <- c.DoQuery(query):
	            default:
	            }
	        }(conn)
	    }
	    return <-ch
	}
```
In this example, the closure does a non-blocking send, which it achieves by using the send operation in `select` statement with a `default` case. If the send cannot go through immediately the default case will be selected. Making the send non-blocking guarantees that none of the goroutines launched in the loop will hang around. However, if the result arrives before the main function has made it to the receive, the send could fail since no one is ready.

This problem is a textbook example of what is known as a [[https://en.wikipedia.org/wiki/Race_condition][race condition]], but the fix is trivial. We just make sure to buffer the channel `ch` (by adding the buffer length as the second argument to [[http://golang.org/pkg/builtin/#make][make]]), guaranteeing that the first send has a place to put the value. This ensures the send will always succeed, and the first value to arrive will be retrieved regardless of the order of execution.

These two examples demonstrate the simplicity with which Go can express complex interactions between goroutines.
