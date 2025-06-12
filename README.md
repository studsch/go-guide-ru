# Uber Go style guide

Перевод [uber go guide][uber-go-guide].

> [!NOTE]
> Перевод соответствует [версии статьи от 09.05.2023][uber-go-guide-changelog].

> [!IMPORTANT]
> Перевод не является дословным: некоторые, по мнению автора, очевидные моменты были опущены.
> Также в тексте могут встречаться авторские пометки.\
> Оформление не следует [стилевым рекмоендациям оригинальной статьи][uber-go-guide-contributing].

## Guidelines

### Pointers to Interfaces

Интерфейсы необходимо передавать по значению

### Verify Interface Compliance

```go
// Bad
type Handler struct {
	// ...
}

func (h *Handler) ServeHTTP(
	w http.ResponseWriter,
	r *http.Request,
) {
	// ...
}
```

```go
// Good
type Handler struct {
  // ...
}

var _ http.Handler = (*Handler)(nil)

func (h *Handler) ServeHTTP(
  w http.ReponseWriter,
  r *http.Request,
) {
  // ...
}
```

Строка `var _ http.Handler = (*Handler)(nil)` вызовет ошибку во время компиляции
если `*Handler` перестанет подходит под `http.Handler` интерфейс.

### Receivers and Interfaces

Методы с _value receivers_ (не знаю как это можно корректно перевести) могут
вызываться как на указателях, так и на значениях. Методы с _pointer receivers_
могут быть вызваны на указателях или адресных значений.

```go
type S struct {
	data string
}

func (s S) Read() string {
	return s.data
}

func (s *S) Write(str string) {
	s.data = str
}

// Нельзя получить указатель для значения в map, так как это не
// адресное значение.
sVals := map[int]S{1: {"A"}}

// Можем вызывать Read потому, что Read имеет value receiver.
sVals[1].Read()

// Нельзя вызвать Write для значения в map потому, что Write
// имеет pointer receiver. Это невозможно получить указатель на
// значение в map.
//
//	sVals[1].Write("test") <- сломает

sPtrs := map[int]*S{1: {"A"}}

// Можно вызывать и Read и Write, если в map указатели.
sPtrs[1].Read()
sPtrs[1].Write("test")
```

### Zero-value Mutexes are Valid

Нулевое значение `sync.Mutex` и `sync.RWMutex` валидны.

```go
// Bad
mu := new(sync.Mutex)
mu.Lock()
```

```go
// Good
var mu sync.Mutex
mu.Lock()
```

Если использовать структуру по указателю, то мьютекс должен быть полем без
указателя в ней. Не стоит встраивать мьютекс в структуру, даже если структура не
экспортируется.

```go
// Bad
type SMap struct {
	sync.Mutex

	data map[string]string
}

func NewSMap() *SMap {
	return &SMap{
		data: make(map[string]string),
	}
}

func (m *SMap) Get(k string) string {
	m.Lock()
	defer m.Unlock()

	return m.data[k]
}
// Поле мьютекса, Lock и Unlock методы являются частью экспортируемого
// API SMap.
```

```go
// Good
type SMap struct {
	mu sync.Mutex

	data map[string]string
}

func NewSMap() *SMap {
	return &SMap{
		data: make(map[string]string),
	}
}

func (m *SMap) Get(k string) string {
	m.mu.Lock()
	defer m.mu.Unlock()

	return m.data[k]
}
// Мьютекс и его методы являются деталями реализации SMap и они скрыты
// от вызывающих его функций.
```

### Copy Slices and Maps at Boundaries

Слайсы и словари содержат указатели на базовые данные.

#### Receiving Slices and Maps

Пользователь может изменить словарь или слайс, которые были получены в качестве
аргумента, если хранить ссылки на них.

```go
// Bad
func (d *Driver) SetTrips(trips []Trips) {
	d.trips = trips
}

trips := ...
d1.SetTrips(trips)

// Изменения коснутся и d1.trips.
trips[0] = ...
```

```go
// Good
func (d *Driver) SetTrips(trips []Trip) {
	d.trips = make([]Trip, len(trips))
	copy(d.trips,trips)
}

trips := ...
d1.SetTrips(trips)

// Можно изменять trips[0] без влияния на d1.trips.
trips[0] = ...
```

#### Returning Slices and Maps

Аналогично с изменениями пользовательских словарей и срезов. Стоит возвращать
копию в таких случаях.

```go
// Bad
type Stats struct {
	mu sync.Mutex
	counters map[string]int
}

// Snapshot возвращает текущее состояние статистики.
func (s *Stats) Snapshot() map[string]int {
	s.mu.Lock()
	defer s.mu.Unlock()

	return s.counters
}

// snapshot больше не защищен мьютексом, любой 
// доступ к snapshot подвержен к гонке данных.
snapshot := stats.Snapshot()
```

```go
// Good
type Stats struct {
	mu sync.Mutex
	counters map[string]int
}

// Snapshot возвращает текущее состояние статистики.
func (s *Stats) Snapshot() map[string]int {
	s.mu.Lock()
	defer s.mu.Unlock()

	result := make(map[string]int, len(s.counters))
	for k, v := range s.counters {
		result[k] = v
	}
	return result
}

// snapshot в данном случае копия.
snapshot := stats.Snapshot()
```

### Defer to Clean Up

Нужно использовать `defer` для очистки ресурсов, таких как файлы и блокировки.

```go
// Bad
p.Lock()
if p.count < 10 {
	p.Unlock()
	return p.count
}

p.count++
newCount := p.count
p.Unlock()

return newCount

// легко пропустить разблокировку из-за множества return.
```

```go
// Good
p.Lock()
defer p.Unlock()

if p.count < 10 {
	return p.count
}

p.count++
return p.count

// более читаемо.
```

### Channel Size is One or None

Каналы в основном должны иметь размер 1 или быть небуферезированными.

```go
// Bad
// Должно быть достаточно для любых целей!
c := make(chan int, 64)
```

```go
// Good
// Размер 1.
c := make(chan int, 1) // или
// Небуферезированный канал, размер 0.
c := make(chan int)
```

### Start Enums at One

Обычно следует начинать перечисления с ненулевого значения.

```go
// Bad
type Operation int

const (
	Add Operation = iota
	Subtract
	Multiply
)

// Add=0, Subtract=1, Multiply=2
```

```go
// Good
type Operation int

const (
	Add Operation = iota + 1
	Subtract
	Multiply
)

// Add=1, Subtract=2, Multiply=3
```

Существуют случаи, когда использование старта с 0 имеет смысл, когда этот случай
является желаемым поведением.

```go
type LogOutput int

const (
	LogToStdout LogOutput = iota
	LogToFile
	LogToRemote
)

// LogToStdout=0, LogToFile=1, LogToRemote=2
```

### Use `"time"` to handle time

Время - это сложная тема. Например, добавление 24 часов к моменту времени не
всегда приведет к новому календарному дню. Поэтому стоит всегда использовать
пакет `"time"` при работе со временем, так как он помогает решать эти проблемы
более безопасно и точно.

#### Use `time.Time` for instants of time

Стоит использовать `time.Time` при работе со временем, и использовать методы,
которые предоставляет этот пакет.

```go
// Bad
func isActive(now, start, stop int) bool {
	return start <= now && now < stop
}
```

```go
// Good
func isActive(now, start, stop time.Time) bool {
	return (start.Before(now) || start.Equal(now)) && now.Before(stop)
}
```

#### Use `time.Duration` for periods of time

Стоит использовать `time.Duration` при работе с периодами времени.

```go
// Bad
func poll(delay int) {
	for {
		// ...
		time.Sleep(time.Duration(delay) * time.Millisecond)
	}
}

poll(10) // не понятно это секунды или миллисекунды
```

```go
// Good
func poll(delay time.Duration) {
	for {
		// ...
		time.Sleep(delay)
	}
}

poll(10*time.Second)
```

Возвращаясь к примеру с добавлением 24 часов к моменту времени, метод, который
используется для добавления времени зависит от намерения. Если нужно получить то
же время суток, но на следующий календарный день, то нужно использовать
`Time.AddDate`. Если нужен момент времени, который гарантированно будет через 24
часа, то нужно `Time.Add`.

```go
newDay := t.AddDate(0 /* years */, 0 /* months */, 1 /* days */)
maybeNewDay := t.Add(24 * time.Hour)
```

#### Use `time.Time` and `time.Duration` with external systems

Стоит использовать `time.Time` и `time.Duration` при работе с внешними
системами, когда это возможно. Например:

- Флаги командной строки (`time.ParseDuration`).
- JSON (`UnmarshallJSON` method).
- SQL (`database/sql` поддерживает конвертацию `DATETIME` или `TIMESTAMP` в
  `time.Time` и обратно).
- YAML (`time.ParseDuration`).

Когда невозможно использовать `time.Duration`, то необходимо использовать `int`
или `float64` и включить единицу измерения в название поля. Например, поскольку
пакет `encoding/json` не поддерживает `time.Duration`, единица измерения должна
быть включена в название поля.

```go
// Bad
// {"interval": 2}
type Config struct {
	Interval int `json:"interval"`
}
```

```go
// Good
// {"intervalMillis": 2000}
type Config struct {
	IntervalMillis int `json:"intervalMillis"`
}
```

Когда нет возможности использовать `time.Time`, необходимо форматировать
временные метки в соответствии с определением RFC 3339. Этот формат используется
по умолчанию в `Time.UnmarshalText` и доступен для использования в `Time.Format`
и `time.Parse` через `time.RFC3339`.

Пакет `"time"` не поддерживает разбор временных меток с високосными секундами и
не учитывает високосные секунды в расчетах.

### Errors

#### Error Types

Существует несколько вариантов для объявления ошибок. Нужно определить следующие
аспекты перед выбором наиболее подходящего варианта:

- Нужно ли вызывающему коду сопоставлять ошибку, чтобы обработать ее? Если да,
  то нужно поддерживать функции `errors.Is` или `errors.As`, объявив переменную
  ошибки верхнего уровня или пользовательский тип.
- Является ли сообщение об ошибке статической или динамической строкой,
  требующей контекстную информацию? В первом случае можно использовать
  `errors.New`, во втором - нужно использовать `fmt.Errorf` или пользовательский
  тип ошибки.
- Нужно ли прокидывать новую ошибку, возвращаемую функцией нижнего уровня? Если
  да, то раздел о [Error Wrapping](#error-wrapping).

| Сопоставление ошибок? | Сообщение об ошибке | Рекомендация                         |
| --------------------- | ------------------- | ------------------------------------ |
| нет                   | статическое         | `errors.New`                         |
| нет                   | динамическое        | `fmt.Errorf`                         |
| да                    | статическое         | верхний уровень `var` с `errors.New` |
| да                    | динамическое        | пользовательский `error` тип         |

Например, использование `errors.New` для ошибок со статической строкой. Нужно
экспортировать эту ошибку как переменную, чтобы поддерживать сопоставление с
помощью `errors.Is`, если вызывающему коду нужно сопоставить и обработать эту
ошибку.

```go
// Не нужно сопоставлять ошибку.
// package foo

func Open() error {
	return errors.New("could not open")
}

// package bar

if err := foo.Open(); err != nil {
	// Нельзя обработать ошибку.
	panic("unknown error")
}
```

```go
// Сопоставление ошибки.
//package foo

var ErrCouldNotOpen = errors.New("could not open")

func Open() error {
	return ErrCouldNotOpen
}

// package bar

if err := foo.Open(); err != nil {
	if errors.Is(err, foo.ErrCouldNotOpen) {
		// Обработка ошибки.
	} else {
		panic("unknown error")
	}
}
```

Для ошибок с динамической строкой - `fmt.Errorf`, если вызывающему коду не нужно
сопоставлять, и пользовательский тип ошибки.

```go
// Не нужно сопоставлять ошибку.
// package foo

func Open(file string) error {
	return fmt.Errorf("file %q not found", file)
}

// package bar

if err := foo.Open("testfile.txt"); err != nil {
	// Нельзя сопоставить ошибку.
	panic("unknown error")
}
```

```go
// Сопоставление ошибки
// package foo

type NotFoundError struct {
	File string
}

func (e *NotFoundError) Error() string {
	return fmt.Sprintf("file %q not found", e.File)
}

func Open(file string) error {
	return &NotFoundError{File: file}
}

// package bar

if err := foo.Open("testfile.txt"); err != nil {
	var notFound *NotFoundError
	if errors.As(err, &notFound) {
		// Обработка ошибки.
	} else {
		panic("unknown error")
	}
}
```

Стоит отметить, что если экспортировать переменные или типы ошибок из пакета,
они станут частью публичного API этого пакета.

#### Error Wrapping

Варианты для прокидывания ошибок, если вызов завершился неудачей:

- Вернуть оригинальную ошибку без изменения.
- Добавить контекст с помощью `fmt.Errorf` и `%w`.
- Добавить контекст с помощью `fmt.Errorf` и `%v`.

Если нет дополнительного контекста, то нужно вернуть оригинальную ошибку без
изменения. Подходит для случаев, когда сообщение об исходной ошибке содержит
достаточную информацию для отслеживания ее источника.

В противном случае необходимо добавить контекст к сообщению об ошибке, когда это
возможно, чтобы вместо неопределенной ошибки по типу "connection refused", была
более полезная ошибка: "call service foo: connection refused".

Использование `%w` и `%v` в `fmt.Errorf`:

- `%w`, если вызывающий код должен иметь доступ к основной ошибке. Это хорошее
  значение по умолчанию для большинства обернутых ошибок, но нужно быть
  осторожным потому, что вызывающий код может начать полагаться на это
  поведение. Для случаев когда обернутая ошибка является `var` или типом,
  следует документировать и тестировать эту часть как контракт функции.
- `%v`, чтобы скрыть основную ошибку. Вызывающий код не сможет сопоставить ее,
  но в будущем можно переключиться на `%w`, если в этом есть необходимость.

При добавлении контекста к возвращаемым ошибка следует сохранять контекст
кратким, избегая фраз подобных "failed to", которые указывают на очевидное и
накапливаются по мере продвижения ошибке вверх по стеку:

```go
// Bad
s, err := store.New()
if err != nil {
	return fmt.Errorf(
		"failed to create new store: %w", err)
}

// failed to x: failed to y: failed to create new store: the error
```

```go
// Good
s, err := store.New()
if err != nil {
	return fmt.Errorf(
		"new store: %w", err)
}

// x: y: new store: the error
```

Однако, как только ошибка отправляется в другую систему, должно быть ясно, что
сообщение является ошибкой (например, тег "err" или префикс "Failed" в логах).

Рекомендованная статья:
[Don't just check errors, handle them gracefully][article-on-error-handling].

#### Error Naming

Для ошибок, хранящихся в глобальных переменных, нужно использовать префикс `Err`
или `err` в зависимости от того, экспортируются они или нет. Эта рекомендация
заменяет правило
[Prefix Unexported Globals with _](#prefix-unexported-globals-with-_).

```go
var (
	// Следующие две ошибки экспортируются
	// для того, чтобы пользователи этого пакета могли сопоставить их
	// с помощью errors.Is.

	ErrBrokenLink = errors.New("link is broken")
	ErrCouldNotOpen = errors.New("could not open")

	// Эта ошибка не экспортируется, потому что
	// мы не хотим, делать ее частью нашего публичного API.
	// Тем не менее, мы можем использовать ее внутри пакета
	// с помощью errors.Is.

	errNotFound = errors.New("not found")
)
```

Для пользовательских типов ошибок нужно использовать суффикс `Error`.

```go
// Эта ошибка экспортируется для того, чтобы
// пользователи этого пакета могли сопоставить ее
// с помощью errors.As.

type NotFoundError struct {
	File string
}

func (e *NotFoundError) Error() string {
	return fmt.Sprintf("file %q not found", e.File)
}

// Эта ошибка не экспортируется, потому что
// мы не хотим делать ее частью публичного API.
// Тем не менее, мы можем использовать ее внутри пакета
// с помощью errors.As.

type resolveError struct {
	Path string
}

func (e *resolveError) Error() string {
	return fmt.Sprintf("resolve %q", e.Path)
}
```

#### Handle Errors Once

Когда вызывающий код получает ошибку от вызываемого, можно обработать ее разными
способами в зависимости от того, что он знает об ошибке.

- Если контракт вызываемого определяет конкретные ошибки, то сопоставить ошибки
  при помощи `errors.Is` или `errors.As` и обработка веток отдельно.
- Если ошибка восстановима, то ее логирование и _degrading gracefully_ (не знаю
  как перевести. Просто логирование, игнорирование?).
- Если ошибка представляет специфичное для домена состояние сбоя, то вернуть
  четко определенную ошибку.
- Возврат ошибки, либо [обернутой](#error-wrapping), либо в первоначальном виде.

В независимости от того, как вызывающий код обрабатывает ошибку, он должен быть
обрабатывать каждую ошибку только один раз. Он не должен логировать ее, а потом
возвращать ее, потому что его вызывающий код могут также обрабатывать эту
ошибку.

Различные случаи:

**Bad** Логирование и возврат ошибки. Вызывающие функции выше по стеку,
вероятно, предпримут аналогичные действия с ошибкой. Это приведет в большому
количеству записей в логе приложения с малой ценностью.

```go
u, err := getUser(id)
if err != nil {
	// BAD: смотреть описание выше.
	log.Printf("Could not get user %q: %v", id, err)
	return err
}
```

**Good** Оборачивание ошибки и ее возврат. Вызывающие функции выше по стеку
будут обрабатывать ошибку. Использование %w гарантирует, что они смогут
сопоставить ошибку с помощью `errors.Is` или `errors.As`, если это уместно.

```go
u, err := getUser(id)
if err != nil {
	return fmt.Errorf("get user %q: %w", id, err)
}
```

**Good** Логирование ошибки и _degrade gracefully_ (просто логирование,
игнорирование?). Если операция не является строго необходимой, можно обеспечить
сниженный, но непрерывный опыт, восстановившись после нее.

```go
if err := emitMetrics(); err != nil {
	// Невозможность записать метрики не должна
	// приводить к сбою приложения.
	log.Printf("Could not emit metrics: %v", err)
}
```

**Good** Сопоставить ошибку и _degrade gracefully_ (просто логирование,
игнорирование?). Если вызываемый код определяет конкретную ошибку в своем
контракте, и сбой можно восстановить, то нужно сопоставить эту ошибку и
обеспечить _degrade gracefully_ (просто логирование, игнорирование?). Для всех
остальных случаев обернуть ошибку и вернуть ее. Вызывающие функции выше по стеку
будут обрабатывать другие ошибки.

```go
tz, err := getUserTimeZone(id)
if err != nil {
	if errors.Is(err, ErrUserNotFound) {
		// Пользователя не существует. Использовать UTC.
		tz = time.UTC
	} else {
		return fmt.Errorf("get user %q: %w", id, err)
	}
}
```

### Handle Type Assertion Failures

При преобразовании типы с одним возвращаемым значением вызовет панику при
неправильном типе. Поэтому всегда нужно использовать идиому "comma ok".

```go
// Bad
t := i.(string)
```

```go
// Good
t, ok := i.(string)
if !ok {
	// обработка ошибки.
}
```

### Don't Panic

Код запущенный в производственной среде, должен избегать паник.Паники являются
основной причиной каскадных сбоев. Если возникает ошибка, функция должна вернуть
ошибку и позволить вызывающему коду решить, как ее обработать.

```go
// Bad
func run(args []string) {
	if len(args) == 0 {
		panic("an argument is required")
	}
	// ...
}

func main() {
	run(os.Args[1:])
}
```

```go
// Good
func run(args []string) error {
	if len(args) == 0 {
		return errors.New("an argument is required")
	}
	// ...
	return nil
}

func main() {
	if err := run(os.Args[1:]); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

Panic/recover не подходит в качестве стратегии обработки ошибок. Программа
должна паниковать только в случаях возникновения чего-то непоправимого, например
разыменовывание nil. Исключением является инициализация программы: серьезные
ошибки при запуске программы, которые должны привести к ее завершению могут
вызвать панику.

```go
var _statusTemplate = template.Must(template.New("name").Parse("_statusHTML"))
```

Даже в тестах предпочтительнее использовать `t.Fatal` или `t.FailNow` вместо
паник, чтобы гарантировать, что тест будет отмечен как провалившийся.

```go
// Bad
// func TestFoo(t *testing.T)

f, err := os.CreateTemp("", "test")
if err != nil {
	panic("failed to set up test")
}
```

```go
// Good
// func TestFoo(t *testing.T)

f, err := os.CreateTemp("", "test")
if err != nil {
	t.Fatal("failed to set up test")
}
```

### Use go.uber.org/atomic

Атомарные операции с пакетом sync/atomic работают с сырыми типами (`int32`,
`int64` и т.д.), поэтому легко забыть использовать атомарную операцию для чтения
или изменения переменных.

Пакет go.uber.org/atomic добавляет безопасность типов к этим операциями, скрывая
базовый тип. Также он включает удобный тип `atomic.Bool`.

```go
// Bad
type foo struct {
	running int32 // atomic
}

func (f* foo) start() {
	if atomic.SwapInt32(&f.running, 1) == 1 {
		// уже запущено...
		return
	}
	// запуск Foo
}

func (f *foo) isRunning() bool {
	return f.running == 1 // гонка!
}
```

```go
// Good
type foo struct {
	running atomic.Bool
}

func (f *foo) start() {
	if f.running.Swap(true) {
		// уже запущено...
		return
	}
	// запуск Foo
}

func (f *foo) isRunning() bool {
	return f.running.Load()
}
```

### Avoid Mutable Globals

Стоит избегать изменения глобальных переменных, вместо этого использовать
dependency injection. Это касается как указателей на функции, так и других типов
значений.

```go
// Bad
//sign.go

var _timeNow = time.Now

func sign(msg string) string {
	now := _timeNow()
	return signWithTime(msg, now)
}

// sign_test.go

func TestSign(t *testing.T) {
	oldTimeNow := _timeNow
	_timeNow = func() time.Time {
		return someFixedTime
	}
	defer func() { _timeNow = oldTimeNow }()

	assert.Equal(t, want, sign(give))
}
```

```go
// Good
// sign.go

type signer struct {
	now func() time.Time
}

func newSigner() *signer {
	return &signer{
		now: time.Now,
	}
}

func (s *signer) Sign(msg string) string {
	now := s.now()
	return signWithTime(msg, now)
}

// sign_test.go

func TestSigner(t *testing.T) {
	s := newSigner()
	s.now = func() time.Time {
		return someFixedTime
	}

	assert.Equal(t, want, s.Sign(give))
}
```

### Avoid Embedding Types in Public Structs

Эти встроенные типы раскрывают детали реализации, препятствуют эволюции типов и
затрудняют документацию.

Предположим, реализованы различные типы списков, используя общий `AbstractList`,
нужно избегать встраивания `AbstractList` в конкретные реализации списков.
Вместо этого нужно вручную написать только те методы для конкретного списка,
которые будут делегировать вызовы абстрактному списку.

```go
type AbstractList struct {}

// Add adds an entity to the list.
func (l *AbstractList) Add(e Entity) {
	// ...
}

// Remove removes an entity from the list.
func (l *AbstractList) Remove(e Entity) {
	// ...
}
```

```go
// Bad
// ConcreateList is a list of entities.
type ConcreateList struct {
	*AbstractList
}
```

```go
// Good
// ConcreateList is a list of entities.
type ConcreateList struct {
	list *AbstractList
}

// Add adds an entity to the list.
func (l *ConcreateList) Add(e Entity) {
	l.list.Add(e)
}

// Remove removes an entity from the list.
func (l *ConcreateList) Remove(e Entity) {
	l.list.Remove(e)
}
```

Go позволяет встраивание типов как компромисс между наследованием и композицией.
Внешний тип получается неявные копии методов встроенного типа. Эти методы по
умолчанию делегируют вызовы тому же методу встроенного экземпляра.

Структура также получает поле с тем же именем, что и тип. Таким образом, если
встроенный тип является публичным, поле также будет публичным. Для поддержания
обратной совместимости каждая будущая версия внешнего типа должна сохранять
встроенный тип.

Встраивание типа редко необходимо. Это удобство, которое помогает избежать
написания утомительных методов делегирования.

Даже встраивание совместимого интерфейса `AbstractList` вместо структуры
предоставило бы разработчику больше гибкости для изменений в будущем, но все
равно раскрыло бы деталь о том, что конкретные списки используют абстрактную
реализацию.

```go
// Bad
// AbstractList is a generalized implementation
// for various kinds of lists of entities.
type AbstractList interface {
	Add(Entity)
	Remove(Entity)
}

// ConcreteList is a list of entities.
type ConcreteList struct {
	AbstractList
}
```

```go
// ConcreteList is a list of entities.
type ConcreteList struct {
	list AbstractList
}

// Add adds an entity to the list.
func (l *ConcreteList) Add(e Entity) {
	l.list.Add(e)
}

// Remove removes an entity from the list.
func (l *ConcreteList) Remove(e Entity) {
	l.list.Remove(e)
}
```

Как при встраивании структуры, так и при встраивании интерфейса, встроенный тип
накладывает ограничения на эволюцию типа.

- Добавление методов к встроенному интерфейсу является изменением, нарушающим
  совместимость.
- Удаление методов из встроенной структуры является изменением, нарушающим
  совместимость.
- Удаление встроенного типа является изменением, нарушающим совместимость.
- Замена встроенного типа, даже на альтернативу, которая удовлетворяет тому же
  интерфейсу, является изменением, нарушающим совместимость.

Хотя написание этих методов делегирования является утомительным, дополнительные
усилия скрывают делать реализации, оставляют больше возможностей для изменений и
также устраняют косвенность при обнаружении полного интерфейса List в
документации.

### Avoid Using Built-In Names

[Спецификация языка Go][the-go-language-specification] описывает несколько
встроенных, [предопределенных идентификаторов][predeclared-identifiers], которые
не следует использовать в качестве имен в Go-программах.

В лучшем случае компилятор выдаст предупреждение, в худшем - такой код может
привести к скрытым, трудно обнаруживаемым ошибкам.

```go
// Bad
var error string
// `error` shadows the builtin

// или

func handleErrorMessage(error string) {
	// `error` shadows the builtin
}

// ---

type Foo struct {
	// Хотя эти поля технически не
	// представляют shadowing, поиск строк 
	// `error` или `string` теперь
	// становится неоднозначным.
	error  error
	string string
}

func (f Foo) Error() error {
	// `error` и `f.error` являются
	// визуально схожими
	return f.error
}

func (f Foo) String() string {
	// `string` и `f.string` являются
	// визуально схожими
	return f.string
}
```

```go
// Good
var errorMessage string
// `error` ссылается на встроенный тип

// или

func handleErrorMessage(msg string) {
	// `error` ссылается на встроенный тип
}

// ---

type Foo struct {
	// Строки `error` и `string` теперь
	// являются однозначными.
	err error
	str string
}

func (f Foo) Error() error {
	return f.err
}

func (f Foo) String() string {
	return f.str
}
```

Стоит отметить, что компилятор не будет генерировать ошибки при использовании
предопределенных идентификаторов, но такие инструменты, как `go vet`, должны
правильно указывать на эти и другие случаи _shadowing_.

### Avoid `init()`

Нужно избегать использования `init()`, где это возможно. Когда `init()`
неизбежен или желателен, код должен стремиться к следующему:

- Быть полностью детерминированным, независимо от окружения программы или
  способа вызова.
- Избегать зависимости от порядка или побочных эффектов других функций `init()`.
  Хотя порядок вызова `init()` хорошо известен, код может изменяться, и,
  следовательно, взаимосвязи между функциями `init()` могут сделать код хрупким
  и подверженным ошибкам.
- Избегать доступа или манипуляций с глобальным или состоянием среды, таким как
  информация о машине, переменные окружения, рабочий каталог, аргументы, входные
  данные программы и т.д.
- Избегать I/O, включая файловую систему, сеть и системные вызовы.

Код, который не может удовлетворить этим требованиям, вероятно, должен быть
реализован как вспомогательная функция, вызываемая в рамках `main()` (или в
другом месте в жизненном цикле программы), или быть написанным как часть самого
`main()`. В частности, библиотеки, предназначенные для использования другими
программами, должны уделять особое внимание тому, чтобы быть полностью
детерминированными и не выполнять "init magic".

```go
// Bad
type Foo struct {
	// ...
}

var _defaultFoo Foo

func init() {
	_defaultFoo = Foo{
		// ...
	}
}

// ---

type Config struct {
	// ...
}

var _config Config

func init() {
	// Bad: основывается на текущей директории.
	cwd, _ := os.Getwd()

	// Bad: I/O
	raw, _ := os.ReadFile(
		path.Join(cwd, "config", "config.yaml"),
	)

	yaml.Unmarshal(raw, &_config)
}
```

```go
// Good
var _defaultFoo = Foo{
	// ...
}

// или, лучше, для тестируемости:

var _defaultFoo = defaultFoo()

func defaultFoo() Foo {
	return Foo{
		// ...
	}
}

// ---

type Config struct {
	// ...
}

func loadConfig() Config {
	cwd, err := os.Getwd()
	// обработка ошибки.

	raw, err := os.ReadFile(
		path.Join(cwd, "config", "config.yaml"),
	)
	// обработка ошибки.

	var config Config
	yaml.Unmarshal(raw, &config)

	return config
}
```

Учитывая вышеизложенное, некоторые ситуации, в которых использование `init()`
может быть предпочтительным или необходимым, могут включать:

- Сложные выражения, которые не могут быть представлены в виде одиночных
  присваиваний.
- Плагины, такие как диалекты `database/sql`, реестры типов кодирования и т.д.
- Оптимизации для Google Cloud Functions и других форм детерминированного
  предварительного вычисления.

### Exit in Main

Программы на Go используют `os.Exit` или `log.Fatal` для немедленного
завершения. (Паника не является хорошим способом для завершения программ.)

`os.Exit` и `log.Fatal` стоит вызывать **только в `main()`**. Все остальные
функции должны возвращать ошибки для сигнализации о сбое.

```go
// Bad
func main() {
	body := readFile(path)
	fmt.Println(body)
}

func readFile(path string) string {
	f, err := os.Open(path)
	if err != nil {
		log.Fatal(err)
	}

	b, err := io.ReadAll(f)
	if err != nil {
		log.Fatal(err)
	}

	return string(b)
}
```

```go
// Good
func main() {
	body, err := readFile(path)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(body)
}

func readFile(path string) (string, error) {
	f, err := os.Open(path)
	if err != nil {
		return "", err
	}

	b, err := io.ReadAll(f)
	if err != nil {
		return "", err
	}

	return string(b), nil
}
```

Обоснование: программы с несколькими функциями, которые могут завершать
выполнение, представляют собой несколько проблем:

- Неочевидный поток управления: любая функция может завершить программу, что
  затрудняет понимание потока управления.
- Сложности с тестированием: функция, которая завершает программу, также
  завершит тест, вызывающий ее. Это делает функцию трудной для тестирования и
  увеличивает риск пропуска других тестов, которые еще не были выполнены с
  помощью `go test`.
- Пропускается очистка: когда функция завершает программу, она пропускает вызовы
  функций, добавленных с помощью `defer`. Это увеличивает риск пропуска важных
  задач по очистке.

#### Exit Once

Если есть возможность, то желательно вызывать `os.Exit` или `log.Fatal` **не
более одного раза** в функции `main()`. Если имеется несколько сценариев ошибок,
которые останавливают выполнение программы, стоит поместить эту логику в
отдельную функцию и возвращать из нее ошибку.

Это позволит сократить функцию `main()` и вынести всю ключевую бизнес-логику в
отдельную, тестируемую функцию.

```go
// Bad
package main

func main() {
	args := os.Args[1:]
	if len(args) != 1 {
		log.Fatal("missing file")
	}
	name := args[0]

	f, err := os.Open(name)
	if err != nil {
		log.Fatal(err)
	}
	defer f.Close()

	// Если вызвать log.Fatal после этой линии,
	// f.Close не будет вызван.

	b, err := io.ReadAll(f)
	if err != nil {
		log.Fatal(err)
	}

	// ...
}
```

```go
// Good
package main

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}

func run() error {
	args := os.Args[1:]
	if len(args) != 1 {
		return errors.New("missing file")
	}
	name := args[0]

	f, err := os.Open(name)
	if err != nil {
		return err
	}
	defer f.Close()

	b, err := io.ReadAll(f)
	if err != nil {
		return err
	}

	// ...
}
```

В приведенном выше примере используется `log.Fatal`, но эти рекомендации также
применимы к `os.Exit` или любому коду библиотек, который вызывает `os.Exit`.

```go
func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

Можно изменить сигнатуру функции `run()` в соответствии с потребностями.
Например, если программа должна завершаться с определенными кодами выхода при
сбоях, `run()` может возвращать код выхода вместо ошибки. Это также позволяет
юнит-тестам напрямую проверять это поведение.

```go
func main() {
	os.Exit(run(args))
}

func run() (exitCode int) {
	// ...
}
```

Стоит отметить, что в общем случае, функция `run()`, которая используется в этих
примерах, не предназначена для того, чтобы быть предписывающей. Существует
гибкость в названии, сигнатуре и настройке функции `run()`. Среди прочего,
можно:

- принимать нераспаршенные аргументы командной строки (например,
  `run(os.Arrgs[1:])`);
- парсить аргументы командной строки в `main()` и передавать их в `run`;
- использовать пользовательский тип ошибки для передачи кода выхода обратно в
  `main()`;
- помещать бизнес-логику в другой уровень абстракции, отличной от
  `package main`.

Эти рекомендации требуют лишь того, чтобы в функция `main()` было одно место,
ответственное за фактическое завершение процесса.

### Use field tags in marshaled structs

Любое поле структуры, которое сериализуется в JSON, YAML или другие форматы,
поддерживающие tag-based field должны быть аннотировано соответствующим тегом.

```go
// Bad
type Stock struct {
	Price int
	Name string
}

bytes, err := json.Marshal(Stock{
	Price: 137,
	Name:  "UBER",
})
```

```go
// Good
type Stock struct {
	Price int	`json:"price"`
	Name string	`json:"name"`
	// Безопасно переименовать Name в Symbol.

bytes, err := json.Marshal(Stock{
	Price: 137,
	Name:  "UBER",
})
```

Обоснование: сериализованная форма структуры является контрактом между
различными системами. Изменения в структуре сериализованной формы, включая имена
полей, нарушают этот контракт. Указание имен полей внутри тегов делает контракт
явным и защищает от случайного его нарушения в результате рефакторинга или
переименования полей.

### Don't fire-and-forget goroutines

Хоть горутины и легковесны, но они не бесплатные: как минимум, они требуют
памяти для своего стек и CPU для планирования. Хотя эти затраты невелики для
типичного использования горутин, они могут вызвать значительные проблемы с
производительностью, когда горутины создаются в большом количестве без контроля
за их временем жизни. Горутины с неконтролируемым временем жизни также могут
вызывать другие проблемы, такие как предотвращение сборки мусора для
неиспользуемых объектов и удержание ресурсов, которые больше не используются.

Поэтому не стоит допускать утечек горутин в производственном коде. Можно
использовать go.uber.org/goleak для тестирования утечек горутин в пакетах,
которые могут создавать горутины.

В общем случае, каждая горутина:

- должна иметь предсказуемое время, когда она прекратит выполнение; или
- должен быть способ сигнализировать горутине, что она должна остановиться.

В обоих случаях должен быть способ, чтобы код мог блокироваться и ждать
завершения горутины.

Например:

```go
// Bad
go func() {
	for {
		flush()
		time.Sleep(delay)
	}
}()

// Нет способа остановить эту горутину. Она будет работать
// до тех пор, пока приложение не завершится.
```

```go
// Good
var (
	stop = make(chan struct{}) // сообщает горутине остановиться
	done = make(chan struct{}) // сообщает нам, что горутина завершила выполнение
)
go func() {
	defer close(done)

	ticker := time.NewTicker(delay)
	defer ticker.Stop()
	for {
		select {
		case <-ticker.C:
			flush()
		case <-stop:
			return
		}
	}
}()

// Где-то в другом месте...
close(stop)	// сигнализируем горутине остановиться
<-done		// и ждем, пока она завершится

// Эту горутину можно остановить с помощью close(stop), и мы
// можем дождаться ее завершения с помощью <-done.
```

#### Wait for goroutines to exit

Должен быть способ дождаться завершения горутины, созданной системой. Существует
два популярных способа сделать это:

Использование `sync.WaitGroup`. Если есть несколько горутин, которых нужно
дождаться.

```go
var wg sync.WaitGroup
for i := 0; i < N; i++ {
	wg.Add(1)
	go func() {
		defer wg.Done()
		// ...
	}()
}

// Чтобы дождаться завершения всех:
wg.Wait()
```

Добавить еще один `chan struct{}`, который горутина закроет, когда завершит
выполнение. Если есть только одна горутина.

```go
done := make(chan struct{})
go func() {
	defer close(done)
	// ...
}()

// Чтобы дождаться завершения горутины:
<-done
```

#### No goroutines in `init()`

Функции `init()` не должны создавать горутины.

Если пакету нужна фоновая горутина, он должен предоставить объект, который
отвечает за управление временем жизни горутины. Этот объект должен предоставлять
метод (`Close`, `Stop`, `Shutdown` и т.д.), который сигнализирует фоновой
горутине остановиться и ждет ее завершения.

```go
// Bad
func init() {
	go doWork()
}

func doWork() {
	for {
		// ..
	}
}

// Создает фоновую горутину без условий, когда пользователь 
// экспортирует этот пакет. У пользователя нет контроля над
// горутиной или способа ее остановить.
```

```go
// Good
type Worker struct{ /* ... */ }

func NewWorker(...) *Worker {
	w := &Worker{
		stop: make(chan struct{}),
		done: make(chan struct{}),
		// ..
	}
	go w.doWork()
}

func (w *Worker) doWork() {
	defer close(w.done)
	for {
		// ...
		case <-w.stop:
			return
	}
}

// Shutdown tells the worker to stop
// and waits until it has finished.
func (w *Worker) Shutdown() {
	close(w.stop)
	<-w.done
}

// Создание worker только в том случае, если пользователь
// запрашивает это. Предоставляет способ остановки worker, чтобы
// пользователь мог освободить ресурсы, используемые worker
//
// Следует использовать `WaitGroup`, если worker управляет
// несколькими горутинами.
```

## Performance

Руководство, касающиеся производительности, применимы только к _hot path_ (в
редких случаях?).

### Prefer strconv over fmt

При преобразовании примитивов в/из строки, `strconv` быстрее чем `fmt`.

```go
// Bad
for i := 0; i < b.N; i++ {
	s := fmt.Sprint(rand.Int())
}

// BenchmarkFmtSprint-4    143 ns/op    2 allocs/op
```

```go
// Good
for i := o; i < b.N; i++ {
	s := strconv.Itoa(rand.Int())
}

// BenchmarkStrconv-4    64.2 ns/op    1 allocs/op
```

### Avoid repeated string-to-byte conversions

Не стоит создавать срезы байт из фиксированной строки повторно. Лучше выполнить
преобразование один раз и сохранить результат.

```go
// Bad
for i := o; i < b.N; i++ {
	w.Write([]byte("Hello, world"))
}

// BenchmarkBad-4   50000000   22.2 ns/op
```

```go
// Good
data := []byte("Hello world")
for i := 0; i < b.N; i++ {
	w.Write(data)
}

// BenchmarkGood-4  500000000   3.25 ns/op
```

### Prefer Specifying Container Capacity

Стоит указывать размер контейнера, где это возможно, чтобы заранее выделить
память для контейнера. Это минимизирует последующие выделения памяти (из-за
копирования и изменения размера контейнера) по мере добавления элементов.

#### Specifying Map Capacity Hints

Где это возможно, рекомендуется указывать подсказки (hint) при инициализации
словарей при помощи `make()`.

```go
make(map[T1]T2, hint)
```

Указание подсказки по емкости для `make()` пытается правильно определить размер
словаря во время инициализации, что снижает необходимость в его увеличении и
выделении памяти по мере добавления элементов.

Стоит отметить, что, в отличие от срезов, подсказки по емкости для словарей не
гарантируют полное, предварительное выделение памяти, а используются для
приблизительной (наверное, из-за этого это и названо _подсказкой_) оценки
количества необходимых бакетов хеш-таблицы. Следовательно, выделения памяти
могут все еще происходить при добавлении элементов в словарь, даже до указанной
емкости.

```go
// Bad
m := make(map[string]os.FileInfo)

files, _ := os.ReadDir("./files")
for _, f := range files {
	m[f.Name()] = f
}

// m создается без подсказки по размеру;
// при присваивании может потребоваться больше
// выделенной памяти.
```

```go
// Good
files, _ := os.ReadDir("./files")

m := make(map[stringos.DirEntry, len(files))
for _, f := range files {
	m[f.Name()] = f
}

// m создается с подсказкой по размеру;
// при присвоении может быть меньше выделений памяти.
```

#### Specifying Slice Capacity

Где это возможно, стоит указывать подсказки по емкости при инициализации срезов
с помощью `make()`, особенно при добавлении элементов.

```go
make([]T, length, capacity)
```

В отличии от словарей, емкость срезов не является подсказкой: компилятор
выделить достаточно памяти для емкости среза, указанной в `make()`, что
означает, что последующие операции `append()` не потребуют выделения памяти (до
тех пор, пока длина среза не совпадет с емкостью, после чего любые добавления
потребуют изменения размера для размещения дополнительных элементов).

```go
// Bad
for n := 0; n < b.N; n++ {
	data := make([]int, 0)
	for k := 0; k < size; k++ {
		data = append(data, k)
	}
}

// BenchmarkBad-4    100000000    2.48s
```

```go
// Good
for n := 0; n < b.N; n++ {
	data := make([]int, 0, size)
	for k := 0; k < size; k++ {
		data = append(data, k)
	}
}

// BenchmarkGood-4   100000000    0.21s
```

## Style

### Avoid overly long lines

Нужно избегать строк кода, которые требуют от читателей горизонтальной прокрутки
или слишком частого поворота головы.

Рекомендуется _soft_ предел длины строки в 99 символов. Авторы должны стремиться
переносить строки до достижения этого предела, но это не жесткий лимит. Код
может превышать этот предел.

# Be Consistent

Некоторые из рекомендаций в этом документе могут быть оценены объективно, другие
ситуативно, контекстуально или субъективно.

Прежде всего стоит быть последовательным.

Последовательный код легче поддерживать, его проще обосновать, от требует
меньших когнитивных затрат и его легче мигрировать или обновлять по мере
появления новых соглашений или исправления ошибок.

При применении этих рекомендаций к кодовой базе рекомендуется вносить изменения
на уровне пакет (или более крупном уровне): применение на уровне подпакета
нарушает вышеуказанную рекомендацию, вводя несколько стилей в один и тот же код.

### Group Similar Declarations

Go поддерживает группировку схожих объявлений.

```go
// Bad
import "a"
import "b"
```

```go
// Good
import (
	"a"
	"b"
)
```

Это также применимо к константам, переменных и объявлениям типов.

```go
// Bad
const a = 1
const b = 2

var a = 1
var b = 2

type Area float64
type Volume float64
```

```go
// Good
const (
	a = 1
	b = 2
)

var (
	a = 1
	b = 2
)

type (
	Area   float64
	Volume float64
)
```

Стоит группировать только связанные объявления. Не группировать объявления,
которые не имеют отношения друг к другу.

```go
// Bad
type Operation int

const (
	Add Operation = iota + 1
	Subtract
	Multiply
	EnvVar = "MY_ENV"
)
```

```go
// Good
type Operation int

const (
	Add Operation = iota + 1
	Subtract
	Multiply
)

const EnvVar = "MY_ENV"
```

Группы не ограничены в том, где их можно использовать. Например, можно
использовать их внутри функций.

```go
// Bad
func f() string {
	red := color.New(0xff0000)
	green := color.New(0x00ff00)
	blue := color.New(0x0000ff)

	// ...
}
```

```go
// Good
func f() string {
	var (
		red   = color.New(0xff0000)
		green = color.New(0x00ff00)
		blue  = color.New(0x0000ff)
	)

	// ...
}
```

Исключение: объявления переменных, особенно внутри функций, должны быть
сгруппированы вместе, если они объявлены рядом с другими переменными. Это стоит
делать для переменных, объявленных вместе, даже если они не связаны.

```go
// Bad
func (c *client) request() {
	caller := c.name
	format := "json"
	timeout := 5*time.Second
	var err error

	// ...
}
```

```go
// Good
func (c *client) request() {
	var (
		caller  = c.name
		format  = "json"
		timeout = 5*time.Second
		err error
	)

	// ...
}
```

### Import Group Ordering

Должно быть две группы импорта:

- стандартная библиотека
- все остальное

Это группировка, которая применяется `goimports` по умолчанию.

```go
// Bad
import (
	"fmt"
	"os"
	"go.uber.org/atomic"
	"golang.org/x/sync/errgroup"
)
```

```go
// Good
import (
	"fmt"
	"os"

	"go.uber.org/atomic"
	"golang.org/x/sync/errgroup"
)
```

_Также используется группировка на 3 группы, 3-ая группа - текущий модуль._

### Package Names

При выборе имен для пакетов следует выбирать имя, которое:

- Полностью написано строчными буквами. Без заглавных букв и подчеркиваний.
- Не требует переименования с использованием именованных импортов в большинстве
  мест вызова.
- Короткое и лаконичное. Стоит помнить, что имя будет полностью указано в каждом
  месте вызова.
- Не во множественном числе. Например, `net/url`, а не `net/urls`.
- Нет "comman", "util", "shared" или "lib". Эти названия плохие и
  неинформативные.

Стоит посмотреть также [Package Names][article-on-package-names-package-names] и
[Style guideline for Go packages][article-on-package-names-style-guideline-for-go-packages].

### Function Names

В `uber` следуют общепринятой в сообществе Go конвенции использования
[MixedCaps для названий функций][article-function-names]. Исключение делается
для тестовых функций, которые могут содержать подчеркивания с целью группировки
связанных тестовых случаев, например, TestMyFunction_WhatIsBeingTested.

### Import Aliasing

Псевдонимы у импортов могут быть использованы, только если имя пакета не
совпадает с последним элементов пути импорта.

```go
import (
	"net/http"

	client "example.com/client-go"
	trace "example.com/trace/v2"
```

Во всех других случаях следует избегать псевдонимов у импорта, если нет прямого
конфликта между импортами.

```go
// Bad
import (
	"fmt"
	"os"
	runtimetrace "runtime/trace"

	nettrace "golang.net/x/trace"
)
```

```go
// Good
import (
  "fmt"
  "os"
  "runtime/trace"

  nettrace "golang.net/x/trace"
)
```

### Function Grouping and Ordering

- Функции должны быть отсортированы в приблизительном порядке вызова.
- Функции в файле должны быть сгруппированы по receiver.

Таким образом, экспортируемые функции должны располагаться первыми в файле,
после определений структур, констант и переменных.

Функции `newXYZ`/`NewXYZ` могут появляться после определения типа, но перед
остальными методами receiver.

Поскольку функции сгруппированы по receiver, обычные утилитарные функции должны
располагаться ближе к концу файла.

```go
// Bad
func (s *something) Cost() {
	return calcCost(s.weights)
}

type something struct { ... }

func calcCost(n []int) int { ... }

func (s *something) Stop() { ... }

func newSomething() *something {
	return &something{}
}
```

```go
// Good
type something struct { ... }

func newSomething() *something {
	return &something{}
}

func (s *something) Cost() {
	return calcCost(s.weights)
}

func (s *something) Stop() { ... }

func calcCost(n []int) int { ... }
```

### Reduce Nesting

Код должен уменьшать уровень вложенности, где это возможно, обрабатывая случаи
ошибок/особые условия в первую очередь и возвращаясь рано или продолжая цикл.
Стоит уменьшать количество кода, который вложен на несколько уровней.

```go
// Bad
for _, v := range data {
	if v.F1 == 1 {
		v = process(v)
		if err := v.Call(); err == nil {
			v.Send()
		} else {
			return err
		}
	} else {
		log.Printf("Invalid v: %v", v)
	}
}
```

```go
// Good
for _, v := range data {
	if v.F1 != 1 {
		log.Printf("Invalid v: %v", v)
		continue
	}

	v = process(v)
	if err := v.Call(); err != nil {
		return err
	}
	v.Send()
}
```

### Unnecessary Else

Если переменная устанавливается в обеих ветках оператора if, ее можно заменить
на один оператор if.

```go
// Bad
var a int
if b {
	a = 100
} else {
	a = 10
}
```

```go
// Good
a := 10
if b {
	a = 100
}
```

### Top-level Variable Declarations

На верхнем уровне стоит использовать стандартное ключевое слово `var`. Не стоит
указывать тип, если он не отличается от типа выражения.

```go
// Bad
var _s string = F()

func F() string { return "A" }
```

```go
// Good
var _s = F()
// Поскольку F уже указывает, что возвращает строку,
// не нужно указывать тип снова.

func F() string { return "A" }
```

Стоит указывать тип, если тип выражения не совпадает с желаемым типом точно.

```go
type myError struct{}

func (myError) Error() string { return "error" }

func F() myError { return myError{} }

var _e error = F()
// F возвращает объект типа myError, но нам нужен error.
```

### Prefix Unexported Globals with _

Стоит ставить префикс неэкспортируемых `var` и `const` верхнего уровня знаком
`_`, чтобы было ясно, что они являются глобальными символами при их
использовании.

Обоснование: переменные и константы верхнего уровня имеют область видимости
пакета. Использование общего имени делает легким случайное использование
неправильного значения в другом файле.

```go
// Bad
// foo.go

const (
	defaultPort = 8080
	defaultUser = "user"
)

// bar.go

func Bar() {
	defaultPort := 9090
	...
	fmt.Println("Default port", defaultPort)

	// Мы не увидим ошибки компиляции, если удалить
	// первую строку функции Bar().
}
```

```go
// Good
// foo.go

const (
	_defaultPort = 8080
	_defaultUser = "user"
)
```

Исключение: неэкспортируемые значения ошибок могут использовать префикс err без
подчеркивания. [Error Naming](#error-naming).

### Embedding in Structs

Встроенные типы должны располагаться вверху списка полей структуры, и между
встроенными полями и обычными полями должна быть пустая строка.

```go
// Bad
type Client struct {
	version int
	http.Client
}
```

```go
// Good
type Client struct {
	http.Client

	version int
}
```

Встраивание должно приносить ощутимую пользу, например, добавляя или расширяя
функциональность семантически уместным образом. Это должно происходить без
негативных последствий для пользователей
([Avoid Embedding Types in Public Structs](#avoid-embedding-types-in-public-structs)).

Исключение: мьютексты не должны быть встроены, даже в неэкспортируемые типы.
[Zero-value Mutexes are Valid](#zero-value-mutexes-are-valid).

Встраивание не должно:

- Быть чисто косметическим или ориентированным на удобство.
- Усложнять создание или использование внешних типов.
- Влиять на нулевые значения внешних типов. Если у внешнего типа есть полезное
  нулевое значение, оно должно оставаться полезным после встраивания внутреннего
  типа.
- Выявлять несвязанные функции или поля внешнего типа как побочный эффект
  встраивания внутреннего типа.
- Выявлять неэкспортируемые типы.
- Влиять на семантику копирования внешних типов.
- Изменять API или семантику типов внешнего типа.
- Встраивать неканоничную форму внутреннего типа.
- Выявлять детали реализации внешнего типа.
- Позволять пользователям контролировать или наблюдать за внутренностями типа.
- Изменять общее поведение внутренних функций через обертку таким образом,
  который мог бы разумно удивить пользователей.

Проще говоря, стоит встраивать осознанно и намеренно. Хороший тест на это - это
"would all of these exported inner methods/fields be added directly to the outer
type" (стоит ли, чтобы все эти экспортируемые внутренние методы/поля добавлялись
напрямую к внешнему типу); если ответ "некоторые" или "нет", не стоит встраивать
внутренний тип - использовать поле вместо этого.

```go
// Bad
type A struct {
	// Bad: A.Lock() и A.Unlock()
	//      теперь доступны, не предоставляя
	//      функциональной пользы, и позволяют
	//      конролировать детали
	//      внутренностей A.
	sync.Mutex
}
```

```go
// Good
type countingWriteCloser struct {
	// Good: Write() предоставляется на этом
	//       внешнем уровне для конкретной
	//       цели и делигирует работу
	//       внутреннего типа Write().
	io.WriteCloser

	count int
}

func (w *countingWriteCloser) Write(bs []byte) (int, error) {
	w.count += len(bs)
	return w.WriteCloser.Write(bs)
}
```

```go
// Bad
type Book struct {
	// Bad: указатель изменяет полезность нулевого значения
	io.ReadWriter

	// другие поля
}

// позже

var b Book
b.Read(...)  // panic: nil pointer
b.String()   // panic: nil pointer
b.Write(...) // panic: nil pointer
```

```go
// Good
type Book struct {
	// Good: имеет полезное нулевоей значение
	bytes.Buffer

	// другие поля
}

// позже

var b Book
b.Read(...)  // ok
b.String()   // ok
b.Write(...) // ok
```

```go
// Bad
type Client struct {
	sync.Mutex
	sync.WaitGroup
	bytes.Buffer
	url.URL
}
```

```go
// Good
type Client struct {
	mtx sync.Mutex
	wg  sync.WaitGroup
	buf bytes.Buffer
	url url.URL
}
```

### Local Variable Declarations

Короткое объявление переменной (`:=`) следует использовать тогда, когда
переменной устанавливается какое-то значение.

```go
// Bad
var s = "foo"
```

```go
// Good
s := "foo"
```

Однако есть случаи, когда значение по умолчанию становится более ясным при
использовании ключевого слова `var`. Например
[Declaring Emtpy Slices][article-on-local-variable-declarations].

```go
// Bad
func f(list []int) {
	filtered := []int{}
	for _, v := range list {
		if v > 10 {
			filtered = append(filtered, v)
		}
	}
}
```

```go
// Good
func f(list []int) {
	var filtered []int
	for _, v := range list {
		if v > 10 {
			filtered = append(filtered, v)
		}
	}
}
```

### nil is a valid slice

`nil` является валидным срезом длиной 0. Это означает, что:

Не стоит возвращать срез длиной 0. Вместо этого стоит возвращать `nil`.

```go
// Bad
if x == "" {
	return []int{}
}
```

```go
// Good
if x == "" {
	return nil
}
```

При проверки пустой ли срез, всегда использовать `len(s) == 0`. Не стоит
проверять на `nil`.

```go
// Bad
func isEmpty(s []string) bool {
	return s == nil
}
```

```go
// Good
func isEmpty(s []string) bool {
	return len(s) == 0
}
```

Нулевое значение (срез, объявленный с помощью `var`) можно использовать сразу
без `make()`.

```go
nums := []int{}
// или nums := make([]int)

if add1 {
	nums = append(nums, 1)
}

if add2 {
	nums = append(nums, 2)
}
```

```go
// Good
var nums []int

if add1 {
	nums = append(nums, 1)
}

if add2 {
	nums = append(nums, 2)
}
```

Стоит помнить, что хотя nil срез является допустимым, он не эквиваленте
выделенному срезу длинной 0 - один является nil, а другой - нет. И с ними могут
обращаться по-разному в различных ситуациях (например, при сериализации).

### Reduce Scope of Variables

Где это возможно, стоит уменьшать область видимости переменных и констант. Не
стоит уменьшать область видимости, если это противоречит принципу
[Reduce Nesting](#reduce-nesting).

```go
// Bad
err := os.WriteFile(name, data, 0644)
if err != nil {
	return err
}
```

```go
// Good
if err := os.WriteFile(name, data, 0644); err != nil {
	return err
}
```

Если нужен результат функции за пределами if, тогда следует уменьшать область
видимости.

```go
// Bad
if data, err := os.ReadFile(name); err == nil {
	err = cfg.Decode(data)
	if err != nil {
		return err
	}

	fmt.Println(cfg)
	return nil
} else {
	return err
}
```

```go
// Good
data, err := os.ReadFile(name)
if err != nil {
	return err
}

if err := cfg.Decode(data); err != nil {
	return err
}

fmt.Println(cfg)
return nil
```

Константы не должны быть глобальными, если они не используются в нескольких
функциях или файлах, или не являются частью внешнего контракта пакета.

```go
// Bad
const (
	_defaultPort = 8080
	_defaultUser = "user"
)

func Bar() {
	fmt.Println("Default port", _defaultPort)
}
```

```go
// Good
func Bar() {
	const (
		defaultPort = 8080
		defaultUser = "user"
	)
	fmt.Println("Default port", defaultPort)
}
```

### Avoid Naked Parameters

Голые параметры в вызовах функций могут ухудшать читаемость. Стоит добавлять
комментарий в стиле C (`/* ... */`) для имен параметров, когда их значение не
очевидно.

```go
// Bad
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true, true)
```

```go
// Good
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true /* isLocal */, true /* done */)
```

Лучше всего заменить голые `bool` на пользовательские типы для более читаемого и
безопасного с точки зрения типов кода. Это позволит в будущем использовать более
двух состояний (true/false) для этого параметра.

```go
type Region int

const (
	UnknownRegion Region = iota
	Local
)

type Status int

const (
	StatusReady Status = iota + 1
	StatusDone
	// Возможно, в будущем появится StatusInProgress
}

func printInfo(name string, region Region, status Status)
```

### Use Raw String Literals to Avoid Escaping

Go поддерживает
[raw string литералы][article-on-use-raw-string-literals-toavoid-escaping],
которые могут занимать несколько строк и включать кавычки. Их стоит
использовать, чтобы избежать строк с ручным экранированием, которые гораздо
труднее читать.

```go
// Bad
wantError := "unknown name:\"test\""
```

```go
// Good
wantError := `unknown error:"test"`
```

### Initializing Structs

#### Use Field Names to Initialize Structs

Практически всегда следует указывать имена полей при инициализации структур. Это
контролируется с помощью `go vet`.

```go
// Bad
k := User{"John", "Doe", true}
```

```go
k := User{
	FirstName: "John",
	LastName: "Doe",
	Admin: true,
}
```

Исключение: имена полей могут быть опущены в тестовых таблицах, когда полей 3
или меньше.

```go
tests := []struct{
	op Operation
	want string
}{
	{Add, "add"},
	{Subtract, "subtract"},
}
```

#### Omit Zero Value Fields in Structs

При инициализации структур с указанием имен полей стоит опускать поля, которые
имеют нулевые значения, если они не предоставляют значимого контекста. В
противном случае стоит позволить Go автоматические установить их в нулевые
значения.

```go
// Bad
user := User{
	FirstName: "John",
	LastName: "Doe",
	MiddleName: "",
	Admin: false,
}
```

```go
// Good
user := User{
	FirstName: "John",
	LastName: "Doe",
}
```

Это помогает уменьшить шум для читателей, опуская значения, которые являются
значениями по умолчанию в данном контексте. Указываются только значимые
значения.

Стоит включать нулевые значения, когда имена полей предоставляют значимый
контекст. Например, тестовые случаи и в [Test Tables](#tast-tables) может
принести пользу от имен полей, даже если они имеют нулевые значения.

```go
tests := []struct{
	give string
	want int
}{
	{give: "0", want: 0},
	// ...
}
```

#### Use `var` for Zero Value Structs

Когда все поля структуры опущены в объявлении, стоит использовать форму `var`
для объявления структуры.

```go
// Bad
user := User{}
```

```go
// Good
var user User
```

Это различает структуры с нулевыми значениями от структур с ненулевыми полями,
аналогично различию, созданному для
[инициализации словарей](#initializing-maps), и соответствует тому, как
предпочтительно
[объявлять пустые срезу][article-on-user-var-for-zero-value-structs].

#### Initializing Struct References

Стоит использовать `&T{}` вместо `new(T)` при инициализации ссылок на структуры,
чтобы это соответствовало инициализации структур.

```go
// Bad
sval := T{Name: "foo"}

// непоследовательно
sptr := new(T)
sptr.Name = "bar"
```

```go
// Good
sval := T{Name: "foo"}

sptr := &T{Name: "bar"}
```

### Initializing Maps

Рекомендуется использовать `make()` для пустых словарей и словарей, заполняемых
программно. Это делает инициализацию словарей визуально отличной от объявления и
упрощает добавление подсказок по размеру позже, если это возможно.

```go
// Bad
var (
	// m1 безопасно читать и записывать;
	// m2 вызовет панику при записи.
	m1 = map[T1]T2{}
	m2 map[T1]T2
)

// Объявление и инициализация визуально схожи.
```

```go
// Good
var (
	// m1 безопасно читать и записывать;
	// m2 вызовет панику при записи.
	m1 = make(map[T1]T2)
	m2 map[T1]T2
)

// Объявление и инициалихация визуально различны.
```

По возможности стоит указывать подсказки по емкости при инициализации словарей с
помощью `make()`.
[Specifying Map Capacity Hints](#specifying-map-capacity-hints).

С другой стороны, если словарь содержит фиксированный список элементов, стоит
использовать литералы словарей для инициализации словарей.

```go
// Bad
m := make(map[T1]T2, 3)
m[k1] = v1
m[k2] = v2
m[k3] = v3
```

```go
// Good
m := map[T1]T2{
	k1: v1,
	k2: v2,
	k3: v3,
}
```

Основное правило заключается в том, чтобы использовать литералы словарей при
добавлении фиксированного набора элементов во время инициализации; в противном
случае стоит использовать `make` (и стоит указывать подсказку по размеру, если
это возможно).

### Format Strings outside Printf

Если объявлять строки формата для функций в стиле `Printf` вне строкового
литерала, то нужно делать их `const`.

Это помогает `go vet` выполнять статический анализ строки формата.

```go
// Bad
msg := "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

```go
// Good
const msg = "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

### Naming Printf-style Functions

При объявлении функций в стиле `Printf`, нужно убедиться, что `go vet` может ее
обнаружить и проверить строку формата.

Это означает, что следует использовать предопределенные имена функций в стиле
`Printf`, если это возможно. `go vet` будет проверять их по умолчанию.
[Printf family][article-on-naming-printf-style-functions].

Если использование предопределенных имен невозможно, стоит завершить выбранное
имя на букву f: `Wrapf`, а не `Wrap`. `go vet` можно попросить проверить
конкретные имена в стиле `Printf`, но они должны заканчиваться на f.

```bash
go vet -printfuncs=wrapf,statusf
```

Статья
[go vet: Printf family check][article-on-naming-printf-style-functions-go-vet].

## Patterns

### Test Tables

Тесты, основанные на таблицах, с [под-тестами][article-on-test-tables-subtests]
могут быть полезным паттерном для написания тестов, чтобы избежать дублирования
кода, когда основная логика теста повторяется.

Если система, подлежащая тестированию, должна быть протестирована по нескольким
условиям, при которых определенные части входных и выходных данных изменяются,
следует использовать тесты, основанные на таблицах, чтобы уменьшить избыточность
и улучшить читаемость.

```go
// Bad
// func TestSplitHostPort(t *testing.T)

host, port, err := net.SplitHostPort("192.0.2.0:8000")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("192.0.2.0:http")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "http", port)

host, port, err = net.SplitHostPort(":8000")
require.NoError(t, err)
assert.Equal(t, "", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("1:8")
require.NoError(t, err)
assert.Equal(t, "1", host)
assert.Equal(t, "8", port)
```

```go
// Good
// func TestSplitHostPort(t *testing.T)

tests := []struct{
	give     string
	wantHost string
	wantPort string
}{
	{
		give:     "192.0.2.0:8000",
		wantHost: "192.0.2.0",
		wantPort: "8000",
	},
	{
		give:     "192.0.2.0:http",
		wantHost: "192.0.2.0",
		wantPort: "http",
	},
	{
		give:     ":8000",
		wantHost: "",
		wantPort: "8000",
	},
	{
		give:     "1:8",
		wantHost: "1",
		wantPort: "8",
	},
}

for _, tt := range tests {
	t.Run(tt.give, func(t *testing.T) {
		host, port, err := net.SplitHostPort(tt.give)
		require.NoError(t, err)
		assert.Equal(t, tt.wantHost, host)
		assert.Equal(t, tt.wantPort, port)
	})
}
```

Тестовые таблицы упрощают добавление контекста к сообщениями об ошибках,
уменьшают дублирование логики и позволяют добавлять новые тестовые случаи.

Стоит следовать соглашению, что срез структур называется `tests`, а каждый
тестовый случай обозначается как `tt`. Кроме того, рекомендуется явно указывать
входные и выходные значения для каждого тестового случая с помощью префиксов
`give` и `want`.

```go
tests := []struct{
	give     string
	wantHost string
	wantPort string
}{
	// ...
}

for _, tt := range tests {
	// ...
}
```

#### Avoid Unnecessary Complexity in Table Tests

Тесты на основе таблиц могут быть трудными для чтения и поддержки, если
под-тесты содержат условные утверждения или другую ветвящуюся логику. Тесты на
основе таблиц НЕ ДОЛЖНЫ использоваться, когда внутри под-тестов требуется
сложная или условная логика (например, сложная логика внутри цикла `for`).

Большие и сложные тесты на основе таблиц ухудшают читаемость и поддерживаемость,
поскольку читателям тестов может быть трудно отлаживать сбои тестов, которые
происходят.

Такие тесты на основе таблиц следует разбивать на несколько тестовых таблиц или
на несколько отдельных функций `Test...`.

Некоторые идеалы, к которым стоит стремиться:

- Сосредоточиться на самом узком единичном поведении.
- Минимизировать "глубину теста" и избегать условные утверждения.
- Убедиться, что все поля таблицы используются во всех тестах.
- Убедиться, что вся логика теста выполняется для всех случаев таблицы.

В этом контексте "глубина теста" означает "в рамках данного теста количество
последовательных утверждений, которые требуют, чтобы предыдущие утверждения были
истинными" (аналогично цикломатической сложности). Наличие "мелких" тестов
означает, что существует меньше взаимосвязей между утверждениями и, что более
важно, что эти утверждения менее вероятно будут условными по умолчанию.

Конкретно, тесты на основе таблиц могут стать запутанными и трудными для чтения,
если они используют несколько ветвящихся путей (например, `shouldError`,
`expectCall` и т.д.), используют много операторов `if` для конкретных ожиданий
моков (например `shouldCallFoo`) или помещают функции внутрь таблицы (например,
`setupMocks func(*FooMock)`).

Тем не менее, при тестировании поведения, которое изменяется только в
зависимости от измененного ввода, может быть предпочтительнее сгруппировать
похожие случаи вместе в тесте на основе таблицы, чтобы лучше проиллюстрировать,
как поведение меняется для всех входных данных, а не разбивать сопоставимые
единицы на отдельные тесты, что усложняет их сравнение и контрастирование.

Если тело теста короткое и простое, допустимо иметь один ветвящийся путь для
случаев успеха и неудачи с полем таблицы, таким как `shouldErr`, чтобы указать
ожидания ошибок.

```go
// Bad
func TestComplicatedTable(t *testing.T) {
	tests := []struct {
		give          string
		want          string
		wantErr       error
		shouldCallX   bool
		shouldCallY   bool
		giveXResponse string
		giveXErr      error
		giveYResponse string
		giveYErr      error
	}{
		// ...
	}

	for _, tt := range tests {
		t.Run(tt.give, func(t *testing.T) {
			// setup mocks
			ctrl := gomock.NewController(t)
			xMock := xmock.NewMockX(ctrl)
			if tt.shouldCallX {
			  xMock.EXPECT().Call().Return(
				tt.giveXResponse, tt.giveXErr,
			  )
			}
			yMock := ymock.NewMockY(ctrl)
			if tt.shouldCallY {
			  yMock.EXPECT().Call().Return(
				tt.giveYResponse, tt.giveYErr,
			  )
			}

			got, err := DoComplexThing(tt.give, xMock, yMock)

			// verify results
			if tt.wantErr != nil {
				require.EqualError(t, err, tt.wantErr)
				return
			}
			require.NoError(t, err)
			assert.Equal(t, want, got)
		})
	}
}
```

```go
// Good
func TestShouldCallX(t *testing.T) {
	// setup mocks
	ctrl := gomock.NewController(t)
	xMock := xmock.NewMockX(ctrl)
	xMock.EXPECT().Call().Return("XResponse", nil)

	yMock := ymock.NewMockY(ctrl)

	got, err := DoComplexThing("inputX", xMock, yMock)

	require.NoError(t, err)
	assert.Equal(t, "want", got)
}

func TestShouldCallYAndFail(t *testing.T) {
	// setup mocks
	ctrl := gomock.NewController(t)
	xMock := xmock.NewMockX(ctrl)

	yMock := ymock.NewMockY(ctrl)
	yMock.EXPECT().Call().Return("YResponse", nil)

	_, err := DoComplexThing("inputY", xMock, yMock)
	assert.EqualError(t, err, "Y failed")
}
```

Эта сложность делает труднее изменять, понимать и доказывать корректность теста.

Хотя строгий рекомендаций нет, читаемость и поддерживаемость всегда должны быть
в центре внимания при выборе между тестами на основе таблиц и отдельными тестами
для нескольких входных/выходных данных системы.

#### Parallel Tests

Параллельные тесты, как и некоторые специализированные циклы (например, те,
которые создают горутины или захватывают ссылки в теле цикла), должны заботиться
о явном присвоении переменных цикла в пределах области видимости цикла, чтобы
гарантировать, что они содержат ожидаемые значения.

```go
tests := []struct{
	give string
	// ...
}{
	// ...
}

for _, tt := range tests {
	tt := tt // for t.Parallel
	t.Run(tt.give, func(t *testing.T) {
		t.Parallel()
		// ...
	})
}
```

В приведенном выше примере нужно объявлять переменную `tt`, область видимости
которой ограничена итерацией цикла, из-за использования `t.Parallel()` ниже.
Если этого не сделать, большинство или все тесты получат неожиданное значение
для `tt` или значение, которое изменяется в процессе выполнения.

### Functional Options

Функциональные опции - это паттер, при котором объявляется непрозрачный тип
`Option`, который записывает информацию в некоторую внутреннюю структуру. Далее
принимается переменное количество этих опций и действуете на основе всей
информации, записанной опциями во внутренней структуре.

Нужно использовать этот паттерн для необязательных аргументов в конструкторах и
других публичных API, которые, могут потребовать расширения, особенно если уже
есть три или более аргументов в этих функциях.

```go
// Bad
// package db

func Open(
	addr string,
	cache bool,
	logger *zap.Logger
) (*Connection, error) {
	// ...
}

// Параметры кэша и логгера всегда должны быть предоставлены,
// даже если пользователь хочет использовать значения по умолчанию.

db.Open(addr, db.DefaultCache, zap.NewNop())
db.Open(addr, db.DefaultCache, log)
db.Open(addr, false /* cache */, zap.NewNop())
db.Open(addr, false /* cache */, log)
```

```go
// Good
// package db

type Option interface {
	// ...
}

func WithCache(c bool) Option {
	// ...
}

func WithLogger(log *zap.Logger) Option {
	// ...
}

// Open creates a connection.
func Open(
	addr string,
	opts ...Option,
) (*Connection, error) {
	// ...
}

// Опции предоставляются только в случае необходимости

db.Open(addr)
db.Open(addr, db.WithLogger(log))
db.Open(addr, db.WithCache(false))
db.Open(
	addr,
	db.WithCache(false),
	db.WithLogger(log),
)
```

Рекомендуемый способ реализации этого паттерна - использовать интерфейс
`Option`, который содержит неэкспортируемый метод, записывающий опции в
неэкспортируемую структуру `options`.

```go
type options struct {
	cache  bool
	logger *zap.Logger
}

type Option interface {
	apply(*options)
}

type cacheOption bool

func (c cacheOption) apply(opts *options) {
	opts.cache = bool(c)
}

func WithCache(c bool) Option {
	return cacheOption(c)
}

type loggerOption struct {
	Log *zap.Logger
}

func (l loggerOption) apply(opts *options) {
	opts.logger = l.Log
}

func WithLogger(log *zap.Logger) Option {
	return loggerOption{Log: log}
}

// Open creates a connection.
func Open(
	addr string,
	opts ...Option,
) (*Connection, error) {
	options := options{
		cache:  defaultCache,
		logger: zap.NewNop(),
	}

	for _, o := range opts {
		o.apply(&options)
	}

	// ...
}
```

Стоит обратить внимание, что существует способ реализации этого паттерна с
помощью замыканий, но считается, что предложенный выше паттер предоставляет
больше гибкости для авторов и проще в отладке и тестировании для пользователей.
В частности, он позволяет сравнивать опции друг с другом в тестах и моках, в
отличие от замыканий, где это невозможно. Кроме того, он позволяет опциям
реализовывать другие интерфейсы, включая `fmt.Stringer`, что позволяет получать
читаемые пользователем строковые представления опций.

Рекомендуется посмотреть:

- [Self-referential functions and the design of options][article-on-functional-options-1]
- [Functional options for friendly APIs][article-on-functional-options-2]

## Linting

Более важно, чем любой "одобренный" набор линтеров - это последовательное
использование линтинга по всей кодовой базе.

Рекомендуется использовать следующие линтеры как минимум, потому что они
помогают выявить наиболее распространенные проблемы и устанавливают высокие
стандарты качества кода, не будучи чрезмерно предписывающими (prescriptive):

- [errcheck][errcheck] для обеспечения обработки ошибок;
- [goimports][goimports] для форматирования кода и управления импортами;
- [golint][golint] для указания на распространенные ошибки стиля;
- [govet][govet] для анализа кода на наличие распространенных ошибок;
- [staticcheck][statickcheck] для выполнения различных проверок статического
  анализа.

### Lint Runners

Рекомендуется использовать [golangci-lint][golangci-lint] в качестве основного
инструмента для линтинга кода на Go, в значительной степени благодаря его
производительности в больших кодовых базах и возможности настраивать и
использовать множество канонических линтеров одновременно. В этом репозитории
есть пример конфигурационного файла [.golangci.yml][.golangci.yml] с
рекомендуемыми линтерами и настройками.

golangci-lint предлагает [различные линтеры][various linters] для использования.
Указанные выше линтеры рекомендуются как базовый набор, и рекомендуется
добавлять любые дополнительные линтеры, которые имеют смысл для проектов.


[uber-go-guide]: https://github.com/uber-go/guide
[uber-go-guide-changelog]: https://github.com/uber-go/guide/blob/master/CHANGELOG.md#2023-05-09
[uber-go-guide-contributing]: https://github.com/uber-go/guide/blob/master/CONTRIBUTING.md
[article-on-error-handling]: https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully
[the-go-language-specification]: https://go.dev/ref/spec
[predeclared-identifiers]: https://go.dev/ref/spec#Predeclared_identifiers
[article-on-package-names-package-names]: https://go.dev/blog/package-names
[article-on-package-names-style-guideline-for-go-packages]: https://rakyll.org/style-packages/
[article-function-names]: https://go.dev/doc/effective_go#mixed-caps
[article-on-local-variable-declarations]: https://go.dev/wiki/CodeReviewComments#declaring-empty-slices
[article-on-use-raw-string-literals-toavoid-escaping]: https://go.dev/ref/spec#raw_string_lit
[article-on-user-var-for-zero-value-structs]: https://go.dev/wiki/CodeReviewComments#declaring-empty-slices
[article-on-naming-printf-style-functions]: https://pkg.go.dev/cmd/vet#hdr-Printf_family
[article-on-naming-printf-style-functions-go-vet]: https://kuzminva.wordpress.com/2017/11/07/go-vet-printf-family-check/
[article-on-test-tables-subtests]: https://go.dev/blog/subtests
[article-on-functional-options-1]: https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html
[article-on-functional-options-2]: https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis
[errcheck]: https://github.com/kisielk/errcheck
[goimports]: https://pkg.go.dev/golang.org/x/tools/cmd/goimports
[golint]: https://github.com/golang/lint
[govet]: https://pkg.go.dev/cmd/vet
[statickcheck]: https://staticcheck.dev/
[golangci-lint]: https://github.com/golangci/golangci-lint
[.golangci.yml]: https://github.com/uber-go/guide/blob/master/.golangci.yml
[various linters]: https://golangci-lint.run/usage/linters/
