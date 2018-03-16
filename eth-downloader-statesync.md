statesync이 지정된 블록 트라이 트리의 피벗 점의 상태를 모두 가져 오는 데 사용, 모든 계정과 일반 계정 계약을 포함하여 정보를 차지하고있다.

## 데이터 구조
예약은 주어진 상태 루트 정의의 요청에 의해 특정 상태의 트라이 stateSync을 다운로드 할 수 있습니다.

	// stateSync schedules requests for downloading a particular state trie defined
	// by a given state root.
	type stateSync struct {
		d *Downloader // Downloader instance to access and manage current peerset
	
		sched  *trie.TrieSync			 // State trie sync scheduler defining the tasks
		keccak hash.Hash				  // Keccak256 hasher to verify deliveries with
		tasks  map[common.Hash]*stateTask // Set of tasks currently queued for retrieval
	
		numUncommitted   int
		bytesUncommitted int
	
		deliver	chan *stateReq // Delivery channel multiplexing peer responses
		cancel	 chan struct{}  // Channel to signal a termination request
		cancelOnce sync.Once	  // Ensures cancel only ever gets called once
		done	   chan struct{}  // Channel to signal termination completion
		err		error		  // Any error hit during sync (set before completion)
	}

생성자

	func newStateSync(d *Downloader, root common.Hash) *stateSync {
		return &stateSync{
			d:	   d,
			sched:   state.NewStateSync(root, d.stateDB),
			keccak:  sha3.NewKeccak256(),
			tasks:   make(map[common.Hash]*stateTask),
			deliver: make(chan *stateReq),
			cancel:  make(chan struct{}),
			done:	make(chan struct{}),
		}
	}

NewStateSync
	
	// NewStateSync create a new state trie download scheduler.
	func NewStateSync(root common.Hash, database trie.DatabaseReader) *trie.TrieSync {
		var syncer *trie.TrieSync
		callback := func(leaf []byte, parent common.Hash) error {
			var obj Account
			if err := rlp.Decode(bytes.NewReader(leaf), &obj); err != nil {
				return err
			}
			syncer.AddSubTrie(obj.Root, 64, parent, nil)
			syncer.AddRawEntry(common.BytesToHash(obj.CodeHash), 64, parent)
			return nil
		}
		syncer = trie.NewTrieSync(root, database, callback)
		return syncer
	}

syncState,이 기능은 다운이라고합니다.

	// syncState starts downloading state with the given root hash.
	func (d *Downloader) syncState(root common.Hash) *stateSync {
		s := newStateSync(d, root)
		select {
		case d.stateSyncStart <- s:
		case <-d.quitCh:
			s.err = errCancelStateFetch
			close(s.done)
		}
		return s
	}

## 출발
우리는 다운로더에 stateFetcher 기능을 실행하기 위해 새로운 goroutine을 시작했다. 이 기능은 제 1 정보에 대한 채널을 stateSyncStart을 시도합니다. 이 함수 syncState stateSyncStart 데이터 전송 채널이 될 것이다.

	// stateFetcher manages the active state sync and accepts requests
	// on its behalf.
	func (d *Downloader) stateFetcher() {
		for {
			select {
			case s := <-d.stateSyncStart:
				for next := s; next != nil; { //for 루프는 다운 개체가 신호를 전송하여 언제든지 동기화해야 변경 나타낸다.
					next = d.runStateSync(next)
				}
			case <-d.stateCh:
				// Ignore state responses while no sync is running.
			case <-d.quitCh:
				return
			}
		}
	}

의 우리가 syncState () 함수를 호출합니다 어디 보자. processFastSyncContent이 기능은 피어 발견 시간의 시작 부분에서 시작됩니다.

	// processFastSyncContent takes fetch results from the queue and writes them to the
	// database. It also controls the synchronisation of state nodes of the pivot block.
	func (d *Downloader) processFastSyncContent(latest *types.Header) error {
		// Start syncing state of the reported head block.
		// This should get us most of the state of the pivot block.
		stateSync := d.syncState(latest.Root)

	

runStateSync는 상태를 얻을 수있는이 방법은 다른 사람들이 제공하는 채널과 거래를 기다리는 그에게 전달 stateCh에서 다운로드되었습니다.
	
	// runStateSync runs a state synchronisation until it completes or another root
	// hash is requested to be switched over to.
	func (d *Downloader) runStateSync(s *stateSync) *stateSync {
		var (
			active   = make(map[string]*stateReq) // Currently in-flight requests
			finished []*stateReq				  // Completed or failed requests
			timeout  = make(chan *stateReq)	   // Timed out active requests
		)
		defer func() {
			// Cancel active request timers on exit. Also set peers to idle so they're
			// available for the next sync.
			for _, req := range active {
				req.timer.Stop()
				req.peer.SetNodeDataIdle(len(req.items))
			}
		}()
		// Run the state sync.
		//상태 동기화를 실행
		go s.run()
		defer s.Cancel()
	
		// Listen for peer departure events to cancel assigned tasks
		peerDrop := make(chan *peerConnection, 1024)
		peerSub := s.d.peers.SubscribePeerDrops(peerDrop)
		defer peerSub.Unsubscribe()
	
		for {
			// Enable sending of the first buffered element if there is one.
			var (
				deliverReq   *stateReq
				deliverReqCh chan *stateReq
			)
			if len(finished) > 0 {
				deliverReq = finished[0]
				deliverReqCh = s.deliver
			}
	
			select {
			// The stateSync lifecycle:
			//또 다른 stateSync 응용 프로그램 실행. 우리는 종료합니다.
			case next := <-d.stateSyncStart:
				return next
	
			case <-s.done:
				return nil
	
			// Send the next finished request to the current sync:
			//데이터를 동기화 보내기 다운로드되었습니다
			case deliverReqCh <- deliverReq:
				finished = append(finished[:0], finished[1:]...)
	
			// Handle incoming state packs:
			//인 커밍 패킷을 처리하는 단계를 포함한다. 수신 된 데이터의 다운 상태 현제 채널로 전송된다.
			case pack := <-d.stateCh:
				// Discard any data not requested (or previsouly timed out)
				req := active[pack.PeerId()]
				if req == nil {
					log.Debug("Unrequested node data", "peer", pack.PeerId(), "len", pack.Items())
					continue
				}
				// Finalize the request and queue up for processing
				req.timer.Stop()
				req.response = pack.(*statePack).states
	
				finished = append(finished, req)
				delete(active, pack.PeerId())
	
				// Handle dropped peer connections:
			case p := <-peerDrop:
				// Skip if no request is currently pending
				req := active[p.id]
				if req == nil {
					continue
				}
				// Finalize the request and queue up for processing
				req.timer.Stop()
				req.dropped = true
	
				finished = append(finished, req)
				delete(active, p.id)
	
			// Handle timed-out requests:
			case req := <-timeout:
				// If the peer is already requesting something else, ignore the stale timeout.
				// This can happen when the timeout and the delivery happens simultaneously,
				// causing both pathways to trigger.
				if active[req.peer.id] != req {
					continue
				}
				// Move the timed out data back into the download queue
				finished = append(finished, req)
				delete(active, req.peer.id)
	
			// Track outgoing state requests:
			case req := <-d.trackStateReq:
				// If an active request already exists for this peer, we have a problem. In
				// theory the trie node schedule must never assign two requests to the same
				// peer. In practive however, a peer might receive a request, disconnect and
				// immediately reconnect before the previous times out. In this case the first
				// request is never honored, alas we must not silently overwrite it, as that
				// causes valid requests to go missing and sync to get stuck.
				if old := active[req.peer.id]; old != nil {
					log.Warn("Busy peer assigned new state fetch", "peer", old.peer.id)
	
					// Make sure the previous one doesn't get siletly lost
					old.timer.Stop()
					old.dropped = true
	
					finished = append(finished, old)
				}
				// Start a timer to notify the sync loop if the peer stalled.
				req.timer = time.AfterFunc(req.timeout, func() {
					select {
					case timeout <- req:
					case <-s.done:
						// Prevent leaking of timer goroutines in the unlikely case where a
						// timer is fired just before exiting runStateSync.
					}
				})
				active[req.peer.id] = req
			}
		}
	}


결과를 얻을 수, 작업을 할당, 실행 및 루프 방법은 작업을 얻을 수 있습니다.

	func (s *stateSync) run() {
		s.err = s.loop()
		close(s.done)
	}
	
	// loop is the main event loop of a state trie sync. It it responsible for the
	// assignment of new tasks to peers (including sending it to them) as well as
	// for the processing of inbound data. Note, that the loop does not directly
	// receive data from peers, rather those are buffered up in the downloader and
	// pushed here async. The reason is to decouple processing from data receipt
	// and timeouts.
	func (s *stateSync) loop() error {
		// Listen for new peer events to assign tasks to them
		newPeer := make(chan *peerConnection, 1024)
		peerSub := s.d.peers.SubscribeNewPeers(newPeer)
		defer peerSub.Unsubscribe()
	
		// Keep assigning new tasks until the sync completes or aborts
		//동기화가 완료되거나 종료 될 때까지 기다립니다
		for s.sched.Pending() > 0 {
			//내부에 영구 저장소에 캐시에서 데이터를 새로 고칩니다. 이 크기를 지정 --cache 명령 줄입니다.
			if err := s.commit(false); err != nil {
				return err
			}
			//할당,
			s.assignTasks()
			// Tasks assigned, wait for something to happen
			select {
			case <-newPeer:
				// New peer arrived, try to assign it download tasks
	
			case <-s.cancel:
				return errCancelStateFetch
	
			case req := <-s.deliver:
				//RunStateSync은 다시 배송 방법 정보를 통해 접수도 성공적으로 반환 요청을 포함 실패한 요청을 포함하고 정보를 확인합니다.
				// Response, disconnect or timeout triggered, drop the peer if stalling
				log.Trace("Received node data response", "peer", req.peer.id, "count", len(req.response), "dropped", req.dropped, "timeout", !req.dropped && req.timedOut())
				if len(req.items) <= 2 && !req.dropped && req.timedOut() {
					// 2 items are the minimum requested, if even that times out, we've no use of
					// this peer at the moment.
					log.Warn("Stalling state sync, dropping peer", "peer", req.peer.id)
					s.d.dropPeer(req.peer.id)
				}
				// Process all the received blobs and check for stale delivery
				stale, err := s.process(req)
				if err != nil {
					log.Warn("Node data write error", "err", err)
					return err
				}
				// The the delivery contains requested data, mark the node idle (otherwise it's a timed out delivery)
				if !stale {
					req.peer.SetNodeDataIdle(len(req.response))
				}
			}
		}
		return s.commit(true)
	}