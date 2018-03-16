다운 블록 사슬이 동기의 시작 주로 담당하고, 현재의 동기화는 두 개의 모드를 가지고, 하나에서,이 모델은 블록 사슬 다운로드 영역 헤더로 구성되는 기존 fullmode, 상기 블록 체 동기화 프로세스이며 헤더 영역의 인증을 포함한 통상의 블록 삽입 과정과 마찬가지로, 트랜잭션, 트랜잭션 실행을 검증하고 다른 연산의 계정 상태 변경이 실제의 처리 인 CPU 집약적 디스크. 또 다른 모델은이 모델을 설명하기 위해 특별한 문서를 가지고, 빠른 동기화 모드 빠른 동기화입니다. 빠른 동기화 문서를 참조하십시오. 간단히 말해, 빠른 동기화 모드는 블록 높이 후 거래를 실행하지 않는 영역 헤더 블록 몸과 영수증 삽입 과정을 다운로드합니다 (가장 높은 블록 높이 - 1024) 뒤에 때 상태 모든 계정을 동기화 1024 개 블록 fullmode 방법을 사용하여 구성됩니다. 계좌 내역 많은 정보를 생성하지 않으면 서이 모델은 블록 삽입 시간을 추가합니다. 그것은 상대적으로 디스크에 저장되지만, 네트워크의 소비를 위해 더 높을 것이다. 영수증 및 상태를 다운로드 할 필요가 있기 때문에.

## 다운 데이터 구조

	
	type Downloader struct {
		mode SyncMode	   // Synchronisation mode defining the strategy used (per sync cycle)
		mux  *event.TypeMux // Event multiplexer to announce sync operation events
		//다운로드 큐 오브젝트 다운로드 후 지역 헤더, 트랜잭션 영수증뿐만 아니라 조립을 전달하는 데 사용
		queue   *queue   // Scheduler for selecting the hashes to download
		//최종 세트
		peers   *peerSet // Set of active peers from which download can proceed
		stateDB ethdb.Database
		//머리에 피벗 포인트 빠른 동기화 블록
		fsPivotLock  *types.Header // Pivot header on critical section entry (cannot change between retries)
		fsPivotFails uint32		// Number of subsequent fast sync failures in the critical section
		//왕복 지연 다운로드
		rttEstimate   uint64 // Round trip time to target for download requests
		rttConfidence uint64 //추정 된 RTT에 대한 신뢰 : (: 백만 원자 작업을 할 수 있습니다 단위) (단위 원자 작전을 허용하는 백만는) RTT의 신뢰를 추정
	
		//통계 통계,
		syncStatsChainOrigin uint64 // Origin block number where syncing started at
		syncStatsChainHeight uint64 // Highest block number known when syncing started
		syncStatsState	   stateSyncStats
		syncStatsLock		sync.RWMutex // Lock protecting the sync stats fields
	
		lightchain LightChain
		blockchain BlockChain
	
		// Callbacks
		dropPeer peerDropFn // Drops a peer for misbehaving
	
		// Status
		synchroniseMock func(id string, hash common.Hash) error // Replacement for synchronise during testing
		synchronising   int32
		notified		int32
	
		// Channels
		headerCh	  chan dataPack	네트워크로부터 다운로드 헤더 입력 채널의 수신 블록 헤더 헤더를 수신하는 채널은이 채널에 전송한다 // [/ ETH 62]
		bodyCh		chan dataPack	// [ETH는 / 62] 수신 채널 블록 체 체 입력 채널을 수신하는 네트워크로부터 다운로드이 채널 바디에 전송 될
		receiptCh	 chan dataPack		// [eth/63] Channel receiving inbound receipts   네트워크로부터 다운로드 영수증 영수증 입력 채널이 채널로 전송 될
		bodyWakeCh	chan bool			// [eth/62] Channel to signal the block body fetcher of new tasks새 작업 가져 오기위한 채널 전송 본체
		receiptWakeCh chan bool			// [eth/63] Channel to signal the receipt fetcher of new tasks   새 작업 가져 오기 접수에 대한 전송 채널
		headerProcCh  chan []*types.Header // [eth/62] Channel to feed the header processor new tasks	채널 헤더 처리기에 대한 새 작업을 제공합니다
	
		// for stateFetcher
		stateSyncStart chan *stateSync   //그것은 새로운 상태 가져 오기를 시작하는 데 사용
		trackStateReq  chan *stateReq	 // TODO
		stateCh		chan dataPack // [eth/63] Channel receiving inbound node state data	   state的输入通道，从网络下载的state会被送到这个通道			
		// Cancellation and termination
		cancelPeer string		// Identifier of the peer currently being used as the master (cancel on drop)
		cancelCh   chan struct{} // Channel to cancel mid-flight syncs
		cancelLock sync.RWMutex  // Lock to protect the cancel channel and peer in delivers
	
		quitCh   chan struct{} // Quit channel to signal termination
		quitLock sync.RWMutex  // Lock to prevent double closes
	
		// Testing hooks
		syncInitHook	 func(uint64, uint64)  // Method to call upon initiating a new sync run
		bodyFetchHook	func([]*types.Header) // Method to call upon starting a block body fetch
		receiptFetchHook func([]*types.Header) // Method to call upon starting a receipt fetch
		chainInsertHook  func([]*fetchResult)  // Method to call upon inserting a chain of blocks (possibly in multiple invocations)
	}


생성자
	
	
	// New creates a new downloader to fetch hashes and blocks from remote peers.
	func New(mode SyncMode, stateDb ethdb.Database, mux *event.TypeMux, chain BlockChain, lightchain LightChain, dropPeer peerDropFn) *Downloader {
		if lightchain == nil {
			lightchain = chain
		}
		dl := &Downloader{
			mode:		   mode,
			stateDB:		stateDb,
			mux:			mux,
			queue:		  newQueue(),
			peers:		  newPeerSet(),
			rttEstimate:	uint64(rttMaxEstimate),
			rttConfidence:  uint64(1000000),
			blockchain:	 chain,
			lightchain:	 lightchain,
			dropPeer:	   dropPeer,
			headerCh:	   make(chan dataPack, 1),
			bodyCh:		 make(chan dataPack, 1),
			receiptCh:	  make(chan dataPack, 1),
			bodyWakeCh:	 make(chan bool, 1),
			receiptWakeCh:  make(chan bool, 1),
			headerProcCh:   make(chan []*types.Header, 1),
			quitCh:		 make(chan struct{}),
			stateCh:		make(chan dataPack),
			stateSyncStart: make(chan *stateSync),
			trackStateReq:  make(chan *stateReq),
		}
		go dl.qosTuner()  //간단하고 주로 rttEstimate rttConfidence을 계산하는 데 사용
		go dl.stateFetcher() //stateFetcher 작업 모니터를 시작하지만, 이번에는 상태 가져 오기 작업을 생성하지.
		return dl
	}

## 동기화 다운로드
동기화는 동기화 프로세스는 약간의 오차가 발생하는 경우가 피어 삭제됩니다 피어 동기화를 시도했으나. 그런 다음이 시도됩니다.

	// Synchronise tries to sync up our local block chain with a remote peer, both
	// adding various sanity checks as well as wrapping it with various log entries.
	func (d *Downloader) Synchronise(id string, head common.Hash, td *big.Int, mode SyncMode) error {
		err := d.synchronise(id, head, td, mode)
		switch err {
		case nil:
		case errBusy:
	
		case errTimeout, errBadPeer, errStallingPeer,
			errEmptyHeaderSet, errPeersUnavailable, errTooOld,
			errInvalidAncestor, errInvalidChain:
			log.Warn("Synchronisation failed, dropping peer", "peer", id, "err", err)
			d.dropPeer(id)
	
		default:
			log.Warn("Synchronisation failed, retrying", "err", err)
		}
		return err
	}


synchronise
	
	// synchronise will select the peer and use it for synchronising. If an empty string is given
	// it will use the best peer possible and synchronize if it's TD is higher than our own. If any of the
	// checks fail an error will be returned. This method is synchronous
	func (d *Downloader) synchronise(id string, hash common.Hash, td *big.Int, mode SyncMode) error {
		// Mock out the synchronisation if testing
		if d.synchroniseMock != nil {
			return d.synchroniseMock(id, hash)
		}
		// Make sure only one goroutine is ever allowed past this point at once
		//만 검사를 실행할 수있는이 방법은 실행 중입니다.
		if !atomic.CompareAndSwapInt32(&d.synchronising, 0, 1) {
			return errBusy
		}
		defer atomic.StoreInt32(&d.synchronising, 0)
	
		// Post a user notification of the sync (only once per session)
		if atomic.CompareAndSwapInt32(&d.notified, 0, 1) {
			log.Info("Block synchronisation started")
		}
		// Reset the queue, peer set and wake channels to clean any internal leftover state
		//상태 큐와 피어를 재설정합니다.
		d.queue.Reset()
		d.peers.Reset()
		//빈 d.bodyWakeCh, d.receiptWakeCh
		for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
			select {
			case <-ch:
			default:
			}
		}
		//빈 d.headerCh, d.bodyCh, d.receiptCh
		for _, ch := range []chan dataPack{d.headerCh, d.bodyCh, d.receiptCh} {
			for empty := false; !empty; {
				select {
				case <-ch:
				default:
					empty = true
				}
			}
		}
		//빈 headerProcCh
		for empty := false; !empty; {
			select {
			case <-d.headerProcCh:
			default:
				empty = true
			}
		}
		// Create cancel channel for aborting mid-flight and mark the master peer
		d.cancelLock.Lock()
		d.cancelCh = make(chan struct{})
		d.cancelPeer = id
		d.cancelLock.Unlock()
	
		defer d.Cancel() // No matter what, we can't leave the cancel channel open
	
		// Set the requested sync mode, unless it's forbidden
		d.mode = mode
		if d.mode == FastSync && atomic.LoadUint32(&d.fsPivotFails) >= fsCriticalTrials {
			d.mode = FullSync
		}
		// Retrieve the origin peer and initiate the downloading process
		p := d.peers.Peer(id)
		if p == nil {
			return errUnknownPeer
		}
		return d.syncWithPeer(p, hash, td)
	}

syncWithPeer

	// syncWithPeer starts a block synchronization based on the hash chain from the
	// specified peer and head hash.
	func (d *Downloader) syncWithPeer(p *peerConnection, hash common.Hash, td *big.Int) (err error) {
		...
		// Look up the sync boundaries: the common ancestor and the target block
		//그것은 헤더 영역을 얻기 위해 해시를 사용하여 네트워크에 액세스하는 방법을 말한다
		latest, err := d.fetchHeight(p)
		if err != nil {
			return err
		}
		height := latest.Number.Uint64()
		//findAncestor 동기화를 시작하는 지점을 찾기 위해, 우리의 공통 조상을 시도.
		origin, err := d.findAncestor(p, height)
		if err != nil {
			return err
		}
		d.syncStatsLock.Lock()
		if d.syncStatsChainHeight <= origin || d.syncStatsChainOrigin > origin {
			d.syncStatsChainOrigin = origin
		}
		d.syncStatsChainHeight = height
		d.syncStatsLock.Unlock()
	
		// Initiate the sync using a concurrent header and content retrieval algorithm
		pivot := uint64(0)
		switch d.mode {
		case LightSync:
			pivot = height
		case FastSync:
			// Calculate the new fast/slow sync pivot point
			//경우 피벗 점은 잠겨 있지 않습니다.
			if d.fsPivotLock == nil {
				pivotOffset, err := rand.Int(rand.Reader, big.NewInt(int64(fsPivotInterval)))
				if err != nil {
					panic(fmt.Sprintf("Failed to access crypto random source: %v", err))
				}
				if height > uint64(fsMinFullBlocks)+pivotOffset.Uint64() {
					pivot = height - uint64(fsMinFullBlocks) - pivotOffset.Uint64()
				}
			} else { //이 시점으로 이미 잠겨 있었다. 그런 다음이 포인트를 사용
				// Pivot point locked in, use this and do not pick a new one!
				pivot = d.fsPivotLock.Number.Uint64()
			}
			// If the point is below the origin, move origin back to ensure state download
			if pivot < origin {
				if pivot > 0 {
					origin = pivot - 1
				} else {
					origin = 0
				}
			}
			log.Debug("Fast syncing until pivot block", "pivot", pivot)
		}
		d.queue.Prepare(origin+1, d.mode, pivot, latest)
		if d.syncInitHook != nil {
			d.syncInitHook(origin, height)
		}
		//헤더, 몸, 영수증, 처리 헤더에 대한 책임이 몇 가져 오기 시작
		fetchers := []func() error{
			func() error { return d.fetchHeaders(p, origin+1) }, // Headers are always retrieved
			func() error { return d.fetchBodies(origin + 1) },   // Bodies are retrieved during normal and fast sync
			func() error { return d.fetchReceipts(origin + 1) }, // Receipts are retrieved during fast sync
			func() error { return d.processHeaders(origin+1, td) },
		}
		if d.mode == FastSync {  //새로운 처리 로직을 추가하는 모드에 따라
			fetchers = append(fetchers, func() error { return d.processFastSyncContent(latest) })
		} else if d.mode == FullSync {
			fetchers = append(fetchers, d.processFullSyncContent)
		}
		err = d.spawnSync(fetchers)
		if err != nil && d.mode == FastSync && d.fsPivotLock != nil {
			// If sync failed in the critical section, bump the fail counter.
			atomic.AddUint32(&d.fsPivotFails, 1)
		}
		return err
	}

각 가져 오기에 spawnSync는하는 goroutine을 시작 가져 오기 오류를 기다리는 차단.

	// spawnSync runs d.process and all given fetcher functions to completion in
	// separate goroutines, returning the first error that appears.
	func (d *Downloader) spawnSync(fetchers []func() error) error {
		var wg sync.WaitGroup
		errc := make(chan error, len(fetchers))
		wg.Add(len(fetchers))
		for _, fn := range fetchers {
			fn := fn
			go func() { defer wg.Done(); errc <- fn() }()
		}
		// Wait for the first error, then terminate the others.
		var err error
		for i := 0; i < len(fetchers); i++ {
			if i == len(fetchers)-1 {
				// Close the queue when all fetchers have exited.
				// This will cause the block processor to end when
				// it has processed the queue.
				d.queue.Close()
			}
			if err = <-errc; err != nil {
				break
			}
		}
		d.queue.Close()
		d.Cancel()
		wg.Wait()
		return err
	}

## 헤더 처리

fetchHeaders 헤더를 획득하는데 사용하는 방법. 그런 다음 인수 헤더에 따라 몸과 영수증 정보를 얻을 이동합니다.

	// fetchHeaders keeps retrieving headers concurrently from the number
	// requested, until no more are returned, potentially throttling on the way. To
	// facilitate concurrency but still protect against malicious nodes sending bad
	// headers, we construct a header chain skeleton using the "origin" peer we are
	// syncing with, and fill in the missing headers using anyone else. Headers from
	// other peers are only accepted if they map cleanly to the skeleton. If no one
	// can fill in the skeleton - not even the origin peer - it's assumed invalid and
	// the origin is dropped.
	fetchHeaders 연속적 모두의 반환을 위해 이러한 동작은 전송 헤더 요청 대기 반복. 헤더 모든 요청까지. 여전히 악의적 인 노드를 방지 할 수있는 것은 잘못된 헤더를 보내는 동안, 동시성을 개선하기 위해, 우리는 &quot;기원&quot;우리는 피어가 헤더 체인 백본을 구성 동기화 및 헤더를 누락 작성 다른 사람의 사용을 사용합니다. 다른 동료의 헤딩슛은 깨끗하게에만 골격에 접수 매핑. 어떤 골격가 작성 할 수없는 경우 - 심지어 원점 피어는 채울 수없는 -이 무효로 간주하고 원점 피어도 폐기.
	func (d *Downloader) fetchHeaders(p *peerConnection, from uint64) error {
		p.log.Debug("Directing header downloads", "origin", from)
		defer p.log.Debug("Header download terminated")
	
		// Create a timeout timer, and the associated header fetcher
		skeleton := true			// Skeleton assembly phase or finishing up
		request := time.Now()	   // time of the last skeleton fetch request
		timeout := time.NewTimer(0) // timer to dump a non-responsive active peer
		<-timeout.C				 // timeout channel should be initially empty
		defer timeout.Stop()
	
		var ttl time.Duration
		getHeaders := func(from uint64) {
			request = time.Now()
	
			ttl = d.requestTTL()
			timeout.Reset(ttl)
	
			if skeleton { //작성 골격
				p.log.Trace("Fetching skeleton headers", "count", MaxHeaderFetch, "from", from)
				go p.peer.RequestHeadersByNumber(from+uint64(MaxHeaderFetch)-1, MaxSkeletonSize, MaxHeaderFetch-1, false)
			} else { //직접 요청
				p.log.Trace("Fetching full headers", "count", MaxHeaderFetch, "from", from)
				go p.peer.RequestHeadersByNumber(from, MaxHeaderFetch, 0, false)
			}
		}
		// Start pulling the header chain skeleton until all is done
		getHeaders(from)
	
		for {
			select {
			case <-d.cancelCh:
				return errCancelHeaderFetch
	
			case packet := <-d.headerCh: //네트워크 헤더 수익률이 채널 headerCh에 게시됩니다
				// Make sure the active peer is giving us the skeleton headers
				if packet.PeerId() != p.id {
					log.Debug("Received skeleton from incorrect peer", "peer", packet.PeerId())
					break
				}
				headerReqTimer.UpdateSince(request)
				timeout.Stop()
	
				// If the skeleton's finished, pull any remaining head headers directly from the origin
				if packet.Items() == 0 && skeleton {
					skeleton = false
					getHeaders(from)
					continue
				}
				// If no more headers are inbound, notify the content fetchers and return
				//더 이상 반환합니다. 그래서 headerProcCh 채널을 말해
				if packet.Items() == 0 {
					p.log.Debug("No more headers available")
					select {
					case d.headerProcCh <- nil:
						return nil
					case <-d.cancelCh:
						return errCancelHeaderFetch
					}
				}
				headers := packet.(*headerPack).headers
	
				// If we received a skeleton batch, resolve internals concurrently
				if skeleton { //당신은 골격, 다음 방법을 작성해야하는 경우 이는 충전 좋은에서
					filled, proced, err := d.fillHeaderSkeleton(from, headers)
					if err != nil {
						p.log.Debug("Skeleton chain invalid", "err", err)
						return errInvalidChain
					}
					headers = filled[proced:]
					//proced 숫자를 대신하여 처리를 완료했습니다. 당신은 proced해야 헤더 뒤에
					from += uint64(proced)
				}
				// Insert all the new headers and fetch the next batch
				if len(headers) > 0 {
					p.log.Trace("Scheduling new headers", "count", len(headers), "from", from)
					//headerProcCh에 게시 다음주기가 계속됩니다.
					select {
					case d.headerProcCh <- headers:
					case <-d.cancelCh:
						return errCancelHeaderFetch
					}
					from += uint64(len(headers))
				}
				getHeaders(from)
	
			case <-timeout.C:
				// Header retrieval timed out, consider the peer bad and drop
				p.log.Debug("Header request timed out", "elapsed", ttl)
				headerTimeoutMeter.Mark(1)
				d.dropPeer(p.id)
	
				// Finish the sync gracefully instead of dumping the gathered data though
				for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
					select {
					case ch <- false:
					case <-d.cancelCh:
					}
				}
				select {
				case d.headerProcCh <- nil:
				case <-d.cancelCh:
				}
				return errBadPeer
			}
		}
	}

processHeaders 방법은이 방법 headerProcCh 채널로부터 헤더를 얻었다. 그리고 예약 할 수있는 큐 헤더에 던져 얻을이 신체 가져 오기 또는 영수증 가져 오기는 작업을 가져받을 수 있습니다.
	
	// processHeaders takes batches of retrieved headers from an input channel and
	// keeps processing and scheduling them into the header chain and downloader's
	// queue until the stream ends or a failure occurs.
	//processHeaders 일괄 취득 헤더,이를 처리하고, 객체 다운의 큐를 통해 예약. 오류가 발생했을 경우, 또는 프로세스가 끝날 때까지.
	func (d *Downloader) processHeaders(origin uint64, td *big.Int) error {
		// Calculate the pivoting point for switching from fast to slow sync
		pivot := d.queue.FastSyncPivot()
	
		// Keep a count of uncertain headers to roll back
		//포인트가 실패 할 경우,이 논리에 대한 처리를 롤백. 당신이 2048 노드를 삽입하기 전에 그래서 롤백합니다. 표준 아래의 보안 때문에 동기 자세한 참조 문서를 빠르게 할 수 있습니다.
		rollback := []*types.Header{}
		defer func() { //기능 때 롤백 오류를 종료하는 데 사용됩니다. TODO
			if len(rollback) > 0 {
				// Flatten the headers and roll them back
				hashes := make([]common.Hash, len(rollback))
				for i, header := range rollback {
					hashes[i] = header.Hash()
				}
				lastHeader, lastFastBlock, lastBlock := d.lightchain.CurrentHeader().Number, common.Big0, common.Big0
				if d.mode != LightSync {
					lastFastBlock = d.blockchain.CurrentFastBlock().Number()
					lastBlock = d.blockchain.CurrentBlock().Number()
				}
				d.lightchain.Rollback(hashes)
				curFastBlock, curBlock := common.Big0, common.Big0
				if d.mode != LightSync {
					curFastBlock = d.blockchain.CurrentFastBlock().Number()
					curBlock = d.blockchain.CurrentBlock().Number()
				}
				log.Warn("Rolled back headers", "count", len(hashes),
					"header", fmt.Sprintf("%d->%d", lastHeader, d.lightchain.CurrentHeader().Number),
					"fast", fmt.Sprintf("%d->%d", lastFastBlock, curFastBlock),
					"block", fmt.Sprintf("%d->%d", lastBlock, curBlock))
	
				// If we're already past the pivot point, this could be an attack, thread carefully
				if rollback[len(rollback)-1].Number.Uint64() > pivot {
					// If we didn't ever fail, lock in the pivot header (must! not! change!)
					if atomic.LoadUint32(&d.fsPivotFails) == 0 {
						for _, header := range rollback {
							if header.Number.Uint64() == pivot {
								log.Warn("Fast-sync pivot locked in", "number", pivot, "hash", header.Hash())
								d.fsPivotLock = header
							}
						}
					}
				}
			}
		}()
	
		// Wait for batches of headers to process
		gotHeaders := false
	
		for {
			select {
			case <-d.cancelCh:
				return errCancelHeaderProcessing
	
			case headers := <-d.headerProcCh:
				// Terminate header processing if we synced up
				if len(headers) == 0 { //프로세스 완료
					// Notify everyone that headers are fully processed
					for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
						select {
						case ch <- false:
						case <-d.cancelCh:
						}
					}
					// If no headers were retrieved at all, the peer violated it's TD promise that it had a
					// better chain compared to ours. The only exception is if it's promised blocks were
					// already imported by other means (e.g. fecher):
					//
					// R <remote peer>, L <local node>: Both at block 10
					// R: Mine block 11, and propagate it to L
					// L: Queue block 11 for import
					// L: Notice that R's head and TD increased compared to ours, start sync
					// L: Import of block 11 finishes
					// L: Sync begins, and finds common ancestor at 11
					// L: Request new headers up from 11 (R's TD was higher, it must have something)
					// R: Nothing to give
					if d.mode != LightSync { //TD 우리보다 다른 측면,하지만 아무것도하지 않았다. 그런 다음 다른 쪽은 잘못 다른 측면입니다. 연결을 끊고 다른 링크
						if !gotHeaders && td.Cmp(d.blockchain.GetTdByHash(d.blockchain.CurrentBlock().Hash())) > 0 {
							return errStallingPeer
						}
					}
					// If fast or light syncing, ensure promised headers are indeed delivered. This is
					// needed to detect scenarios where an attacker feeds a bad pivot and then bails out
					// of delivering the post-pivot blocks that would flag the invalid content.
					//
					// This check cannot be executed "as is" for full imports, since blocks may still be
					// queued for processing when the header download completes. However, as long as the
					// peer gave us something useful, we're already happy/progressed (above check).
					if d.mode == FastSync || d.mode == LightSync {
						if td.Cmp(d.lightchain.GetTdByHash(d.lightchain.CurrentHeader().Hash())) > 0 {
							return errStallingPeer
						}
					}
					// Disable any rollback and return
					rollback = nil
					return nil
				}
				// Otherwise split the chunk of headers into batches and process them
				gotHeaders = true
	
				for len(headers) > 0 {
					// Terminate if something failed in between processing chunks
					select {
					case <-d.cancelCh:
						return errCancelHeaderProcessing
					default:
					}
					// Select the next chunk of headers to import
					limit := maxHeadersProcess
					if limit > len(headers) {
						limit = len(headers)
					}
					chunk := headers[:limit]
	
					// In case of header only syncing, validate the chunk immediately
					if d.mode == FastSync || d.mode == LightSync { //이 빠른 동기 모드 또는 동기 모드가 가벼운 경우 (전용 섹션 헤더를 다운로드)
						// Collect the yet unknown headers to mark them as uncertain
						unknown := make([]*types.Header, 0, len(headers))
						for _, header := range chunk {
							if !d.lightchain.HasHeader(header.Hash(), header.Number.Uint64()) {
								unknown = append(unknown, header)
							}
						}
						// If we're importing pure headers, verify based on their recentness
						//얼마나 많은 블록마다 검증
						frequency := fsHeaderCheckFrequency
						if chunk[len(chunk)-1].Number.Uint64()+uint64(fsHeaderForceVerify) > pivot {
							frequency = 1
						}
						//lightchain 기본 사슬 같다. 헤더 영역을 삽입합니다. 그 다음에 실패하면 당신은 롤백 할 필요가있다.
						if n, err := d.lightchain.InsertHeaderChain(chunk, frequency); err != nil {
							// If some headers were inserted, add them too to the rollback list
							if n > 0 {
								rollback = append(rollback, chunk[:n]...)
							}
							log.Debug("Invalid header encountered", "number", chunk[n].Number, "hash", chunk[n].Hash(), "err", err)
							return errInvalidChain
						}
						// All verifications passed, store newly found uncertain headers
						rollback = append(rollback, unknown...)
						if len(rollback) > fsHeaderSafetyNet {
							rollback = append(rollback[:0], rollback[len(rollback)-fsHeaderSafetyNet:]...)
						}
					}
					// If we're fast syncing and just pulled in the pivot, make sure it's the one locked in
					if d.mode == FastSync && d.fsPivotLock != nil && chunk[0].Number.Uint64() <= pivot && chunk[len(chunk)-1].Number.Uint64() >= pivot { //PivotLock 경우, 해시가 동일한 확인.
						if pivot := chunk[int(pivot-chunk[0].Number.Uint64())]; pivot.Hash() != d.fsPivotLock.Hash() {
							log.Warn("Pivot doesn't match locked in one", "remoteNumber", pivot.Number, "remoteHash", pivot.Hash(), "localNumber", d.fsPivotLock.Number, "localHash", d.fsPivotLock.Hash())
							return errInvalidChain
						}
					}
					// Unless we're doing light chains, schedule the headers for associated content retrieval
					//우리는 가벼운 체인을 처리 한 경우. 헤더 스케줄링 관련 데이터를 얻었다. 몸, 영수증
					if d.mode == FullSync || d.mode == FastSync {
						// If we've reached the allowed number of pending headers, stall a bit
						//현재 수신 큐의 용량보다 작은 경우. 그래서 기다립니다.
						for d.queue.PendingBlocks() >= maxQueuedHeaders || d.queue.PendingReceipts() >= maxQueuedHeaders {
							select {
							case <-d.cancelCh:
								return errCancelHeaderProcessing
							case <-time.After(time.Second):
							}
						}
						// Otherwise insert the headers for content retrieval
						//예약 통화, 다운로드 몸과 영수증을 대기열
						inserts := d.queue.Schedule(chunk, origin)
						if len(inserts) != len(chunk) {
							log.Debug("Stale headers")
							return errBadPeer
						}
					}
					headers = headers[limit:]
					origin += uint64(limit)
				}
				// Signal the content downloaders of the availablility of new tasks
				//d.bodyWakeCh 채널하려면 d.receiptWakeCh 메시지 처리 스레드 웨이크 업을 보냅니다.
				for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
					select {
					case ch <- true:
					default:
					}
				}
			}
		}
	}


## 몸 과정
fetchBodies 기능은 일부 폐쇄 함수를 정의하고 함수 fetchParts를 호출
	
	// fetchBodies iteratively downloads the scheduled block bodies, taking any
	// available peers, reserving a chunk of blocks for each, waiting for delivery
	// and also periodically checking for timeouts.
	//진행중인 다운로드 블록 fetchBodies, 중간는 링크의 각 부위에 대해 예약 된 블록의 어떠한 사용도 연결 블록이 전송 될 때까지 기다릴 사용하고, 정기적으로 초과하는지 여부를 확인한다.
	func (d *Downloader) fetchBodies(from uint64) error {
		log.Debug("Downloading block bodies", "origin", from)
	
		var (
			deliver = func(packet dataPack) (int, error) { //펑션 블록 전달의 완전한 몸을 다운로드
				pack := packet.(*bodyPack)
				return d.queue.DeliverBodies(pack.peerId, pack.transactions, pack.uncles)
			}
			expire   = func() map[string]int { return d.queue.ExpireBodies(d.requestTTL()) }  //시간 제한
			fetch= Func을 (p *의 용의 PeerConnection, REQ * fetchRequest) 에러가 반환 p.FetchBodies {(REQ)} 함수를 페치 //
			capacity = func(p *peerConnection) int { return p.BlockCapacity(d.requestRTT()) } //최종 처리량
			setIdle  = func(p *peerConnection, accepted int) { p.SetBodiesIdle(accepted) } //유휴 피어 설정
		)
		err := d.fetchParts(errCancelBodyFetch, d.bodyCh, deliver, d.bodyWakeCh, expire,
			d.queue.PendingBlocks, d.queue.InFlightBlocks, d.queue.ShouldThrottleBlocks, d.queue.ReserveBodies,
			d.bodyFetchHook, fetch, d.queue.CancelBodies, capacity, d.peers.BodyIdlePeers, setIdle, "bodies")
	
		log.Debug("Block body download terminated", "err", err)
		return err
	}


fetchParts


	// fetchParts iteratively downloads scheduled block parts, taking any available
	// peers, reserving a chunk of fetch requests for each, waiting for delivery and
	// also periodically checking for timeouts.
	//반복적 다른 신체 각 추출 부에 대한 예약 요청을 다수 사용할 수있게 소정의 블록 부분을 다운로드 및 전송을 기다리고 fetchParts 주기적으로 조사했다.
	// As the scheduling/timeout logic mostly is the same for all downloaded data
	// types, this method is used by each for data gathering and is instrumented with
	// various callbacks to handle the slight differences between processing them.
	//모든 다운로드 된 데이터 유형의 대부분을 스케줄링 / 제한 로직 동일하기 때문에, 따라서이 방법은 블록의 종류를 다운로드하는 데 사용되며, 콜백 함수는 그들 사이의 미묘한 차이의 다양한 방법에 관한 것이다.
	// The instrumentation parameters:
	//- errCancel 오류 유형은 (대부분 멋진 로그인하게) 인출 작업이 취소되는 경우 동작을 인출하는 경우, 이는 상기 데이터 채널을 반환 전송 될 것이다 취소 할
	//- deliveryCh : 이상 (모든 동시 동료로부터 합병) 데이터가 전달 대상을 완료 다운로드 다운로드 데이터 패킷을 검색 할 채널
	//  - deliver: (일반적 queue``내) 유형 특정 다운로드 큐에 처리 된 데이터를 데이터 패킷을 제공하는 콜백을 처리 한 후에 큐에 전달
	//  - wakeCh:  새로운 태스크 (또는 동기 완료) 새로운 태스크 페처의 도착을 통지하거나, 동기화가 완료 사용할 수있는 가져 오기를 깨우기위한 알림 채널
	//  - expire:  작업 콜백 방법은 너무 오래 걸려서 요청을 중단하고 시간 초과로 인해 콜백 요청의 종료 결함이있는 동료 (트래픽 쉐이핑)을 반환합니다.
	//  - pending: 여전히 (완료 / 비 completability을 감지)도 작업을 다운로드해야 다운로드를 필요로하는 요청의 수에 대한 번호 작업 콜백.
	//  - inFlight:진행중인 요청의 과정에서 요청 수의 수에 대한 작업 콜백 처리를 (모든 활성 다운로드가 완료 될 때까지 기다립니다)
	//  - throttle:처리 큐가 가득 스로틀 (결합 된 메모리를 사용)를 활성화 할 경우 작업 콜백 프로세싱 큐가 가득 콜백 함수인지 여부를 확인하는 데 사용되는 확인한다.
	//  - reserve: 태스크 콜백 피어 콜백 함수 태스크를 예약하는 데 사용되는 특정 피어 (또한 부분적으로 완료 신호)에 새로운 다운로드 작업을 예약
	//  - fetchHook:   tester callback to notify of new tasks being initiated (allows testing the scheduling logic) 
	//  - fetch:   네트워크 콜백은 실제로 물리적 원격 피어 // 요청 전송 네트워크에 특정 다운로드 요청을 콜백 함수를 보낼 수
	//  - cancel:  작업 콜백은 기내 다운로드 요청을 중단하고 (손실 피어의 경우)를 재조정 할 수 작업을 취소 할 콜백 함수를 처리하는합니다
	//  - capacity:네트워크 콜백 피어 (트래픽 쉐이핑) 네트워크 용량 또는 대역폭의 추정 유형 특정 대역폭 용량을 취득한다.
	//  - idle:	네트워크 콜백 작업 피어 할당 할 수 있습니다 현재 (타입 별) 유휴 동료를 검색하는 것은 유휴 콜백 함수입니다
	//  - setIdle: 네트워크 콜백 유휴 콜백 피어 설정의 예상 용량 (트래픽 쉐이핑)을 유휴하고 업데이트 다시 피어를 설정합니다
	//  - kind:	유형의 텍스트 레이블은 로그 로그 mesages 다운로드 형태로 표시 다운로드되는
	func (d *Downloader) fetchParts(errCancel error, deliveryCh chan dataPack, deliver func(dataPack) (int, error), wakeCh chan bool,
		expire func() map[string]int, pending func() int, inFlight func() bool, throttle func() bool, reserve func(*peerConnection, int) (*fetchRequest, bool, error),
		fetchHook func([]*types.Header), fetch func(*peerConnection, *fetchRequest) error, cancel func(*fetchRequest), capacity func(*peerConnection) int,
		idle func() ([]*peerConnection, int), setIdle func(*peerConnection, int), kind string) error {
	
		// Create a ticker to detect expired retrieval tasks
		ticker := time.NewTicker(100 * time.Millisecond)
		defer ticker.Stop()
	
		update := make(chan struct{}, 1)
	
		// Prepare the queue and fetch block parts until the block header fetcher's done
		finished := false
		for {
			select {
			case <-d.cancelCh:
				return errCancel
	
			case packet := <-deliveryCh:
				// If the peer was previously banned and failed to deliver it's pack
				// in a reasonable time frame, ignore it's message.
				//피어가 전에 금지하고 적시에 데이터를 제공하지 않은 경우,이 데이터를 무시
				if peer := d.peers.Peer(packet.PeerId()); peer != nil {
					// Deliver the received chunk of data and check chain validity
					accepted, err := deliver(packet)
					if err == errInvalidChain {
						return err
					}
					// Unless a peer delivered something completely else than requested (usually
					// caused by a timed out request which came through in the end), set it to
					// idle. If the delivery's stale, the peer should have already been idled.
					if err != errStaleDelivery {
						setIdle(peer, accepted)
					}
					// Issue a log to the user to see what's going on
					switch {
					case err == nil && packet.Items() == 0:
						peer.log.Trace("Requested data not delivered", "type", kind)
					case err == nil:
						peer.log.Trace("Delivered new batch of data", "type", kind, "count", packet.Stats())
					default:
						peer.log.Trace("Failed to deliver retrieved data", "type", kind, "err", err)
					}
				}
				// Blocks assembled, try to update the progress
				select {
				case update <- struct{}{}:
				default:
				}
	
			case cont := <-wakeCh:
				// The header fetcher sent a continuation flag, check if it's done
				//모든 작업은 큐에 기록을 완료합니다.
				if !cont {
					finished = true
				}
				// Headers arrive, try to update the progress
				select {
				case update <- struct{}{}:
				default:
				}
	
			case <-ticker.C:
				// Sanity check update the progress
				select {
				case update <- struct{}{}:
				default:
				}
	
			case <-update:
				// Short circuit if we lost all our peers
				if d.peers.Len() == 0 {
					return errNoPeers
				}
				// Check for fetch request timeouts and demote the responsible peers
				for pid, fails := range expire() {
					if peer := d.peers.Peer(pid); peer != nil {
						// If a lot of retrieval elements expired, we might have overestimated the remote peer or perhaps
						// ourselves. Only reset to minimal throughput but don't drop just yet. If even the minimal times
						// out that sync wise we need to get rid of the peer.
						//검색 요소의 많은이 만료되면, 우리는 원격 객체 나 자신을 과대 평가되었을 수 있습니다. 그것은 단지 최소한의 처리량으로 재설정 할 수 있지만, 폐기하지 않습니다. 심지어 최소한의 제한 시간 동기화 어떤 물론, 우리는 피어를 제거해야합니다.
						// The reason the minimum threshold is 2 is because the downloader tries to estimate the bandwidth
						// and latency of a peer separately, which requires pushing the measures capacity a bit and seeing
						// how response times reacts, to it always requests one more than the minimum (i.e. min 2).
						//최소 임계치는 시도가 약간 밀어 용량을 측정하고, 반응의 반응 시간은, 항상 (즉, 최소 2) 요구 된 최소값보다 큰 방법을 참조해야하는 각 피어 추정 대역폭 및 레이턴시를 다운로드하기 때문에 2 이유이다.
						if fails > 2 {
							peer.log.Trace("Data delivery timed out", "type", kind)
							setIdle(peer, 0)
						} else {
							peer.log.Debug("Stalling delivery, dropping", "type", kind)
							d.dropPeer(pid)
						}
					}
				}
				// If there's nothing more to fetch, wait or terminate
				//작업이 완료되었습니다. 그래서 종료
				if pending() == 0 { //할당 대기중인 작업이없는 경우, 휴식. 다음 코드를 실행하지 마십시오.
					if !inFlight() && finished {
						log.Debug("Data fetching completed", "type", kind)
						return nil
					}
					break
				}
				// Send a download request to all idle peers, until throttled
				progressed, throttled, running := false, false, inFlight()
				idles, total := idle()
	
				for _, peer := range idles {
					// Short circuit if throttling activated
					if throttle() {
						throttled = true
						break
					}
					// Short circuit if there is no more available task.
					if pending() == 0 {
						break
					}
					// Reserve a chunk of fetches for a peer. A nil can mean either that
					// no more headers are available, or that the peer is known not to
					// have them.
					//할당 작업은 피어 요청합니다.
					request, progress, err := reserve(peer, capacity(peer))
					if err != nil {
						return err
					}
					if progress {
						progressed = true
					}
					if request == nil {
						continue
					}
					if request.From > 0 {
						peer.log.Trace("Requesting new batch of data", "type", kind, "from", request.From)
					} else if len(request.Headers) > 0 {
						peer.log.Trace("Requesting new batch of data", "type", kind, "count", len(request.Headers), "from", request.Headers[0].Number)
					} else {
						peer.log.Trace("Requesting new batch of data", "type", kind, "count", len(request.Hashes))
					}
					// Fetch the chunk and make sure any errors return the hashes to the queue
					if fetchHook != nil {
						fetchHook(request.Headers)
					}
					if err := fetch(peer, request); err != nil {
						// Although we could try and make an attempt to fix this, this error really
						// means that we've double allocated a fetch task to a peer. If that is the
						// case, the internal state of the downloader and the queue is very wrong so
						// better hard crash and note the error instead of silently accumulating into
						// a much bigger issue.
						panic(fmt.Sprintf("%v: %s fetch assignment failed", peer, kind))
					}
					running = true
				}
				// Make sure that we have peers available for fetching. If all peers have been tried
				// and all failed throw an error
				if !progressed && !throttled && !running && len(idles) == total && pending() > 0 {
					return errPeersUnavailable
				}
			}
		}
	}


## 영수증 처리
마찬가지로 수신 처리 체.
	
	// fetchReceipts iteratively downloads the scheduled block receipts, taking any
	// available peers, reserving a chunk of receipts for each, waiting for delivery
	// and also periodically checking for timeouts.
	func (d *Downloader) fetchReceipts(from uint64) error {
		log.Debug("Downloading transaction receipts", "origin", from)
	
		var (
			deliver = func(packet dataPack) (int, error) {
				pack := packet.(*receiptPack)
				return d.queue.DeliverReceipts(pack.peerId, pack.receipts)
			}
			expire   = func() map[string]int { return d.queue.ExpireReceipts(d.requestTTL()) }
			fetch	= func(p *peerConnection, req *fetchRequest) error { return p.FetchReceipts(req) }
			capacity = func(p *peerConnection) int { return p.ReceiptCapacity(d.requestRTT()) }
			setIdle  = func(p *peerConnection, accepted int) { p.SetReceiptsIdle(accepted) }
		)
		err := d.fetchParts(errCancelReceiptFetch, d.receiptCh, deliver, d.receiptWakeCh, expire,
			d.queue.PendingReceipts, d.queue.InFlightReceipts, d.queue.ShouldThrottleReceipts, d.queue.ReserveReceipts,
			d.receiptFetchHook, fetch, d.queue.CancelReceipts, capacity, d.peers.ReceiptIdlePeers, setIdle, "receipts")
	
		log.Debug("Transaction receipt download terminated", "err", err)
		return err
	}


## processFastSyncContent 및 processFullSyncContent



	// processFastSyncContent takes fetch results from the queue and writes them to the
	// database. It also controls the synchronisation of state nodes of the pivot block.
	func (d *Downloader) processFastSyncContent(latest *types.Header) error {
		// Start syncing state of the reported head block.
		// This should get us most of the state of the pivot block.
		//동기화 상태를 시작합니다
		stateSync := d.syncState(latest.Root)
		defer stateSync.Cancel()
		go func() {
			if err := stateSync.Wait(); err != nil {
				d.queue.Close() // wake up WaitResults
			}
		}()
	
		pivot := d.queue.FastSyncPivot()
		for {
			results := d.queue.WaitResults() //처리 된 출력 큐 블록
			if len(results) == 0 {
				return stateSync.Cancel()
			}
			if d.chainInsertHook != nil {
				d.chainInsertHook(results)
			}
			P, beforeP, afterP := splitAroundPivot(pivot, results)
			//삽입 빠른 동기화 데이터
			if err := d.commitFastSyncData(beforeP, stateSync); err != nil {
				return err
			}
			if P != nil {
				//당신이 피벗 포인트에 도달 한 경우 상태 동기화가 완료에 대한 다음 기다립니다
				stateSync.Cancel()
				if err := d.commitPivotBlock(P); err != nil {
					return err
				}
			}
			//피봇 포인트 이후 모든 노드를 들어, 우리는 전체 치료를 따라야합니다.
			if err := d.importBlockResults(afterP); err != nil {
				return err
			}
		}
	}

processFullSyncContent은 비교적 간단하다. 큐로부터 취득 블록은 내부에 삽입된다.
	
	// processFullSyncContent takes fetch results from the queue and imports them into the chain.
	func (d *Downloader) processFullSyncContent() error {
		for {
			results := d.queue.WaitResults()
			if len(results) == 0 {
				return nil
			}
			if d.chainInsertHook != nil {
				d.chainInsertHook(results)
			}
			if err := d.importBlockResults(results); err != nil {
				return err
			}
		}
	}
