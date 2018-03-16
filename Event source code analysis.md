이벤트 패키지는 이벤트 및 구독 모델 내부 자료의 프로세스를 구현합니다.

## event.go

이 코드는 현재 사용되지 않는로, 피드는 사용자 개체를 알리는 표시됩니다. 그러나, 비록 사용하는 코드가있다. 그리고 코드의이 부분은 많이 없습니다. 간단한 소개.

데이터 구조
TypeMux의 주요 사용하는 것입니다. subm는 모든 가입자를 기록합니다. 각 유형에서 볼 수있는 많은 가입자를 가질 수 있습니다.
	
	// TypeMuxEvent is a time-tagged notification pushed to subscribers.
	type TypeMuxEvent struct {
		Time time.Time
		Data interface{}
	}
	
	// A TypeMux dispatches events to registered receivers. Receivers can be
	// registered to handle events of certain type. Any operation
	// called after mux is stopped will return ErrMuxClosed.
	//
	// The zero value is ready to use.
	//
	// Deprecated: use Feed
	type TypeMux struct {
		mutex   sync.RWMutex
		subm	map[reflect.Type][]*TypeMuxSubscription
		stopped bool
	}


구독을 작성, 당신은 동시에 여러 종류의 구독 할 수 있습니다.

	// Subscribe creates a subscription for events of the given types. The
	// subscription's channel is closed when it is unsubscribed
	// or the mux is closed.
	func (mux *TypeMux) Subscribe(types ...interface{}) *TypeMuxSubscription {
		sub := newsub(mux)
		mux.mutex.Lock()
		defer mux.mutex.Unlock()
		if mux.stopped {
			// set the status to closed so that calling Unsubscribe after this
			// call will short circuit.
			sub.closed = true
			close(sub.postC)
		} else {
			if mux.subm == nil {
				mux.subm = make(map[reflect.Type][]*TypeMuxSubscription)
			}
			for _, t := range types {
				rtyp := reflect.TypeOf(t)
				oldsubs := mux.subm[rtyp]
				if find(oldsubs, sub) != -1 {
					panic(fmt.Sprintf("event: duplicate type %s in Subscribe", rtyp))
				}
				subs := make([]*TypeMuxSubscription, len(oldsubs)+1)
				copy(subs, oldsubs)
				subs[len(oldsubs)] = sub
				mux.subm[rtyp] = subs
			}
		}
		return sub
	}

	// TypeMuxSubscription is a subscription established through TypeMux.
	type TypeMuxSubscription struct {
		mux	 *TypeMux
		created time.Time
		closeMu sync.Mutex
		closing chan struct{}
		closed  bool
	
		// these two are the same channel. they are stored separately so
		// postC can be set to nil without affecting the return value of
		// Chan.
		postMu sync.RWMutex
		//readC 및 postC 실제로있는 채널입니다. 그러나 하나의 채널에서 쓰기 전용 채널에서 읽고
		//단방향 채널
		readC  <-chan *TypeMuxEvent
		postC  chan<- *TypeMuxEvent
	}
	
	func newsub(mux *TypeMux) *TypeMuxSubscription {
		c := make(chan *TypeMuxEvent)
		return &TypeMuxSubscription{
			mux:	 mux,
			created: time.Now(),
			readC:   c,
			postC:   c,
			closing: make(chan struct{}),
		}
	}

위의 TypeMux하는 이벤트를 게시, 이번에는 모든 가입자 뉴스의이 유형을 받게됩니다.

	// Post sends an event to all receivers registered for the given type.
	// It returns ErrMuxClosed if the mux has been stopped.
	func (mux *TypeMux) Post(ev interface{}) error {
		event := &TypeMuxEvent{
			Time: time.Now(),
			Data: ev,
		}
		rtyp := reflect.TypeOf(ev)
		mux.mutex.RLock()
		if mux.stopped {
			mux.mutex.RUnlock()
			return ErrMuxClosed
		}
		subs := mux.subm[rtyp]
		mux.mutex.RUnlock()
		for _, sub := range subs {
			//전달을 차단.
			sub.deliver(event)
		}
		return nil
	}

	
	func (s *TypeMuxSubscription) deliver(event *TypeMuxEvent) {
		// Short circuit delivery if stale event
		if s.created.After(event.Time) {
			return
		}
		// Otherwise deliver the event
		s.postMu.RLock()
		defer s.postMu.RUnlock()
	
		select {// 길을 차단하는 방법
		case s.postC <- event:
		case <-s.closing:
		}
	}


## feed.go
현재 개체의 주요 사용. 말했다 이전 내부 event.go의 TypeMux 교체

피드 데이터 구조

	// Feed implements one-to-many subscriptions where the carrier of events is a channel.
	// Values sent to a Feed are delivered to all subscribed channels simultaneously.
	//피드 일대 가입 모델, 이벤트를 전달하는 채널의 사용을 제공합니다. 값으로 공급이 동시에 채널에서 모든 가입자에게 전달됩니다.
	// Feeds can only be used with a single type. The type is determined by the first Send or
	// Subscribe operation. Subsequent calls to these methods panic if the type does not
	// match.
	//피드는 하나의 유형을 사용할 수 있습니다. 이 이전 다른 이벤트는, 이벤트는 하나 개 이상의 유형을 사용할 수 있습니다. 첫 번째 유형은 송신라고하거나 결정을 호출 구독한다. 후속 통화 및 불일치의 유형은 당황됩니다
	// The zero value is ready to use.
	type Feed struct {
		once	  sync.Once		// ensures that init only runs once
		sendLock  chan struct{}	// sendLock has a one-element buffer and is empty when held.It protects sendCases.
		removeSub chan interface{} // interrupts Send
		sendCases caseList		 // the active set of select cases used by Send
	
		// The inbox holds newly subscribed channels until they are added to sendCases.
		mu	 sync.Mutex
		inbox  caseList
		etype  reflect.Type
		closed bool
	}

초기화 초기화는 보호가 한 번만 실행됩니다 보장하기 위해 한 번있을 것입니다.

	func (f *Feed) init() {
		f.removeSub = make(chan interface{})
		f.sendLock = make(chan struct{}, 1)
		f.sendLock <- struct{}{}
		f.sendCases = caseList{{Chan: reflect.ValueOf(f.removeSub), Dir: reflect.SelectRecv}}
	}

, 구독 배달 채널에 가입. 상대의 이벤트와는 달리. 가입 유형에 소개 된 이벤트는 가입 후 건물 내부에 가입 코드의 경우 채널하고 반환을 필요로한다. 이 직접 배달 채널 모델은 더 유연 할 수있다.
수신 채널 SelectCase 생성에이어서 방법. 받은 편지함 속으로.
	
	// Subscribe adds a channel to the feed. Future sends will be delivered on the channel
	// until the subscription is canceled. All channels added must have the same element type.
	//
	// The channel should have ample buffer space to avoid blocking other subscribers.
	// Slow subscribers are not dropped.
	func (f *Feed) Subscribe(channel interface{}) Subscription {
		f.once.Do(f.init)
	
		chanval := reflect.ValueOf(channel)
		chantyp := chanval.Type()
		if chantyp.Kind() != reflect.Chan || chantyp.ChanDir()&reflect.SendDir == 0 { //종류는 채널이 아닌 경우. 또는 채널의 방향은 데이터를 전송할 수 없습니다. 그래서 잘못 종료합니다.
			panic(errBadChannel)
		}
		sub := &feedSub{feed: f, channel: chanval, err: make(chan error, 1)}
	
		f.mu.Lock()
		defer f.mu.Unlock()
		if !f.typecheck(chantyp.Elem()) {
			panic(feedTypeError{op: "Subscribe", got: chantyp, want: reflect.ChanOf(reflect.SendDir, f.etype)})
		}
		// Add the select case to the inbox.
		// The next Send will add it to f.sendCases.
		cas := reflect.SelectCase{Dir: reflect.SelectSend, Chan: chanval}
		f.inbox = append(f.inbox, cas)
		return sub
	}


공급 방법은 다음 방법을 차단하는 모든 채널을 통해 전송되지 않는 전송, 전송 방법. 이것은 클라이언트에 영향을 빠른 속도가 느린 클라이언트 발생할 수 있습니다. 그러나 SelectCase를 사용하여 반사 모드를 사용하여. 첫째, 비 차단 보내려고하는 TrySend를 호출합니다. 이 느린 클라이언트를하지 않습니다합니다. 모든 데이터를 직접 전송됩니다. TrySend 경우 일부 클라이언트는 실패합니다. 그런 다음 전송 사이클의 선택 방법. 나는이 이벤트를 대체 할 피드의 이유 같아요.


	// Send delivers to all subscribed channels simultaneously.
	// It returns the number of subscribers that the value was sent to.
	func (f *Feed) Send(value interface{}) (nsent int) {
		f.once.Do(f.init)
		<-f.sendLock
	
		// Add new cases from the inbox after taking the send lock.
		f.mu.Lock()
		f.sendCases = append(f.sendCases, f.inbox...)
		f.inbox = nil
		f.mu.Unlock()
	
		// Set the sent value on all channels.
		rvalue := reflect.ValueOf(value)
		if !f.typecheck(rvalue.Type()) {
			f.sendLock <- struct{}{}
			panic(feedTypeError{op: "Send", got: rvalue.Type(), want: f.etype})
		}
		for i := firstSubSendCase; i < len(f.sendCases); i++ {
			f.sendCases[i].Send = rvalue
		}
	
		// Send until all channels except removeSub have been chosen.
		cases := f.sendCases
		for {
			// Fast path: try sending without blocking before adding to the select set.
			// This should usually succeed if subscribers are fast enough and have free
			// buffer space.
			for i := firstSubSendCase; i < len(cases); i++ {
				if cases[i].Chan.TrySend(rvalue) {
					nsent++
					cases = cases.deactivate(i)
					i--
				}
			}
			if len(cases) == firstSubSendCase {
				break
			}
			// Select on all the receivers, waiting for them to unblock.
			chosen, recv, _ := reflect.Select(cases)
			if chosen == 0 /* <-f.removeSub */ {
				index := f.sendCases.find(recv.Interface())
				f.sendCases = f.sendCases.delete(index)
				if index >= 0 && index < len(cases) {
					cases = f.sendCases[:len(cases)-1]
				}
			} else {
				cases = cases.deactivate(chosen)
				nsent++
			}
		}
	
		// Forget about the sent value and hand off the send lock.
		for i := firstSubSendCase; i < len(f.sendCases); i++ {
			f.sendCases[i].Send = reflect.Value{}
		}
		f.sendLock <- struct{}{}
		return nsent
	}
	
