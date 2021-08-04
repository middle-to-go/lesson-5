---
customTheme : "presentation"

---

## Лекция 5. Асинхронные сущности и паттерны в Go
### OZON

### Москва, 2021


---

### Лекции

1. <span style="color:gray">Введение. Рабочее окружение. Структура программы. Инструментарий.</span>
2. <span style="color:gray">Базовые конструкции и операторы. Встроенные типы и структуры данных.</span>
3. <span style="color:gray">Структуры данных, отложенные вызовы, обработка ошибок и основы тестирования</span>
4. <span style="color:gray">Интерфейсы, моки и тестирование с ними</span>
5. Асинхронные сущности и паттерны в Go
6. <span style="color:gray">Protobuf и gRPC</span>
7. <span style="color:gray">Работа с БД в Go</span>
8. <span style="color:gray">Брокеры сообщений. Трассировка. Метрики. </span>

---

### Темы

Сегодня мы поговорим про:

1. Параллелизм и конкурентность<!-- .element: class="fragment" -->
1. Горутины <!-- .element: class="fragment" -->
1. Каналы <!-- .element: class="fragment" -->
1. Примитивы синхронизации <!-- .element: class="fragment" -->
1. Гонки приоритетов <!-- .element: class="fragment" -->
1. Контексты <!-- .element: class="fragment" -->


---

### Обозначения

* 📽️ - посмотри воркшоу
* ⚗️ - проведи эксперимент
* 🔬 - изучи внимательно
* 📖 - прочитай документация
* 🪙 - подумай о сложности
* 🐞 - запомни ошибку
* 🔨 - запомни решение
* 🏔️ - обойди камень предкновенья
* ⏰ - сделай перерыв
* 🏡 - попробуй дома
* 💡 - обсуди светлые идеи
* 🙋 - задай вопрос
* ⚡ - запомни панику

---

### Параллелизм и конкурентность

- Параллелизм -- это исполнение задач параллельно на физическом уровне, то есть когда у каждого параллельного процесса есть свой набор ресурсов, которые не разделяются. Аналогия: параллельные прямые. <!-- .element: class="fragment" -->
- Конкурентность -- это исполнение задач параллельно на логическом уровне. То есть когда мы можем управлять общими ресурсами так, чтобы задачи эффективно их использовали.  <!-- .element: class="fragment" -->

⚡ В русскоязычной литературе эти два термина путаются. Смотрите на определения выше. <!-- .element: class="fragment" -->

---

### Параллелизм и конкурентность

<img src="/images/lesson_5/parallel_and_concurency.png" />

source: https://joearms.github.io/#Index

---

### Горутины

Goroutine — это, по сути, Coroutine, но это Go, поэтому мы заменяем букву C на G и получаем слово Goroutine. <!-- .element: class="fragment" -->

Как и в ОС в Go есть свой планировщик, который определяет когда какая горутина выполняется: <!-- .element: class="fragment" -->
1. В Go <= 1.13 был кооперативный планировщик, т.е. каждая горутина исполнялась, пока не подходила к месту, где можно переключить контекст исполнения на другую горутину (системные вызовы, мьютексы и т.д.) <!-- .element: class="fragment" -->
2. С Go 1.14 планировщик стал вытесняющим. То есть он может переключить контекст тогда, когда захочет. Его поведение недетерминировано. <!-- .element: class="fragment" -->
 
🏡 https://habr.com/ru/post/489862/ <!-- .element: class="fragment" -->

---

### Горутины

🏡 Сравни как будет исполняться на Go 1.13 и Go >= 1.14.

```
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    runtime.GOMAXPROCS(1)

    fmt.Println("The program starts ...")

    go func() {
        for {
        }
    }()

    time.Sleep(time.Second)
    fmt.Println("I got scheduled!")
}
```

---

### Горутины

Планировщик может переключать контекст в следующих ситуациях:
* Использование ключевого слова go <!-- .element: class="fragment" -->
* Cборщик мусора <!-- .element: class="fragment" -->
* Системные вызовы <!-- .element: class="fragment" -->
* Синхронизация <!-- .element: class="fragment" -->

Но не обязательно, что он это сделает. <!-- .element: class="fragment" -->

---

### Горутины

<img src="/images/lesson_5/books.jpg" />

---

### Запуск горутины

Каждая горутина имеет свой стек, но всё остальное она делит с системой.

```go
go func(trolley Trolley) {
    burnBooks(trolley)
}(trolley)
```

```go
go func() {
    burnBooks(trolley)
}()
```

```go
go burnBooks(trolley)
```

```go
go trolley.Load(pile)
```

---

### Сколько тут горутин?

```
func main() {
    fmt.Printf(
        "Goroutines: %d",
        runtime.NumGoroutine(),
    )
}
```

🏡 ⚗️ Когда количество горутин будет меняться?

---

### Что напечатает программа?

N.b.: "что напечатает программа?", это примерно то же самое, что и "что хотел сказать автор?".

```go
func main() {
    go fmt.Printf("Hello")
}
```

---

### Замыкание

Замыкание это когда внутренняя функция имеет доступ к переменным родительской функции, даже после того как родительская функция выполнена.

```go
func main() {
	for i := 0; i < 5; i++ {
		go func() {
			fmt.Print(i)
		}()
	}
	time.Sleep(2 * time.Second)
}
```
🏡 ⚗️ А что если `runtime.GOMAXPROCS(1)`?


---

### Какой адрес будет у i?

```go
func main() {
	for i := 0; i < 5; i++ {
		go func() {
			fmt.Printf("%v, %T, %v\n", i, i, &i)
		}()
	}
	time.Sleep(2 * time.Second)
}
```


---

### Каналы

```go
chan T
```

* работают с конкретным типом <!-- .element: class="fragment" -->
* потокобезопасны <!-- .element: class="fragment" -->
* похожи на очереди FIFO <!-- .element: class="fragment" -->

---

### Операции над каналами

```go

ch := make(chan int) // создать канал
ch <- 10 // записать в канал
v := <-ch // прочитать из канала
close(ch) // закрыть канал

```

🏡 https://habr.com/ru/post/308070/


---

### Небуферизированные каналы

```go
    ch := make(chan int)
```
<img src="/images/lesson_5/chan_1.png" />

---

### Буферизированные каналы

```go
ch := make(chan int, 4)
```
<img src="/images/lesson_5/chan_2.png" />

---

### Чему равен буфер небуферизированного канала?

```go
    ch := make(chan int, ?)
```

```go
    func main() {
        ch := make(chan int)
        fmt.Println(cap(ch))
    }
```

---

### Что будет, если читать из пустого канала?

<img  src="/images/lesson_5/chan_3.png" />

---

### Что будет, если читать из пустого канала?

<img  src="/images/lesson_5/chan_4.png" />

---

### Что будет, если писать в закрытый канал?

<img  src="/images/lesson_5/chan_5.png" />

---

### Что будет, если писать в закрытый канал?

<img  src="/images/lesson_5/chan_6.png" />

---

### Что будет, если читать из закрытого канала?

<img  src="/images/lesson_5/chan_7.png" />

---

### Что будет, если читать из закрытого канала?

<img  src="/images/lesson_5/chan_8.png" />

---

### Что будет, если читать из пустого закрытого канала?

<img  src="/images/lesson_5/chan_9.png" />

---

### Что будет, если читать из пустого закрытого канала?

<img  src="/images/lesson_5/chan_10.png" />

---

### Синхронизация горутин каналами

```go

func main() {
    var ch = make(chan struct{})
    go func() {
        fmt.Printf("Hello")
        ch <- struct{}{}
    }()
    <-ch
}
```

```go
func main() {
    var ch = make(chan struct{})
    go func() {
        fmt.Printf("Hello")
        <-ch
    }()
    ch <- struct{}{}
}
```

---

### Работа с каналами

* Чтение из канала, пока он не закрыт

```go
v, ok := <-ch // значение и флаг "открытости" канала
```

Producer:
```go
for _, t := range tasks {
    ch <- t
}
close(ch)
```
Consumer:

```go
for {
    x, ok := <-ch
    if !ok {
        break
    }
    fmt.Println(x)
}
```

---

### Работа с каналами

Producer:

```go
for _, t := range tasks {
    ch <- t
}
close(ch)
```

Consumer:
```go
for x := range ch {
    fmt.Println(x)
}
```

---

### Правила закрытия канала

Кто закрывает канал?

- Канал закрывает тот, кто в него пишет. <!-- .element: class="fragment" -->
- Если несколько писателей, то тот, кто создал писателей и канал. <!-- .element: class="fragment" -->

---

### Каналы: однонаправленные

```go
    chan<- T // только запись
    <-chan T // только чтение
```

Что произойдет?

```go
func f(out chan<- int) {
    <-out
}
func main() {
    var ch = make(chan int)
    f(ch)
}
```

---

### Каналы: таймаут

```go
timer := time.NewTimer(10 * time.Second)
select {
case data := <-ch:
    fmt.Printf("received: %v", data)
case <-timer.C:
    fmt.Printf("failed to receive in 10s")
}
```

---

### Каналы: периодик

```go

ticker := time.NewTicker(10 * time.Second)
defer ticker.Stop()
for {
    select {
    case <-ticker.C:
        fmt.Printf("tick")
    case <-doneCh:
        return
    }
}
```

---

### Каналы: как сигналы

```go
make(chan struct{}, 1)
```

Источник сигнала:
```go
select {
    case notifyCh <- struct{}{}:
    default:
}
```

Приемник сигнала:

```go
select {
    case <-notifyCh:
    case ...
}
```

---

### Каналы: graceful shutdown

```go

interruptCh := make(chan os.Signal, 1)
signal.Notify(interruptCh, os.Interrupt, syscall.SIGTERM)
fmt.Printf("Got %v...\n", <-interruptCh)

```

---

### Примитивы синхронизации

1. sync.WaitGroup
2. sync.Once
3. sync.Mutex
4. sync.RWMutex

---

### sync.WaitGroup: какую проблему решает?

```go

func main() {
    const goCount = 5
    ch := make(chan struct{})
    for i := 0; i < goCount; i++ {
    go func() {
        fmt.Println("go-go-go")
        ch <- struct{}{}
        }()
    }
    for i := 0; i < goCount; i++ {
        <-ch
    }
}
```

---

### sync.WaitGroup: ожидание горутин

```go
func main() {
    const goCount = 5
    wg := sync.WaitGroup{}
    wg.Add(goCount) // <===
    for i := 0; i < goCount; i++ {
        go func() {
            fmt.Println("go-go-go")
            wg.Done() // <===
        }()
    }
    wg.Wait()
}

---

### sync.WaitGroup: ожидание горутин

```go
func main() {
    wg := sync.WaitGroup{}
    for i := 0; i < 5; i++ {
        wg.Add(1) // <===
        go func() {
            fmt.Println("go-go-go")
            wg.Done() // <===
        }()
    }
    wg.Wait()
}
```

---

### sync.WaitGroup: API

```go

type WaitGroup struct {
}

func (wg *WaitGroup) Add(delta int) // увеличивает счетчик WaitGroup.
func (wg *WaitGroup) Done() // уменьшает счетчик на 1.
func (wg *WaitGroup) Wait() // блокируется, пока счетчик WaitGroup не обнулится.
```

---

### sync.Once: какую проблему решает?

```go

func main() {
    var once sync.Once
    onceBody := func() {
        fmt.Println("Only once")
    }
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            once.Do(onceBody)
            wg.Done()
        }()
    }
    wg.Wait()
}
```

---

### sync.Once: ленивая инициализация (пример)

```
type List struct {
    once sync.Once
...
}

func (l *List) PushFront(v interface{}) {
    l.init()
...
}

func (l *List) init() {
    l.once.Do(func() {
    ...
    })
}
```

---

### sync.Once: синглтон (пример)

```go

type singleton struct {
}

var instance *singleton
var once sync.Once

func GetInstance() *singleton {
    once.Do(func() {
        instance = &singleton{}
    })
    return instance
}
```

---

### sync.Mutex: какую проблему решает?

```go

func main() {
    wg := sync.WaitGroup{}
    v := 0
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            v++
            wg.Done()
        }()
    }
    wg.Wait()
    fmt.Println(v)
}
```

---

### sync.Mutex

```go

$ GOMAXPROCS=1 go run mu.go
1000
$ GOMAXPROCS=4 go run mu.go
947
$ GOMAXPROCS=4 go run mu.go
956
```

---

### sync.Mutex

```go 

func main() {
    wg := sync.WaitGroup{}
    mu := sync.Mutex{}
    v := 0
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            mu.Lock() // <===
            v++
            mu.Unlock() // <===
            wg.Done()
        }()
    }
    wg.Wait()
    fmt.Println(v)
}
```

---

### sync.Mutex: паттерны использования

Помещайте мьютекс выше тех полей, доступ к которым он будет защищать

```
var sum struct {
    mu sync.Mutex // <=== этот мьютекс защищает
    i int // <=== поле под ним
}
```

Используйте defer, если есть несколько точек выхода
```
func doSomething() {
    mu.Lock()
    defer mu.Unlock()
    err := ...
    if err != nil {
        return // <===
    }
    err = ...
    if err != nil {
        return // <===
    }
    return // <===
}
```

---

### Копирование мьютексов

```go

type Container struct {
	sync.Mutex
	counters map[string]int
}

func (c Container) inc(name string) {
	c.Lock()
	defer c.Unlock()
	c.counters[name]++
}

func main() {
	c := Container{counters: map[string]int{"a": 0, "b": 0}}

	doIncrement := func(name string, n int) {
		for i := 0; i < n; i++ {
			c.inc(name)
		}
	}

	go doIncrement("a", 100000)
	go doIncrement("a", 100000)

	// Wait a bit for the goroutines to finish
	time.Sleep(300 * time.Millisecond)
	fmt.Println(c.counters)
}

```

---

### Дедлоки

<img  src="/images/lesson_5/deadlock.png" />

---

### Гонки приоритетов

В чем проблема, кроме неопределенного поведения?

```go
func main() {
    wg := sync.WaitGroup{}
    text := ""
    wg.Add(2)
    go func() {
        text = "hello world"
        wg.Done()
    }()
    go func() {
        fmt.Println(text)
        wg.Done()
    }()
    wg.Wait()
}
```

---

### Определение гонок

```go

$ go test -race mypkg
$ go run -race mysrc.go
$ go build -race mycmd
$ go install -race mypkg

```

---

### Определение гонок

Ограничение race детектора:

```go
func main() {
    for i := 0; i < 10000; i++ {
        go func() {
            time.Sleep(time.Second)
        }()
    }
    time.Sleep(time.Second)
}
```

Можно исключить тесты:

```
// +build !race
package foo
// The test contains a data race. See issue 123.
func TestFoo(t *testing.T) {
// ...
}
```

---

### Context

> A Context carries a deadline, a cancellation signal, and other values across API boundaries

```go
type Context interface {
  Deadline() (deadline time.Time, ok bool)
  Done() <-chan struct{}
  Err() error
  Value(key interface{}) interface{}
}
```

```go
package context
​
var Canceled = errors.New("context canceled")
​
var DeadlineExceeded error = deadlineExceededError{}
// Stream generates values with DoSomething and sends them to out
// until DoSomething returns an error or ctx.Done is closed.
func Stream(ctx context.Context, out chan<- Value) error {
  for {
    v, err := DoSomething(ctx)
    if err != nil {
      return err
    }
    select {
    case <-ctx.Done():
      return ctx.Err()
    case out <- v:
    }
  }
}
```

---

### Context

Контексты

```go
context.Background()
context.TODO()
```

- Background: возвращает ненулевой пустой контекст. Он никогда не отменяется, не имеет никаких значений и не имеет крайнего срока. Обычно он используется основной функцией, инициализацией и тестами, а также в качестве контекста верхнего уровня для входящих запросов.

- TODO: возвращает ненулевой пустой контекст. Код должен использовать `context.TODO()`, когда неясно, какой контекст использовать, или он еще недоступен (поскольку окружающая функция еще не была расширена для приема параметра контекста).

---

### Context: WithTimeout

WithTimeout:
```go
    ctx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()

    resp, err := client.DescribeTaskV1(ctx, req)
```

---

### Context: WithCancel

```
gen := func(ctx context.Context) <-chan int {
    dst := make(chan int)
    n := 1
    go func() {
      for {
        select {
        case <-ctx.Done():
          return // returning not to leak the goroutine
        case dst <- n:
          n++
        }
      }
    }()
    return dst
  }
​
  ctx, cancel := context.WithCancel(context.Background())
  defer cancel() // cancel when we are finished consuming integers
​
  for n := range gen(ctx) {
    fmt.Println(n)
    if n == 5 {
      break
    }
  }
```

---

### Context: WithDeadline

```go
    return WithDeadline(parent, time.Now().Add(timeout))
```

---

### Context: WithValue

```go
package user
​
import "context"
​
type User struct {...}
​
type key int
​
var userKey key
​
func NewContext(ctx context.Context, u *User) context.Context {
    return context.WithValue(ctx, userKey, u)
}
​
func FromContext(ctx context.Context) (*User, bool) {
  u, ok := ctx.Value(userKey).(*User)
  return u, ok
}
```

---

### Context: metadata

Метаданные - частный случай для WithValue

Получение:
```go
import "google.golang.org/grpc/metadata"
​
var user string
meta, found := metadata.FromIncomingContext(ctx)
if found {
  for key, values := range meta {
    if !strings.Contains(key, "ocp") {
      continue
    }
    if len(values) == 0 {
      continue
    }
    value := values[0]
    span.SetTag(key, value)
  }
}
```

---

### Context: metadata

Отправка:

```go
md := metadata.New(map[string]string{
  "key1": "val1",
  "key2": "val2",
})
md := metadata.Pairs(
    "key1", "val1",
    "key1", "val1-2", // "key1" will have map value []string{"val1", "val1-2"}
    "key2", "val2",
)
ctx := metadata.NewOutgoingContext(context.Background(), md)
```

---

### Context: metadata

Что из себя представляет метаданные 🔬

```go
type MD map[string][]string
​
func FromIncomingContext(ctx context.Context) (md MD, ok bool) {
  md, ok = ctx.Value(mdIncomingKey{}).(MD)
  return
}
```

> HTTP Authorization header is added as authorization gRPC request header

> HTTP headers that start with 'Grpc-Metadata-' are mapped to gRPC metadata (prefixed with grpcgateway-)