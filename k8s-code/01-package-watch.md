# 介绍
我们一起来研究下kubernetes的watch包，这有助于理解kubernetes的list-watch机制，本文基于**v1.9.7**，代码目录`
/vendor/k8s.io/apimachinery/pkg/watch`
# watch.go
watch.go定义了Event结构体和watch.Interface接口
## Event struct
`Event struct`包括事件类型`Type`和事件发生的对象`Object`
```go
// EventType defines the possible types of events.
type EventType string

const (
	Added    EventType = "ADDED"
	Modified EventType = "MODIFIED"
	Deleted  EventType = "DELETED"
	Error    EventType = "ERROR"

	DefaultChanSize int32 = 100
)

// Event represents a single event to a watched resource.
// +k8s:deepcopy-gen=true
type Event struct {
	Type EventType

	// Object is:
	//  * If Type is Added or Modified: the new state of the object.
	//  * If Type is Deleted: the state of the object immediately before deletion.
	//  * If Type is Error: *api.Status is recommended; other types may make sense
	//    depending on context.
	Object runtime.Object
}
```
## watch.Interface interface
`watch.Interface`包括两个方法：返回事件监听结果的chan `ResultChan()`和停止监听的方法`Stop()`
```go
// Interface can be implemented by anything that knows how to watch and report changes.
type Interface interface {
	// Stops watching. Will close the channel returned by ResultChan(). Releases
	// any resources used by the watch.
	Stop()

	// Returns a chan which will receive all the events. If an error occurs
	// or Stop() is called, this channel will be closed, in which case the
	// watch should be completely cleaned up.
	ResultChan() <-chan Event
}
```

watch.go还包括三种Interface实现：`emptyWatch`，`FakeWatcher`，`RaceFreeFakeWatcher`，其中FakeWatcher和RaceFreeFakeWatcher是线程安全的
```go
type emptyWatch chan Event

// NewEmptyWatch returns a watch interface that returns no results and is closed.
// May be used in certain error conditions where no information is available but
// an error is not warranted.
func NewEmptyWatch() Interface {
	ch := make(chan Event)
	close(ch)
	return emptyWatch(ch)
}

// FakeWatcher lets you test anything that consumes a watch.Interface; threadsafe.
type FakeWatcher struct {
	result  chan Event
	Stopped bool
	sync.Mutex
}

func NewFake() *FakeWatcher {
	return &FakeWatcher{
		result: make(chan Event),
	}
}

func NewFakeWithChanSize(size int, blocking bool) *FakeWatcher {
	return &FakeWatcher{
		result: make(chan Event, size),
	}
}

// RaceFreeFakeWatcher lets you test anything that consumes a watch.Interface; threadsafe.
type RaceFreeFakeWatcher struct {
	result  chan Event
	Stopped bool
	sync.Mutex
}

func NewRaceFreeFake() *RaceFreeFakeWatcher {
	return &RaceFreeFakeWatcher{
		result: make(chan Event, DefaultChanSize),
	}
}
```
# mux.go
mux.go主要包括事件广播器`Broadcaster`和广播器观察者`broadcasterWatcher`
## Broadcaster struct
Broadcaster包括一个watchers map，Broadcaster有一个协程接收所有的事件并发送事件到所有注册的watcher。保证Broadcaster的所有watcher一直都能不断的接收到Broadcaster发送过来的事件
```go
// Broadcaster distributes event notifications among any number of watchers. Every event
// is delivered to every watcher.
type Broadcaster struct {
	// TODO: see if this lock is needed now that new watchers go through
	// the incoming channel.
	lock sync.Mutex

	watchers     map[int64]*broadcasterWatcher
	nextWatcher  int64
	distributing sync.WaitGroup

	incoming chan Event

	// How large to make watcher's channel.
	watchQueueLength int
	// If one of the watch channels is full, don't wait for it to become empty.
	// Instead just deliver it to the watchers that do have space in their
	// channels and move on to the next event.
	// It's more fair to do this on a per-watcher basis than to do it on the
	// "incoming" channel, which would allow one slow watcher to prevent all
	// other watchers from getting new events.
	fullChannelBehavior FullChannelBehavior
}

// NewBroadcaster creates a new Broadcaster. queueLength is the maximum number of events to queue per watcher.
// It is guaranteed that events will be distributed in the order in which they occur,
// but the order in which a single event is distributed among all of the watchers is unspecified.
func NewBroadcaster(queueLength int, fullChannelBehavior FullChannelBehavior) *Broadcaster {
	m := &Broadcaster{
		watchers:            map[int64]*broadcasterWatcher{},
		incoming:            make(chan Event, incomingQueueLength),
		watchQueueLength:    queueLength,
		fullChannelBehavior: fullChannelBehavior,
	}
	m.distributing.Add(1)
	go m.loop()
	return m
}
```
## broadcasterWatcher struct
`broadcasterWatcher`实现了watch.Interface interface
```go
// broadcasterWatcher handles a single watcher of a broadcaster
type broadcasterWatcher struct {
	result  chan Event
	stopped chan struct{}
	stop    sync.Once
	id      int64
	m       *Broadcaster
}
```
# filter.go
## filteredWatch struct
`filteredWatch`实现了watch.Interface interface，通过`FilterFunc`可以只watch满足一定条件的事件
```go
// FilterFunc should take an event, possibly modify it in some way, and return
// the modified event. If the event should be ignored, then return keep=false.
type FilterFunc func(in Event) (out Event, keep bool)

func Filter(w Interface, f FilterFunc) Interface {
	fw := &filteredWatch{
		incoming: w,
		result:   make(chan Event),
		f:        f,
	}
	go fw.loop()
	return fw
}

type filteredWatch struct {
	incoming Interface
	result   chan Event
	f        FilterFunc
}
```
## Recorder struct
Recorder结构体中的Interface是filteredWatch，记录watcher所接收到的所有的事件
```go
// Recorder records all events that are sent from the watch until it is closed.
type Recorder struct {
	Interface

	lock   sync.Mutex
	events []Event
}

var _ Interface = &Recorder{}

// NewRecorder wraps an Interface and records any changes sent across it.
func NewRecorder(w Interface) *Recorder {
	r := &Recorder{}
	r.Interface = Filter(w, r.record)
	return r
}

// record is a FilterFunc and tracks each received event.
func (r *Recorder) record(in Event) (Event, bool) {
	r.lock.Lock()
	defer r.lock.Unlock()
	r.events = append(r.events, in)
	return in, true
}
```

# streamwatcher.go
`StreamWatcher`封装一个实现`Decoder` interface的stream，将其转换为watch.Interface实现
```go
// Decoder allows StreamWatcher to watch any stream for which a Decoder can be written.
type Decoder interface {
	// Decode should return the type of event, the decoded object, or an error.
	// An error will cause StreamWatcher to call Close(). Decode should block until
	// it has data or an error occurs.
	Decode() (action EventType, object runtime.Object, err error)

	// Close should close the underlying io.Reader, signalling to the source of
	// the stream that it is no longer being watched. Close() must cause any
	// outstanding call to Decode() to return with an error of some sort.
	Close()
}

// StreamWatcher turns any stream for which you can write a Decoder interface
// into a watch.Interface.
type StreamWatcher struct {
	sync.Mutex
	source  Decoder
	result  chan Event
	stopped bool
}

// NewStreamWatcher creates a StreamWatcher from the given decoder.
func NewStreamWatcher(d Decoder) *StreamWatcher {
	sw := &StreamWatcher{
		source: d,
		// It's easy for a consumer to add buffering via an extra
		// goroutine/channel, but impossible for them to remove it,
		// so nonbuffered is better.
		result: make(chan Event),
	}
	go sw.receive()
	return sw
}
```

# until.go
`Until`用于满足conditions条件的watcher的Event过滤
```go
// ConditionFunc returns true if the condition has been reached, false if it has not been reached yet,
// or an error if the condition cannot be checked and should terminate. In general, it is better to define
// level driven conditions over edge driven conditions (pod has ready=true, vs pod modified and ready changed
// from false to true).
type ConditionFunc func(event Event) (bool, error)

// Until reads items from the watch until each provided condition succeeds, and then returns the last watch
// encountered. The first condition that returns an error terminates the watch (and the event is also returned).
// If no event has been received, the returned event will be nil.
// Conditions are satisfied sequentially so as to provide a useful primitive for higher level composition.
// A zero timeout means to wait forever.
func Until(timeout time.Duration, watcher Interface, conditions ...ConditionFunc) (*Event, error) {}
```
