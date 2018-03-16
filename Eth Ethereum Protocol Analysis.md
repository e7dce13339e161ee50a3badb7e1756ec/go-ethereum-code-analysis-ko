
에서 서비스 노드의 정의, ETH 실제로 서비스를 구현한다.

	type Service interface {
		// Protocols retrieves the P2P protocols the service wishes to start.
		Protocols() []p2p.Protocol
	
		// APIs retrieves the list of RPC descriptors the service provides
		APIs() []rpc.API
	
		// Start is called after all services have been constructed and the networking
		// layer was also initialized to spawn any goroutines required by the service.
		Start(server *p2p.Server) error
	
		// Stop terminates all goroutines belonging to the service, blocking until they
		// are all terminated.
		Stop() error
	}

ETH 디렉토리의 에테 리움은 광장 이더넷 서비스를 달성하는 것입니다 이동합니다. 광장 프로토콜 분사의 등록 방법에 의해 이더넷 노드이다.


	// RegisterEthService adds an Ethereum client to the stack.
	func RegisterEthService(stack *node.Node, cfg *eth.Config) {
		var err error
		if cfg.SyncMode == downloader.LightSync {
			err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
				return les.New(ctx, cfg)
			})
		} else {
			err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
				fullNode, err := eth.New(ctx, cfg)
				if fullNode != nil && cfg.LightServ > 0 {
					ls, _ := les.NewLesServer(fullNode, cfg)
					fullNode.AddLesServer(ls)
				}
				return fullNode, err
			})
		}
		if err != nil {
			Fatalf("Failed to register the Ethereum service: %v", err)
		}
	}

광장 이더넷 프로토콜 데이터 구조
	
	// Ethereum implements the Ethereum full node service.
	type Ethereum struct {
		config	  *Config				구성
		chainConfig *params.ChainConfig	체인 구성
	
		// Channel for shutting down the service
		shutdownChan  chan bool	// Channel for shutting down the ethereum
		stopDbUpgrade func() error // stop chain db sequential key upgrade
	
		// Handlers
		txPool		  *core.TxPool		무역 풀
		blockchain	  *core.BlockChain	블록 체인
		protocolManager *ProtocolManager	관리 계약
		lesServer	   LesServer			경량 클라이언트 서버
	
		// DB interfaces
		chainDb ethdb.Database // Block chain database블록 체인 데이터베이스
	
		eventMux	   *event.TypeMux
		engine		 consensus.Engine			일관성 엔진. 그것은 탕의 일부가되어야
		accountManager *accounts.Manager		계정 관리
	
		bloomRequests chan chan *bloombits.Retrieval // Channel receiving bloom data retrieval requests채널 수신기 꽃 필터 데이터 요청
		bloomIndexer  *core.ChainIndexer		 블록 가져 오기시 블룸 인덱서 작업을 수행 // 수입 일시적으로 차단하는 동안 운영 // 블룸 인덱서 무엇을 명확하지 않다
	
		ApiBackend *EthApiBackend	// 백 엔드 API의 RPC 서비스 이용에 제공
	
		miner	 *miner.Miner		// 광부
		gasPrice  *big.Int			// gasPrice의 최소값을 수신하는 노드. 현재 노드의 값보다 작다이 거부 될
		etherbase common.Address	// 광부 주소
	
		networkId	 uint64		네트워크 ID testnet 1 @ 0이다 mainnet
		netRPCService *ethapi.PublicNetAPI// RPC 서비스
	
		lock sync.RWMutex // Protects the variadic fields (e.g. gas price and etherbase)
	}

핵심 내용을 포함하지 않는 첫 번째 서비스를 제공합니다. 새로운 광장 이더넷 프로토콜을 만듭니다. 단지에 대해 알려주십시오. 후속 내부 핵심 내용을 분석한다.

	// New creates a new Ethereum object (including the
	// initialisation of the common Ethereum object)
	func New(ctx *node.ServiceContext, config *Config) (*Ethereum, error) {
		if config.SyncMode == downloader.LightSync {
			return nil, errors.New("can't run eth.Ethereum in light sync mode, use les.LightEthereum")
		}
		if !config.SyncMode.IsValid() {
			return nil, fmt.Errorf("invalid sync mode %d", config.SyncMode)
		}
		//leveldb 만들기. 열거 나 새로운 디렉토리 chaindata를 만들
		chainDb, err := CreateDB(ctx, config, "chaindata")
		if err != nil {
			return nil, err
		}
		//업그레이드 데이터베이스 형식
		stopDbUpgrade := upgradeDeduplicateData(chainDb)
		//설정 창조 블록. 그렇다면 데이터베이스 내부로부터 (개인 쇄)를 제거한 데이터베이스 작성 블럭. 아니면 디폴트 값은 내부의 코드에서 인수했다.
		chainConfig, genesisHash, genesisErr := core.SetupGenesisBlock(chainDb, config.Genesis)
		if _, ok := genesisErr.(*params.ConfigCompatError); genesisErr != nil && !ok {
			return nil, genesisErr
		}
		log.Info("Initialised chain configuration", "config", chainConfig)
	
		eth := &Ethereum{
			config:		 config,
			chainDb:		chainDb,
			chainConfig:	chainConfig,
			eventMux:	   ctx.EventMux,
			accountManager: ctx.AccountManager,
			engine:	 CreateConsensusEngine (CTX, 설정, chainConfig, chainDb), // 일관성 엔진. 여기 내 이해 탕입니다
			shutdownChan:   make(chan bool),
			stopDbUpgrade:  stopDbUpgrade,
			networkId:  config.NetworkId은, // 네트워크 ID는 네트워크를 구분합니다. 테스트 네트워크는 주 네트워크가 0.1이다
			gasPrice:   고객이 거래를 승인 --gasprice config.GasPrice은, // 당신은 gasprice 최소 구성 할 수 있습니다. 그 노드가 폐기 될이 값보다 작은 경우.
			etherbase:  config.Etherbase, // 광산 수혜자
			bloomRequests:  make(chan chan *bloombits.Retrieval),  //꽃 요청
			bloomIndexer:   NewBloomIndexer(chainDb, params.BloomBitsBlocks),
		}
	
		log.Info("Initialising Ethereum protocol", "versions", ProtocolVersions, "network", config.NetworkId)
	
		if !config.SkipBcVersionCheck { //데이터베이스의 BlockChainVersion 버전은 저장 BlockChainVersion 내부를 확인하고 클라이언트가 동일
			bcVersion := core.GetBlockChainVersion(chainDb)
			if bcVersion != core.BlockChainVersion && bcVersion != 0 {
				return nil, fmt.Errorf("Blockchain DB version mismatch (%d / %d). Run geth upgradedb.\n", bcVersion, core.BlockChainVersion)
			}
			core.WriteBlockChainVersion(chainDb, core.BlockChainVersion)
		}
	
		vmConfig := vm.Config{EnablePreimageRecording: config.EnablePreimageRecording}
		//블록 체인을 사용하여 데이터베이스를 생성
		eth.blockchain, err = core.NewBlockChain(chainDb, eth.chainConfig, eth.engine, vmConfig)
		if err != nil {
			return nil, err
		}
		// Rewind the chain in case of an incompatible config upgrade.
		if compat, ok := genesisErr.(*params.ConfigCompatError); ok {
			log.Warn("Rewinding chain to upgrade configuration", "err", compat)
			eth.blockchain.SetHead(compat.RewindTo)
			core.WriteChainConfig(chainDb, genesisHash, chainConfig)
		}
		//bloomIndexer 나는 많이하지 포함하는 것을 무엇인지 몰랐다. 상관없이 첫번째 있다는
		eth.bloomIndexer.Start(eth.blockchain.CurrentHeader(), eth.blockchain.SubscribeChainEvent)
	
		if config.TxPool.Journal != "" {
			config.TxPool.Journal = ctx.ResolvePath(config.TxPool.Journal)
		}
		//거래 구덩이를 만듭니다. 기억 트랜잭션은 로컬 또는 네트워크를 통해 수신.
		eth.txPool = core.NewTxPool(config.TxPool, eth.chainConfig, eth.blockchain)
		//프로토콜 관리자 만들기
		if eth.protocolManager, err = NewProtocolManager(eth.chainConfig, config.SyncMode, config.NetworkId, eth.eventMux, eth.txPool, eth.engine, eth.blockchain, chainDb); err != nil {
			return nil, err
		}
		//광부 만들기
		eth.miner = miner.New(eth, eth.chainConfig, eth.EventMux(), eth.engine)
		eth.miner.SetExtra(makeExtraData(config.ExtraData))
		//ApiBackend는 백엔드 지원 RPC 호출을 제공하는 데 사용
		eth.ApiBackend = &EthApiBackend{eth, nil}
		//gpoParams GPO 가스 가격 오라클의 약어. GasPrice이 예상된다. 최근 트랜잭션에 의해 현재의 값 GasPrice을 예측합니다. 이 값은 다음과 같이 사용할 수 있습니다 후 참조 거래 비용을 보낼 수 있습니다.
		gpoParams := config.GPO
		if gpoParams.Default == nil {
			gpoParams.Default = config.GasPrice
		}
		eth.ApiBackend.gpo = gasprice.NewOracle(eth.ApiBackend, gpoParams)
	
		return eth, nil
	}

api_backend.go 파일에 정의 ApiBackend. 그것은 몇 가지 기능을 캡슐화합니다.

	// EthApiBackend implements ethapi.Backend for full nodes
	type EthApiBackend struct {
		eth *Ethereum
		gpo *gasprice.Oracle
	}
	func (b *EthApiBackend) SetHead(number uint64) {
		b.eth.protocolManager.downloader.Cancel()
		b.eth.blockchain.SetHead(number)
	}

어떤 방법의 핵심에 추가하여 새로운 방법, 이더넷 프로토콜에서 더 중요한 목적 ProtocolManager 광장이, 이더넷 광장은 원래 프로토콜이었다. ProtocolManager 또한 여러 하위 프로토콜 이더넷 광장을 관리 할 수 ​​있습니다.

	
	// NewProtocolManager returns a new ethereum sub protocol manager. The Ethereum sub protocol manages peers capable
	// with the ethereum network.
	func NewProtocolManager(config *params.ChainConfig, mode downloader.SyncMode, networkId uint64, mux *event.TypeMux, txpool txPool, engine consensus.Engine, blockchain *core.BlockChain, chaindb ethdb.Database) (*ProtocolManager, error) {
		// Create the protocol manager with the base fields
		manager := &ProtocolManager{
			networkId:   networkId,
			eventMux:	mux,
			txpool:	  txpool,
			blockchain:  blockchain,
			chaindb:	 chaindb,
			chainconfig: config,
			peers:	   newPeerSet(),
			newPeerCh:   make(chan *peer),
			noMorePeers: make(chan struct{}),
			txsyncCh:	make(chan *txsync),
			quitSync:	make(chan struct{}),
		}
		// Figure out whether to allow fast sync or not
		if mode == downloader.FastSync && blockchain.CurrentBlock().NumberU64() > 0 {
			log.Warn("Blockchain not empty, fast sync disabled")
			mode = downloader.FullSync
		}
		if mode == downloader.FastSync {
			manager.fastSync = uint32(1)
		}
		// Initiate a sub-protocol for every implemented version we can handle
		manager.SubProtocols = make([]p2p.Protocol, 0, len(ProtocolVersions))
		for i, version := range ProtocolVersions {
			// Skip protocol version if incompatible with the mode of operation
			if mode == downloader.FastSync && version < eth63 {
				continue
			}
			// Compatible; initialise the sub-protocol
			version := version // Closure for the run
			manager.SubProtocols = append(manager.SubProtocols, p2p.Protocol{
				Name:	ProtocolName,
				Version: version,
				Length:  ProtocolLengths[i],
				//프로토콜 내부에 어떤 P2P를 기억하십시오. P2P는 피어가 성공적으로 연결 한 후 실행 메소드를 호출
				Run: func(p *p2p.Peer, rw p2p.MsgReadWriter) error {
					peer := manager.newPeer(int(version), p, rw)
					select {
					case manager.newPeerCh <- peer:
						manager.wg.Add(1)
						defer manager.wg.Done()
						return manager.handle(peer)
					case <-manager.quitSync:
						return p2p.DiscQuitting
					}
				},
				NodeInfo: func() interface{} {
					return manager.NodeInfo()
				},
				PeerInfo: func(id discover.NodeID) interface{} {
					if p := manager.peers.Peer(fmt.Sprintf("%x", id[:8])); p != nil {
						return p.Info()
					}
					return nil
				},
			})
		}
		if len(manager.SubProtocols) == 0 {
			return nil, errIncompatibleConfig
		}
		// Construct the different synchronisation mechanisms
		//다운로더는 다른 피어에서 데이터를 동기화 할 책임이있다.
		//로더는 전체 체인 동기화 도구
		manager.downloader = downloader.New(mode, chaindb, manager.eventMux, blockchain, nil, manager.removePeer)
		//검사기 함수는 헤더 영역의 일관성을 검증 엔진을 사용하는 것이다
		validator := func(header *types.Header) error {
			return engine.VerifyHeader(blockchain, header, true)
		}
		//기능 블록의 높이를 반환
		heighter := func() uint64 {
			return blockchain.CurrentBlock().NumberU64()
		}
		//빠른 동기화가 켜져있는 경우. 그것은 삽입를 호출하지 않습니다.
		inserter := func(blocks types.Blocks) (int, error) {
			// If fast sync is running, deny importing weird blocks
			if atomic.LoadUint32(&manager.fastSync) == 1 {
				log.Warn("Discarded bad propagated block", "number", blocks[0].Number(), "hash", blocks[0].Hash())
				return 0, nil
			}
			//무역 수신을 시작하도록 설정
			atomic.StoreUint32(&manager.acceptTxs, 1) // Mark initial sync done on any fetcher import
			//삽입 블록
			return manager.blockchain.InsertChain(blocks)
		}
		//가져 오기를 생성
		//각 피어 통지에서 블록의 축적에 대한 책임을 가져 오기 및 검색을 배열합니다.
		manager.fetcher = fetcher.New(blockchain.GetBlockByHash, validator, manager.BroadcastBlock, heighter, inserter, manager.removePeer)
	
		return manager, nil
	}


API를 서비스 () 메소드는 노출 된 RPC 서비스 방법을 반환합니다.

	// APIs returns the collection of RPC services the ethereum package offers.
	// NOTE, some of these services probably need to be moved to somewhere else.
	func (s *Ethereum) APIs() []rpc.API {
		apis := ethapi.GetAPIs(s.ApiBackend)
	
		// Append any APIs exposed explicitly by the consensus engine
		apis = append(apis, s.engine.APIs(s.BlockChain())...)
	
		// Append all the local APIs and return
		return append(apis, []rpc.API{
			{
				Namespace: "eth",
				Version:   "1.0",
				Service:   NewPublicEthereumAPI(s),
				Public:	true,
			},
			...
			, {
				Namespace: "net",
				Version:   "1.0",
				Service:   s.netRPCService,
				Public:	true,
			},
		}...)
	}

서비스 메서드가 반환하는 서비스의 프로토콜은 프로토콜을 P2P의 사람들을 제공합니다. 내부의 모든 하위 프로토콜 프로토콜 매니저를 돌려줍니다.이 lesServer 경우도 프로토콜 lesServer을 제공합니다. 당신은 볼 수 있습니다. 모든 네트워크 기능은 길에서 프로토콜에 의해 제공됩니다.

	// Protocols implements node.Service, returning all the currently configured
	// network protocols to start.
	func (s *Ethereum) Protocols() []p2p.Protocol {
		if s.lesServer == nil {
			return s.protocolManager.SubProtocols
		}
		return append(s.protocolManager.SubProtocols, s.lesServer.Protocols()...)
	}


에테 리움 서비스 시작 방법 작성 후, 서비스를 호출됩니다. 우리가 시작하는 방법을 살펴 보자
	
	// Start implements node.Service, starting all internal goroutines needed by the
	// Ethereum protocol implementation.
	func (s *Ethereum) Start(srvr *p2p.Server) error {
		// Start the bloom bits servicing goroutines
		//블룸 필터 goroutine TODO 시작 요청 프로세스
		s.startBloomHandlers()
	
		// Start the RPC service
		//만들기 API 순 네트워크
		s.netRPCService = ethapi.NewPublicNetAPI(srvr, s.NetVersion())
	
		// Figure out a max peers count based on the server limits
		maxPeers := srvr.MaxPeers
		if s.config.LightServ > 0 {
			maxPeers -= s.config.LightPeers
			if maxPeers < srvr.MaxPeers/2 {
				maxPeers = srvr.MaxPeers / 2
			}
		}
		// Start the networking layer and the light server if requested
		//개시 프로토콜 관리자
		s.protocolManager.Start(maxPeers)
		if s.lesServer != nil {
			//시작되지 않으면 lesServer은 전무하다.
			s.lesServer.Start(srvr)
		}
		return nil
	}

패브릭 프로토콜 데이터 관리자

	type ProtocolManager struct {
		networkId uint64
	
		fastSync  uint32 // Flag whether fast sync is enabled (gets disabled if we already have blocks)
		acceptTxs uint32 // Flag whether we're considered synchronised (enables transaction processing)
	
		txpool	  txPool
		blockchain  *core.BlockChain
		chaindb	 ethdb.Database
		chainconfig *params.ChainConfig
		maxPeers	int
	
		downloader *downloader.Downloader
		fetcher	*fetcher.Fetcher
		peers	  *peerSet
	
		SubProtocols []p2p.Protocol
	
		eventMux	  *event.TypeMux
		txCh		  chan core.TxPreEvent
		txSub		 event.Subscription
		minedBlockSub *event.TypeMuxSubscription
	
		// channels for fetcher, syncer, txsyncLoop
		newPeerCh   chan *peer
		txsyncCh	chan *txsync
		quitSync	chan struct{}
		noMorePeers chan struct{}
	
		// wait group is used for graceful shutdowns during downloading
		// and processing
		wg sync.WaitGroup
	}

방법 프로토콜 관리자를 시작합니다. 거래의 다양한 처리 할 수 ​​goroutine의 번호를 시작한이 방법은,이 클래스는 기본 구현 클래스 이더넷 서비스의 광장해야한다고 추측 할 수있다.
	
	func (pm *ProtocolManager) Start(maxPeers int) {
		pm.maxPeers = maxPeers
		
		// broadcast transactions
		//방송 채널 거래. TxPreEvent txCh는 txpool 가입 채널로합니다. 이 메시지와 txpool는 txCh을 알려 드리고자합니다. goroutine 거래는 방송 뉴스를 방송합니다.
		pm.txCh = make(chan core.TxPreEvent, txChanSize)
		//가입 영수증
		pm.txSub = pm.txpool.SubscribeTxPreEvent(pm.txCh)
		//방송 시작 Goroutine
		go pm.txBroadcastLoop()
	
		// broadcast mined blocks
		//뉴스 광산 구독합니다. 새로운 블록은 뉴스에서 파고되어야 할 것이다 때. 구독하고 구독되지 않는 것으로 표시되고 두 개의 상이한 모드를 사용하여 가입 상술.
		pm.minedBlockSub = pm.eventMux.Subscribe(core.NewMinedBlockEvent{})
		//필요가 가능한 한 빨리 가기 네트워크를 방송 할 때 발굴 때 방송 goroutine 광업.
		go pm.minedBroadcastLoop()
	
		// start sync handlers
		//동기화는 주기적으로 네트워크 및 처리 블록과 해시 다운로드 알림 핸들러와 동기화 할 책임이 있습니다.
		go pm.syncer()
		//원래 트랜잭션 동기화 각 새 연결에 대한 책임을 txsyncLoop. 새로운 피어가 나타나면, 우리는 현재 계류중인 트랜잭션을 전달합니다. 대역폭 사용 수출을 최소화하기 위해, 우리는 한 번만 패킷을 보낼 수 있습니다.
		go pm.txsyncLoop()
	}


P2P의 서버가 시작되면 연결에 노드를 찾기 위해 주도권을 쥐고, 또는 다른 노드에 연결됩니다. 프로세스는 핸드 쉐이크 프로토콜 다음에 먼저 암호화 된 채널 핸드 쉐이킹을 연결되어 있습니다. 각 프로토콜이 최종 합의에 제어를 전송하기 위해 마지막으로, goroutine 실행 실행 방법을 시작합니다. 실행 방법은 먼저 피어 개체를 만든 다음이 핸들 피어를 처리하는 방법을 호출
	
	Run: func(p *p2p.Peer, rw p2p.MsgReadWriter) error {
						peer := manager.newPeer(int(version), p, rw)
						select {
						case manager.newPeerCh <- peer:  //채널 newPeerCh에 피어를 전송
							manager.wg.Add(1)
							defer manager.wg.Done()
							return manager.handle(peer)  //방법 handlo 전화
						case <-manager.quitSync:
							return p2p.DiscQuitting
						}
					},


핸들 방법,

	
	// handle is the callback invoked to manage the life cycle of an eth peer. When
	// this function terminates, the peer is disconnected.
	//핸들은 피어 ETH의 라이프 사이클 관리를 관리하기위한 콜백 방법이다. 이 방법은 종료하면 피어 연결이 끊어집니다.
	func (pm *ProtocolManager) handle(p *peer) error {
		if pm.peers.Len() >= pm.maxPeers {
			return p2p.DiscTooManyPeers
		}
		p.Log().Debug("Ethereum peer connected", "name", p.Name())
	
		// Execute the Ethereum handshake
		td, head, genesis := pm.blockchain.Status()
		//TD 총 어려운 헤드 영역이 현재 헤더 창 정보 작성 부이다. 창조는 같은 핸드 쉐이크가 성공적으로 차단할 수 있습니다.
		if err := p.Handshake(pm.networkId, td, head, genesis); err != nil {
			p.Log().Debug("Ethereum handshake failed", "err", err)
			return err
		}
		if rw, ok := p.rw.(*meteredMsgReadWriter); ok {
			rw.Init(p.version)
		}
		// Register the peer locally
		//지역에 피어 등록
		if err := pm.peers.Register(p); err != nil {
			p.Log().Error("Ethereum peer registration failed", "err", err)
			return err
		}
		defer pm.removePeer(p.id)
	
		// Register the peer in the downloader. If the downloader considers it banned, we disconnect
		//이 피어 다운 이렇게 분리 금지 생각하면 피어 다운 로더. 등록합니다.
		if err := pm.downloader.RegisterPeer(p.id, p.version, p); err != nil {
			return err
		}
		// Propagate existing transactions. new transactions appearing
		// after this will be sent via broadcasts.
		//다른 측면에 현재 계류중인 트랜잭션에 보낸,이 연결이 방금 만든 경우에만 발생
		pm.syncTransactions(p)
	
		// If we're DAO hard-fork aware, validate any remote peer with regard to the hard-fork
		//DAO의 피어 하드 분기점 확인
		if daoBlock := pm.chainconfig.DAOForkBlock; daoBlock != nil {
			// Request the peer's DAO fork header for extra-data validation
			if err := p.RequestHeadersByNumber(daoBlock.Uint64(), 1, 0, false); err != nil {
				return err
			}
			// Start a timer to disconnect if the peer doesn't reply in time
			//경우 응답은 15 초 내에 수신되지 않습니다. 그런 다음 연결.
			p.forkDrop = time.AfterFunc(daoChallengeTimeout, func() {
				p.Log().Debug("Timed out DAO fork-check, dropping")
				pm.removePeer(p.id)
			})
			// Make sure it's cleaned up if the peer dies off
			defer func() {
				if p.forkDrop != nil {
					p.forkDrop.Stop()
					p.forkDrop = nil
				}
			}()
		}
		// main loop. handle incoming messages.
		//주요 루프. 수신 메시지를 처리.
		for {
			if err := pm.handleMsg(p); err != nil {
				p.Log().Debug("Ethereum message handling failed", "err", err)
				return err
			}
		}
	}


Handshake
	
	// Handshake executes the eth protocol handshake, negotiating version number,
	// network IDs, difficulties, head and genesis blocks.
	func (p *peer) Handshake(network uint64, td *big.Int, head common.Hash, genesis common.Hash) error {
		// Send out own handshake in a new thread
		//오차의 크기는 채널 2, 일회성 처리의 두 가지 방법에 따라 goroutine하는
		errc := make(chan error, 2)
		var status statusData // safe to read after two values have been received from errc
	
		go func() {
			errc <- p2p.Send(p.rw, StatusMsg, &statusData{
				ProtocolVersion: uint32(p.version),
				NetworkId:	   network,
				TD:			  td,
				CurrentBlock:	head,
				GenesisBlock:	genesis,
			})
		}()
		go func() {
			errc <- p.readStatus(network, &status, genesis)
		}()
		timeout := time.NewTimer(handshakeTimeout)
		defer timeout.Stop()
		//어떤 에러 (송신, 수신) 또는 타임 아웃을 수신한다면, 분리.
		for i := 0; i < 2; i++ {
			select {
			case err := <-errc:
				if err != nil {
					return err
				}
			case <-timeout.C:
				return p2p.DiscReadTimeout
			}
		}
		p.td, p.head = status.TD, status.CurrentBlock
		return nil
	}

readStatus는 다양한 조건의 시험에서 반환

	func (p *peer) readStatus(network uint64, status *statusData, genesis common.Hash) (err error) {
		msg, err := p.rw.ReadMsg()
		if err != nil {
			return err
		}
		if msg.Code != StatusMsg {
			return errResp(ErrNoStatusMsg, "first msg has code %x (!= %x)", msg.Code, StatusMsg)
		}
		if msg.Size > ProtocolMaxMsgSize {
			return errResp(ErrMsgTooLarge, "%v > %v", msg.Size, ProtocolMaxMsgSize)
		}
		// Decode the handshake and make sure everything matches
		if err := msg.Decode(&status); err != nil {
			return errResp(ErrDecode, "msg %v: %v", msg, err)
		}
		if status.GenesisBlock != genesis {
			return errResp(ErrGenesisBlockMismatch, "%x (!= %x)", status.GenesisBlock[:8], genesis[:8])
		}
		if status.NetworkId != network {
			return errResp(ErrNetworkIdMismatch, "%d (!= %d)", status.NetworkId, network)
		}
		if int(status.ProtocolVersion) != p.version {
			return errResp(ErrProtocolVersionMismatch, "%d (!= %d)", status.ProtocolVersion, p.version)
		}
		return nil
	}

지도에서 자신의 동료로 피어 간단 등록

	// Register injects a new peer into the working set, or returns an error if the
	// peer is already known.
	func (ps *peerSet) Register(p *peer) error {
		ps.lock.Lock()
		defer ps.lock.Unlock()
	
		if ps.closed {
			return errClosed
		}
		if _, ok := ps.peers[p.id]; ok {
			return errAlreadyRegistered
		}
		ps.peers[p.id] = p
		return nil
	}


손 일련의 검사 후 흔들어, 전화 handleMsg 방법의주기는 이벤트 루프를 처리합니다. 이 방법은 주로 다양한 메시지를받은 후 응답을 다루는 매우 깁니다.
	
	// handleMsg is invoked whenever an inbound message is received from a remote
	// peer. The remote connection is turn down upon returning any error.
	func (pm *ProtocolManager) handleMsg(p *peer) error {
		// Read the next message from the remote peer, and ensure it's fully consumed
		msg, err := p.rw.ReadMsg()
		if err != nil {
			return err
		}
		if msg.Size > ProtocolMaxMsgSize {
			return errResp(ErrMsgTooLarge, "%v > %v", msg.Size, ProtocolMaxMsgSize)
		}
		defer msg.Discard()
	
		// Handle the message depending on its contents
		switch {
		case msg.Code == StatusMsg:
			// Status messages should never arrive after the handshake
			//StatusMsg는 HandleShake 단계에서 받아야합니다. HandleShake는 메시지를 수신 할 수 없습니다해야 후.
			return errResp(ErrExtraStatusMsg, "uncontrolled status message")
	
		// Block header query, collect the requested headers and reply
		//상기 요청 메시지의 헤더 영역을 수신하는 정보 영역 헤더 요청에 따라 반환한다.
		case msg.Code == GetBlockHeadersMsg:
			// Decode the complex header query
			var query getBlockHeadersData
			if err := msg.Decode(&query); err != nil {
				return errResp(ErrDecode, "%v: %v", msg, err)
			}
			hashMode := query.Origin.Hash != (common.Hash{})
	
			// Gather headers until the fetch or network limits is reached
			var (
				bytes   common.StorageSize
				headers []*types.Header
				unknown bool
			)
			for !unknown && len(headers) < int(query.Amount) && bytes < softResponseLimit && len(headers) < downloader.MaxHeaderFetch {
				// Retrieve the next header satisfying the query
				var origin *types.Header
				if hashMode {
					origin = pm.blockchain.GetHeaderByHash(query.Origin.Hash)
				} else {
					origin = pm.blockchain.GetHeaderByNumber(query.Origin.Number)
				}
				if origin == nil {
					break
				}
				number := origin.Number.Uint64()
				headers = append(headers, origin)
				bytes += estHeaderRlpSize
	
				// Advance to the next header of the query
				switch {
				case query.Origin.Hash != (common.Hash{}) && query.Reverse:
					// Hash based traversal towards the genesis block
					//생성 블록으로 이동하기 시작부터 지정된 해시. 즉, 운동을 반대한다. 해시로 찾기
					for i := 0; i < int(query.Skip)+1; i++ {
						if header := pm.blockchain.GetHeader(query.Origin.Hash, number); header != nil {//헤더 영역 및 숫자 해시 전에 의한 취득
						
							query.Origin.Hash = header.ParentHash
							number--
						} else {
							unknown = true
							break //브레이크 스위치를 벗어났습니다. 알 수없는 루프에서 사용된다.
						}
					}
				case query.Origin.Hash != (common.Hash{}) && !query.Reverse:
					// Hash based traversal towards the leaf block
					//해시하여 찾아
					var (
						current = origin.Number.Uint64()
						next	= current + query.Skip + 1
					)
					if next <= current { //앞으로 있지만, 현재보다 작은 다음, 준비 정수 오버 플로우 공격.
						infos, _ := json.MarshalIndent(p.Peer.Info(), "", "  ")
						p.Log().Warn("GetBlockHeaders skip overflow attack", "current", current, "skip", query.Skip, "next", next, "attacker", infos)
						unknown = true
					} else {
						if header := pm.blockchain.GetHeaderByNumber(next); header != nil {
							if pm.blockchain.GetBlockHashesFromHash(header.Hash(), query.Skip+1)[query.Skip] == query.Origin.Hash {
								//당신은 같은 체인에이 헤더,이 헤더와 기원을 찾을 수 있다면.
								query.Origin.Hash = header.Hash()
							} else {
								unknown = true
							}
						} else {
							unknown = true
						}
					}
				case query.Reverse:	// 번호로 찾기
					// Number based traversal towards the genesis block
					//  query.Origin.Hash == (common.Hash{}) 
					if query.Origin.Number >= query.Skip+1 {
						query.Origin.Number -= (query.Skip + 1)
					} else {
						unknown = true
					}
	
				case !query.Reverse: // 번호로 찾기
					// Number based traversal towards the leaf block
					query.Origin.Number += (query.Skip + 1)
				}
			}
			return p.SendBlockHeaders(headers)
	
		case msg.Code == BlockHeadersMsg: //GetBlockHeadersMsg는 회신을 받았다.
			// A batch of headers arrived to one of our previous requests
			var headers []*types.Header
			if err := msg.Decode(&headers); err != nil {
				return errResp(ErrDecode, "msg %v: %v", msg, err)
			}
			// If no headers were received, but we're expending a DAO fork check, maybe it's that
			//피어가 어떤 헤더를 반환하지 않으며, forkDrop이 비어 있지 않으면 우리가 핸드 셰이크에 요청 DAO 헤더를 보내기 전에, 그 요청은 우리의 DAO 체크해야한다.
			if len(headers) == 0 && p.forkDrop != nil {
				// Possibly an empty reply to the fork header checks, sanity check TDs
				verifyDAO := true
	
				// If we already have a DAO header, we can check the peer's TD against it. If
				// the peer's ahead of this, it too must have a reply to the DAO check
				if daoHeader := pm.blockchain.GetHeaderByNumber(pm.chainconfig.DAOForkBlock.Uint64()); daoHeader != nil {
					if _, td := p.Head(); td.Cmp(pm.blockchain.GetTd(daoHeader.Hash(), daoHeader.Number.Uint64())) >= 0 {
						//총 시간은 피어 어려운 TD 값을 초과하는 경우, 대향 면적이 헤더에 존재한다는 것을 나타내는 상기 DAO 분기점 블록을 초과하지만, 빈 리턴 하였는지를 확인하기 위해 검증이 실패. 여기에하지 말아야 할 것입니다. 당신이 끝을 전송하지 않으면 타임 아웃 종료 될 것입니다.
						verifyDAO = false
					}
				}
				// If we're seemingly on the same chain, disable the drop timer
				if verifyDAO { //검증이 성공하면, 다음 타이머를 삭제하고 돌아갑니다.
					p.Log().Debug("Seems to be on the same side of the DAO fork")
					p.forkDrop.Stop()
					p.forkDrop = nil
					return nil
				}
			}
			// Filter out any explicitly requested headers, deliver the rest to the downloader
			//어떤 아주 명확한 요청을 필터링 한 후 다운 로더의 나머지 부분에 전달
			//길이가 1 인 경우 필터는 사실이다
			filter := len(headers) == 1
			if filter {
				// If it's a potential DAO fork check, validate against the rules
				if p.forkDrop != nil && pm.chainconfig.DAOForkBlock.Cmp(headers[0].Number) == 0 {  //DAO 검사
					// Disable the fork drop timer
					p.forkDrop.Stop()
					p.forkDrop = nil
	
					// Validate the header and either drop the peer or continue
					if err := misc.VerifyDAOHeaderExtraData(pm.chainconfig, headers[0]); err != nil {
						p.Log().Debug("Verified to be on the other side of the DAO fork, dropping")
						return err
					}
					p.Log().Debug("Verified to be on the same side of the DAO fork")
					return nil
				}
				// Irrelevant of the fork checks, send the header to the fetcher just in case
				//요청이 DAO없는 경우, 필터는 필터링합니다. 필터는 반환 헤더 처리를 계속해야합니다,이 헤더는 다운로더에 배포됩니다.
				headers = pm.fetcher.FilterHeaders(p.id, headers, time.Now())
			}
			if len(headers) > 0 || !filter {
				err := pm.downloader.DeliverHeaders(p.id, headers)
				if err != nil {
					log.Debug("Failed to deliver headers", "err", err)
				}
			}
	
		case msg.Code == GetBlockBodiesMsg:
			//블록 바디의 요청이 상대적으로 간단하다. 내부 blockchain에서 반환하는 줄에 몸을 가져옵니다.
			// Decode the retrieval message
			msgStream := rlp.NewStream(msg.Payload, uint64(msg.Size))
			if _, err := msgStream.List(); err != nil {
				return err
			}
			// Gather blocks until the fetch or network limits is reached
			var (
				hash   common.Hash
				bytes  int
				bodies []rlp.RawValue
			)
			for bytes < softResponseLimit && len(bodies) < downloader.MaxBlockFetch {
				// Retrieve the hash of the next block
				if err := msgStream.Decode(&hash); err == rlp.EOL {
					break
				} else if err != nil {
					return errResp(ErrDecode, "msg %v: %v", msg, err)
				}
				// Retrieve the requested block body, stopping if enough was found
				if data := pm.blockchain.GetBodyRLP(hash); len(data) != 0 {
					bodies = append(bodies, data)
					bytes += len(data)
				}
			}
			return p.SendBlockBodiesRLP(bodies)
	
		case msg.Code == BlockBodiesMsg:
			// A batch of block bodies arrived to one of our previous requests
			var request blockBodiesData
			if err := msg.Decode(&request); err != nil {
				return errResp(ErrDecode, "msg %v: %v", msg, err)
			}
			// Deliver them all to the downloader for queuing
			trasactions := make([][]*types.Transaction, len(request))
			uncles := make([][]*types.Header, len(request))
	
			for i, body := range request {
				trasactions[i] = body.Transactions
				uncles[i] = body.Uncles
			}
			// Filter out any explicitly requested bodies, deliver the rest to the downloader
			//모든 요청 쇼 및 다운로더로 나머지를 필터링
			filter := len(trasactions) > 0 || len(uncles) > 0
			if filter {
				trasactions, uncles = pm.fetcher.FilterBodies(p.id, trasactions, uncles, time.Now())
			}
			if len(trasactions) > 0 || len(uncles) > 0 || !filter {
				err := pm.downloader.DeliverBodies(p.id, trasactions, uncles)
				if err != nil {
					log.Debug("Failed to deliver bodies", "err", err)
				}
			}
	
		case p.version >= eth63 && msg.Code == GetNodeDataMsg:
			//버전의 끝 요청 및 eth63 NODEDATA입니다
			// Decode the retrieval message
			msgStream := rlp.NewStream(msg.Payload, uint64(msg.Size))
			if _, err := msgStream.List(); err != nil {
				return err
			}
			// Gather state data until the fetch or network limits is reached
			var (
				hash  common.Hash
				bytes int
				data  [][]byte
			)
			for bytes < softResponseLimit && len(data) < downloader.MaxStateFetch {
				// Retrieve the hash of the next state entry
				if err := msgStream.Decode(&hash); err == rlp.EOL {
					break
				} else if err != nil {
					return errResp(ErrDecode, "msg %v: %v", msg, err)
				}
				// Retrieve the requested state entry, stopping if enough was found
				//모든 요청 해시 값이 서로에게 반환됩니다.
				if entry, err := pm.chaindb.Get(hash.Bytes()); err == nil {
					data = append(data, entry)
					bytes += len(entry)
				}
			}
			return p.SendNodeData(data)
	
		case p.version >= eth63 && msg.Code == NodeDataMsg:
			// A batch of node state data arrived to one of our previous requests
			var data [][]byte
			if err := msg.Decode(&data); err != nil {
				return errResp(ErrDecode, "msg %v: %v", msg, err)
			}
			// Deliver all to the downloader
			//다운로더에 대한 데이터
			if err := pm.downloader.DeliverNodeData(p.id, data); err != nil {
				log.Debug("Failed to deliver node state data", "err", err)
			}
	
		case p.version >= eth63 && msg.Code == GetReceiptsMsg:
			//영수증 요청
			// Decode the retrieval message
			msgStream := rlp.NewStream(msg.Payload, uint64(msg.Size))
			if _, err := msgStream.List(); err != nil {
				return err
			}
			// Gather state data until the fetch or network limits is reached
			var (
				hash	 common.Hash
				bytes	int
				receipts []rlp.RawValue
			)
			for bytes < softResponseLimit && len(receipts) < downloader.MaxReceiptFetch {
				// Retrieve the hash of the next block
				if err := msgStream.Decode(&hash); err == rlp.EOL {
					break
				} else if err != nil {
					return errResp(ErrDecode, "msg %v: %v", msg, err)
				}
				// Retrieve the requested block's receipts, skipping if unknown to us
				results := core.GetBlockReceipts(pm.chaindb, hash, core.GetBlockNumber(pm.chaindb, hash))
				if results == nil {
					if header := pm.blockchain.GetHeaderByHash(hash); header == nil || header.ReceiptHash != types.EmptyRootHash {
						continue
					}
				}
				// If known, encode and queue for response packet
				if encoded, err := rlp.EncodeToBytes(results); err != nil {
					log.Error("Failed to encode receipt", "err", err)
				} else {
					receipts = append(receipts, encoded)
					bytes += len(encoded)
				}
			}
			return p.SendReceiptsRLP(receipts)
	
		case p.version >= eth63 && msg.Code == ReceiptsMsg:
			// A batch of receipts arrived to one of our previous requests
			var receipts [][]*types.Receipt
			if err := msg.Decode(&receipts); err != nil {
				return errResp(ErrDecode, "msg %v: %v", msg, err)
			}
			// Deliver all to the downloader
			if err := pm.downloader.DeliverReceipts(p.id, receipts); err != nil {
				log.Debug("Failed to deliver receipts", "err", err)
			}
	
		case msg.Code == NewBlockHashesMsg:
			//메시지 BlockHashesMsg 받기
			var announces newBlockHashesData
			if err := msg.Decode(&announces); err != nil {
				return errResp(ErrDecode, "%v: %v", msg, err)
			}
			// Mark the hashes as present at the remote node
			for _, block := range announces {
				p.MarkBlock(block.Hash)
			}
			// Schedule all the unknown hashes for retrieval
			unknown := make(newBlockHashesData, 0, len(announces))
			for _, block := range announces {
				if !pm.blockchain.HasBlock(block.Hash, block.Number) {
					unknown = append(unknown, block)
				}
			}
			for _, block := range unknown {
				//가져 오기 통지 다운로드 할 수있는 잠재적 인 블록이
				pm.fetcher.Notify(p.id, block.Hash, block.Number, time.Now(), p.RequestOneHeader, p.RequestBodies)
			}
	
		case msg.Code == NewBlockMsg:
			// Retrieve and decode the propagated block
			var request newBlockData
			if err := msg.Decode(&request); err != nil {
				return errResp(ErrDecode, "%v: %v", msg, err)
			}
			request.Block.ReceivedAt = msg.ReceivedAt
			request.Block.ReceivedFrom = p
	
			// Mark the peer as owning the block and schedule it for import
			p.MarkBlock(request.Block.Hash())
			pm.fetcher.Enqueue(p.id, request.Block)
	
			// Assuming the block is importable by the peer, but possibly not yet done so,
			// calculate the head hash and TD that the peer truly must have.
			var (
				trueHead = request.Block.ParentHash()
				trueTD   = new(big.Int).Sub(request.TD, request.Block.Difficulty())
			)
			// Update the peers total difficulty if better than the previous
			if _, td := p.Head(); trueTD.Cmp(td) > 0 {
				//TD와 다른 레코드의 우리 측의 경우는 true 피어 머리, 설정 피어 실제 머리와 TD,
				p.SetHead(trueHead, trueTD)
	
				// Schedule a sync if above ours. Note, this will not fire a sync for a gap of
				// a singe block (as the true TD is below the propagated block), however this
				// scenario should easily be covered by the fetcher.
				//우리 TD보다 진정한 TD 큰 경우, 요청 및 피어 동기화.
				currentBlock := pm.blockchain.CurrentBlock()
				if trueTD.Cmp(pm.blockchain.GetTd(currentBlock.Hash(), currentBlock.NumberU64())) > 0 {
					go pm.synchronise(p)
				}
			}
	
		case msg.Code == TxMsg:
			// Transactions arrived, make sure we have a valid and fresh chain to handle them
			//광고로 돌아 가기. 우리는 트랜잭션 동기화를받을 수 없습니다 전에 쓸모없는 정보를 입력합니다.
			if atomic.LoadUint32(&pm.acceptTxs) == 0 {
				break
			}
			// Transactions can be processed, parse all of them and deliver to the pool
			var txs []*types.Transaction
			if err := msg.Decode(&txs); err != nil {
				return errResp(ErrDecode, "msg %v: %v", msg, err)
			}
			for i, tx := range txs {
				// Validate and mark the remote transaction
				if tx == nil {
					return errResp(ErrDecode, "transaction %d is nil", i)
				}
				p.MarkTransaction(tx.Hash())
			}
			//txpool에 추가
			pm.txpool.AddRemotes(txs)
	
		default:
			return errResp(ErrInvalidMsgCode, "%v", msg.Code)
		}
		return nil
	}

여러 동기화 동기화, 노드가 아닌 다른 노드의 발견이 방법의 동기화를 호출 자신의 시간을 업데이트하기 전에,


	// synchronise tries to sync up our local block chain with a remote peer.
	//원격 단부와 로컬 블록 사슬 동기를 얻으려고 동기화.
	func (pm *ProtocolManager) synchronise(peer *peer) {
		// Short circuit if no peers are available
		if peer == nil {
			return
		}
		// Make sure the peer's TD is higher than our own
		currentBlock := pm.blockchain.CurrentBlock()
		td := pm.blockchain.GetTd(currentBlock.Hash(), currentBlock.NumberU64())
	
		pHead, pTd := peer.Head()
		if pTd.Cmp(td) <= 0 {
			return
		}
		// Otherwise try to sync with the downloader
		mode := downloader.FullSync
		if atomic.LoadUint32(&pm.fastSync) == 1 { //그것은 명시 적으로 빨리 것을 주장하는 경우
			// Fast sync was explicitly requested, and explicitly granted
			mode = downloader.FastSync
		} else if currentBlock.NumberU64() == 0 && pm.blockchain.CurrentFastBlock().NumberU64() > 0 {  //데이터베이스가 비어있는 경우
			// The database seems empty as the current block is the genesis. Yet the fast
			// block is ahead, so fast sync was enabled for this node at a certain point.
			// The only scenario where this can happen is if the user manually (or via a
			// bad block) rolled back a fast sync node below the sync point. In this case
			// however it's safe to reenable fast sync.
			atomic.StoreUint32(&pm.fastSync, 1)
			mode = downloader.FastSync
		}
		// Run the sync cycle, and disable fast sync if we've went past the pivot block
		err := pm.downloader.Synchronise(peer.id, pHead, pTd, mode)
	
		if atomic.LoadUint32(&pm.fastSync) == 1 {
			// Disable fast sync if we indeed have something in our chain
			if pm.blockchain.CurrentBlock().NumberU64() > 0 {
				log.Info("Fast sync complete, auto disabling")
				atomic.StoreUint32(&pm.fastSync, 0)
			}
		}
		if err != nil {
			return
		}
		atomic.StoreUint32(&pm.acceptTxs, 1) // Mark initial sync done
		//동기화를 수신이 완료 거래입니다 시작합니다.
		if head := pm.blockchain.CurrentBlock(); head.NumberU64() > 0 {
			// We've completed a sync cycle, notify all peers of new state. This path is
			// essential in star-topology networks where a gateway node needs to notify
			// all its out-of-date peers of the availability of a new block. This failure
			// scenario will most often crop up in private and hackathon networks with
			// degenerate connectivity, but it should be healthy for the mainnet too to
			// more reliably update peers or the local TD state.
			//우리는 우리의 피어 상태 모두를 말한다.
			go pm.BroadcastBlock(head, false)
		}
	}


무역 방송. 시작 goroutine의 시작을 txBroadcastLoop. 이벤트가 위에 기록 될 때 txCh는 txpool에 법적 거래를 받았다. 그런 다음 트랜잭션은 모든 피어에 방송된다

	func (self *ProtocolManager) txBroadcastLoop() {
		for {
			select {
			case event := <-self.txCh:
				self.BroadcastTx(event.Tx.Hash(), event.Tx)
	
			// Err() channel will be closed when unsubscribing.
			case <-self.txSub.Err():
				return
			}
		}
	}


광업 방송. 새로운 방송 광석의 수신 가입 이벤트는 파낸합니다.

	// Mined broadcast loop
	func (self *ProtocolManager) minedBroadcastLoop() {
		// automatically stops if unsubscribe
		for obj := range self.minedBlockSub.Chan() {
			switch ev := obj.Data.(type) {
			case core.NewMinedBlockEvent:
				self.BroadcastBlock(ev.Block, true)  // First propagate block to peers
				self.BroadcastBlock(ev.Block, false) // Only then announce to the rest
			}
		}
	}

일반 및 네트워크 동기화에 대한 책임 긴 링크,

	// syncer is responsible for periodically synchronising with the network, both
	// downloading hashes and blocks as well as handling the announcement handler.
	//동기화는 주기적으로 네트워크 및 처리 블록과 해시 다운로드 알림 핸들러와 동기화 할 책임이 있습니다.
	func (pm *ProtocolManager) syncer() {
		// Start and ensure cleanup of sync mechanisms
		pm.fetcher.Start()
		defer pm.fetcher.Stop()
		defer pm.downloader.Terminate()
	
		// Wait for different events to fire synchronisation operations
		forceSync := time.NewTicker(forceSyncCycle)
		defer forceSync.Stop()
	
		for {
			select {
			case <-pm.newPeerCh: //새 피어 시간을 추가 할 때이 동기화됩니다. 이번에는 블록 방송을 트리거 할 수 있습니다.
				// Make sure we have peers to select from, then sync
				if pm.peers.Len() < minDesiredPeerCount {
					break
				}
				go pm.synchronise(pm.peers.BestPeer())
	
			case <-forceSync.C:
				//정기적으로 10 초 트리거
				// Force a sync even if not enough peers are present
				//BestPeer ()는 항상 어려운 최대 노드를 선택합니다.
				go pm.synchronise(pm.peers.BestPeer())
	
			case <-pm.noMorePeers: //종료 신호
				return
			}
		}
	}

연결이 설정 될 때까지 보류중인 트랜잭션을 전송에 대한 책임 txsyncLoop.


	// txsyncLoop takes care of the initial transaction sync for each new
	// connection. When a new peer appears, we relay all currently pending
	// transactions. In order to minimise egress bandwidth usage, we send
	// the transactions in small packs to one peer at a time.

	원래 트랜잭션 동기화 각 새 연결에 대한 책임을 txsyncLoop. 새로운 피어가 나타나면, 우리는 현재 계류중인 트랜잭션을 전달합니다. 대역폭 사용 수출을 최소화하기 위해, 우리는 피어에 하나의 트랜잭션에 패킷을 보낼 것입니다.
	func (pm *ProtocolManager) txsyncLoop() {
		var (
			pending = make(map[discover.NodeID]*txsync)
			sending = false			   // whether a send is active
			pack	= new(txsync)		 // the pack that is being sent
			done	= make(chan error, 1) // result of the send
		)
	
		// send starts a sending a pack of transactions from the sync.
		send := func(s *txsync) {
			// Fill pack with transactions up to the target size.
			size := common.StorageSize(0)
			pack.p = s.p
			pack.txs = pack.txs[:0]
			for i := 0; i < len(s.txs) && size < txsyncPackSize; i++ {
				pack.txs = append(pack.txs, s.txs[i])
				size += s.txs[i].Size()
			}
			// Remove the transactions that will be sent.
			s.txs = s.txs[:copy(s.txs, s.txs[len(pack.txs):])]
			if len(s.txs) == 0 {
				delete(pending, s.p.ID())
			}
			// Send the pack in the background.
			s.p.Log().Trace("Sending batch of transactions", "count", len(pack.txs), "bytes", size)
			sending = true
			go func() { done <- pack.p.SendTransactions(pack.txs) }()
		}
	
		// pick chooses the next pending sync.
		//무작위 txsync를 보내도록 선택했습니다.
		pick := func() *txsync {
			if len(pending) == 0 {
				return nil
			}
			n := rand.Intn(len(pending)) + 1
			for _, s := range pending {
				if n--; n == 0 {
					return s
				}
			}
			return nil
		}
	
		for {
			select {
			case s := <-pm.txsyncCh: //여기 txsyncCh에서 메시지를 수신하는 단계를 포함한다.
				pending[s.p.ID()] = s
				if !sending {
					send(s)
				}
			case err := <-done:
				sending = false
				// Stop tracking peers that cause send failures.
				if err != nil {
					pack.p.Log().Debug("Transaction send failed", "err", err)
					delete(pending, pack.p.ID())
				}
				// Schedule the next send.
				if s := pick(); s != nil {
					send(s)
				}
			case <-pm.quitSync:
				return
			}
		}
	}

txsyncCh 큐 생산자 syncTransactions되는 핸들 메소드 호출한다. 때 방금 만든 새 링크를 한 번 호출됩니다.

	// syncTransactions starts sending all currently pending transactions to the given peer.
	func (pm *ProtocolManager) syncTransactions(p *peer) {
		var txs types.Transactions
		pending, _ := pm.txpool.Pending()
		for _, batch := range pending {
			txs = append(txs, batch...)
		}
		if len(txs) == 0 {
			return
		}
		select {
		case pm.txsyncCh <- &txsync{p, txs}:
		case <-pm.quitSync:
		}
	}


요약. 우리는 지금 과정의 큰 숫자입니다.

블록 동기화

1. 당신은 당신의 자신의 광산을 파고 경우. )은 (goroutine minedBroadcastLoop으로 방송합니다.
이 다른 사람에 의해 수신되면 2 피어 방송 블록 (NewBlockHashesMsg / NewBlockMsg) 여부를 알 페처? TODO
BestPeer와 () 주기적 3. goroutine 긴 링크에서 ()의 정보를 동기화합니다.

무역 동기화

1. 새로운 연결이 설정됩니다. 그에게 보류중인 거래를 보냅니다.
2. 지역 트랜잭션, 또는 다른 사람에 의해 전송받은 거래 정보를 전송. txpool 메시지는 txCh 채널로 전달되는 메시지를 생성한다. 그런 다음 다른 피어에 송신 goroutine txBroadcastLoop () 처리는이 거래를 알 수 없습니다.