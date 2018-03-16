P2P는 서버의 가장 중요한 부분입니다. 그것은 이전의 모든 구성 요소를 함께 제공합니다.

서버의 구조를 먼저 살펴

	
	// Server manages all peer connections.
	type Server struct {
		// Config fields may not be modified while the server is running.
		Config
	
		// Hooks for testing. These are useful because we can inhibit
		// the whole protocol stack.
		newTransport func(net.Conn) transport
		newPeerHook  func(*Peer)
	
		lock	sync.Mutex // protects running
		running bool
	
		ntab		 discoverTable
		listener	 net.Listener
		ourHandshake *protoHandshake
		lastLookup   time.Time
		DiscV5	   *discv5.Network
	
		// These are for Peers, PeerCount (and nothing else).
		peerOp	 chan peerOpFunc
		peerOpDone chan struct{}
	
		quit		  chan struct{}
		addstatic	 chan *discover.Node
		removestatic  chan *discover.Node
		posthandshake chan *conn
		addpeer	   chan *conn
		delpeer	   chan peerDrop
		loopWG		sync.WaitGroup // loop, listenLoop
		peerFeed	  event.Feed
	}

	// conn wraps a network connection with information gathered
	// during the two handshakes.
	type conn struct {
		fd net.Conn
		transport
		flags connFlag
		cont  chan error	  // The run loop uses cont to signal errors to SetupConn.
		id	discover.NodeID // valid after the encryption handshake
		caps  []Cap		   // valid after the protocol handshake
		name  string		  // valid after the protocol handshake
	}

	type transport interface {
		// The two handshakes.
		doEncHandshake(prv *ecdsa.PrivateKey, dialDest *discover.Node) (discover.NodeID, error)
		doProtoHandshake(our *protoHandshake) (*protoHandshake, error)
		// The MsgReadWriter can only be used after the encryption
		// handshake has completed. The code uses conn.id to track this
		// by setting it to a non-nil value after the encryption handshake.
		MsgReadWriter
		// transports must provide Close because we use MsgPipe in some of
		// the tests. Closing the actual network connection doesn't do
		// anything in those tests because NsgPipe doesn't use it.
		close(err error)
	}

newServer의 방법은 없습니다. 초기화 시작 (방법)에 작업 할 수 있습니다.


	// Start starts running the server.
	// Servers can not be re-used after stopping.
	func (srv *Server) Start() (err error) {
		srv.lock.Lock()
		defer srv.lock.Unlock()
		if srv.running { //복수의 시작을 피하십시오. 반복 시작 멀티 스레딩을 방지하기 위해 srv.lock
			return errors.New("server already running")
		}
		srv.running = true
		log.Info("Starting P2P networking")
	
		// static fields
		if srv.PrivateKey == nil {
			return fmt.Errorf("Server.PrivateKey must be set to a non-nil key")
		}
		if srv.newTransport == nil {	// 여기 newRLPX 운송의 사용은 네트워크 프로토콜에 사용 rlpx.go 지적했다.
			srv.newTransport = newRLPX
		}
		if srv.Dialer == nil { //TCLPDialer를 사용하여
			srv.Dialer = TCPDialer{&net.Dialer{Timeout: defaultDialTimeout}}
		}
		srv.quit = make(chan struct{})
		srv.addpeer = make(chan *conn)
		srv.delpeer = make(chan peerDrop)
		srv.posthandshake = make(chan *conn)
		srv.addstatic = make(chan *discover.Node)
		srv.removestatic = make(chan *discover.Node)
		srv.peerOp = make(chan peerOpFunc)
		srv.peerOpDone = make(chan struct{})
	
		// node table
		if !srv.NoDiscovery {  //시작 네트워크를 발견 할 수 있습니다. UDP 열기 듣기.
			ntab, err := discover.ListenUDP(srv.PrivateKey, srv.ListenAddr, srv.NAT, srv.NodeDatabase, srv.NetRestrict)
			if err != nil {
				return err
			}
			//노드의 시작을 시작하는 설정합니다. 때 시간의 다른 노드. 그럼 당신은이 노드를 연결하기 시작합니다. 이 노드의 정보는 하드 코딩 된 내부 구성 파일에 있습니다.
			if err := ntab.SetFallbackNodes(srv.BootstrapNodes); err != nil {
				return err
			}
			srv.ntab = ntab
		}
	
		if srv.DiscoveryV5 {//이것은 새로운 노드 탐색 프로토콜입니다. 아무 소용이 없습니다. 일시적으로 여기에 분석되지.
			ntab, err := discv5.ListenUDP(srv.PrivateKey, srv.DiscoveryV5Addr, srv.NAT, "", srv.NetRestrict) //srv.NodeDatabase)
			if err != nil {
				return err
			}
			if err := ntab.SetFallbackNodes(srv.BootstrapNodesV5); err != nil {
				return err
			}
			srv.DiscV5 = ntab
		}
	
		dynPeers := (srv.MaxPeers + 1) / 2
		if srv.NoDiscovery {
			dynPeers = 0
		}	
		//dialerstate 만들기.
		dialer := newDialState(srv.StaticNodes, srv.BootstrapNodes, srv.ntab, dynPeers, srv.NetRestrict)
	
		// handshake
		//우리 자신의 프로토콜을 핸드 셰이크
		srv.ourHandshake = &protoHandshake{Version: baseProtocolVersion, Name: srv.Name, ID: discover.PubkeyID(&srv.PrivateKey.PublicKey)}
		for _, p := range srv.Protocols {//모든 계약은 모자를 증가
			srv.ourHandshake.Caps = append(srv.ourHandshake.Caps, p.cap())
		}
		// listen/dial
		if srv.ListenAddr != "" {
			//TCP 포트에서 수신 시작
			if err := srv.startListening(); err != nil {
				return err
			}
		}
		if srv.NoDial && srv.ListenAddr == "" {
			log.Warn("P2P server will be useless, neither dialing nor listening")
		}
	
		srv.loopWG.Add(1)
		//Goroutine 핸들러를 시작합니다.
		go srv.run(dialer)
		srv.running = true
		return nil
	}


리스너를 시작합니다. 당신은 TCP 프로토콜을 볼 수 있습니다. 다음은 수신 포트이고 UDP 포트는 동일합니다. 기본값은 30303입니다
	
	func (srv *Server) startListening() error {
		// Launch the TCP listener.
		listener, err := net.Listen("tcp", srv.ListenAddr)
		if err != nil {
			return err
		}
		laddr := listener.Addr().(*net.TCPAddr)
		srv.ListenAddr = laddr.String()
		srv.listener = listener
		srv.loopWG.Add(1)
		go srv.listenLoop()
		// Map the TCP listening port if NAT is configured.
		if !laddr.IP.IsLoopback() && srv.NAT != nil {
			srv.loopWG.Add(1)
			go func() {
				nat.Map(srv.NAT, srv.quit, "tcp", laddr.Port, laddr.Port, "ethereum p2p")
				srv.loopWG.Done()
			}()
		}
		return nil
	}

listenLoop (). 이 goroutine의 무한 루프입니다. 포트에서 수신 및 외부 요청을받을 수 있습니다.
	
	// listenLoop runs in its own goroutine and accepts
	// inbound connections.
	func (srv *Server) listenLoop() {
		defer srv.loopWG.Done()
		log.Info("RLPx listener up", "self", srv.makeSelf(srv.listener, srv.ntab))
	
		// This channel acts as a semaphore limiting
		// active inbound connections that are lingering pre-handshake.
		// If all slots are taken, no further connections are accepted.
		tokens := maxAcceptConns
		if srv.MaxPendingPeers > 0 {
			tokens = srv.MaxPendingPeers
		}
		//maxAcceptConns 슬롯 만들기. 우리는 많은 동시 연결을 처리합니다. 더하지 않습니다.
		slots := make(chan struct{}, tokens)
		//슬롯은 가득 찼다.
		for i := 0; i < tokens; i++ {
			slots <- struct{}{}
		}
	
		for {
			// Wait for a handshake slot before accepting.
			<-slots
	
			var (
				fd  net.Conn
				err error
			)
			for {
				fd, err = srv.listener.Accept()
				if tempErr, ok := err.(tempError); ok && tempErr.Temporary() {
					log.Debug("Temporary read error", "err", err)
					continue
				} else if err != nil {
					log.Debug("Read error", "err", err)
					return
				}
				break
			}
	
			// Reject connections that do not match NetRestrict.
			//화이트리스트. 내부의 화이트리스트에 그렇지 않은 경우. 그런 다음 연결을 닫습니다.
			if srv.NetRestrict != nil {
				if tcp, ok := fd.RemoteAddr().(*net.TCPAddr); ok && !srv.NetRestrict.Contains(tcp.IP) {
					log.Debug("Rejected conn (not whitelisted in NetRestrict)", "addr", fd.RemoteAddr())
					fd.Close()
					slots <- struct{}{}
					continue
				}
			}
	
			fd = newMeteredConn(fd, true)
			log.Trace("Accepted connection", "addr", fd.RemoteAddr())
	
			// Spawn the handler. It will give the slot back when the connection
			// has been established.
			go func() {
				//그것은 연결이 완료 후 설립되는 한 것으로 보인다. 슬롯가 반환됩니다. 우리가 다시 기억이 기능을 SetupConn 및 전화 dialTask.Do이 있었다,이 기능은 주로 여러 번에게 악수 연결을 수행한다.
				srv.SetupConn(fd, inboundConn, nil)
				slots <- struct{}{}
			}()
		}
	}

SetupConn,이 기능은 악수 계약을 수행하고 피어 연결 개체의 비트를 만들어보십시오.


	// SetupConn runs the handshakes and attempts to add the connection
	// as a peer. It returns when the connection has been added as a peer
	// or the handshakes have failed.
	func (srv *Server) SetupConn(fd net.Conn, flags connFlag, dialDest *discover.Node) {
		// Prevent leftover pending conns from entering the handshake.
		srv.lock.Lock()
		running := srv.running
		srv.lock.Unlock()
		//그것은 CONN 개체를 만듭니다. newTransport newRLPx 방법은 실제로 포인터가 가리키는입니다. rlpx 조금 포장에 실제로 FD의 계약.
		c := &conn{fd: fd, transport: srv.newTransport(fd), flags: flags, cont: make(chan error)}
		if !running {
			c.close(errServerStopped)
			return
		}
		// Run the encryption handshake.
		var err error
		//수송 통신용 익명 필드이므로 여기서 실제로. doEncHandshake 내부 rlpx.go를 행한다. 직접 CONN 방법 으로서는 방법 익명 필드.
		if c.id, err = c.doEncHandshake(srv.PrivateKey, dialDest); err != nil {
			log.Trace("Failed RLPx handshake", "addr", c.fd.RemoteAddr(), "conn", c.flags, "err", err)
			c.close(err)
			return
		}
		clog := log.New("id", c.id, "addr", c.fd.RemoteAddr(), "conn", c.flags)
		// For dialed connections, check that the remote public key matches.
		//ID와 해당 연결 ID는 핸드 쉐이크와 일치하지 않는 경우
		if dialDest != nil && c.id != dialDest.ID {
			c.close(DiscUnexpectedIdentity)
			clog.Trace("Dialed identity mismatch", "want", c, dialDest.ID)
			return
		}
		//사실,이 검사는 제 파라미터 첫번째 파라미터 큐 지정 보내는 것이다. 그리고 c.cout에서 반환 정보를 수신하는 단계를 포함한다. 이 동기 방식이다.
		//여기에 관해서는, 후속 작업은 그냥 합법적 인 연결이 반환되어 있는지 확인.
		if err := srv.checkpoint(c, srv.posthandshake); err != nil {
			clog.Trace("Rejected peer before protocol handshake", "err", err)
			c.close(err)
			return
		}
		// Run the protocol handshake
		phs, err := c.doProtoHandshake(srv.ourHandshake)
		if err != nil {
			clog.Trace("Failed proto handshake", "err", err)
			c.close(err)
			return
		}
		if phs.ID != c.id {
			clog.Trace("Wrong devp2p handshake identity", "err", phs.ID)
			c.close(DiscUnexpectedIdentity)
			return
		}
		c.caps, c.name = phs.Caps, phs.Name
		//여기에 두 개의 핸드 셰이크가 완료됩니다. C의는 addpeer는 큐를 보낼 수 있습니다. 때 백그라운드 처리 큐, 연결 처리합니다
		if err := srv.checkpoint(c, srv.addpeer); err != nil {
			clog.Trace("Rejected peer", "err", err)
			c.close(err)
			return
		}
		// If the checks completed successfully, runPeer has now been
		// launched by run.
	}


위에서 언급 한 공정은 listenLoop, listenLoop 주로 외부 그 활성 연결을 수신하는데 사용되는 방법이다. 노드가 외부 노드를 연결하는 프로세스 연결을 시작해야하는 경우가 있습니다. 바로 위의 검사 처리의 흐름은 정보를 큐. server.run이 goroutine 내부 코드의이 부분.



	func (srv *Server) run(dialstate dialer) {
		defer srv.loopWG.Done()
		var (
			peers		= make(map[discover.NodeID]*Peer)
			trusted	  = make(map[discover.NodeID]bool, len(srv.TrustedNodes))
			taskdone	 = make(chan task, maxActiveDialTasks)
			runningTasks []task
			queuedTasks  []task // tasks that can't run yet
		)
		// Put trusted nodes into a map to speed up checks.
		// Trusted peers are loaded on startup and cannot be
		// modified while the server is running.
		//연결이 너무 후 다른 노드가 밖으로 거부 될 경우 신뢰할 수있는 노드는 이러한 특성을 가지고있다. 그러나 신뢰할 수있는 노드가 수신됩니다.
		for _, n := range srv.TrustedNodes {
			trusted[n.ID] = true
		}
	
		// removes t from runningTasks
		//그것은 runningTasks에서 작업 대기열을 삭제하는 함수를 정의
		delTask := func(t task) {
			for i := range runningTasks {
				if runningTasks[i] == t {
					runningTasks = append(runningTasks[:i], runningTasks[i+1:]...)
					break
				}
			}
		}
		// starts until max number of active tasks is satisfied
		//동시에 노드 (16)의 수는 시작에 연결되어 있습니다. 순회의 runningTasks는 대기 행렬에 이러한 작업을 시작합니다.
		startTasks := func(ts []task) (rest []task) {
			i := 0
			for ; len(runningTasks) < maxActiveDialTasks && i < len(ts); i++ {
				t := ts[i]
				log.Trace("New dial task", "task", t)
				go func() { t.Do(srv); taskdone <- t }()
				runningTasks = append(runningTasks, t)
			}
			return ts[i:]
		}
		scheduleTasks := func() {
			// Start from queue first.
			//첫째, startTasks 부분을 호출 시작, 나머지는 queuedTasks에 반환됩니다.
			queuedTasks = append(queuedTasks[:0], startTasks(queuedTasks)...)
			// Query dialer for new tasks and start as many as possible now.
			//작업을 생성하고, startTasks 시작하려고 newTasks를 호출합니다. 그리고 큐 queuedTasks로 부팅 일시적으로없는에
			if len(runningTasks) < maxActiveDialTasks {
				nt := dialstate.newTasks(len(runningTasks)+len(queuedTasks), peers, time.Now())
				queuedTasks = append(queuedTasks, startTasks(nt)...)
			}
		}
	
	running:
		for {
			//새 작업을 생성 할 수 dialstate.newTasks를 호출합니다. 그리고 startTasks 새로운 작업을 시작 호출합니다.
			//dialTask ​​모든 시작한 경우, 그것은 수면 시간 제한 작업을 생성합니다.
			scheduleTasks()
	
			select {
			case <-srv.quit:
				// The server was stopped. Run the cleanup logic.
				break running
			case n := <-srv.addstatic:
				// This channel is used by AddPeer to add to the
				// ephemeral static peer list. Add it to the dialer,
				// it will keep the node connected.
				log.Debug("Adding static node", "node", n)
				dialstate.addStatic(n)
			case n := <-srv.removestatic:
				// This channel is used by RemovePeer to send a
				// disconnect request to a peer and begin the
				// stop keeping the node connected
				log.Debug("Removing static node", "node", n)
				dialstate.removeStatic(n)
				if p, ok := peers[n.ID]; ok {
					p.Disconnect(DiscRequested)
				}
			case op := <-srv.peerOp:
				// This channel is used by Peers and PeerCount.
				op(peers)
				srv.peerOpDone <- struct{}{}
			case t := <-taskdone:
				// A task got done. Tell dialstate about it so it
				// can update its state and remove it from the active
				// tasks list.
				log.Trace("Dial task done", "task", t)
				dialstate.taskDone(t, time.Now())
				delTask(t)
			case c := <-srv.posthandshake:
				// A connection has passed the encryption handshake so
				// the remote identity is known (but hasn't been verified yet).
				//검사 점 방법 전에 전화하는 것을 잊지 연결이이 채널로 전송됩니다.
				if trusted[c.id] {
					// Ensure that the trusted flag is set before checking against MaxPeers.
					c.flags |= trustedConn
				}
				// TODO: track in-progress inbound node IDs (pre-Peer) to avoid dialing them.
				select {
				case c.cont <- srv.encHandshakeChecks(peers, c):
				case <-srv.quit:
					break running
				}
			case c := <-srv.addpeer:
				// At this point the connection is past the protocol handshake.
				// Its capabilities are known and the remote identity is verified.
				//그것은이 채널을는 addpeer로 전송 두 방향 핸드 셰이크 연결 후 검사를 호출합니다.
				//그런 다음 newPeer은 피어 개체를 만들었습니다.
				//시작 피어 개체를 시작 Goroutine. 방법 peer.run 호출합니다.
				err := srv.protoHandshakeChecks(peers, c)
				if err == nil {
					// The handshakes are done and it passed all checks.
					p := newPeer(c, srv.Protocols)
					// If message events are enabled, pass the peerFeed
					// to the peer
					if srv.EnableMsgEvents {
						p.events = &srv.peerFeed
					}
					name := truncateName(c.name)
					log.Debug("Adding p2p peer", "id", c.id, "name", name, "addr", c.fd.RemoteAddr(), "peers", len(peers)+1)
					peers[c.id] = p
					go srv.runPeer(p)
				}
				// The dialer logic relies on the assumption that
				// dial tasks complete after the peer has been added or
				// discarded. Unblock the task last.
				select {
				case c.cont <- err:
				case <-srv.quit:
					break running
				}
			case pd := <-srv.delpeer:
				// A peer disconnected.
				d := common.PrettyDuration(mclock.Now() - pd.created)
				pd.log.Debug("Removing p2p peer", "duration", d, "peers", len(peers)-1, "req", pd.requested, "err", pd.err)
				delete(peers, pd.ID())
			}
		}
	
		log.Trace("P2P networking is spinning down")
	
		// Terminate discovery. If there is a running lookup it will terminate soon.
		if srv.ntab != nil {
			srv.ntab.Close()
		}
		if srv.DiscV5 != nil {
			srv.DiscV5.Close()
		}
		// Disconnect all peers.
		for _, p := range peers {
			p.Disconnect(DiscQuitting)
		}
		// Wait for peers to shut down. Pending connections and tasks are
		// not handled here and will terminate soon-ish because srv.quit
		// is closed.
		for len(peers) > 0 {
			p := <-srv.delpeer
			p.log.Trace("<-delpeer (spindown)", "remainingTasks", len(runningTasks))
			delete(peers, p.ID())
		}
	}


runPeer 방법

	// runPeer runs in its own goroutine for each peer.
	// it waits until the Peer logic returns and removes
	// the peer.
	func (srv *Server) runPeer(p *Peer) {
		if srv.newPeerHook != nil {
			srv.newPeerHook(p)
		}
	
		// broadcast peer add
		srv.peerFeed.Send(&PeerEvent{
			Type: PeerEventTypeAdd,
			Peer: p.ID(),
		})
	
		// run the protocol
		remoteRequested, err := p.run()
	
		// broadcast peer drop
		srv.peerFeed.Send(&PeerEvent{
			Type:  PeerEventTypeDrop,
			Peer:  p.ID(),
			Error: err.Error(),
		})
	
		// Note: run waits for existing peers to be sent on srv.delpeer
		// before returning, so this send should not select on srv.quit.
		srv.delpeer <- peerDrop{p, err, remoteRequested}
	}


요약 :

서버 개체의 모든 주요 구성 요소는 결합의 도입 전에 작업을 완료합니다. Rlpx.go 암호화 된 링크를 처리하는 데 사용. 노드 검색 프로세스를 사용하여 발견하고 찾을 수 있습니다. 상기 접속 노드를 사용하여 전화를 생성하는 단계에 연결한다. 피어 개체를 사용하여 각 연결을 처리합니다.

를 수신하고 새로운 연결을받을 수있는 listenLoop 서버를 출시했다. 새 작업 다이얼을 생성하고 연결 dialstate 전화 goroutine의 실행을 시작합니다. 의사 소통 및 협력 goroutine 사이의 채널을 사용합니다.