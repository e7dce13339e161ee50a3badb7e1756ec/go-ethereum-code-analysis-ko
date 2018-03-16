## 공식 문서의 RPC 패키지

Package rpc provides access to the exported methods of an object across a network
or other I/O connection. After creating a server instance objects can be registered,
making it visible from the outside. Exported methods that follow specific
conventions can be called remotely. It also has support for the publish/subscribe
pattern.

RPC 패키지는 네트워크 또는 다른 I에 의해 접속 될 수있다 이러한 기능을 제공 / O 개체를 액세스 할 수있는 방법이 반출된다. 서버를 생성 한 후, 개체가 서버에 등록하고 외부 세계에 대한 액세스를 허용 할 수 있습니다. 이 방법은 파생 지방의 방법에 의해 원격으로 호출 할 수 있습니다. 또한 모델을 공개 / 등록 지원합니다.

Methods that satisfy the following criteria are made available for remote access:

- object must be exported
- method must be exported
- method returns 0, 1 (response or error) or 2 (response and error) values
- method argument(s) must be exported or builtin types
- method returned value(s) must be exported or builtin types

이 방법은 다음과 같은 기준 원격 액세스를 위해 사용될 수있다 만족

- 객체는 내 보내야합니다
- 방법은 내 보내야합니다
- 메소드가 복귀 0 (또는 에러 응답) 또는 제 2 (및 에러 응답) 값
- 메서드 매개 변수를 내보내거나 내장 유형이어야합니다
- 방법은 값을 내보내거나 내장 타입되어야한다 반환

An example method:

	func (s *CalcService) Add(a, b int) (int, error)

When the returned error isn't nil the returned integer is ignored and the error is
send back to the client. Otherwise the returned integer is send back to the client.

반환 값이 형성 무시 될 때 반환 된 오류가 전무 동일하지 않을 경우, 오류가 클라이언트로 전송됩니다. 그렇지 않으면 형성 수익률은 다시 클라이언트로 전송됩니다.

Optional arguments are supported by accepting pointer values as arguments. E.g.
if we want to do the addition in an optional finite field we can accept a mod
argument as pointer value.
메서드 매개 변수의 같은 포인터 타입을 제공함으로써 선택적 매개 변수를 지원할 수 있습니다. 나는 조금 뒤에 읽어 보시기 바랍니다.

	 func (s *CalService) Add(a, b int, mod *int) (int, error)

This RPC method can be called with 2 integers and a null value as third argument.
In that case the mod argument will be nil. Or it can be called with 3 integers,
in that case mod will be pointing to the given third argument. Since the optional
argument is the last argument the RPC package will also accept 2 integers as
arguments. It will pass the mod argument as nil to the RPC method.

RPC 방법은 두 정수 널값을 통과 호출 세번째 파라미터로서 사용될 수있다. 이 경우, 모드 파라미터는 전무로 설정된다. 모드는 세 번째 인수를 가리 키도록 설정할 수 있도록 아니면, 세 정수를 전달할 수 있습니다. 선택적 매개 변수는 어떤 자연의 정수를 수신 마지막 매개 변수, RPC 패킷 전송이 있었지만, 그래서 모드 매개 변수는 전무로 설정됩니다.

The server offers the ServeCodec method which accepts a ServerCodec instance. It will
read requests from the codec, process the request and sends the response back to the
client using the codec. The server can execute requests concurrently. Responses
can be sent back to the client out of order.

ServerCodec 서버는이 방법은 파라미터로서 ServerCodec 인스턴스를 수신하는 방법을 제공한다. 코덱 서버는 읽기 요청을 사용하여 요청을 처리 한 다음 코덱하여 클라이언트에 응답을 보냅니다. 서버는 동시에 실행 요구 될 수있다. 주문 응답은 요청 및 일관성을 주문할 수 있습니다.

	//An example server which uses the JSON codec:
	 type CalculatorService struct {}
	
	 func (s *CalculatorService) Add(a, b int) int {
		return a + b
	 }
	
	 func (s *CalculatorService Div(a, b int) (int, error) {
		if b == 0 {
			return 0, errors.New("divide by zero")
		}
		return a/b, nil
	 }
	calculator := new(CalculatorService)
	 server := NewServer()
	 server.RegisterName("calculator", calculator")
	
	 l, _ := net.ListenUnix("unix", &net.UnixAddr{Net: "unix", Name: "/tmp/calculator.sock"})
	 for {
		c, _ := l.AcceptUnix()
		codec := v2.NewJSONCodec(c)
		go server.ServeCodec(codec)
	 }


The package also supports the publish subscribe pattern through the use of subscriptions.
A method that is considered eligible for notifications must satisfy the following criteria:

 - object must be exported
 - method must be exported
 - first method argument type must be context.Context
 - method argument(s) must be exported or builtin types
 - method must return the tuple Subscription, error


이 패키지는 또한 구독 모델에 구독을 사용하여 게시를 지원합니다.
이 방법은 다음과 같은 조건을 충족해야합니다 통지의 조건을 충족하는 것으로 간주된다 :

- 객체는 내 보내야합니다
- 방법은 내 보내야합니다
- 첫 번째 방법은 인수 유형은 context.Context해야합니다
- 메서드 매개 변수를 내보내거나 내장 유형해야
- 방법은 튜플 가입 오류를 반환해야합니다

An example method:

	 func (s *BlockChainService) NewBlocks(ctx context.Context) (Subscription, error) {
		 ...
	 }

Subscriptions are deleted when:

 - the user sends an unsubscribe request
 - the connection which was used to create the subscription is closed. This can be initiated
   by the client and server. The server will close the connection on an write error or when
   the queue of buffered notifications gets too big.

다음과 같은 상황에서 구독이 삭제됩니다

- 사용자가 취소 요청을 보낸다
- 가입 연결이 닫혀 만듭니다. 이 클라이언트 나 서버에서 실행할 수 있습니다. 서버는 오류 알림 큐 길이가 너무 큰 기록하거나 연결을 닫 선택합니다합니다.

## RPC 패킷 구조 실질적
요청 및 응답 인코딩 및 디코딩 JSON 형식 및 네트워크 프로토콜 채널 동시에 서버와 클라이언트 클래스 다루고있다. 네트워크 프로토콜 및 데이터 전송 채널을 연결하는 주 기능을 제공한다. JSON 형식으로 인코딩 및 디코딩 요청 및 응답은 주로 직렬화 및 역 직렬화 제공 JSON (이 -&gt; 객체를 이동).

![image](picture/rpc_1.png)


## 소스 해결

### server.go
RPC 서버의 코어 로직의 주요 성과를 server.go. RPC 등록 방법에있어서, 판독 요청, 요청, 전송 응답 및 기타 로직 처리.
핵심 데이터 구조 서버는 서버 구조입니다. 서비스 필드는지도, 등록 된 모든 메소드 및 클래스의 기록이다. 실행 매개 변수를 제어하고 실행 서버를 중지하는 데 사용됩니다. 코덱은 집합입니다. 상점 모든 코덱을, 사실 모든 연결. codecsMu 잠금 장치는 멀티 스레드 액세스 코덱을 보호하는 데 사용됩니다.

값 유형 필드 서비스는 서비스 유형입니다. 등록 서버의 서비스 센터의 예는 객체 및 방법들의 조합이다. 서비스 필드 이름이 구독하는 방법 인스턴스에 가입하는 것입니다, 서비스의 네임 스페이스를 입력 일반 인스턴스, 콜백 콜백 메소드의 인스턴스를 나타냅니다.


	type serviceRegistry map[string]*service // collection of services
	type callbacks map[string]*callback	  // collection of RPC callbacks
	type subscriptions map[string]*callback 
	type Server struct {
		services serviceRegistry
	
		run	  int32
		codecsMu sync.Mutex
		codecs   *set.Set
	}
	
	// callback is a method callback which was registered in the server
	type callback struct {
		rcvr		reflect.Value  // receiver of method
		method	  reflect.Method // callback
		argTypes	[]reflect.Type // input argument types
		hasCtx	  bool		   // method's first argument is a context (not included in argTypes)
		errPos	  int			// err return idx, of -1 when method cannot return error
		isSubscribe bool		   // indication if the callback is a subscription
	}
	
	// service represents a registered object
	type service struct {
		name		  string		// name for service
		typ		   reflect.Type  // receiver type
		callbacks	 callbacks	 // registered handlers
		subscriptions subscriptions // available subscriptions/notifications
	}


서버 만들기, 서버는 자신의 예를 몇 가지 메타 정보 RPC 서비스를 제공하기 위해 등록 server.RegisterName를 호출하여 시간을 만듭니다.
		
	const MetadataApi = "rpc"
	// NewServer will create a new server instance with no registered handlers.
	func NewServer() *Server {
		server := &Server{
			services: make(serviceRegistry),
			codecs:   set.New(),
			run:	  1,
		}
	
		// register a default service which will provide meta information about the RPC service such as the services and
		// methods it offers.
		rpcService := &RPCService{server}
		server.RegisterName(MetadataApi, rpcService)
	
		return server
	}

서비스 등록 server.RegisterName는 RegisterName 방법은 들어오는 RCVR 인스턴스가 적당한 방법을 찾지 못하는 이상, 그것은 오류를 반환으로 전달하는 매개 변수에 의해 서비스 객체를 생성하는 것입니다. 오류가없는 경우, 인스턴스가되어 ServiceRegistry에 가입 만드는 서비스를했습니다.


	// RegisterName will create a service for the given rcvr type under the given name. When no methods on the given rcvr
	// match the criteria to be either a RPC method or a subscription an error is returned. Otherwise a new service is
	// created and added to the service collection this server instance serves.
	func (s *Server) RegisterName(name string, rcvr interface{}) error {
		if s.services == nil {
			s.services = make(serviceRegistry)
		}
	
		svc := new(service)
		svc.typ = reflect.TypeOf(rcvr)
		rcvrVal := reflect.ValueOf(rcvr)
	
		if name == "" {
			return fmt.Errorf("no service name for type %s", svc.typ.String())
		}
		//클래스 이름의 인스턴스 (클래스의 이름의 대문자 첫 글자) 파생되지 않은 경우, 오류를 반환합니다.
		if !isExported(reflect.Indirect(rcvrVal).Type().Name()) {
			return fmt.Errorf("%s is not exported", reflect.Indirect(rcvrVal).Type().Name())
		}
		//적절한 방법을 찾기 반사 정보와 구독을 콜백
		methods, subscriptions := suitableCallbacks(rcvrVal, svc.typ)
		//현재 이름이 등록되어있는 경우, 그 직접 삽입 여부, 새로운 대안으로 같은 이름을 사용하는 방법이 있는지.
		// already a previous service register under given sname, merge methods/subscriptions
		if regsvc, present := s.services[name]; present {
			if len(methods) == 0 && len(subscriptions) == 0 {
				return fmt.Errorf("Service %T doesn't have any suitable methods/subscriptions to expose", rcvr)
			}
			for _, m := range methods {
				regsvc.callbacks[formatName(m.method.Name)] = m
			}
			for _, s := range subscriptions {
				regsvc.subscriptions[formatName(s.method.Name)] = s
			}
			return nil
		}
	
		svc.name = name
		svc.callbacks, svc.subscriptions = methods, subscriptions
	
		if len(svc.callbacks) == 0 && len(svc.subscriptions) == 0 {
			return fmt.Errorf("Service %T doesn't have any suitable methods/subscriptions to expose", rcvr)
		}
	
		s.services[svc.name] = svc
		return nil
	}

적합한 방법 suitableCallbacks, utils.go하는이 방법을 찾는 정보에 의해 반사. 이 방법은이 유형의 모든 방법을 통과 적응 RPC 콜백 메소드 또는 표준 구독 콜백 유형과 수익을 찾을 수 있습니다. RPC 표준에 대한 문서의 시작 부분에 RPC 표준을 참조하시기 바랍니다.

	// suitableCallbacks iterates over the methods of the given type. It will determine if a method satisfies the criteria
	// for a RPC callback or a subscription callback and adds it to the collection of callbacks or subscriptions. See server
	// documentation for a summary of these criteria.
	func suitableCallbacks(rcvr reflect.Value, typ reflect.Type) (callbacks, subscriptions) {
		callbacks := make(callbacks)
		subscriptions := make(subscriptions)
	
	METHODS:
		for m := 0; m < typ.NumMethod(); m++ {
			method := typ.Method(m)
			mtype := method.Type
			mname := formatName(method.Name)
			if method.PkgPath != "" { // method must be exported
				continue
			}
	
			var h callback
			h.isSubscribe = isPubSub(mtype)
			h.rcvr = rcvr
			h.method = method
			h.errPos = -1
	
			firstArg := 1
			numIn := mtype.NumIn()
			if numIn >= 2 && mtype.In(1) == contextType {
				h.hasCtx = true
				firstArg = 2
			}
	
			if h.isSubscribe {
				h.argTypes = make([]reflect.Type, numIn-firstArg) // skip rcvr type
				for i := firstArg; i < numIn; i++ {
					argType := mtype.In(i)
					if isExportedOrBuiltinType(argType) {
						h.argTypes[i-firstArg] = argType
					} else {
						continue METHODS
					}
				}
	
				subscriptions[mname] = &h
				continue METHODS
			}
	
			// determine method arguments, ignore first arg since it's the receiver type
			// Arguments must be exported or builtin types
			h.argTypes = make([]reflect.Type, numIn-firstArg)
			for i := firstArg; i < numIn; i++ {
				argType := mtype.In(i)
				if !isExportedOrBuiltinType(argType) {
					continue METHODS
				}
				h.argTypes[i-firstArg] = argType
			}
	
			// check that all returned values are exported or builtin types
			for i := 0; i < mtype.NumOut(); i++ {
				if !isExportedOrBuiltinType(mtype.Out(i)) {
					continue METHODS
				}
			}
	
			// when a method returns an error it must be the last returned value
			h.errPos = -1
			for i := 0; i < mtype.NumOut(); i++ {
				if isErrorType(mtype.Out(i)) {
					h.errPos = i
					break
				}
			}
	
			if h.errPos >= 0 && h.errPos != mtype.NumOut()-1 {
				continue METHODS
			}
	
			switch mtype.NumOut() {
			case 0, 1, 2:
				if mtype.NumOut() == 2 && h.errPos == -1 { // method must one return value and 1 error
					continue METHODS
				}
				callbacks[mname] = &h
			}
		}
	
		return callbacks, subscriptions
	}


시작 ipc.go.에서 본원에 참고로 코드의 서비스 서버, 서버 시작 및 서비스 부 당신은 또한 연결의 레이어에 싸여 장식 패턴 코덱 비슷한 JsonCodec 기능을 볼 수 있습니다 서비스에 goroutine srv.ServeCodec 통화를 시작하는) (모든 동의 링크를 볼 수 있습니다. 코덱 볼 간략히 소개 후속 배치됩니다.

	func (srv *Server) ServeListener(l net.Listener) error {
		for {
			conn, err := l.Accept()
			if err != nil {
				return err
			}
			log.Trace(fmt.Sprint("accepted conn", conn.RemoteAddr()))
			go srv.ServeCodec(NewJSONCodec(conn), OptionMethodInvocation|OptionSubscriptions)
		}
	}

ServeCodec,이 방법은 매우 간단 폐쇄 codec.Close 기능을 제공한다. serveRequest SINGLESHOT 두번째 파라미터 SINGLESHOT이 참이면, 제어 연결 또는 접속 파라미터들의 짧은 길이의 요청을 처리 한 후 종료한다. 그러나 우리의 serveRequest 방법은 무한 루프도 예외는 발생하지, 또는 클라이언트를 종료 주도권을 쥐고있다,있다, 서버가 종료되지 않습니다. RPC 함수 오랫동안 연결을 제공한다.

	// ServeCodec reads incoming requests from codec, calls the appropriate callback and writes the
	// response back using the given codec. It will block until the codec is closed or the server is
	// stopped. In either case the codec is closed.
	func (s *Server) ServeCodec(codec ServerCodec, options CodecOption) {
		defer codec.Close()
		s.serveRequest(codec, false, options)
	}

우리의 무거운 방법은 결국 serveRequest이 방법은 주 처리 흐름 서버입니다했다. 코덱의 요청을 읽고 해당 메서드 호출을 찾은 다음 기록 된 코덱에 응답합니다.

코드 온라인 자습서의 사용을 참조 할 수 있습니다 표준 라이브러리의 일부입니다, sync.WaitGroup는 세마포어 기능을 구현합니다. 컨텍스트 컨텍스트 관리를 실현.

	
	// serveRequest will reads requests from the codec, calls the RPC callback and
	// writes the response to the given codec.
	//
	// If singleShot is true it will process a single request, otherwise it will handle
	// requests until the codec returns an error when reading a request (in most cases
	// an EOF). It executes requests in parallel when singleShot is false.
	func (s *Server) serveRequest(codec ServerCodec, singleShot bool, options CodecOption) error {
		var pend sync.WaitGroup
		defer func() {
			if err := recover(); err != nil {
				const size = 64 << 10
				buf := make([]byte, size)
				buf = buf[:runtime.Stack(buf, false)]
				log.Error(string(buf))
			}
			s.codecsMu.Lock()
			s.codecs.Remove(codec)
			s.codecsMu.Unlock()
		}()
	
		ctx, cancel := context.WithCancel(context.Background())
		defer cancel()
	
		// if the codec supports notification include a notifier that callbacks can use
		// to send notification to clients. It is thight to the codec/connection. If the
		// connection is closed the notifier will stop and cancels all active subscriptions.
		if options&OptionSubscriptions == OptionSubscriptions {
			ctx = context.WithValue(ctx, notifierKey{}, newNotifier(codec))
		}
		s.codecsMu.Lock()
		if atomic.LoadInt32(&s.run) != 1 { // server stopped
			s.codecsMu.Unlock()
			return &shutdownError{}
		}
		s.codecs.Add(codec)
		s.codecsMu.Unlock()
	
		// test if the server is ordered to stop
		for atomic.LoadInt32(&s.run) == 1 {
			reqs, batch, err := s.readRequest(codec)
			if err != nil {
				// If a parsing error occurred, send an error
				if err.Error() != "EOF" {
					log.Debug(fmt.Sprintf("read error %v\n", err))
					codec.Write(codec.CreateErrorResponse(nil, err))
				}
				// Error or end of stream, wait for requests and tear down
				//여기에 주요 고려 사항은, 멀티 스레드의 모든 요청 처리를 위해 대기가 완료되는 동안 처리입니다
				//각 스레드 호출 pend.Add 이동 시작 (1).
				//처리가 완료된 후 pend.Done ()를 호출 한 빼기 것입니다. 시간이 0 일 때, () 메소드가 반환을 기다립니다.
				pend.Wait()
				return nil
			}
	
			// check if server is ordered to shutdown and return an error
			// telling the client that his request failed.
			if atomic.LoadInt32(&s.run) != 1 {
				err = &shutdownError{}
				if batch {
					resps := make([]interface{}, len(reqs))
					for i, r := range reqs {
						resps[i] = codec.CreateErrorResponse(&r.id, err)
					}
					codec.Write(resps)
				} else {
					codec.Write(codec.CreateErrorResponse(&reqs[0].id, err))
				}
				return nil
			}
			// If a single shot request is executing, run and return immediately
			//한 번만, 다음 실행 후 반환하면 완료됩니다.
			if singleShot {
				if batch {
					s.execBatch(ctx, codec, reqs)
				} else {
					s.exec(ctx, codec, reqs[0])
				}
				return nil
			}
			// For multi-shot connections, start a goroutine to serve and loop back
			pend.Add(1)
			//요청을 처리하는 스레드를 시작합니다.
			go func(reqs []*serverRequest, batch bool) {
				defer pend.Done()
				if batch {
					s.execBatch(ctx, codec, reqs)
				} else {
					s.exec(ctx, codec, reqs[0])
				}
			}(reqs, batch)
		}
		return nil
	}


readRequest 방법 코덱에서 판독 요청 후 오브젝트 요청을 조립하는 방법에 따라 해당 요청을 찾을.
rpcRequest是codec返回的请求类型。j

	type rpcRequest struct {
		service  string
		method   string
		id	   interface{}
		isPubSub bool
		params   interface{}
		err	  Error // invalid batch element
	}

처리 요구 ServerRequest에 복귀 후

	// serverRequest is an incoming request
	type serverRequest struct {
		id			interface{}
		svcname	   string
		callb		 *callback
		args		  []reflect.Value
		isUnsubscribe bool
		err		   Error
	}

readRequest 방법 코덱에서 판독 요청은 요청 객체를 생성하는 프로세스를 ServerRequest에 반환한다.

	// readRequest requests the next (batch) request from the codec. It will return the collection
	// of requests, an indication if the request was a batch, the invalid request identifier and an
	// error when the request could not be read/parsed.
	func (s *Server) readRequest(codec ServerCodec) ([]*serverRequest, bool, Error) {
		reqs, batch, err := codec.ReadRequestHeaders()
		if err != nil {
			return nil, batch, err
		}
		requests := make([]*serverRequest, len(reqs))
		//REQS에 따라 요청의 건설
		// verify requests
		for i, r := range reqs {
			var ok bool
			var svc *service
	
			if r.err != nil {
				requests[i] = &serverRequest{id: r.id, err: r.err}
				continue
			}
			//요청이 전송 요청 / 가입 기간이며, 방법은 _unsubscribe 이름 접미사가있는 경우.
			if r.isPubSub && strings.HasSuffix(r.method, unsubscribeMethodSuffix) {
				requests[i] = &serverRequest{id: r.id, isUnsubscribe: true}
				argTypes := []reflect.Type{reflect.TypeOf("")} // expect subscription id as first arg
				if args, err := codec.ParseRequestArguments(argTypes, r.params); err == nil {
					requests[i].args = args
				} else {
					requests[i].err = &invalidParamsError{err.Error()}
				}
				continue
			}
			//이 서비스에 등록하지 않은 경우.
			if svc, ok = s.services[r.service]; !ok { // rpc method isn't available
				requests[i] = &serverRequest{id: r.id, err: &methodNotFoundError{r.service, r.method}}
				continue
			}
			//이 경우 게시 및 모델을 가입. 구독 메서드를 호출합니다.
			if r.isPubSub { // eth_subscribe, r.method contains the subscription method name
				if callb, ok := svc.subscriptions[r.method]; ok {
					requests[i] = &serverRequest{id: r.id, svcname: svc.name, callb: callb}
					if r.params != nil && len(callb.argTypes) > 0 {
						argTypes := []reflect.Type{reflect.TypeOf("")}
						argTypes = append(argTypes, callb.argTypes...)
						if args, err := codec.ParseRequestArguments(argTypes, r.params); err == nil {
							requests[i].args = args[1:] // first one is service.method name which isn't an actual argument
						} else {
							requests[i].err = &invalidParamsError{err.Error()}
						}
					}
				} else {
					requests[i] = &serverRequest{id: r.id, err: &methodNotFoundError{r.method, r.method}}
				}
				continue
			}
	
			if callb, ok := svc.callbacks[r.method]; ok { // lookup RPC method
				requests[i] = &serverRequest{id: r.id, svcname: svc.name, callb: callb}
				if r.params != nil && len(callb.argTypes) > 0 {
					if args, err := codec.ParseRequestArguments(callb.argTypes, r.params); err == nil {
						requests[i].args = args
					} else {
						requests[i].err = &invalidParamsError{err.Error()}
					}
				}
				continue
			}
	
			requests[i] = &serverRequest{id: r.id, err: &methodNotFoundError{r.service, r.method}}
		}
	
		return requests, batch, nil
	}

간부와 execBatch 방법은 요청 처리 s.handle 메소드를 호출한다.

	// exec executes the given request and writes the result back using the codec.
	func (s *Server) exec(ctx context.Context, codec ServerCodec, req *serverRequest) {
		var response interface{}
		var callback func()
		if req.err != nil {
			response = codec.CreateErrorResponse(&req.id, req.err)
		} else {
			response, callback = s.handle(ctx, codec, req)
		}
	
		if err := codec.Write(response); err != nil {
			log.Error(fmt.Sprintf("%v\n", err))
			codec.Close()
		}
	
		// when request was a subscribe request this allows these subscriptions to be actived
		if callback != nil {
			callback()
		}
	}
	
	// execBatch executes the given requests and writes the result back using the codec.
	// It will only write the response back when the last request is processed.
	func (s *Server) execBatch(ctx context.Context, codec ServerCodec, requests []*serverRequest) {
		responses := make([]interface{}, len(requests))
		var callbacks []func()
		for i, req := range requests {
			if req.err != nil {
				responses[i] = codec.CreateErrorResponse(&req.id, req.err)
			} else {
				var callback func()
				if responses[i], callback = s.handle(ctx, codec, req); callback != nil {
					callbacks = append(callbacks, callback)
				}
			}
		}
	
		if err := codec.Write(responses); err != nil {
			log.Error(fmt.Sprintf("%v\n", err))
			codec.Close()
		}
	
		// when request holds one of more subscribe requests this allows these subscriptions to be activated
		for _, c := range callbacks {
			c()
		}
	}

, 방법을 처리 요청을 수행하고 응답을 반환
	
	// handle executes a request and returns the response from the callback.
	func (s *Server) handle(ctx context.Context, codec ServerCodec, req *serverRequest) (interface{}, func()) {
		if req.err != nil {
			return codec.CreateErrorResponse(&req.id, req.err), nil
		}
		//구독 취소 메시지를합니다. NotifierFromContext 우리가 CTX에 통지를 얻을 전에 (CTX).
		if req.isUnsubscribe { // cancel subscription, first param must be the subscription id
			if len(req.args) >= 1 && req.args[0].Kind() == reflect.String {
				notifier, supported := NotifierFromContext(ctx)
				if !supported { // interface doesn't support subscriptions (e.g. http)
					return codec.CreateErrorResponse(&req.id, &callbackError{ErrNotificationsUnsupported.Error()}), nil
				}
	
				subid := ID(req.args[0].String())
				if err := notifier.unsubscribe(subid); err != nil {
					return codec.CreateErrorResponse(&req.id, &callbackError{err.Error()}), nil
				}
	
				return codec.CreateResponse(req.id, true), nil
			}
			return codec.CreateErrorResponse(&req.id, &invalidParamsError{"Expected subscription id as first argument"}), nil
		}
		//구독 메시지의 경우. 그런 다음 구독을 만들 수 있습니다. 그리고 구독을 활성화합니다.
		if req.callb.isSubscribe {
			subid, err := s.createSubscription(ctx, codec, req)
			if err != nil {
				return codec.CreateErrorResponse(&req.id, &callbackError{err.Error()}), nil
			}
	
			// active the subscription after the sub id was successfully sent to the client
			activateSub := func() {
				notifier, _ := NotifierFromContext(ctx)
				notifier.activate(subid, req.svcname)
			}
	
			return codec.CreateResponse(req.id, subid), activateSub
		}
	
		// regular RPC call, prepare arguments
		if len(req.args) != len(req.callb.argTypes) {
			rpcErr := &invalidParamsError{fmt.Sprintf("%s%s%s expects %d parameters, got %d",
				req.svcname, serviceMethodSeparator, req.callb.method.Name,
				len(req.callb.argTypes), len(req.args))}
			return codec.CreateErrorResponse(&req.id, rpcErr), nil
		}
	
		arguments := []reflect.Value{req.callb.rcvr}
		if req.callb.hasCtx {
			arguments = append(arguments, reflect.ValueOf(ctx))
		}
		if len(req.args) > 0 {
			arguments = append(arguments, req.args...)
		}
		//제공되는 메소드를 호출하고 응답을받을 RPC
		// execute RPC method and return result
		reply := req.callb.method.Func.Call(arguments)
		if len(reply) == 0 {
			return codec.CreateResponse(req.id, nil), nil
		}
	
		if req.callb.errPos >= 0 { // test if method returned an error
			if !reply[req.callb.errPos].IsNil() {
				e := reply[req.callb.errPos].Interface().(error)
				res := codec.CreateErrorResponse(&req.id, &callbackError{e.Error()})
				return res, nil
			}
		}
		return codec.CreateResponse(req.id, reply[0].Interface()), nil
	}

### subscription.go 게시 및 모델을 가입.
이전 server.go에 초점이 정교한 모델을 구독 게시 할 코드의 일부에 출연했다.

우리는 코드를 serveRequest 같은 코드가있다.

	코덱 지원하는 경우 클라이언트에 메시지를 보내 알리미라는 개체를 통해 콜백 함수를 실행할 수 있습니다.
	그와 코덱 / 연결 관계는 매우 가까운 거리에 있습니다. 연결이 종료되는 경우, 신고자는 닫고, 모든 활성 구독을 취소합니다.
	// if the codec supports notification include a notifier that callbacks can use
	// to send notification to clients. It is thight to the codec/connection. If the
	// connection is closed the notifier will stop and cancels all active subscriptions.
	if options&OptionSubscriptions == OptionSubscriptions {
		ctx = context.WithValue(ctx, notifierKey{}, newNotifier(codec))
	}

클라이언트 연결이 newNotifier 메소드를 호출하는 통지 자 객체를 생성하는 서비스 CTX에 저장된다. 알리미 오브젝트가 저장된 예 코덱을 관찰 할 수있다, 즉 통보는 필요할 때 데이터를 전송하기위한 저장 네트워크 연결 개체.

	// newNotifier creates a new notifier that can be used to send subscription
	// notifications to the client.
	func newNotifier(codec ServerCodec) *Notifier {
		return &Notifier{
			codec:	codec,
			active:   make(map[ID]*Subscription),
			inactive: make(map[ID]*Subscription),
		}
	}

그런 접근 방식을 처리, 우리는로 식별 방법의 특별한 종류, 다루고있는 isSubscribe. CreateSubscription 호출 방법은 구독을 만들고 내부의 통지 활성화 대기열을 저장하는 notifier.activate 방법을 문의하십시오. 코드는 그 기술이있다. 직접 전화를 활성화하지 않습니다이 방법은 가입하지만, 다시 함수 리턴 코드와 활성 섹션을 완료합니다. 그런 다음이 응답 호출 된 후 클라이언트로 전송되는 임원 또는 execBatch codec.CreateResponse (req.id, 하위 ID) 내부의 코드를 기다립니다. 그는 구독 정보를 받고 클라이언트가 가입 ID를 수신하지 마십시오.

	if req.callb.isSubscribe {
		subid, err := s.createSubscription(ctx, codec, req)
		if err != nil {
			return codec.CreateErrorResponse(&req.id, &callbackError{err.Error()}), nil
		}

		// active the subscription after the sub id was successfully sent to the client
		activateSub := func() {
			notifier, _ := NotifierFromContext(ctx)
			notifier.activate(subid, req.svcname)
		}

		return codec.CreateResponse(req.id, subid), activateSub
	}

createSubscription 방법까지 등록 지정된 메소드를 호출하고 응답을받을.

	// createSubscription will call the subscription callback and returns the subscription id or error.
	func (s *Server) createSubscription(ctx context.Context, c ServerCodec, req *serverRequest) (ID, error) {
		// subscription have as first argument the context following optional arguments
		args := []reflect.Value{req.callb.rcvr, reflect.ValueOf(ctx)}
		args = append(args, req.args...)
		reply := req.callb.method.Func.Call(args)
	
		if !reply[1].IsNil() { // subscription creation failed
			return "", reply[1].Interface().(error)
		}
	
		return reply[0].Interface().(*Subscription).ID, nil
	}

구독을 활성화 우리의 활성화 방법을 살펴보십시오. 활성화가 가입 구독 ID에 클라이언트에 전송되면, 클라이언트는 수신 된 정보에 가입 구독 ID를 수신하지 않은 시간을 방지한다.
	
	// activate enables a subscription. Until a subscription is enabled all
	// notifications are dropped. This method is called by the RPC server after
	// the subscription ID was sent to client. This prevents notifications being
	// send to the client before the subscription ID is send to the client.
	func (n *Notifier) activate(id ID, namespace string) {
		n.subMu.Lock()
		defer n.subMu.Unlock()
		if sub, found := n.inactive[id]; found {
			sub.namespace = namespace
			n.active[id] = sub
			delete(n.inactive, id)
		}
	}

우리가 수신 거부 기능을 살펴 보자

	// unsubscribe a subscription.
	// If the subscription could not be found ErrSubscriptionNotFound is returned.
	func (n *Notifier) unsubscribe(id ID) error {
		n.subMu.Lock()
		defer n.subMu.Unlock()
		if s, found := n.active[id]; found {
			close(s.err)
			delete(n.active, id)
			return nil
		}
		return ErrSubscriptionNotFound
	}

마지막으로, 클라이언트에 데이터를 보내려면이 함수를 호출 구독 기능을 보내,이 비교적 간단하다.

	// Notify sends a notification to the client with the given data as payload.
	// If an error occurs the RPC connection is closed and the error is returned.
	func (n *Notifier) Notify(id ID, data interface{}) error {
		n.subMu.RLock()
		defer n.subMu.RUnlock()
	
		sub, active := n.active[id]
		if active {
			notification := n.codec.CreateNotification(string(id), sub.namespace, data)
			if err := n.codec.Write(notification); err != nil {
				n.codec.Close()
				return err
			}
		}
		return nil
	}


어떻게 TestNotifications subscription_test.go 프로세스에 의해 전체 권장 사항을 볼 수 있습니다.

### Client.go RPC 클라이언트 측 소스 코드 분석.

클라이언트의 주요 기능 서버에 요청을 전송하는 것, 그리고 그 반응은 호출자에게 전달되는 응답을 수신한다.

클라이언트 데이터 구조

	// Client represents a connection to an RPC server.
	type Client struct {
		idCounter   uint32
		//함수 발생기에 연결이 함수를 호출 클라이언트가 네트워크 연결 객체를 생성한다.
		connectFunc func(ctx context.Context) (net.Conn, error)
		//HTTP 프로토콜이 아닌 HTTP 프로토콜은 프로세스의 범위를 가지고, HTTP 프로토콜이 긴 연결을 지원하지 않는 단 하나의 모드는이 요청에 대한 응답에 해당하는 지원, 그것은 또한 모델을 공개 / 등록 지원하지 않습니다.
		isHTTP	  bool
	
		// writeConn is only safe to access outside dispatch, with the
		// write lock held. The write lock is taken by sending on
		// requestOp and released by sending on sendDone.
		//여기에 주목하여 볼 수있는, writeConn이 네트워크 연결이 쓰기 요청을 대상으로 사용되는 전화
		//단지 그것을 발송 방법을 호출하는 것이 안전하고 잠금 요청을 획득 할 필요가 큐 requestOp로 전송됩니다,
		//요청은 다른 요청에 대한 잠금 sendDone를 해제 큐에 대한 쓰기 요청을 전송 완료 후, 잠금을 획득 한 후 네트워크에서 쓸 수 있습니다.
		writeConn net.Conn
	
		// for dispatch
		//여기에 코드가 채널을 사용하는 방법에 대해 설명 따를로 많은 채널이 채널은 일반적으로 goroutine 사이의 통신 채널을 사용 있습니다.
		close	   chan struct{}
		didQuit	 chan struct{}				  // closed when client quits
		reconnected chan net.Conn				  // where write/reconnect sends the new connection
		readErr	 chan error					 // errors from read
		readResp	chan []*jsonrpcMessage		 // valid messages from read
		requestOp   chan *requestOp				// for registering response IDs
		sendDone	chan error					 // signals write completion, releases write lock
		respWait	map[string]*requestOp		  // active requests
		subs		map[string]*ClientSubscription // active subscriptions
	}


newClient, 새로운 클라이언트. 네트워크 접속을 기반으로하는 경우 connectFunc 메소드를 호출하여 네트워크에 연결하도록 httpConn 오브젝트 후 isHTTP true로 설정. 이 HTTP 연결, 직접 반환의 경우 사람이 goroutine 호출 발송 방법을 시작할 것인지 그런 다음 개체를 초기화합니다. 채널 언급 - 상기와 같은 goroutine에 의해 전체 클라이언트 명령 센터, 정보를 얻을, 통신의 다양한 정보를 기반으로 결정을 내릴 수있는 발송 방법. 후속 의지 세부 파견. HTTP 호출은 매우 간단하기 때문에, 여기에 방법의 간단한 HTTP 박람회를 할 수 있습니다.

	
	func newClient(initctx context.Context, connectFunc func(context.Context) (net.Conn, error)) (*Client, error) {
		conn, err := connectFunc(initctx)
		if err != nil {
			return nil, err
		}
		_, isHTTP := conn.(*httpConn)
	
		c := &Client{
			writeConn:   conn,
			isHTTP:	  isHTTP,
			connectFunc: connectFunc,
			close:	   make(chan struct{}),
			didQuit:	 make(chan struct{}),
			reconnected: make(chan net.Conn),
			readErr:	 make(chan error),
			readResp:	make(chan []*jsonrpcMessage),
			requestOp:   make(chan *requestOp),
			sendDone:	make(chan error, 1),
			respWait:	make(map[string]*requestOp),
			subs:		make(map[string]*ClientSubscription),
		}
		if !isHTTP {
			go c.dispatch(conn)
		}
		return c, nil
	}


RPC 호출 통화 방법 클라이언트를 호출하여 호출에 요청.

	// Call performs a JSON-RPC call with the given arguments and unmarshals into
	// result if no error occurred.
	//
	// The result must be a pointer so that package json can unmarshal into it. You
	// can also pass nil, in which case the result is ignored.
	JSON 객체로 값을 변환 할 수 있도록 반환 값은, 포인터이어야합니다. 당신이 반환 값을 걱정하지 않는 경우, 그것은 또한 패스 무기 호에 의해 무시 될 수있다.
	func (c *Client) Call(result interface{}, method string, args ...interface{}) error {
		ctx := context.Background()
		return c.CallContext(ctx, result, method, args...)
	}
	
	func (c *Client) CallContext(ctx context.Context, result interface{}, method string, args ...interface{}) error {
		msg, err := c.newMessage(method, args...)
		if err != nil {
			return err
		}
		//requestOp 객체를 구축합니다. RESP 판독 리턴 큐는 큐 길이는 1이다.
		op := &requestOp{ids: []json.RawMessage{msg.ID}, resp: make(chan *jsonrpcMessage, 1)}
		
		if c.isHTTP {
			err = c.sendHTTP(ctx, op, msg)
		} else {
			err = c.send(ctx, op, msg)
		}
		if err != nil {
			return err
		}
	
		// dispatch has accepted the request and will close the channel it when it quits.
		switch resp, err := op.wait(ctx); {
		case err != nil:
			return err
		case resp.Error != nil:
			return resp.Error
		case len(resp.Result) == 0:
			return ErrNoResult
		default:
			return json.Unmarshal(resp.Result, &result)
		}
	}

sendHTTP,이 방법은 응답을 얻기 위해 직접 doRequest 방법 요청을 호출합니다. 그런 다음 반환 RESP에 큐에 기록됩니다.

	func (c *Client) sendHTTP(ctx context.Context, op *requestOp, msg interface{}) error {
		hc := c.writeConn.(*httpConn)
		respBody, err := hc.doRequest(ctx, msg)
		if err != nil {
			return err
		}
		defer respBody.Close()
		var respmsg jsonrpcMessage
		if err := json.NewDecoder(respBody).Decode(&respmsg); err != nil {
			return err
		}
		op.resp <- &respmsg
		return nil
	}


또 다른 방법은 두 개의 큐의 정보를 볼 수 있습니다 위의 op.wait () 메소드를 볼 수 있습니다. 그 다음 응답에 대한 HTTP RESP 큐에서 얻은 경우 직접 반환됩니다. 그래야 전체 HTTP 요청 프로세스가 완료됩니다. 중간은 스레드 완료 내에서 모두 멀티 스레딩 문제와 관련이 없습니다.

	func (op *requestOp) wait(ctx context.Context) (*jsonrpcMessage, error) {
		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		case resp := <-op.resp:
			return resp, op.err
		}
	}

그것은 HTTP 요청이 아닌 경우. HTTP 요청을 기억하지 않을 경우 공정의 흐름은 더 복잡하다. newClient는 goroutine 호출 발송 방법을 시작하는 경우. 우리는 보낼-HTTP가 아닌 방법을 살펴 봅니다.

보기의 의견에서. 이 방법은 영업 requestOp이 큐를 작성하는 것, 그것은 대기열이 시간이 사람의 큐가 처리되지 않으면 통화가 여기 차단되도록, 버퍼가 없다는 것을 밝혔다. 잠금을 가지고 성공적인 작전을 requestOp을 보낸 경우이 잠금 동일합니다, 당신은 다음 단계로 계속, 다음 단계는 네트워크에 요청의 전체 내용을 전송하기 위해 write 메소드를 호출하는 것입니다. SendDone는 큐에 메시지를 보냅니다. 록을 해제 sendDone 알 수있는 후속 디스패치하는 방법이 공정의 상세한 분석. 그리고 돌아갑니다. op.wait있어서 내부 차단 방법 돌아온. 그것은 op.resp 큐로부터 응답을 수신하거나 ctx.Done () 메시지를 수신 (일반적으로 완료에 획득 한이 메시지를하거나 종료하도록 강요)까지

	// send registers op with the dispatch loop, then sends msg on the connection.
	// if sending fails, op is deregistered.
	func (c *Client) send(ctx context.Context, op *requestOp, msg interface{}) error {
		select {
		case c.requestOp <- op:
			log.Trace("", "msg", log.Lazy{Fn: func() string {
				return fmt.Sprint("sending ", msg)
			}})
			err := c.write(ctx, msg)
			c.sendDone <- err
			return err
		case <-ctx.Done():
			// This can happen if the client is overloaded or unable to keep up with
			// subscription notifications.
			return ctx.Err()
		case <-c.didQuit:
			//철회했다, 그것은 닫기를 호출 할 수 있습니다
			return ErrClientQuit
		}
	}

발송 방법
	

	// dispatch is the main loop of the client.
	// It sends read messages to waiting calls to Call and BatchCall
	// and subscription notifications to registered subscriptions.
	func (c *Client) dispatch(conn net.Conn) {
		// Spawn the initial read loop.
		go c.read(conn)
	
		var (
			lastOp		*requestOp	// tracks last send operation
			requestOpLock = c.requestOp // nil while the send lock is held
			reading	   = true		// if true, a read loop is running
		)
		defer close(c.didQuit)
		defer func() {
			c.closeRequestOps(ErrClientQuit)
			conn.Close()
			if reading {
				// Empty read channels until read is dead.
				for {
					select {
					case <-c.readResp:
					case <-c.readErr:
						return
					}
				}
			}
		}()
	
		for {
			select {
			case <-c.close:
				return
	
			// Read path.
			case batch := <-c.readResp:
				//응답을 읽어보십시오. 다루는 적절한 방법을 호출
				for _, msg := range batch {
					switch {
					case msg.isNotification():
						log.Trace("", "msg", log.Lazy{Fn: func() string {
							return fmt.Sprint("<-readResp: notification ", msg)
						}})
						c.handleNotification(msg)
					case msg.isResponse():
						log.Trace("", "msg", log.Lazy{Fn: func() string {
							return fmt.Sprint("<-readResp: response ", msg)
						}})
						c.handleResponse(msg)
					default:
						log.Debug("", "msg", log.Lazy{Fn: func() string {
							return fmt.Sprint("<-readResp: dropping weird message", msg)
						}})
						// TODO: maybe close
					}
				}
	
			case err := <-c.readErr:
				//읽기 장애 정보를 수신,이 스레드를 통해 전달됩니다 읽어 보시기 바랍니다.
				log.Debug(fmt.Sprintf("<-readErr: %v", err))
				c.closeRequestOps(err)
				conn.Close()
				reading = false
	
			case newconn := <-c.reconnected:
				//어소시에이션 메시지를 수신
				log.Debug(fmt.Sprintf("<-reconnected: (reading=%t) %v", reading, conn.RemoteAddr()))
				if reading {
					//연결 대기 전에 완료 읽기.
					// Wait for the previous read loop to exit. This is a rare case.
					conn.Close()
					<-c.readErr
				}
				//열기 읽기 goroutine
				go c.read(newconn)
				reading = true
				conn = newconn
	
			// Send path.
			case op := <-requestOpLock:
				// Stop listening for further send ops until the current one is done.
				//RequestOp는 다음 세트가 비어 requestOpLock입니다 메시지를 수신
				//아무도 치료가 차단하지 않기 때문에 requestOp에 작전을 보낼 다른 사람들이이 시간이 있다면.
				requestOpLock = nil
				lastOp = op
				//이 연산은 대기열에 추가됩니다.
				for _, id := range op.ids {
					c.respWait[string(id)] = op
				}
	
			case err := <-c.sendDone:
				//요청 된 정보는 영업 네트워크로 전송 된 경우. 그것은 sendDone에 메시지를 보내드립니다. 오류를 보내는 과정은 다음 잘못되면! = 무기 호.
				if err != nil {
					// Remove response handlers for the last send. We remove those here
					// because the error is already handled in Call or BatchCall. When the
					// read loop goes down, it will signal all other current operations.
					//큐에서 삭제 된 모든 ID입니다.
					for _, id := range lastOp.ids {
						delete(c.respWait, string(id))
					}
				}
				// Listen for send ops again.
				//재시작 메시지 처리 RequestOp.
				requestOpLock = c.requestOp
				lastOp = nil
			}
		}
	}


다음 메인 흐름이도 디스패치 설명한다. 다음 그림은 스레드를 반올림됩니다. 파란색 사각형은 채널입니다. 화살표는 채널 방향에서의 데이터의 흐름을 나타낸다.

![image](picture/rpc_2.png)

- 네트워크상에서 멀티 스레드 직렬 흐름에 대한 요청을 송신하는 제 requestOp 네트워크에 다음 기록 요구 메시지 잠금을 획득하고 해제 디스패치 sendDone에 정보를 송신 전달하라는 요청을 전송한다. 이 두 채널 sendDone requestOp 파견하고 완전한 기능 코드 시리얼 네트워크를 피팅하여 요청을 보낼 수 있습니다.
- 정보를 읽은 다음 호출자 프로세스를 다시 돌아갑니다. 네트워크에 전송 요청 정보를 한 후, 연속 goroutine 읽기의 내부 네트워크에서 정보를 읽습니다. 자세한 정보를 알아볼를 읽을 돌아온 후, readResp를 통해 디스패치 큐에 보냈다. 해당 파견 발신자 및 발신자의 반환 정보 쓰기 큐 RESP를 찾습니다. 반환 정보의 처리를 완료합니다.
- 재 연결 과정. 재 연결 적극적으로 쓰기에 실패 외부 호출자의 경우 외부 발신자를 호출한다. 전화는 파견에 대한 새 연결을 보낼 완료된 후. 파견 새 연결을받은 후 다음 새 연결에 대한 새로운 읽기 goroutine의 정보를 읽기 시작하기 전에, 연결이 종료됩니다.
- 폐쇄 과정. 호출자는 큐를 닫 정보를 기록 할 닫기 방법 닫기 메소드를 호출합니다. 정보 발신 가까이받은 후. 닫기 didQuit 큐, 연결이 닫힐, 읽기 goroutine을 기다리고 중지합니다. didQuit는 모든 클라이언트 호출을 반환 위의 모든 대기열에서 대기.


#### 클라이언트 구독 모델의 특수 처리
상기 방법의 주요 공정 플로우가 호출된다. 이더넷 광장 RPC 프레임 워크는 또한 게시와 모델을 구독 지원합니다.

우리는보고 가입 방법을, 이더넷 광장은 여러 주요 (EthSubscribe ShhSubscribe)의 가입 서비스를 제공합니다. 또한, (구독) 서비스를 사용자 정의에 가입하는 방법을 제공


	// EthSubscribe registers a subscripion under the "eth" namespace.
	func (c *Client) EthSubscribe(ctx context.Context, channel interface{}, args ...interface{}) (*ClientSubscription, error) {
		return c.Subscribe(ctx, "eth", channel, args...)
	}
	
	// ShhSubscribe registers a subscripion under the "shh" namespace.
	func (c *Client) ShhSubscribe(ctx context.Context, channel interface{}, args ...interface{}) (*ClientSubscription, error) {
		return c.Subscribe(ctx, "shh", channel, args...)
	}
	
	// Subscribe calls the "<namespace>_subscribe" method with the given arguments,
	// registering a subscription. Server notifications for the subscription are
	// sent to the given channel. The element type of the channel must match the
	// expected type of content returned by the subscription.
	//
	// The context argument cancels the RPC request that sets up the subscription but has no
	// effect on the subscription after Subscribe has returned.
	//
	// Slow subscribers will be dropped eventually. Client buffers up to 8000 notifications
	// before considering the subscriber dead. The subscription Err channel will receive
	// ErrSubscriptionQueueOverflow. Use a sufficiently large buffer on the channel or ensure
	// that the channel usually has at least one reader to prevent this issue.
	//구독 &quot;매개 변수가 호출에 전달합니다 <namespace> _subscribe &quot;접근 방식은 지정된 메시지를 구독 할 수 있습니다.
	//Notification Server는 지정된 큐 채널 매개 변수를 작성합니다. 채널 매개 변수 및 반환의 동일한 유형이어야합니다.
	//CTX 매개 변수는 RPC 요청을 취소하는 데 사용할 수 있지만, 등록이 완료된 경우 효과가있을 것입니다.
	//가입자 메시지 처리 속도는 각 클라이언트는 캐시에게 8000 메시지가 삭제됩니다 너무 느립니다.
	func (c *Client) Subscribe(ctx context.Context, namespace string, channel interface{}, args ...interface{}) (*ClientSubscription, error) {
		// Check type of channel first.
		chanVal := reflect.ValueOf(channel)
		if chanVal.Kind() != reflect.Chan || chanVal.Type().ChanDir()&reflect.SendDir == 0 {
			panic("first argument to Subscribe must be a writable channel")
		}
		if chanVal.IsNil() {
			panic("channel given to Subscribe must not be nil")
		}
		if c.isHTTP {
			return nil, ErrNotificationsUnsupported
		}
	
		msg, err := c.newMessage(namespace+subscribeMethodSuffix, args...)
		if err != nil {
			return nil, err
		}
		//다른 매개 변수를 RequestOp하고 전화를 호출합니다. 또 하나 개의 매개 변수 하위.
		op := &requestOp{
			ids:  []json.RawMessage{msg.ID},
			resp: make(chan *jsonrpcMessage),
			sub:  newClientSubscription(c, namespace, chanVal),
		}
	
		// Send the subscription request.
		// The arrival and validity of the response is signaled on sub.quit.
		if err := c.send(ctx, op, msg); err != nil {
			return nil, err
		}
		if _, err := op.wait(ctx); err != nil {
			return nil, err
		}
		return op.sub, nil
	}

새로운 객체를 생성 ClientSubscription newClientSubscription 방법은, 이러한 목적은 수신 채널 파라미터에 저장된다. 그런 다음 자신은 세 가지 찬 물체를 만들었습니다. 자세한 사항은이 세 가지 오브젝트 찬 따를 것이다


	func newClientSubscription(c *Client, namespace string, channel reflect.Value) *ClientSubscription {
		sub := &ClientSubscription{
			client:	c,
			namespace: namespace,
			etype:	 channel.Type().Elem(),
			channel:   channel,
			quit:	  make(chan struct{}),
			err:	   make(chan error, 1),
			in:		make(chan json.RawMessage),
		}
		return sub
	}

바와 같이 위의 코드에서 볼 수 있습니다. 가입 요청을 구축, 거의 같은 루트에 가입 프로세스를 호출합니다. 네트워크에 보낼 보내 전화 후 반환 기다립니다. 우리는 다른 가입을보고 파견에 의해 호출하는 반환에게 그 결과를 처리합니다.


	func (c *Client) handleResponse(msg *jsonrpcMessage) {
		op := c.respWait[string(msg.ID)]
		if op == nil {
			log.Debug(fmt.Sprintf("unsolicited response %v", msg))
			return
		}
		delete(c.respWait, string(msg.ID))
		// For normal responses, just forward the reply to Call/BatchCall.
		op.sub는 무기 호, 일반 RPC 요청이있는 경우,이 필드의 값은 비어 만 가입 요청은 가치가있다.
		if op.sub == nil {
			op.resp <- msg
			return
		}
		// For subscription responses, start the subscription if the server
		// indicates success. EthSubscribe gets unblocked in either case through
		// the op.resp channel.
		defer close(op.resp)
		if msg.Error != nil {
			op.err = msg.Error
			return
		}
		if op.err = json.Unmarshal(msg.Result, &op.sub.subid); op.err == nil {
			//새로운 goroutine를 시작하고 op.sub.subid까지 기록한다.
			go op.sub.start()
			c.subs[op.sub.subid] = op.sub
		}
	}


op.sub.start 방법. 이 goroutine 구독 메시지를 처리 ​​할 수 ​​있도록 설계. 주요 기능은 내부의 버퍼 대기열의 내부에 그 가입 메시지를 가입 얻는 것이다. 데이터가 가능한 경우는 전송 될 수있다. 일부 데이터는 그 버퍼에서 수신되는 채널 내부 사용자에게 전송된다. 버퍼가 특정 크기를 초과하는 경우, 이것은 폐기된다.

	
	func (sub *ClientSubscription) start() {
		sub.quitWithError(sub.forward())
	}
	
	func (sub *ClientSubscription) forward() (err error, unsubscribeServer bool) {
		cases := []reflect.SelectCase{
			{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(sub.quit)},
			{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(sub.in)},
			{Dir: reflect.SelectSend, Chan: sub.channel},
		}
		buffer := list.New()
		defer buffer.Init()
		for {
			var chosen int
			var recv reflect.Value
			if buffer.Len() == 0 {
				// Idle, omit send case.
				chosen, recv, _ = reflect.Select(cases[:2])
			} else {
				// Non-empty buffer, send the first queued item.
				cases[2].Send = reflect.ValueOf(buffer.Front().Value)
				chosen, recv, _ = reflect.Select(cases)
			}
	
			switch chosen {
			case 0: // <-sub.quit
				return nil, false
			case 1: // <-sub.in
				val, err := sub.unmarshal(recv.Interface().(json.RawMessage))
				if err != nil {
					return err, true
				}
				if buffer.Len() == maxClientSubscriptionBuffer {
					return ErrSubscriptionQueueOverflow, true
				}
				buffer.PushBack(val)
			case 2: // sub.channel<-
				cases[2].Send = reflect.Value{} // Don't hold onto the value.
				buffer.Remove(buffer.Front())
			}
		}
	}


알림 메시지를 수신 할 때의 handleNotification 메소드를 호출합니다. 메시지 큐에 넣어.

	func (c *Client) handleNotification(msg *jsonrpcMessage) {
		if !strings.HasSuffix(msg.Method, notificationMethodSuffix) {
			log.Debug(fmt.Sprint("dropping non-subscription message: ", msg))
			return
		}
		var subResult struct {
			ID	 string		  `json:"subscription"`
			Result json.RawMessage `json:"result"`
		}
		if err := json.Unmarshal(msg.Params, &subResult); err != nil {
			log.Debug(fmt.Sprint("dropping invalid subscription message: ", msg))
			return
		}
		if c.subs[subResult.ID] != nil {
			c.subs[subResult.ID].deliver(subResult.Result)
		}
	}
	func (sub *ClientSubscription) deliver(result json.RawMessage) (ok bool) {
		select {
		case sub.in <- result:
			return true
		case <-sub.quit:
			return false
		}
	}

