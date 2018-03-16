노드의 이동의 에테 리움의 노드를 나타냅니다. 전체 노드 수 있음, 노드는 경량 일 가능성이있다. 노드는 세계 많은 주변 노드의 유형에 이더넷 광장 조성 운영하는 과정으로 이해 될 수있다.

일반적인 노드는 P2P 노드입니다. 다른 비즈니스 계층 프로토콜 (피어를 사용하는 P2P의 인터페이스 프로토콜을 참조 구분하기 위해 네트워크 계층 프로토콜)를 실행, 노드의 다른 유형에 따라 반면, P2P 네트워크 프로토콜을 실행합니다.
	
노드 구조.
	
	// Node is a container on which services can be registered.
	type Node struct {
		eventmux *event.TypeMux // Event multiplexer used between the services of a stack
		config   *Config
		accman   *accounts.Manager
	
		ephemeralKeystore string		 // if non-empty, the key directory that will be removed by Stop
		instanceDirLock   flock.Releaser // prevents concurrent use of instance directory
	
		serverConfig p2p.Config
		server	   *p2p.Server // Currently running P2P networking layer
	
		serviceFuncs []ServiceConstructor	 // Service constructors (in dependency order)
		services	 map[reflect.Type]Service // Currently running services
	
		rpcAPIs	   []rpc.API   // List of APIs currently provided by the node
		inprocHandler *rpc.Server // In-process RPC request handler to process the API requests
	
		ipcEndpoint string	   // IPC endpoint to listen at (empty = IPC disabled)
		ipcListener net.Listener // IPC RPC listener socket to serve API requests
		ipcHandler  *rpc.Server  // IPC RPC request handler to process the API requests
	
		httpEndpoint  string	   // HTTP endpoint (interface + port) to listen at (empty = HTTP disabled)
		httpWhitelist []string	 // HTTP RPC modules to allow through this endpoint
		httpListener  net.Listener // HTTP RPC listener socket to server API requests
		httpHandler   *rpc.Server  // HTTP RPC request handler to process the API requests
	
		wsEndpoint string	   // Websocket endpoint (interface + port) to listen at (empty = websocket disabled)
		wsListener net.Listener // Websocket RPC listener socket to server API requests
		wsHandler  *rpc.Server  // Websocket RPC request handler to process the API requests
	
		stop chan struct{} // Channel to wait for termination notifications
		lock sync.RWMutex
	}


노드 초기화 초기화, 기타 외부 구성 요소에 의존하지 않는 노드 만 구성 개체에 따라 달라집니다.
	
	// New creates a new P2P node, ready for protocol registration.
	func New(conf *Config) (*Node, error) {
		// Copy config and resolve the datadir so future changes to the current
		// working directory don't affect the node.
		confCopy := *conf
		conf = &confCopy
		if conf.DataDir != "" {  //절대 경로로.
			absdatadir, err := filepath.Abs(conf.DataDir)
			if err != nil {
				return nil, err
			}
			conf.DataDir = absdatadir
		}
		// Ensure that the instance name doesn't cause weird conflicts with
		// other files in the data directory.
		if strings.ContainsAny(conf.Name, `/\`) {
			return nil, errors.New(`Config.Name must not contain '/' or '\'`)
		}
		if conf.Name == datadirDefaultKeyStore {
			return nil, errors.New(`Config.Name cannot be "` + datadirDefaultKeyStore + `"`)
		}
		if strings.HasSuffix(conf.Name, ".ipc") {
			return nil, errors.New(`Config.Name cannot end in ".ipc"`)
		}
		// Ensure that the AccountManager method works before the node has started.
		// We rely on this in cmd/geth.
		am, ephemeralKeystore, err := makeAccountManager(conf)
		if err != nil {
			return nil, err
		}
		// Note: any interaction with Config that would create/touch files
		// in the data directory or instance directory is delayed until Start.
		return &Node{
			accman:			am,
			ephemeralKeystore: ephemeralKeystore,
			config:			conf,
			serviceFuncs:	  []ServiceConstructor{},
			ipcEndpoint:	   conf.IPCEndpoint(),
			httpEndpoint:	  conf.HTTPEndpoint(),
			wsEndpoint:		conf.WSEndpoint(),
			eventmux:		  new(event.TypeMux),
		}, nil
	}


### 노드 서비스 및 프로토콜 서명
노드는 특정 비즈니스 로직을 담당하지 않기 때문에. 따라서, 특정 비즈니스 로직은 노드 내부에 와서 등록하여 등록합니다.
다른 모듈은 서비스 생성자를 등록하는 방법을 등록합니다. 서비스를 생성 할 수있는이 서비스 생성자를 사용.


	// Register injects a new service into the node's stack. The service created by
	// the passed constructor must be unique in its type with regard to sibling ones.
	func (n *Node) Register(constructor ServiceConstructor) error {
		n.lock.Lock()
		defer n.lock.Unlock()
	
		if n.server != nil {
			return ErrNodeRunning
		}
		n.serviceFuncs = append(n.serviceFuncs, constructor)
		return nil
	}

무엇 서비스입니다

	type ServiceConstructor func(ctx *ServiceContext) (Service, error)
	// Service is an individual protocol that can be registered into a node.
	//
	// Notes:
	//
	//• 서비스 라이프 사이클 관리는 노드에 위임되어있다.이 서비스는 허용된다
	// initialize itself upon creation, but no goroutines should be spun up outside of the
	// Start method.
	//
	//• 다시 시작 로직은 새로운 인스턴스를 생성 할 노드로 필요하지 않습니다
	// every time a service is started.

	//라이프 사이클 관리 서비스는 노드 관리에 표현했다. 당신이 만들 때이 서비스는 자동으로 초기화 할 수 있지만 그것은 시작 방법 밖에 goroutines을 시작해서는 안된다.
	//노드가 새로운 인스턴스에게이 서비스를 시작할 때마다 만들 수 있기 때문에 다시 시작 논리가 필요하지 않습니다.
	type Service interface {
		// Protocols retrieves the P2P protocols the service wishes to start.
		//P2P 프로토콜 서비스 제공 희망
		Protocols() []p2p.Protocol
		
		// APIs retrieves the list of RPC descriptors the service provides
		//서비스의 희망에 설명 된 RPC 방법 제공
		APIs() []rpc.API
	
		// Start is called after all services have been constructed and the networking
		// layer was also initialized to spawn any goroutines required by the service.
		//모든 서비스가 구축 된 후에는 전화를 시작하고, 네트워크 계층은 어떤 goroutine 필요한 서비스를 생산하기 위해 초기화됩니다.
		Start(server *p2p.Server) error
	
		// Stop terminates all goroutines belonging to the service, blocking until they
		// are all terminated.
		
		//서비스가 모두 goroutine을 중지합니다이 방법을 중지합니다. 모든 goroutine이 차단 될 필요가 종료되었습니다
		Stop() error
	}


### 노드 시작
노드 시작 프로세스를 생성하고 P2P의 노드를 실행합니다.

	// Start create a live P2P node and starts running it.
	func (n *Node) Start() error {
		n.lock.Lock()
		defer n.lock.Unlock()
	
		// Short circuit if the node's already running
		if n.server != nil {
			return ErrNodeRunning
		}
		if err := n.openDataDir(); err != nil {
			return err
		}
	
		// Initialize the p2p server. This creates the node key and
		// discovery databases.
		n.serverConfig = n.config.P2P
		n.serverConfig.PrivateKey = n.config.NodeKey()
		n.serverConfig.Name = n.config.NodeName()
		if n.serverConfig.StaticNodes == nil {
			//처리 구성 파일 정전기 nodes.json
			n.serverConfig.StaticNodes = n.config.StaticNodes()
		}
		if n.serverConfig.TrustedNodes == nil {
			//처리 프로파일 신뢰할-nodes.json
			n.serverConfig.TrustedNodes = n.config.TrustedNodes()
		}
		if n.serverConfig.NodeDatabase == "" {
			n.serverConfig.NodeDatabase = n.config.NodeDB()
		}
		//P2P는 서버 만들기
		running := &p2p.Server{Config: n.serverConfig}
		log.Info("Starting peer-to-peer node", "instance", n.serverConfig.Name)
	
		// Otherwise copy and specialize the P2P configuration
		services := make(map[reflect.Type]Service)
		for _, constructor := range n.serviceFuncs {
			// Create a new context for the particular service
			ctx := &ServiceContext{
				config:		 n.config,
				services:	   make(map[reflect.Type]Service),
				EventMux:	   n.eventmux,
				AccountManager: n.accman,
			}
			for kind, s := range services { // copy needed for threaded access
				ctx.services[kind] = s
			}
			// Construct and save the service
			//등록 된 모든 서비스를 만들기.
			service, err := constructor(ctx)
			if err != nil {
				return err
			}
			kind := reflect.TypeOf(service)
			if _, exists := services[kind]; exists {
				return &DuplicateServiceError{Kind: kind}
			}
			services[kind] = service
		}
		// Gather the protocols and start the freshly assembled P2P server
		//모든 P2P 프로토콜을 수집하고 p2p.Rrotocols를 삽입
		for _, service := range services {
			running.Protocols = append(running.Protocols, service.Protocols()...)
		}
		//P2P는 서버 출시
		if err := running.Start(); err != nil {
			return convertFileLockError(err)
		}
		// Start each of the services
		//각 서비스를 시작합니다
		started := []reflect.Type{}
		for kind, service := range services {
			// Start the next service, stopping all previous upon failure
			if err := service.Start(running); err != nil {
				for _, kind := range started {
					services[kind].Stop()
				}
				running.Stop()
	
				return err
			}
			// Mark the service started for potential cleanup
			started = append(started, kind)
		}
		// Lastly start the configured RPC interfaces
		//마지막으로, RPC 서비스를 시작
		if err := n.startRPC(services); err != nil {
			for _, service := range services {
				service.Stop()
			}
			running.Stop()
			return err
		}
		// Finish initializing the startup
		n.services = services
		n.server = running
		n.stop = make(chan struct{})
	
		return nil
	}


startRPC,이 방법은 모든 API를 수집하는 것입니다. 그리고 차례로 각 RPC 서버의 시작, 기본값에서 InProc 및 IPC를 시작하는 것입니다 호출합니다. 당신은 또한 HTTP 및 웹 소켓을 활성화할지 여부를 구성 할 수 있습니다 지정합니다.

	// startRPC is a helper method to start all the various RPC endpoint during node
	// startup. It's not meant to be called at any time afterwards as it makes certain
	// assumptions about the state of the node.
	func (n *Node) startRPC(services map[reflect.Type]Service) error {
		// Gather all the possible APIs to surface
		apis := n.apis()
		for _, service := range services {
			apis = append(apis, service.APIs()...)
		}
		// Start the various API endpoints, terminating all in case of errors
		if err := n.startInProc(apis); err != nil {
			return err
		}
		if err := n.startIPC(apis); err != nil {
			n.stopInProc()
			return err
		}
		if err := n.startHTTP(n.httpEndpoint, apis, n.config.HTTPModules, n.config.HTTPCors); err != nil {
			n.stopIPC()
			n.stopInProc()
			return err
		}
		if err := n.startWS(n.wsEndpoint, apis, n.config.WSModules, n.config.WSOrigins, n.config.WSExposeAll); err != nil {
			n.stopHTTP()
			n.stopIPC()
			n.stopInProc()
			return err
		}
		// All API endpoints started successfully
		n.rpcAPIs = apis
		return nil
	}


startXXX 특정 RPC의 시작이다. 프로세스는 유사하다. 여기에 보면 StartWS

	// startWS initializes and starts the websocket RPC endpoint.
	func (n *Node) startWS(endpoint string, apis []rpc.API, modules []string, wsOrigins []string, exposeAll bool) error {
		// Short circuit if the WS endpoint isn't being exposed
		if endpoint == "" {
			return nil
		}
		// Generate the whitelist based on the allowed modules
		//화이트리스트를 생성
		whitelist := make(map[string]bool)
		for _, module := range modules {
			whitelist[module] = true
		}
		// Register all the APIs exposed by the services
		handler := rpc.NewServer()
		for _, api := range apis {
			if exposeAll || whitelist[api.Namespace] || (len(whitelist) == 0 && api.Public) {
			//이런 상황에서 만 등록이 API에 초점을 맞출 것이다.
				if err := handler.RegisterName(api.Namespace, api.Service); err != nil {
					return err
				}
				log.Debug(fmt.Sprintf("WebSocket registered %T under '%s'", api.Service, api.Namespace))
			}
		}
		// All APIs registered, start the HTTP listener
		var (
			listener net.Listener
			err	  error
		)
		if listener, err = net.Listen("tcp", endpoint); err != nil {
			return err
		}
		go rpc.NewWSServer(wsOrigins, handler).Serve(listener)
		log.Info(fmt.Sprintf("WebSocket endpoint opened: ws://%s", listener.Addr()))
	
		// All listeners booted successfully
		n.wsEndpoint = endpoint
		n.wsListener = listener
		n.wsHandler = handler
	
		return nil
	}