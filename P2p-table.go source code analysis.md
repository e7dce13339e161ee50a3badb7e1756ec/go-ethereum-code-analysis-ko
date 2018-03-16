Kademlia의 P2P 프로토콜의 주요 성과를 table.go.

### 약 Kademlia 프로토콜 (PDF 문서 내의 권장 독서 참조)
(이하 Kad 추가로 언급) Kademlia 프로토콜은 PetarP. Maymounkov와 데이빗 마지 어스 뉴욕 대학이다.
2002 &quot;Kademlia에 발표 된 연구 : 정보 시스템 -peer peerto을 기반으로
&quot;는 XOR 메트릭.
간단히 말해, Kad 추가는 분산 해시 테이블 (DHT) 기술이며,보다 DHT와 같은 다른 기술을 달성하기 위해
코드 등, 수, 과자, 배타적 OR (XOR)에 고유 한 알고리즘 Kad 추가 기반 형 거리 측정, 설립
다른 알고리즘에 비해 새로운 DHT 토폴로지는 크게 라우팅 쿼리 속도를 향상시킬 수있다.


### 테이블 구조 및 필드

	const (
		alpha	  = 3  // Kademlia concurrency factor
		bucketSize = 16 // Kademlia bucket size
		hashBits   = len(common.Hash{}) * 8
		nBuckets   = hashBits + 1 // Number of buckets
	
		maxBondingPingPongs = 16
		maxFindnodeFailures = 5
	
		autoRefreshInterval = 1 * time.Hour
		seedCount		   = 30
		seedMaxAge		  = 5 * 24 * time.Hour
	)
	
	type Table struct {
		mutex   sync.Mutex		// protects buckets, their content, and nursery
		buckets [nBuckets]*bucket // index of known nodes by distance
		nursery []*Node		   // bootstrap nodes
		db	  *nodeDB		   // database of known nodes
	
		refreshReq chan chan struct{}
		closeReq   chan struct{}
		closed	 chan struct{}
	
		bondmu	sync.Mutex
		bonding   map[NodeID]*bondproc
		bondslots chan struct{} // limits total number of active bonding processes
	
		nodeAddedHook func(*Node) // for testing
	
		net  transport
		self *Node // metadata of the local node
	}


### 초기화


	func newTable(t transport, ourID NodeID, ourAddr *net.UDPAddr, nodeDBPath string) (*Table, error) {
		// If no node database was given, use an in-memory one
		//database.go가 도입 전에. 열기 leveldb. 경로가 비어있는 경우. 그 후, 메모리 기반 DB를 열고
		db, err := newNodeDB(nodeDBPath, Version, ourID)
		if err != nil {
			return nil, err
		}
		tab := &Table{
			net:		t,
			db:		 db,
			self:	   NewNode(ourID, ourAddr.IP, uint16(ourAddr.Port), uint16(ourAddr.Port)),
			bonding:	make(map[NodeID]*bondproc),
			bondslots:  make(chan struct{}, maxBondingPingPongs),
			refreshReq: make(chan chan struct{}),
			closeReq:   make(chan struct{}),
			closed:	 make(chan struct{}),
		}
		for i := 0; i < cap(tab.bondslots); i++ {
			tab.bondslots <- struct{}{}
		}
		for i := range tab.buckets {
			tab.buckets[i] = new(bucket)
		}
		go tab.refreshLoop()
		return tab, nil
	}

위의 초기화,이 기능은 다음 작업을 완료하는 데 주로 ()는 goroutine의 refreshLoop을 시작했다.

1. (autoRefreshInterval)를 한 번 작업 시간마다 갱신
2. 요청이 refreshReq을받은 경우. 그런 다음 작업을 새로 고칩니다.
3. 상기 수신 된 메시지가 폐쇄되는 경우. 그래서이 닫힙니다.

따라서, 주요 직무는 작업 새로 고침을 시작하는 것입니다. doRefresh


	// refreshLoop schedules doRefresh runs and coordinates shutdown.
	func (tab *Table) refreshLoop() {
		var (
			timer   = time.NewTicker(autoRefreshInterval)
			waiting []chan struct{} // accumulates waiting callers while doRefresh runs
			done	chan struct{}   // where doRefresh reports completion
		)
	loop:
		for {
			select {
			case <-timer.C:
				if done == nil {
					done = make(chan struct{})
					go tab.doRefresh(done)
				}
			case req := <-tab.refreshReq:
				waiting = append(waiting, req)
				if done == nil {
					done = make(chan struct{})
					go tab.doRefresh(done)
				}
			case <-done:
				for _, ch := range waiting {
					close(ch)
				}
				waiting = nil
				done = nil
			case <-tab.closeReq:
				break loop
			}
		}
	
		if tab.net != nil {
			tab.net.close()
		}
		if done != nil {
			<-done
		}
		for _, ch := range waiting {
			close(ch)
		}
		tab.db.close()
		close(tab.closed)
	}


doRefresh 기능

	// doRefresh performs a lookup for a random target to keep buckets
	// full. seed nodes are inserted if the table is empty (initial
	// bootstrap or discarded faulty peers).
	//doRefresh 전체 버킷을 유지하기 위해, 임의의 대상을 찾을 수 있습니다. 테이블이 비어있는 경우, 시드 노드가 삽입됩니다. (예 : 후 시작하거나 시작 잘못된 노드를 삭제)
	func (tab *Table) doRefresh(done chan struct{}) {
		defer close(done)
	
		// The Kademlia paper specifies that the bucket refresh should
		// perform a lookup in the least recently used bucket. We cannot
		// adhere to this because the findnode target is a 512bit value
		// (not hash-sized) and it is not easily possible to generate a
		// sha3 preimage that falls into a chosen bucket.
		// We perform a lookup with a random target instead.
		//여기에 일시적으로 이해하지 못했다
		var target NodeID
		rand.Read(target[:])
		result := tab.lookup(target, false) //조회 가장 가까운 k 개의 노드에서 대상을 찾을 수 있습니다
		if len(result) > 0 {  //결과가 0 설명 테이블이없는 비어있는 경우, 직접 돌아갑니다.
			return
		}
	
		// The table is empty. Load nodes from the database and insert
		// them. This should yield a few previously seen nodes that are
		// (hopefully) still alive.
		//querySeeds는 database.go 장 내부 데이터베이스에서 무작위로 발견 가능한 종자 노드를 도입 작동합니다.
		//데이터베이스의 시작 부분에서 시작 비어 있습니다. 즉, 처음부터 반환 된 종자가 비어있다.
		seeds := tab.db.querySeeds(seedCount, seedMaxAge)
		//bondall 함수를 호출. 그것은 이러한 노드에 접속을 시도하고, 테이블에 삽입.
		//tab.nursery는 명령 행 씨 노드에 지정됩니다.
		//대부분은 부팅시 시작합니다. Tab.nursery 값은 코드 내부에 내장되어 있습니다. 여기서 값이있다.
		//C:\GOPATH\src\github.com\ethereum\go-ethereum\mobile\params.go
		//이 값은 내부 작성 죽었다고한다. 이 값은 SetFallbackNodes 방법을 통해 기록된다. 후속이 방법을 분석합니다.
		//다음은 탁구의 양방향 교환 될 것입니다. 결과는 다음 데이터베이스에 저장됩니다.
		seeds = tab.bondall(append(seeds, tab.nursery...))
	
		if len(seeds) == 0 { //어떤 씨앗 노드가 발견되지 않는, 다음 새로 고침 기다려야 할 수도 있습니다.
			log.Debug("No discv4 seed nodes found")
		}
		for _, n := range seeds {
			age := log.Lazy{Fn: func() time.Duration { return time.Since(tab.db.lastPong(n.ID)) }}
			log.Trace("Found seed node in database", "id", n.ID, "addr", n.addr(), "age", age)
		}
		tab.mutex.Lock()
		//모든 시드 결합을 통해이 방법은 버킷에 추가된다 (버킷이 완전하지 않다한다)
		tab.stuff(seeds) 
		tab.mutex.Unlock()
	
		// Finally, do a self lookup to fill up the buckets.
		tab.lookup(tab.self.ID, false) //시드 노드. 그래서 버킷을 채워 자신을 찾을 수 있습니다.
	}

본드 호출 bondall있어서, 멀티 - 스레드이다.

	// bondall bonds with all given nodes concurrently and returns
	// those nodes for which bonding has probably succeeded.
	func (tab *Table) bondall(nodes []*Node) (result []*Node) {
		rc := make(chan *Node, len(nodes))
		for i := range nodes {
			go func(n *Node) {
				nn, _ := tab.bond(false, n.ID, n.addr(), uint16(n.TCP))
				rc <- nn
			}(nodes[i])
		}
		for range nodes {
			if n := <-rc; n != nil {
				result = append(result, n)
			}
		}
		return result
	}

결합 방법. 에서 udp.go을 기억하십시오. 이 방법을 호출 할 수 있습니다 때 우리는 핑 방법을받을 때


	// bond ensures the local node has a bond with the given remote node.
	// It also attempts to insert the node into the table if bonding succeeds.
	// The caller must not hold tab.mutex.
	//결합은 로컬 노드와 원격 노드가 주신 결합되도록. (ID와 말단부 원격 IP).
	//바인딩이 성공하면 노드 표를 삽입하려고합니다. 호출자는 tab.mutex 잠금을 보유해야합니다
	// A bond is must be established before sending findnode requests.
	// Both sides must have completed a ping/pong exchange for a bond to
	// exist. The total number of active bonding processes is limited in
	// order to restrain network use.
	// 发送findnode请求之前必须建立一个绑定。양 당사자는 결합을 완료하기 위해 양방향 핑 / 퐁 프로세스를 완료해야합니다.
	// 为了节约网路资源。 同时存在的bonding处理流程的总数量是受限的。
	// bond is meant to operate idempotently in that bonding with a remote
	// node which still remembers a previously established bond will work.
	// The remote node will simply not send a ping back, causing waitping
	// to time out.
	//결합 동작은 이전 원격 노드 본드 결합도 수행 할 수 회수에이어서, 멱등된다. 원격 노드는 단순히 핑을 전송하지 않습니다. 타임 아웃을 waitping 기다립니다.
	// If pinged is true, the remote node has just pinged us and one half
	// of the process can be skipped.
	//핑 경우는 사실이다. 그런 다음 원격 노드가 우리에게 메시지를 보냈습니다 핑 (ping). 공정의 이러한 절반은 생략 될 수있다.
	func (tab *Table) bond(pinged bool, id NodeID, addr *net.UDPAddr, tcpPort uint16) (*Node, error) {
		if id == tab.self.ID {
			return nil, errors.New("is self")
		}
		// Retrieve a previously known node and any recent findnode failures
		node, fails := tab.db.node(id), 0
		if node != nil {
			fails = tab.db.findFails(id)
		}
		// If the node is unknown (non-bonded) or failed (remotely unknown), bond from scratch
		var result error
		age := time.Since(tab.db.lastPong(id))
		if node == nil || fails > 0 || age > nodeDBNodeExpiration {
			//데이터베이스는이 노드가없는 경우. 또는 오류의 수는 0 또는 노드 시간 제한보다 큽니다.
			log.Trace("Starting bonding ping/pong", "id", id, "known", node != nil, "failcount", fails, "age", age)
	
			tab.bondmu.Lock()
			w := tab.bonding[id]
			if w != nil {
				// Wait for an existing bonding process to complete.
				tab.bondmu.Unlock()
				<-w.done
			} else {
				// Register a new bonding process.
				w = &bondproc{done: make(chan struct{})}
				tab.bonding[id] = w
				tab.bondmu.Unlock()
				// Do the ping/pong. The result goes into w.
				tab.pingpong(w, pinged, id, addr, tcpPort)
				// Unregister the process after it's done.
				tab.bondmu.Lock()
				delete(tab.bonding, id)
				tab.bondmu.Unlock()
			}
			// Retrieve the bonding results
			result = w.err
			if result == nil {
				node = w.n
			}
		}
		if node != nil {
			// Add the node to the table even if the bonding ping/pong
			// fails. It will be relaced quickly if it continues to be
			// unresponsive.
			//이 방법은 중요합니다. 해당 버킷 공간 경우, 즉 버킷에 삽입된다. 양동이 가득합니다. 노드가 공간을 만들기 위해 시도에 핑 버킷을 테스트하는 데 사용됩니다.
			tab.add(node)
			tab.db.updateFindFails(id, 0)
		}
		return node, result
	}

탁구 방법

	func (tab *Table) pingpong(w *bondproc, pinged bool, id NodeID, addr *net.UDPAddr, tcpPort uint16) {
		// Request a bonding slot to limit network usage
		<-tab.bondslots
		defer func() { tab.bondslots <- struct{}{} }()
	
		// Ping the remote side and wait for a pong.
		//핑 원격 노드. 그리고 탁구 메시지를 기다리는
		if w.err = tab.ping(id, addr); w.err != nil {
			close(w.done)
			return
		}
		//이 핑에 해당하는 경우 메시지를 수신 UDP에 제공됩니다. 이 시간 우리는 핑 메시지의 다른 측면을 받았습니다.
		//그런 다음 우리는 핑 (ping) 다른 소식을 기다립니다. 그렇지 않으면 다른 쪽 (우리가 핑 메시지를 개시)에서 보낸 핑 메시지를 기다릴 필요가있다.
		if !pinged {
			// Give the remote node a chance to ping us before we start
			// sending findnode requests. If they still remember us,
			// waitping will simply time out.
			tab.net.waitping(id)
		}
		// Bonding succeeded, update the node database.
		//채권 프로세스를 완료합니다. 노드는 데이터베이스에 삽입됩니다. 데이터베이스 작업은 여기 다. 동작은 내부 tab.add 양동이 완료됩니다. 버킷은 메모리 집약적 인 작업이다. 데이터베이스는 지속적 씨 노드입니다. 부팅 과정을 가속화하는 데 사용됩니다.
		w.n = NewNode(id, addr.IP, uint16(addr.Port), tcpPort)
		tab.db.updateNode(w.n)
		close(w.done)
	}

tab.add 방법

	// add attempts to add the given node its corresponding bucket. If the
	// bucket has space available, adding the node succeeds immediately.
	// Otherwise, the node is added if the least recently active node in
	// the bucket does not respond to a ping packet.
	//I는 해당 버킷에 삽입 주어진 노드를 추가했습니다. 버킷이 공간이 있다면, 직접 삽입. 최근 활동의 양동이에 노드가 핑에 응답하지 않는 경우 그렇지 않으면, 우리는 그것을 대체이 노드를 사용할 수 있습니다.
	// The caller must not hold tab.mutex.
	func (tab *Table) add(new *Node) {
		b := tab.buckets[logdist(tab.self.sha, new.sha)]
		tab.mutex.Lock()
		defer tab.mutex.Unlock()
		if b.bump(new) { //노드가 존재하는 경우. 그런 다음 그 값을 업데이트합니다. 그런 다음 종료합니다.
			return
		}
		var oldest *Node
		if len(b.entries) == bucketSize {
			oldest = b.entries[bucketSize-1]
			if oldest.contested {
				// The node is already being replaced, don't attempt
				// to replace it.
				//다른 goroutine은이 노드에서 테스트중인 경우. 그래서 종료를 교체를 취소 할 수 있습니다.
				//긴 시간의 핑 때문에. 그래서 이번에는 잠겨 있지 않습니다. 국가에 의해 경쟁는 이러한 경우를 식별합니다.
				return
			}
			oldest.contested = true
			// Let go of the mutex so other goroutines can access
			// the table while we ping the least recently active node.
			tab.mutex.Unlock()
			err := tab.ping(oldest.ID, oldest.addr())
			tab.mutex.Lock()
			oldest.contested = false
			if err == nil {
				// The node responded, don't replace it.
				return
			}
		}
		added := b.replace(new, oldest)
		if added && tab.nodeAddedHook != nil {
			tab.nodeAddedHook(new)
		}
	}



물건은 비교적 간단하다. 해당 노드가 양동이에 삽입해야 찾을 수 있습니다. 버킷 전체가 아닌 경우는, 양동이를 삽입합니다. 그렇지 않으면, 아무것도하지 않습니다. 그것은 그 logdist ()이 메소드에 대해 말했다 될 필요가있다. 이것은 다음 배타적 OR의 위치에 따라 두 값의 방법과, 가장 높은 인덱스를 반환한다. 예 logdist (101,010) = 3 logdist (100, 100) = 0 logdist 옵션 (100,110) = 2

	// stuff adds nodes the table to the end of their corresponding bucket
	// if the bucket is not full. The caller must hold tab.mutex.
	func (tab *Table) stuff(nodes []*Node) {
	outer:
		for _, n := range nodes {
			if n.ID == tab.self.ID {
				continue // don't add self
			}
			bucket := tab.buckets[logdist(tab.self.sha, n.sha)]
			for i := range bucket.entries {
				if bucket.entries[i].ID == n.ID {
					continue outer // already in bucket
				}
			}
			if len(bucket.entries) < bucketSize {
				bucket.entries = append(bucket.entries, n)
				if tab.nodeAddedHook != nil {
					tab.nodeAddedHook(n)
				}
			}
		}
	}


에 대해 살펴보기 전에 조회 기능. 이 기능은 특정 노드에 대한 정보를 조회하는 데 사용됩니다. 이 함수는 먼저 로컬 노드에서 최단 거리로 모두 16 개 개의 노드를 얻을 수 있습니다. Findnode 모든 노드로 요청을 보냅니다. 그런 다음 bondall 반환 프로세스를 정의. 그리고 모든 노드를 반환합니다.



	func (tab *Table) lookup(targetID NodeID, refreshIfEmpty bool) []*Node {
		var (
			target		 = crypto.Keccak256Hash(targetID[:])
			asked		  = make(map[NodeID]bool)
			seen		   = make(map[NodeID]bool)
			reply		  = make(chan []*Node, alpha)
			pendingQueries = 0
			result		 *nodesByDistance
		)
		// don't query further if we hit ourself.
		// unlikely to happen often in practice.
		asked[tab.self.ID] = true
		자신을 묻지 마세요
		for {
			tab.mutex.Lock()
			// generate initial result set
			result = tab.closest(target, bucketSize)
			//스트라이크 16 개 노드의 최근 대상
			tab.mutex.Unlock()
			if len(result.entries) > 0 || !refreshIfEmpty {
				break
			}
			// The result set is empty, all nodes were dropped, refresh.
			// We actually wait for the refresh to complete here. The very
			// first query will hit this case and run the bootstrapping
			// logic.
			<-tab.refresh()
			refreshIfEmpty = false
		}
	
		for {
			// ask the alpha closest nodes that we haven't asked yet
			//여기서 동시 쿼리 것 (pendingQueries 파라미터에 의해 제어되는) 매 3 goroutine 동시성
			//각 반복 쿼리 결과와 가장 가까운 세 개의 노드를 대상으로합니다.
			for i := 0; i < len(result.entries) && pendingQueries < alpha; i++ {
				n := result.entries[i]
				if !asked[n.ID] { //당신이 물었다하지 않으면이 result.entries 때문에 //주기가 여러 번 반복됩니다. 그래서 이미 처리 된이 변수 컨트롤을 사용합니다.
					asked[n.ID] = true
					pendingQueries++
					go func() {
						// Find potential neighbors to bond with
						r, err := tab.net.findnode(n.ID, n.addr(), targetID)
						if err != nil {
							// Bump the failure counter to detect and evacuate non-bonded entries
							fails := tab.db.findFails(n.ID) + 1
							tab.db.updateFindFails(n.ID, fails)
							log.Trace("Bumping findnode failure counter", "id", n.ID, "failcount", fails)
	
							if fails >= maxFindnodeFailures {
								log.Trace("Too many findnode failures, dropping", "id", n.ID, "failcount", fails)
								tab.delete(n)
							}
						}
						reply <- tab.bondall(r)
					}()
				}
			}
			if pendingQueries == 0 {
				// we have asked all closest nodes, stop the search
				break
			}
			// wait for the next reply
			for _, n := range <-reply {
				if n != nil && !seen[n.ID] { //다른 원격 노드는 동일한 노드를 반환 할 수 있기 때문이다. 모든]은 [알 중복으로한다.
					seen[n.ID] = true
					//이 곳은 결과가 큐에 가입 할 결과를 알아 주목해야 할 필요가있다. 이것은 끊임없이 새로운 것을 추가 결과 노드만큼 찾을 수있는 프로세스 루프이라고. 이 사이클을 종료하지 않습니다.
					result.push(n, bucketSize)
				}
			}
			pendingQueries--
		}
		return result.entries
	}
	
	// closest returns the n nodes in the table that are closest to the
	// given id. The caller must hold tab.mutex.
	func (tab *Table) closest(target common.Hash, nresults int) *nodesByDistance {
		// This is a very wasteful way to find the closest nodes but
		// obviously correct. I believe that tree-based buckets would make
		// this easier to implement efficiently.
		close := &nodesByDistance{target: target}
		for _, b := range tab.buckets {
			for _, n := range b.entries {
				close.push(n, nresults)
			}
		}
		return close
	}

모든 노드에서 대상으로부터의 거리에 따라 정렬 될 result.push 방법. 그것은 먼 길 근처에서 순서에 따라 새 노드를 삽입하기로 결정합니다. (최대 큐 (16 개) 요소를 포함한다). 이 거리 요소 내부 대기열과 더 가까워지고 대상으로 이어질 것입니다. 상대적인 거리는 큐 쫓겨한다.
	
	// nodesByDistance is a list of nodes, ordered by
	// distance to target.
	type nodesByDistance struct {
		entries []*Node
		target  common.Hash
	}
	
	// push adds the given node to the list, keeping the total size below maxElems.
	func (h *nodesByDistance) push(n *Node, maxElems int) {
		ix := sort.Search(len(h.entries), func(i int) bool {
			return distcmp(h.target, h.entries[i].sha, n.sha) > 0
		})
		if len(h.entries) < maxElems {
			h.entries = append(h.entries, n)
		}
		if ix == len(h.entries) {
			// farther away than all nodes we already have.
			// if there was room for it, the node is now the last element.
		} else {
			// slide existing entries down to make room
			// this will overwrite the entry we just appended.
			copy(h.entries[ix+1:], h.entries[ix:])
			h.entries[ix] = n
		}
	}


### 일부 방법은 수출을 table.go
해결 및 조회 방법

	// Resolve searches for a specific node with the given ID.
	// It returns nil if the node could not be found.
	//노드 ID를 얻기 위해 사용 해결 방법은 특정된다. 로컬 노드합니다. 그런 다음 로컬 노드로 돌아갑니다. 그렇지 않으면 실행
	//네트워크에서 조회 쿼리 시간. 노드에 쿼리합니다. 그리고 돌아왔다. 그렇지 않으면 전무
	func (tab *Table) Resolve(targetID NodeID) *Node {
		// If the node is present in the local table, no
		// network interaction is required.
		hash := crypto.Keccak256Hash(targetID[:])
		tab.mutex.Lock()
		cl := tab.closest(hash, 1)
		tab.mutex.Unlock()
		if len(cl.entries) > 0 && cl.entries[0].ID == targetID {
			return cl.entries[0]
		}
		// Otherwise, do a network lookup.
		result := tab.Lookup(targetID)
		for _, n := range result {
			if n.ID == targetID {
				return n
			}
		}
		return nil
	}
	
	// Lookup performs a network search for nodes close
	// to the given target. It approaches the target by querying
	// nodes that are closer to it on each iteration.
	// The given target does not need to be an actual node
	// identifier.
	func (tab *Table) Lookup(targetID NodeID) []*Node {
		return tab.lookup(targetID, true)
	}

노드 링크 초기화를 설정 SetFallbackNodes 방법. 데이터베이스 테이블이 비어와 네트워크 연결을 도울 수있는 알려진 노드가없는에서,

	// SetFallbackNodes sets the initial points of contact. These nodes
	// are used to connect to the network if the table is empty and there
	// are no known nodes in the database.
	func (tab *Table) SetFallbackNodes(nodes []*Node) error {
		for _, n := range nodes {
			if err := n.validateComplete(); err != nil {
				return fmt.Errorf("bad bootstrap/fallback node %q (%v)", n, err)
			}
		}
		tab.mutex.Lock()
		tab.nursery = make([]*Node, 0, len(nodes))
		for _, n := range nodes {
			cpy := *n
			// Recompute cpy.sha because the node might not have been
			// created by NewNode or ParseNode.
			cpy.sha = crypto.Keccak256Hash(n.ID[:])
			tab.nursery = append(tab.nursery, &cpy)
		}
		tab.mutex.Unlock()
		tab.refresh()
		return nil
	}


### 개요

이러한 방법으로, Kademlia P2P 네트워크 프로토콜 끝에 올 것입니다. 기본적으로는 종이에 따라 구현된다. UDP 네트워크 통신. 지금까지 스토리지 노드 데이터베이스 링크. 표 Kademlia의 핵심을 구현합니다. XOR의 거리에 따라 노드를 찾을 수 있습니다. 검색 및 업데이트 프로세스 노드입니다.