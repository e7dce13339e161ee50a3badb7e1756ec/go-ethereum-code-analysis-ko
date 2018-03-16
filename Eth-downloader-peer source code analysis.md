피어 다운 모듈은 포장 처리량이 유휴 상태, 피어 노드가 사용 포함되어 있으며,이 정보는 오류가 발생하기 전에 기록됩니다.


## peer
	
	// peerConnection represents an active peer from which hashes and blocks are retrieved.
	type peerConnection struct {
		id string // Unique identifier of the peer
	
		headerIdle  int32 //(1 = 활성 유휴 = 0), 취득한 헤더의 현재 동작 상태 피어의 현재 헤더 활동 상태.
		blockIdle   int32 // Current block activity state of the peer (idle = 0, active = 1)현재 블록의 작동 상태를 얻는다
		receiptIdle int32 //취득한 수신 피어의 현재 수신 활동 상태 (유휴 = 0, 활성 = 1) 현재의 운전 상태
		stateIdle   int32 //상태 노드의 피어 노드의 현재 데이터 활동 상태 (유휴 = 0, 활성 = 1) 현재의 운전 상태
	
		headerThroughput  float64 // Number of headers measured to be retrievable per second// 초당받을 수 있습니다 얼마나 많은 기록 헤더 영역 측정
		blockThroughput   float64 //측정 블록 (기관)의 수는 두 번째 메트릭 초당 // 수신 할 수있는 방법을 많은 레코드 블록 당 검색 할 수하는
		receiptThroughput float64 //많은 영수증이 두 번째 메트릭 당 접수 방법 측정 영수증의 수를 기록 할 수있는 초당 검색 할 수하는
		stateThroughput   float64 //계좌 번호 2 기록 상태에 따라 검색 할 수 측정 노드 데이터 부분의 수는 1 미터 당 수신 할 수있다
	
		rtt time.Duration //응답 (QoS)을 요청 응답 시간을 추적하기 위해 왕복 시간 요청
	
		headerStarted  time.Time // Time instance when the last header fetch was started헤더의 마지막 기록은 요청 시간을 가져
		blockStarted   time.Time // Time instance when the last block (body) fetch was started
		receiptStarted time.Time // Time instance when the last receipt fetch was started
		stateStarted   time.Time // Time instance when the last node data fetch was started
		
		lacking map[common.Hash]struct{} //요청하지 해시 세트 해시 값이 요청에 가지 않을 기록 (이전하지 않았다), 일반적으로 이전 요청이 실패했기 때문에
	
		peer Peer		의 // 피어 ETH
	
		version int		// Eth protocol version number to switch strategies
		log	 log.Logger // Contextual logger to add extra infos to peer logs
		lock	sync.RWMutex
	}



FetchXXX
FetchHeaders FetchBodies 주요 함수 호출 eth.peer 함수는 데이터 요청을 송신한다.

	// FetchHeaders sends a header retrieval request to the remote peer.
	func (p *peerConnection) FetchHeaders(from uint64, count int) error {
		// Sanity check the protocol version
		if p.version < 62 {
			panic(fmt.Sprintf("header fetch [eth/62+] requested on eth/%d", p.version))
		}
		// Short circuit if the peer is already fetching
		if !atomic.CompareAndSwapInt32(&p.headerIdle, 0, 1) {
			return errAlreadyFetching
		}
		p.headerStarted = time.Now()
	
		// Issue the header retrieval request (absolut upwards without gaps)
		go p.peer.RequestHeadersByNumber(from, count, 0, false)
	
		return nil
	}

SetXXXIdle 기능
SetHeadersIdle는 SetBlocksIdle 다른 기능은 새로운 요청을 수행 할 수 있도록, 유휴 상태로 피어 상태를 제공했다. 데이터의 양이 시간을하여도 있지만하면 전송 링크의 처리량을 다시 평가합니다.
	
	// SetHeadersIdle sets the peer to idle, allowing it to execute new header retrieval
	// requests. Its estimated header retrieval throughput is updated with that measured
	// just now.
	func (p *peerConnection) SetHeadersIdle(delivered int) {
		p.setIdle(p.headerStarted, delivered, &p.headerThroughput, &p.headerIdle)
	}

setIdle

	// setIdle sets the peer to idle, allowing it to execute new retrieval requests.
	// Its estimated retrieval throughput is updated with that measured just now.
	func (p *peerConnection) setIdle(started time.Time, delivered int, throughput *float64, idle *int32) {
		// Irrelevant of the scaling, make sure the peer ends up idle
		defer atomic.StoreInt32(idle, 0)
	
		p.lock.Lock()
		defer p.lock.Unlock()
	
		// If nothing was delivered (hard timeout / unavailable data), reduce throughput to minimum
		if delivered == 0 {
			*throughput = 0
			return
		}
		// Otherwise update the throughput with a new measurement
		elapsed := time.Since(started) + 1 // +1 (ns) to ensure non-zero divisor
		measured := float64(delivered) / (float64(elapsed) / float64(time.Second))
		
		//measurementImpact = 0.1, 새로운 처리량 = 처리량 된 * 0.9 + 0.1 * 처리량
		*throughput = (1-measurementImpact)*(*throughput) + measurementImpact*measured	
		//업데이트 RTT
		p.rtt = time.Duration((1-measurementImpact)*float64(p.rtt) + measurementImpact*float64(elapsed))
	
		p.log.Trace("Peer throughput measurements updated",
			"hps", p.headerThroughput, "bps", p.blockThroughput,
			"rps", p.receiptThroughput, "sps", p.stateThroughput,
			"miss", len(p.lacking), "rtt", p.rtt)
	}


현재 링크를 반환 XXXCapacity 기능은 처리량을 할 수 있습니다.

	// HeaderCapacity retrieves the peers header download allowance based on its
	// previously discovered throughput.
	func (p *peerConnection) HeaderCapacity(targetRTT time.Duration) int {
		p.lock.RLock()
		defer p.lock.RUnlock()
		//여기에 이상한 큰 targetRTT, 요청의 더 많은 수입니다.
		return int(math.Min(1+math.Max(1, p.headerThroughput*float64(targetRTT)/float64(time.Second)), float64(MaxHeaderFetch)))
	}


다음 요청이 동일한 피어를 통과하지 않도록 그것은 실패 여부를 마지막으로 시간을 표시하는 데 사용 결여
		
		// MarkLacking appends a new entity to the set of items (blocks, receipts, states)
		// that a peer is known not to have (i.e. have been requested before). If the
		// set reaches its maximum allowed capacity, items are randomly dropped off.
		func (p *peerConnection) MarkLacking(hash common.Hash) {
			p.lock.Lock()
			defer p.lock.Unlock()
		
			for len(p.lacking) >= maxLackingHashes {
				for drop := range p.lacking {
					delete(p.lacking, drop)
					break
				}
			}
			p.lacking[hash] = struct{}{}
		}
		
		// Lacks retrieves whether the hash of a blockchain item is on the peers lacking
		// list (i.e. whether we know that the peer does not have it).
		func (p *peerConnection) Lacks(hash common.Hash) bool {
			p.lock.RLock()
			defer p.lock.RUnlock()
		
			_, ok := p.lacking[hash]
			return ok
		}


## peerSet

	// peerSet represents the collection of active peer participating in the chain
	// download procedure.
	type peerSet struct {
		peers		map[string]*peerConnection
		newPeerFeed  event.Feed
		peerDropFeed event.Feed
		lock		 sync.RWMutex
	}


등록 및 등록 취소

	// Register injects a new peer into the working set, or returns an error if the
	// peer is already known.
	//
	// The method also sets the starting throughput values of the new peer to the
	// average of all existing peers, to give it a realistic chance of being used
	// for data retrievals.
	func (ps *peerSet) Register(p *peerConnection) error {
		// Retrieve the current median RTT as a sane default
		p.rtt = ps.medianRTT()
	
		// Register the new peer with some meaningful defaults
		ps.lock.Lock()
		if _, ok := ps.peers[p.id]; ok {
			ps.lock.Unlock()
			return errAlreadyRegistered
		}
		if len(ps.peers) > 0 {
			p.headerThroughput, p.blockThroughput, p.receiptThroughput, p.stateThroughput = 0, 0, 0, 0
	
			for _, peer := range ps.peers {
				peer.lock.RLock()
				p.headerThroughput += peer.headerThroughput
				p.blockThroughput += peer.blockThroughput
				p.receiptThroughput += peer.receiptThroughput
				p.stateThroughput += peer.stateThroughput
				peer.lock.RUnlock()
			}
			p.headerThroughput /= float64(len(ps.peers))
			p.blockThroughput /= float64(len(ps.peers))
			p.receiptThroughput /= float64(len(ps.peers))
			p.stateThroughput /= float64(len(ps.peers))
		}
		ps.peers[p.id] = p
		ps.lock.Unlock()
	
		ps.newPeerFeed.Send(p)
		return nil
	}
	
	// Unregister removes a remote peer from the active set, disabling any further
	// actions to/from that particular entity.
	func (ps *peerSet) Unregister(id string) error {
		ps.lock.Lock()
		p, ok := ps.peers[id]
		if !ok {
			defer ps.lock.Unlock()
			return errNotRegistered
		}
		delete(ps.peers, id)
		ps.lock.Unlock()
	
		ps.peerDropFeed.Send(p)
		return nil
	}

XXXIdlePeers

	// HeaderIdlePeers retrieves a flat list of all the currently header-idle peers
	// within the active peer set, ordered by their reputation.
	func (ps *peerSet) HeaderIdlePeers() ([]*peerConnection, int) {
		idle := func(p *peerConnection) bool {
			return atomic.LoadInt32(&p.headerIdle) == 0
		}
		throughput := func(p *peerConnection) float64 {
			p.lock.RLock()
			defer p.lock.RUnlock()
			return p.headerThroughput
		}
		return ps.idlePeers(62, 64, idle, throughput)
	}

	// idlePeers retrieves a flat list of all currently idle peers satisfying the
	// protocol version constraints, using the provided function to check idleness.
	// The resulting set of peers are sorted by their measure throughput.
	func (ps *peerSet) idlePeers(minProtocol, maxProtocol int, idleCheck func(*peerConnection) bool, throughput func(*peerConnection) float64) ([]*peerConnection, int) {
		ps.lock.RLock()
		defer ps.lock.RUnlock()
	
		idle, total := make([]*peerConnection, 0, len(ps.peers)), 0
		for _, p := range ps.peers { //또래 아이들의 첫 추출
			if p.version >= minProtocol && p.version <= maxProtocol {
				if idleCheck(p) {
					idle = append(idle, p)
				}
				total++
			}
		}
		for i := 0; i < len(idle); i++ { //작은 높은 처리량에서 버블 정렬은, 처리량합니다.
			for j := i + 1; j < len(idle); j++ {
				if throughput(idle[i]) < throughput(idle[j]) {
					idle[i], idle[j] = idle[j], idle[i]
				}
			}
		}
		return idle, total
	}

medianRTT는, RTT의 동종 업체의 평균을 획득,
	
	// medianRTT returns the median RTT of te peerset, considering only the tuning
	// peers if there are more peers available.
	func (ps *peerSet) medianRTT() time.Duration {
		// Gather all the currnetly measured round trip times
		ps.lock.RLock()
		defer ps.lock.RUnlock()
	
		rtts := make([]float64, 0, len(ps.peers))
		for _, p := range ps.peers {
			p.lock.RLock()
			rtts = append(rtts, float64(p.rtt))
			p.lock.RUnlock()
		}
		sort.Float64s(rtts)
	
		median := rttMaxEstimate
		if qosTuningPeers <= len(rtts) {
			median = time.Duration(rtts[qosTuningPeers/2]) // Median of our tuning peers
		} else if len(rtts) > 0 {
			median = time.Duration(rtts[len(rtts)/2]) // Median of our connected peers (maintain even like this some baseline qos)
		}
		// Restrict the RTT into some QoS defaults, irrelevant of true RTT
		if median < rttMinEstimate {
			median = rttMinEstimate
		}
		if median > rttMaxEstimate {
			median = rttMaxEstimate
		}
		return median
	}
