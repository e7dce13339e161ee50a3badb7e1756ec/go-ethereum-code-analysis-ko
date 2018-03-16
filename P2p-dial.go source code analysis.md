링크를 설정하는 작업의 한 부분을 담당하는 P2P에 dial.go. 예를 들어, 링크 노드의 설립 것을 발견했다. 노드와 링크를 설정합니다. 발견하여 노드의 주소를 찾을 수 있습니다. 그리고 다른 기능.


중간 상태, 코어 데이터 구조 내부 다이얼 기능을 저장하는 데이터 구조의 내부 dailstate dial.go.

	// dialstate schedules dials and discovery lookups.
	// it get's a chance to compute new tasks on every iteration
	// of the main loop in Server.run.
	type dialstate struct {
		maxDynDials int					// 노드 링크의 최대 동적 수
		ntab		discoverTable		// discoverTable 쿼리 노드를 만드는 데 사용
		netrestrict *netutil.Netlist
	
		lookupRunning bool
		dialing	   map[discover.NodeID]connFlag	// 노드가 연결되어
		lookupBuf [* Discover.Node // 현재 검색 조회 결과 // 현재 검색 질의 결과
		randomNodes   []*discover.Node //discoverTable에서 표 // 임의 쿼리에서 노드 작성
		static	[discover.NodeID * dialTask ​​// 정적 노드 매핑.
		hist		  *dialHistory
	
		start	 time.Time		// time when the dialer was first used
		bootnodes []*discover.Node //기본 다이얼이 더 동료가 없을 때 //이 내장 된 노드입니다. 다른 노드를 찾을 수없는 경우. 그런 다음이 노드를 연결합니다.
	}

dailstate의 창조.

	func newDialState(static []*discover.Node, bootnodes []*discover.Node, ntab discoverTable, maxdyn int, netrestrict *netutil.Netlist) *dialstate {
		s := &dialstate{
			maxDynDials: maxdyn,
			ntab:		ntab,
			netrestrict: netrestrict,
			static:	  make(map[discover.NodeID]*dialTask),
			dialing:	 make(map[discover.NodeID]connFlag),
			bootnodes:   make([]*discover.Node, len(bootnodes)),
			randomNodes: make([]*discover.Node, maxdyn/2),
			hist:		new(dialHistory),
		}
		copy(s.bootnodes, bootnodes)
		for _, n := range static {
			s.addStatic(n)
		}
		return s
	}

가장 중요한 방법 데일은 newTasks 방법입니다. 이 방법은 작업을 생성하는 데 사용됩니다. 작업 인터페이스입니다. 우리는 방법이 있습니까.
	
	type task interface {
		Do(*Server)
	}

	func (s *dialstate) newTasks(nRunning int, peers map[discover.NodeID]*Peer, now time.Time) []task {
		if s.start == (time.Time{}) {
			s.start = now
		}
	
		var newtasks []task
		//addDial 먼저 checkDial 검사 노드에 의해 내부 방법이다. 이어서 newtasks 내부 큐에 추가 마지막 노드의 상태를 설정.
		addDial := func(flag connFlag, n *discover.Node) bool {
			if err := s.checkDial(n, peers); err != nil {
				log.Trace("Skipping dial candidate", "id", n.ID, "addr", &net.TCPAddr{IP: n.IP, Port: int(n.TCP)}, "err", err)
				return false
			}
			s.dialing[n.ID] = flag
			newtasks = append(newtasks, &dialTask{flags: flag, dest: n})
			return true
		}
	
		// Compute number of dynamic dials necessary at this point.
		needDynDials := s.maxDynDials
		//먼저 설정된 연결의 유형을 결정합니다. 경우 동적으로 입력됩니다. 우리는 수를 줄이기 위해 동적 링크를 설정해야합니다.
		for _, p := range peers {
			if p.rw.is(dynDialedConn) {
				needDynDials--
			}
		}
		//그리고 링크가 설립되고 결정합니다. 경우 동적으로 입력됩니다. 우리는 수를 줄이기 위해 동적 링크를 설정해야합니다.
		for _, flag := range s.dialing {
			if flag&dynDialedConn != 0 {
				needDynDials--
			}
		}
	
		// Expire the dial history on every invocation.
		s.hist.expire(now)
	
		// Create dials for static nodes if they are not connected.
		//모든 정적 유형을 참조하십시오. 당신은 다음 링크를 생성 할 수 있습니다.
		for id, t := range s.static {
			err := s.checkDial(t.dest, peers)
			switch err {
			case errNotWhitelisted, errSelf:
				log.Warn("Removing static dial candidate", "id", t.dest.ID, "addr", &net.TCPAddr{IP: t.dest.IP, Port: int(t.dest.TCP)}, "err", err)
				delete(s.static, t.dest.ID)
			case nil:
				s.dialing[id] = t.flags
				newtasks = append(newtasks, t)
			}
		}
		// If we don't have any peers whatsoever, try to dial a random bootnode. This
		// scenario is useful for the testnet (and private networks) where the discovery
		// table might be full of mostly bad peers, making it hard to find good ones.
		//당신은 현재 어떤 링크가없는 경우. 20 초 (fallbackInterval) 내의 링크를 생성하지 않습니다. 그런 다음 링크를 생성 bootnode 사용합니다.
		if len(peers) == 0 && len(s.bootnodes) > 0 && needDynDials > 0 && now.Sub(s.start) > fallbackInterval {
			bootnode := s.bootnodes[0]
			s.bootnodes = append(s.bootnodes[:0], s.bootnodes[1:]...)
			s.bootnodes = append(s.bootnodes, bootnode)
	
			if addDial(dynDialedConn, bootnode) {
				needDynDials--
			}
		}
		// Use random nodes from the table for half of the necessary
		// dynamic dials.
		//그렇지 않으면, 임의의 노드 1/2를 사용하여 링크를 만들 수 있습니다.
		randomCandidates := needDynDials / 2
		if randomCandidates > 0 {
			n := s.ntab.ReadRandomNodes(s.randomNodes)
			for i := 0; i < randomCandidates && i < n; i++ {
				if addDial(dynDialedConn, s.randomNodes[i]) {
					needDynDials--
				}
			}
		}
		// Create dynamic dials from random lookup results, removing tried
		// items from the result buffer.
		i := 0
		for ; i < len(s.lookupBuf) && needDynDials > 0; i++ {
			if addDial(dynDialedConn, s.lookupBuf[i]) {
				needDynDials--
			}
		}
		s.lookupBuf = s.lookupBuf[:copy(s.lookupBuf, s.lookupBuf[i:])]
		// Launch a discovery lookup if more candidates are needed.
		//심지어이 충분한 동적 링크를 생성 할 수 없습니다. 그런 다음 discoverTask를 만들 네트워크의 다른 노드를 찾는 데 사용. lookupBuf를 넣어
		if len(s.lookupBuf) < needDynDials && !s.lookupRunning {
			s.lookupRunning = true
			newtasks = append(newtasks, &discoverTask{})
		}
	
		// Launch a timer to wait for the next node to expire if all
		// candidates have been tried and no task is currently active.
		// This should prevent cases where the dialer logic is not ticked
		// because there are no pending events.
		//모든 작업, 수면 수익을 만드는 다음 작업을 할 필요가없는 경우.
		if nRunning == 0 && len(newtasks) == 0 && s.hist.Len() > 0 {
			t := &waitExpireTask{s.hist.min().exp.Sub(now)}
			newtasks = append(newtasks, t)
		}
		return newtasks
	}


태스크 링크를 만들어야하는지 여부를 checkDial 방법은 확인합니다.

	func (s *dialstate) checkDial(n *discover.Node, peers map[discover.NodeID]*Peer) error {
		_, dialing := s.dialing[n.ID]
		switch {
		case dialing:				// 만들기
			return errAlreadyDialing
		case peers[n.ID] != nil:	// 이미 연결
			return errAlreadyConnected
		case s.ntab != nil && n.ID == s.ntab.Self().ID:객체를 생성 // 것은 자신 아니다
			return errSelf
		case s.netrestrict != nil && !s.netrestrict.Contains(n.IP): //네트워크 제한. 서로의 IP 주소는 내부에 화이트리스트에 없습니다.
			return errNotWhitelisted
		case s.hist.contains(n.ID):// ID는 이전에 연결되어 있습니다.
			return errRecentlyDialed
		}
		return nil
	}

taskDone 방법. 작업이 완료되면이 메서드는 다시 호출됩니다. 작업의보기 유형입니다. 작업이 연결되어 있으면, 내부 HIST 증가. 그리고는 링크 큐에서 제거됩니다. 쿼리 작업하는 경우. 내부 lookupBuf의 단점 쿼리.

	func (s *dialstate) taskDone(t task, now time.Time) {
		switch t := t.(type) {
		case *dialTask:
			s.hist.add(t.dest.ID, now.Add(dialHistoryExpiration))
			delete(s.dialing, t.dest.ID)
		case *discoverTask:
			s.lookupRunning = false
			s.lookupBuf = append(s.lookupBuf, t.results...)
		}
	}



dialTask.Do 방법, 다른 작업은 다른 방법을 수행하십시오. 연결을 설정하는 주된 책임 dailTask. t.dest 경우에는 IP 주소입니다. 그런 다음 IP 주소 결의를 조회하려고합니다. 그런 다음 링크를 생성하는 방법을 다이얼, 통화. 정적 노드. 첫 번째 실패하면 다시 정적 노드를 해결하기 위해 노력할 것입니다. 그리고 전화를 시도 (노드 변경의 정적 IP 주소가. 그래서 우리는 새 주소 정적 노드를 해결 한 다음 링크를 호출하려고합니다. IP와 같은 정적 노드가 구성되어 있습니다.)

	func (t *dialTask) Do(srv *Server) {
		if t.dest.Incomplete() {
			if !t.resolve(srv) {
				return
			}
		}
		success := t.dial(srv, t.dest)
		// Try resolving the ID of static nodes if dialing failed.
		if !success && t.flags&staticDialedConn != 0 {
			if t.resolve(srv) {
				t.dial(srv, t.dest)
			}
		}
	}

방법을 해결. 이 방법은 주요 방법 해결 네트워크를 발견 호출합니다. 즉 실패하면 타임 아웃을 시도

	// resolve attempts to find the current endpoint for the destination
	// using discovery.
	//
	// Resolve operations are throttled with backoff to avoid flooding the
	// discovery network with useless queries for nodes that don't exist.
	// The backoff delay resets when the node is found.
	func (t *dialTask) resolve(srv *Server) bool {
		if srv.ntab == nil {
			log.Debug("Can't resolve node", "id", t.dest.ID, "err", "discovery is disabled")
			return false
		}
		if t.resolveDelay == 0 {
			t.resolveDelay = initialResolveDelay
		}
		if time.Since(t.lastResolved) < t.resolveDelay {
			return false
		}
		resolved := srv.ntab.Resolve(t.dest.ID)
		t.lastResolved = time.Now()
		if resolved == nil {
			t.resolveDelay *= 2
			if t.resolveDelay > maxResolveDelay {
				t.resolveDelay = maxResolveDelay
			}
			log.Debug("Resolving node failed", "id", t.dest.ID, "newdelay", t.resolveDelay)
			return false
		}
		// The node was found.
		t.resolveDelay = initialResolveDelay
		t.dest = resolved
		log.Debug("Resolved node", "id", t.dest.ID, "addr", &net.TCPAddr{IP: t.dest.IP, Port: int(t.dest.TCP)})
		return true
	}
	

방법에있어서, 네트워크 연결의 동작의 실제 방법 다이얼. 주로 srv.SetupConn 방법,이 방법의 후속 재분석 Server.go 재분석 내지.

	// dial performs the actual connection attempt.
	func (t *dialTask) dial(srv *Server, dest *discover.Node) bool {
		fd, err := srv.Dialer.Dial(dest)
		if err != nil {
			log.Trace("Dial error", "task", t, "err", err)
			return false
		}
		mfd := newMeteredConn(fd, false)
		srv.SetupConn(mfd, t.flags, dest)
		return true
	}

, 방법의 discoverTask 및 waitExpireTask를 수행

	func (t *discoverTask) Do(srv *Server) {
		// newTasks generates a lookup task whenever dynamic dials are
		// necessary. Lookups need to take some time, otherwise the
		// event loop spins too fast.
		next := srv.lastLookup.Add(lookupInterval)
		if now := time.Now(); now.Before(next) {
			time.Sleep(next.Sub(now))
		}
		srv.lastLookup = time.Now()
		var target discover.NodeID
		rand.Read(target[:])
		t.results = srv.ntab.Lookup(target)
	}

	
	func (t waitExpireTask) Do(*Server) {
		time.Sleep(t.Duration)
	}