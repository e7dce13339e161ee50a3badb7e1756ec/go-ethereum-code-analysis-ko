페처 동기 블록 기반 알림을 포함한다. 우리가 NewBlockHashesMsg 시간이 소식을 받았을 때, 우리는 해시 값의 많은 블록을 받았다. 당신은 블록 해시 값으로 동기화 할 필요가 있고, 다음 로컬 블록 체인을 업데이트합니다. 가져 오기는 기능을 제공합니다.


데이터 구조

	// announce is the hash notification of the availability of a new block in the
	// network.
	//네트워크가 가지고있는 적합한 새로운 블록이 나타납니다 해시 통지를 발표합니다.
	type announce struct {
		hash   common.Hash   //블록의 해시는 새로운 블록의 해시 값 // 발표되고
		number uint64	// 블록의 번호가 발표되고 (0 = 알 수없는 | 오래된 프로토콜) 블록의 높이 값,
		header *types.Header // Header of the block partially reassembled (new protocol)헤더 영역을 재 조립
		time   time.Time	 // Timestamp of the announcement
	
		origin string // Identifier of the peer originating the notification
	
		fetchHeader headerRequesterFn //가져 오기 기능은 피어에 대한 정보가 포함 된 발표 블록의 헤더의 헤더 인수 함수 포인터 영역을 검색 할 수 있습니다. 즉, 헤더 영역을 설정하는 것입니다
		fetchBodies bodyRequesterFn   //발표 된 블록의 신체 기능 포인터 취득 블록 본문을 검색 할 수 가져 오기 기능
	}
	
	// headerFilterTask represents a batch of headers needing fetcher filtering.
	type headerFilterTask struct {
		peer	string		  // The source peer of block headers
		headers []*types.Header // Collection of headers to filter
		time	time.Time	   // Arrival time of the headers
	}
	
	// headerFilterTask represents a batch of block bodies (transactions and uncles)
	// needing fetcher filtering.
	type bodyFilterTask struct {
		peer		 string				 // The source peer of block bodies
		transactions [][]*types.Transaction // Collection of transactions per block bodies
		uncles	   [][]*types.Header	  // Collection of uncles per block bodies
		time		 time.Time			  // Arrival time of the blocks' contents
	}
	
	// inject represents a schedules import operation. 
	//노드가 메시지를 수신하면 NewBlockMsg 블록이 삽입 될 때
	type inject struct {
		origin string
		block  *types.Block
	}
	
	// Fetcher is responsible for accumulating block announcements from various peers
	// and scheduling them for retrieval.
	type Fetcher struct {
		// Various event channels
		notify chan *announce채널을 발표 //,
		inject chan *inject	채널 주입 //
	
		blockFilter  chan chan []*types.Block // 채널 채널?
		headerFilter chan chan *headerFilterTask
		bodyFilter   chan chan *bodyFilterTask
	
		done chan common.Hash
		quit chan struct{}
	
		// Announce states
		announces  map[string]int		  // 당 메모리 고갈 키를 방지하기 위해 카운트를 발표 피어하는 피어 이름이고, 값은 큰 메모리 풋 프린트를 방지하기 위해, 카운트를 발표합니다.
		announced  map[common.Hash][]*announce //대기 가져 오는 예정 발표 블록, 발표의 인출을 예약 할 수 있습니다
		fetching   map[common.Hash]*announce   //발표 블록은, 현재 발표 페치되고 가져 오는
		fetched헤더 맵 [common.Hash] * 발표 // 블록은 블록 체 취득을 대기 이미 헤더 영역을 취득 // 본체 검색 스케줄, 페치
		completing map[common.Hash]*announce   //헤더 블록은 현재 몸 완료 머리와 몸을 얻을 //이 발표 완료된
	
		// Block cache
		queue  *prque.Prque		// 큐 임포트 조작 (구성에있어서, 블록 번호) 반입 동작을 (블록 번호 정렬) // 큐 포함 함유
		queues map[string]int	  피어 블록 카운트 당 // 메모리 고갈 키 피어 방지하기 위해, 값은 블록의 수이다. 너무 많은 메모리를 사용하지 마십시오.
		queued map[common.Hash]*inject //이미 대기중인 블록 (수입 DEDUP하는) 세트 블록 큐에 넣어왔다. 무거운 이동합니다.
	
		//콜백의 수에 콜백 의존.
		getBlock	   blockRetrievalFn   // Retrieves a block from the local chain
		verifyHeader   headerVerifierFn   // Checks if a block's headers have a valid proof of work
		broadcastBlock blockBroadcasterFn // Broadcasts a block to connected peers
		chainHeight	chainHeightFn	  // Retrieves the current chain's height
		insertChain	chainInsertFn	  // Injects a batch of blocks into the chain
		dropPeer	   peerDropFn		 // Drops a peer for misbehaving
	
		//테스트 목적으로 만 테스트 후크.
		announceChangeHook func(common.Hash, bool) // Method to call upon adding or deleting a hash from the announce list
		queueChangeHook	func(common.Hash, bool) // Method to call upon adding or deleting a block from the import queue
		fetchingHook	   func([]common.Hash)	 // Method to call upon starting a block (eth/61) or header (eth/62) fetch
		completingHook	 func([]common.Hash)	 // Method to call upon starting a block body fetch (eth/62)
		importedHook	   func(*types.Block)	  // Method to call upon successful block import (both eth/61 and eth/62)
	}

가져 오기를 시작 다루는 직접 goroutine을 시작했다. 이 기능은 약간 깁니다. 이후 다시 분석.

	// Start boots up the announcement based synchroniser, accepting and processing
	// hash notifications and block fetches until termination requested.
	func (f *Fetcher) Start() {
		go f.loop()
	}


루프 기능 기능이 너무 오래. 나는 버전을 생략 게시 할 예정입니다. 가져 오기는 (대기 가져 오기 가져 오기, 몸을 인출 완료 헤더 기다릴 가져 오기, 가져 오기 완료입니다)를 발표 상태를 기록 사지도 (발표, 반입, 반입, 완료)에 의해. 사실, 루프는 내부 ​​타이머에 의해 다양한 상태 전환을지도하고 다양한 메시지를 발표합니다.


	// Loop is the main fetcher loop, checking and processing various notification
	// events.
	func (f *Fetcher) loop() {
		// Iterate the block fetching until a quit is requested
		fetchTimer := time.NewTimer(0)  //타이머를 가져옵니다.
		completeTimer := time.NewTimer(0) //compelte 타이머.
	
		for {
			// Clean up any expired block fetches
			//오초을 통해 가져 오는 시간은 다음이 가져 오는을 포기하는 경우
			for hash, announce := range f.fetching {
				if time.Since(announce.time) > fetchTimeout {
					f.forgetHash(hash)
				}
			}
			// Import any queued blocks that could potentially fit
			//캐시 블록을 페치이 fetcher.queue는 순서 대기 로컬 삽입 블록 사슬을 완료
			//fetcher.queue는 우선 순위 큐이다. 우선 순위는 블록 제외 수 있으므로 상단 블록 소수이다.
			height := f.chainHeight()
			for !f.queue.Empty() { // 
				op := f.queue.PopItem().(*inject)
				if f.queueChangeHook != nil {
					f.queueChangeHook(op.block.Hash(), false)
				}
				// If too high up the chain or phase, continue later
				number := op.block.NumberU64()
				if number > height+1 { //현재 블록의 높이가 너무 높으면, 당신은 가져올 수 없습니다
					f.queue.Push(op, -float32(op.block.NumberU64()))
					if f.queueChangeHook != nil {
						f.queueChangeHook(op.block.Hash(), true)
					}
					break
				}
				// Otherwise if fresh and still unknown, try and import
				hash := op.block.Hash()
				if number+maxUncleDist < height || f.getBlock(hash) != nil {
					//블록의 높이가 현재 높이 maxUncleDist 아래 너무 낮
					//또는 블록은 수입하고있다
					f.forgetBlock(hash)
					continue
				}
				//삽입 블록
				f.insert(op.origin, op.block)
			}
			// Wait for an outside event to occur
			select {
			case <-f.quit:
				// Fetcher terminating, abort all operations
				return
	
			case notification := <-f.notify: //언제 수신 NewBlockHashesMsg, 페처 채널을 통지하는 송신 방법 호출 알림 막지 한 해시 값에 대한 로컬 블록 사슬.
				...
	
			case op := <-f.inject: //호출이 NewBlockMsg 페처의 엔큐 방법을 수신하면,이 방법은 채널을 주입 현재 수신 된 블록을 전송한다.
				...
				f.enqueue(op.origin, op.block)
	
			case hash := <-f.done: //블록이 블록 다 채널의 해시 값에 전송 될 때 가져 오기가 완료되면.
				...
	
			case <-fetchTimer.C: //fetchTimer 타이머는 정기적으로 가져 오기 수행 헤더 영역을 가져올 필요
				...
	
			case <-completeTimer.C: //completeTimer 타이머는 주기적으로 블록 몸을 가져 오기 가져 오기 위해 필요
				...
	
			case filter := <-f.headerFilter: //메시지가 BlockHeadersMsg (일부받은 헤더 영역)을 수신되면, 큐 headerFilter에 이러한 메시지를 전달합니다. 사이드는 다른 시스템에 다른를 반환합니다, 왼쪽 요청 된 데이터 가져 오기에 속하는 것입니다.
				...
	
			case filter := <-f.bodyFilter: //BlockBodiesMsg이 메시지를 받았을 때, 이러한 메시지는 큐에 전달 bodyFilter됩니다. 사이드는 다른 시스템에 다른를 반환합니다, 왼쪽 요청 된 데이터 가져 오기에 속하는 것입니다.
				...
			}
		}
	}

### 여과 공정의 헤더 영역
#### FilterHeaders 요청
BlockHeadersMsg 받았을 때 FilterHeaders 메서드가 호출됩니다. headerFilter에 채널 필터의이 방법은 제 배달. 그런 다음 headerFilterTask 작업을 제공하기 위해 필터링 할 수 있습니다. 그런 다음 리턴 메시지 큐 필터를 기다리는 차단.


	// FilterHeaders extracts all the headers that were explicitly requested by the fetcher,
	// returning those that should be handled differently.
	func (f *Fetcher) FilterHeaders(peer string, headers []*types.Header, time time.Time) []*types.Header {
		log.Trace("Filtering headers", "peer", peer, "headers", len(headers))
	
		// Send the filter channel to the fetcher
		filter := make(chan *headerFilterTask)
	
		select {
		case f.headerFilter <- filter:
		case <-f.quit:
			return nil
		}
		// Request the filtering of the header list
		select {
		case filter <- &headerFilterTask{peer: peer, headers: headers, time: time}:
		case <-f.quit:
			return nil
		}
		// Retrieve the headers remaining after filtering
		select {
		case task := <-filter:
			return task.headers
		case <-f.quit:
			return nil
		}
	}


#### headerFilter 처리
goroutine이 처리 루프 (12).

	case filter := <-f.headerFilter:
				// Headers arrived from a remote peer. Extract those that were explicitly
				// requested by the fetcher, and return everything else so it's delivered
				// to other parts of the system.
				var task *headerFilterTask
				select {
				case task = <-filter:
				case <-f.quit:
					return
				}
				headerFilterInMeter.Mark(int64(len(task.headers)))
	
				// Split the batch of headers into unknown ones (to return to the caller),
				// known incomplete ones (requiring body retrievals) and completed blocks.
				unknown, incomplete, complete := []*types.Header{}, []*announce{}, []*types.Block{}
				for _, header := range task.headers {
					hash := header.Hash()
	
					// Filter fetcher-requested headers from other synchronisation algorithms
					//이것은 우리가 요청하는 정보가 반환됩니다 있는지 확인하기 위해 상황에 따라.
					if announce := f.fetching[hash]; announce != nil && announce.origin == task.peer && f.fetched[hash] == nil && f.completing[hash] == nil && f.queued[hash] == nil {
						// If the delivered header does not match the promised number, drop the announcer
						//헤더 다른 높이의 블록과 우리의 요청을 반환하면, 반환 피어의 헤더를 삭제합니다. 그리고 (블록 정보를 재 취득하기 위해)이 해시에서 잊지
						if header.Number.Uint64() != announce.number {
							log.Trace("Invalid block number fetched", "peer", announce.origin, "hash", header.Hash(), "announced", announce.number, "provided", header.Number)
							f.dropPeer(announce.origin)
							f.forgetHash(hash)
							continue
						}
						// Only keep if not imported by other means
						if f.getBlock(hash) == nil {
							announce.header = header
							announce.time = task.time
	
							// If the block is empty (header only), short circuit into the final import queue
							//이 블록은 어떤 거래 나 삼촌 블록을 포함하지 않는 경우,보기 영역 헤더에 따르면. 그래서 우리는 몸 블록을하지 않습니다. 그런 다음 목록에 직접 삽입.
							if header.TxHash == types.DeriveSha(types.Transactions{}) && header.UncleHash == types.CalcUncleHash([]*types.Header{}) {
								log.Trace("Block empty, skipping body retrieval", "peer", announce.origin, "number", header.Number, "hash", header.Hash())
	
								block := types.NewBlockWithHeader(header)
								block.ReceivedAt = task.time
	
								complete = append(complete, block)
								f.completing[hash] = announce
								continue
							}
							// Otherwise add to the list of blocks needing completion
							//그렇지 않으면, 미완성 대기자 명단에 삽입하는 것은 blockbody 가져
							incomplete = append(incomplete, announce)
						} else {
							log.Trace("Block already imported, discarding header", "peer", announce.origin, "number", header.Number, "hash", header.Hash())
							f.forgetHash(hash)
						}
					} else {
						// Fetcher doesn't know about it, add to the return list
						//가져 오기는이 헤더를 알 수 없습니다. 증가 반환 반환을 기다리는 목록을 표시합니다.
						unknown = append(unknown, header)
					}
				}
				headerFilterOutMeter.Mark(int64(len(unknown)))
				select {
				//반환 된 결과가 반환됩니다.
				case filter <- &headerFilterTask{headers: unknown, time: task.time}:
				case <-f.quit:
					return
				}
				// Schedule the retrieved headers for body completion
				for _, announce := range incomplete {
					hash := announce.header.Hash()
					if _, ok := f.completing[hash]; ok { //당신은 완료하는 데 다른 곳에있는 경우
						continue
					}
					//기다리고 대기하는 본체를 얻기 위해지도로 처리한다.
					f.fetched[hash] = append(f.fetched[hash], announce)
					if len(f.fetched) == 1 { //요소가 맵을 가져온 경우 단지에 합류했다. 그런 다음 타이머를 다시 설정합니다.
						f.rescheduleComplete(completeTimer)
					}
				}
				// Schedule the header-only blocks for import
				//수입을 기다리는 큐에 유일한 헤더 블록
				for _, block := range complete {
					if announce := f.completing[block.Hash()]; announce != nil {
						f.enqueue(announce.origin, block)
					}
				}


#### bodyFilter 처리
그리고, 상기와 마찬가지의 처리.

		case filter := <-f.bodyFilter:
			// Block bodies arrived, extract any explicitly requested blocks, return the rest
			var task *bodyFilterTask
			select {
			case task = <-filter:
			case <-f.quit:
				return
			}
			bodyFilterInMeter.Mark(int64(len(task.transactions)))

			blocks := []*types.Block{}
			for i := 0; i < len(task.transactions) && i < len(task.uncles); i++ {
				// Match up a body to any possible completion request
				matched := false

				for hash, announce := range f.completing {
					if f.queued[hash] == nil {
						txnHash := types.DeriveSha(types.Transactions(task.transactions[i]))
						uncleHash := types.CalcUncleHash(task.uncles[i])

						if txnHash == announce.header.TxHash && uncleHash == announce.header.UncleHash && announce.origin == task.peer {
							// Mark the body matched, reassemble if still unknown
							matched = true
							
							if f.getBlock(hash) == nil {
								block := types.NewBlockWithHeader(announce.header).WithBody(task.transactions[i], task.uncles[i])
								block.ReceivedAt = task.time

								blocks = append(blocks, block)
							} else {
								f.forgetHash(hash)
							}
						}
					}
				}
				if matched {
					task.transactions = append(task.transactions[:i], task.transactions[i+1:]...)
					task.uncles = append(task.uncles[:i], task.uncles[i+1:]...)
					i--
					continue
				}
			}

			bodyFilterOutMeter.Mark(int64(len(task.transactions)))
			select {
			case filter <- task:
			case <-f.quit:
				return
			}
			// Schedule the retrieved blocks for ordered import
			for _, block := range blocks {
				if announce := f.completing[block.Hash()]; announce != nil {
					f.enqueue(announce.origin, block)
				}
			}

#### 통지 프로세스
언제 수신 NewBlockHashesMsg, 페처 채널을 통지하는 송신 방법 호출 알림 막지 한 해시 값에 대한 로컬 블록 사슬.


	// Notify announces the fetcher of the potential availability of a new block in
	// the network.
	func (f *Fetcher) Notify(peer string, hash common.Hash, number uint64, time time.Time,
		headerFetcher headerRequesterFn, bodyFetcher bodyRequesterFn) error {
		block := &announce{
			hash:		hash,
			number:	  number,
			time:		time,
			origin:	  peer,
			fetchHeader: headerFetcher,
			fetchBodies: bodyFetcher,
		}
		select {
		case f.notify <- block:
			return nil
		case <-f.quit:
			return errTerminated
		}
	}

루프 처리에있어서, 주로 용기가 발표 후 웨이팅 타이머 처리를 가하고 확인.

	case notification := <-f.notify:
			// A block was announced, make sure the peer isn't DOSing us
			propAnnounceInMeter.Mark(1)

			count := f.announces[notification.origin] + 1
			if count > hashLimit {  //이 기껏 hashLimit 256 만 원격 256 발표
				log.Debug("Peer exceeded outstanding announces", "peer", notification.origin, "limit", hashLimit)
				propAnnounceDOSMeter.Mark(1)
				break
			}
			// If we have a valid block number, check that it's potentially useful
			//잠재적 유용성을 확인하십시오. 지역 거리 블록 번호와 블록 체인에 따르면, 너무 작은 우리를 위해 이해가되지 않았다.
			if notification.number > 0 {
				if dist := int64(notification.number) - int64(f.chainHeight()); dist < -maxUncleDist || dist > maxQueueDist {
					log.Debug("Peer discarded announcement", "peer", notification.origin, "number", notification.number, "hash", notification.hash, "distance", dist)
					propAnnounceDropMeter.Mark(1)
					break
				}
			}
			// All is well, schedule the announce if block's not yet downloading
			//우리가 존재 여부를 확인합니다.
			if _, ok := f.fetching[notification.hash]; ok {
				break
			}
			if _, ok := f.completing[notification.hash]; ok {
				break
			}
			f.announces[notification.origin] = count
			f.announced[notification.hash] = append(f.announced[notification.hash], notification)
			if f.announceChangeHook != nil && len(f.announced[notification.hash]) == 1 {
				f.announceChangeHook(notification.hash, true)
			}
			if len(f.announced) == 1 {
				f.rescheduleFetch(fetchTimer)
			}

#### 인큐 과정
호출이 NewBlockMsg 페처의 엔큐 방법을 수신하면,이 방법은 채널을 주입 현재 수신 된 블록을 전송한다. 이 오브젝트를 생성하는 방법은 다음 분사 주입 채널로 전송되는 것을 알 수있다
	
	// Enqueue tries to fill gaps the the fetcher's future import queue.
	func (f *Fetcher) Enqueue(peer string, block *types.Block) error {
		op := &inject{
			origin: peer,
			block:  block,
		}
		select {
		case f.inject <- op:
			return nil
		case <-f.quit:
			return errTerminated
		}
	}

주입 채널 프로세싱, 그것은 수입 큐에 직접 첨가 매우 간단하다

	case op := <-f.inject:
			// A direct block insertion was requested, try and fill any pending gaps
			propBroadcastInMeter.Mark(1)
			f.enqueue(op.origin, op.block)

enqueue

	// enqueue schedules a new future import operation, if the block to be imported
	// has not yet been seen.
	func (f *Fetcher) enqueue(peer string, block *types.Block) {
		hash := block.Hash()
	
		// Ensure the peer isn't DOSing us
		count := f.queues[peer] + 1
		서로의 캐시가 너무 많은 경우 계산&gt; blockLimit {블록은 64 blockLimit합니다.
			log.Debug("Discarded propagated block, exceeded allowance", "peer", peer, "number", block.Number(), "hash", hash, "limit", blockLimit)
			propBroadcastDOSMeter.Mark(1)
			f.forgetHash(hash)
			return
		}
		// Discard any past or too distant blocks
		//우리는 너무 멀리 블록 체인에서입니다.
		if dist := int64(block.NumberU64()) - int64(f.chainHeight()); dist < -maxUncleDist || dist > maxQueueDist { 
			log.Debug("Discarded propagated block, too far away", "peer", peer, "number", block.Number(), "hash", hash, "distance", dist)
			propBroadcastDropMeter.Mark(1)
			f.forgetHash(hash)
			return
		}
		// Schedule the block for future importing
		//큐에 삽입.
		if _, ok := f.queued[hash]; !ok {
			op := &inject{
				origin: peer,
				block:  block,
			}
			f.queues[peer] = count
			f.queued[hash] = op
			f.queue.Push(op, -float32(block.NumberU64()))
			if f.queueChangeHook != nil {
				f.queueChangeHook(op.block.Hash(), true)
			}
			log.Debug("Queued propagated block", "peer", peer, "number", block.Number(), "hash", hash, "queued", f.queue.Size())
		}
	}

#### 타이머 처리
모두 2 개 개의 타이머가있다. fetchTimer 및 completeTimer는 블록 본체에 헤더 영역에 액세스를 획득 할 책임이있다.

(headerFilter가) - -&gt; 가져 --completeTimer (몸을 가져 오기) -&gt; 완료 - (bodyFilter) -&gt; 대기열 --task.done- 상태 전이는 --fetchTimer (가져 오기 헤더) ---&gt; 가져 오는 발표 -&gt; forgetHash

나는 문제를 발견했다. 컨테이너 가능한 누출을 완료. 당신은 요청 본문의 해시를 보내는 경우. 요청이 실패하지만, 다른 하나는 반환하지 않았습니다. 이번에 완료 한 용기가없는 청소. 이 문제를 일으킬 수 있습니다.

		case <-fetchTimer.C:
			// At least one block's timer ran out, check for needing retrieval
			request := make(map[string][]common.Hash)

			for hash, announces := range f.announced {
				//여기 TODO 시간 제한은 무엇을 의미입니다
				//초기에는 오랜 시간이 발표 받고, arriveTimeout - gatherSlack 후.
				if time.Since(announces[0].time) > arriveTimeout-gatherSlack {
					// Pick a random peer to retrieve from, reset all others
					//복수의 같은 블록을 대신하여 발표하는 것은 여러 피어에서 발표
					announce := announces[rand.Intn(len(announces))]
					f.forgetHash(hash)

					// If the block still didn't arrive, queue for fetching
					if f.getBlock(hash) == nil {
						request[announce.origin] = append(request[announce.origin], hash)
						f.fetching[hash] = announce
					}
				}
			}
			// Send out all block header requests
			//모든 요청을 보내기.
			for peer, hashes := range request {
				log.Trace("Fetching scheduled headers", "peer", peer, "list", hashes)

				// Create a closure of the fetch and schedule in on a new thread
				fetchHeader, hashes := f.fetching[hashes[0]].fetchHeader, hashes
				go func() {
					if f.fetchingHook != nil {
						f.fetchingHook(hashes)
					}
					for _, hash := range hashes {
						headerFetchMeter.Mark(1)
						fetchHeader(hash) // Suboptimal, but protocol doesn't allow batch header retrievals
					}
				}()
			}
			// Schedule the next fetch if blocks are still pending
			f.rescheduleFetch(fetchTimer)

		case <-completeTimer.C:
			// At least one header's timer ran out, retrieve everything
			request := make(map[string][]common.Hash)

			for hash, announces := range f.fetched {
				// Pick a random peer to retrieve from, reset all others
				announce := announces[rand.Intn(len(announces))]
				f.forgetHash(hash)

				// If the block still didn't arrive, queue for completion
				if f.getBlock(hash) == nil {
					request[announce.origin] = append(request[announce.origin], hash)
					f.completing[hash] = announce
				}
			}
			// Send out all block body requests
			for peer, hashes := range request {
				log.Trace("Fetching scheduled bodies", "peer", peer, "list", hashes)

				// Create a closure of the fetch and schedule in on a new thread
				if f.completingHook != nil {
					f.completingHook(hashes)
				}
				bodyFetchMeter.Mark(int64(len(hashes)))
				go f.completing[hashes[0]].fetchBodies(hashes)
			}
			// Schedule the next fetch if blocks are still pending
			f.rescheduleComplete(completeTimer)



#### 일부 다른 방법

가져 오기 삽입하는 방법. 이 방법은 로컬 블록 사슬로 지정된 블록을 추가한다.

	// insert spawns a new goroutine to run a block insertion into the chain. If the
	// block's number is at the same height as the current import phase, if updates
	// the phase states accordingly.
	func (f *Fetcher) insert(peer string, block *types.Block) {
		hash := block.Hash()
	
		// Run the import on a new thread
		log.Debug("Importing propagated block", "peer", peer, "number", block.Number(), "hash", hash)
		go func() {
			defer func() { f.done <- hash }()
	
			// If the parent's unknown, abort insertion
			parent := f.getBlock(block.ParentHash())
			if parent == nil {
				log.Debug("Unknown parent of propagated block", "peer", peer, "number", block.Number(), "hash", hash, "parent", block.ParentHash())
				return
			}
			// Quickly validate the header and propagate the block if it passes
			//헤더 영역이 확인되면, 즉시 방송을 차단합니다. NewBlockMsg
			switch err := f.verifyHeader(block.Header()); err {
			case nil:
				// All ok, quickly propagate to our peers
				propBroadcastOutTimer.UpdateSince(block.ReceivedAt)
				go f.broadcastBlock(block, true)
	
			case consensus.ErrFutureBlock:
				// Weird future block, don't fail, but neither propagate
	
			default:
				// Something went very wrong, drop the peer
				log.Debug("Propagated block verification failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
				f.dropPeer(peer)
				return
			}
			// Run the actual import and log any issues
			if _, err := f.insertChain(types.Blocks{block}); err != nil {
				log.Debug("Propagated block import failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
				return
			}
			// If import succeeded, broadcast the block
			//삽입에 성공, 다음 방송 블록 인 경우, 두 번째 인수는 false입니다. 만 블록 방송을 해시합니다. NewBlockHashesMsg
			propAnnounceOutTimer.UpdateSince(block.ReceivedAt)
			go f.broadcastBlock(block, false)
	
			// Invoke the testing hook if needed
			if f.importedHook != nil {
				f.importedHook(block)
			}
		}()
	}