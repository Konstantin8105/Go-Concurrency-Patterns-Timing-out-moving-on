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

(В данном примере мы использовали [`time.Sleep`](http://golang.org/pkg/time/#Sleep) для демонстрации работы горутин и каналов. В реальных программах вы должны использовать [`time.After`](http://golang.org/pkg/time/#After), функция, которая возвращает канал и отправляет по этому каналу после указанной продолжительности.)

Давайте рассмотрим ещё один вариант данного шаблона. В этом примере у нас есть программа, которая одновременно считывает данные из нескольких реплицируемых баз данных. Программа нуждается только в одном из ответов, и она должна принять ответ, который приходит первым.

Функция `Query` принимает срез(slice) соединений с базой данных и строку` query`. Он запрашивает каждую из баз данных параллельно и возвращает первый ответ, который он получает:
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
В этом примере закрытие делает неблокирующую отправку, которую она достигает, используя операцию отправки в операторе `select` со значением ` default`. Если отправка не может пройти сразу, будет выбран случай по умолчанию. Выполнение отправки без блокировки гарантирует, что ни один из запущенных в цикле горутин не будет висеть. Однако, если результат доходит до того, как основная функция добралась до приема, отправка может завершиться неудачей, так как никто не готов.

Эта проблема представляет собой пример учебника того, что известно как [`race`](https://en.wikipedia.org/wiki/Race_condition), но исправить просто. Мы просто закачаем канал `ch` (добавив длину буфера в качестве второго аргумента в [make](http://golang.org/pkg/builtin/#make)), гарантируем, что первая отправка имеет место, чтобы положить значение. Это гарантирует, что отправка всегда будет успешной, и первое полученное значение будет получено независимо от порядка выполнения.

*После добавления буферизованного канала (добавлено при переводе)*
```golang
	func Query(conns []Conn, query string) Result {
	    ch := make(chan Result, 1)
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

Эти два примера демонстрируют простоту, с которой Go может решить сложные взаимодействия между гортанами.
