다운 큐 스케줄링 기능 제한 기능을 제공한다. 일정 / ScheduleSkeleton를 호출하여 작업 예약을 신청하려면, 다음, 큐에 데이터를 다운로드의 호출 DeliverXXX 방법을 스케줄링 작업을 수신하고, 스레드 내에서 다운을 수행 할 수 ReserveXXX 방법을 문의하십시오. 마지막으로, WaitResults 완료 작업을 얻을 수 있습니다. 몇 가지 추가 제어 작업 거기 중간은 ExpireXXX는 태스크 시간 초과, CancelXXX가 작업을 취소 할 수 있는지 여부를 제어하는 ​​데 사용됩니다.



## 예약 방법
예약 전화는 지역 헤더 다운로드 일정의 일부 적용합니다. 어떤 정당성을 수행 후에는 blockTaskPool, receiptTaskPool, receiptTaskQueue, receiptTaskPool에 작업을 확인 볼 수 있습니다.
TaskPool을 해시 존재의 헤더를 기록하는 데 사용지도입니다. 작업 대기열 우선 순위 스케쥴링은 제 작은 작업 기능을 달성 할 수 있도록 우선 음극 블록의 높이가 높을수록 작은 블록의 높이, 우선 순위 큐이다.
	
	// Schedule adds a set of headers for the download queue for scheduling, returning
	// the new headers encountered.
	//블록의 높이에서 내측 헤더의 첫 번째 요소를 나타낸다. 반환 값은 모든 헤더를받은 돌려줍니다
	func (q *queue) Schedule(headers []*types.Header, from uint64) []*types.Header {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		// Insert all the headers prioritised by the contained block number
		inserts := make([]*types.Header, 0, len(headers))
		for _, header := range headers {
			// Make sure chain order is honoured and preserved throughout
			hash := header.Hash()
			if header.Number == nil || header.Number.Uint64() != from {
				log.Warn("Header broke chain ordering", "number", header.Number, "hash", hash, "expected", from)
				break
			}
			//headerHead 헤더 영역 점포 마지막 블록 올바른 연결 여부를 확인 삽입.
			if q.headerHead != (common.Hash{}) && q.headerHead != header.ParentHash {
				log.Warn("Header broke chain ancestry", "number", header.Number, "hash", hash)
				break
			}
			// Make sure no duplicate requests are executed
			//바로 여기 계속 중복 확인이되지 위쪽부터하지 않았다.
			if _, ok := q.blockTaskPool[hash]; ok {
				log.Warn("Header  already scheduled for block fetch", "number", header.Number, "hash", hash)
				continue
			}
			if _, ok := q.receiptTaskPool[hash]; ok {
				log.Warn("Header already scheduled for receipt fetch", "number", header.Number, "hash", hash)
				continue
			}
			// Queue the header for content retrieval
			q.blockTaskPool[hash] = header
			q.blockTaskQueue.Push(header, -float32(header.Number.Uint64()))
	
			if q.mode == FastSync && header.Number.Uint64() <= q.fastSyncPivot {
				// Fast phase of the fast sync, retrieve receipts too
				//이 빠른 동기화 모드, 블록 피벗 포인트의 높이보다도 작은 경우. 그래서 심지어 영수증을받을
				q.receiptTaskPool[hash] = header
				q.receiptTaskQueue.Push(header, -float32(header.Number.Uint64()))
			}
			inserts = append(inserts, header)
			q.headerHead = hash
			from++
		}
		return inserts
	}


## ReserveXXX
ReserveXXX 방법 안에 수행 큐에서 일부 작업을 수신하는 데 사용된다. goroutine 내부 다운로더가 수행하는 작업의 번호를 받기 위해이 메소드를 호출합니다. 이 방법은 reserveHeaders 방법을 직접 호출한다. 모든 ReserveXXX 방법은 약간의 차이가 입력 매개 변수에 추가하여, reserveHeaders 메소드를 호출합니다.

	// ReserveBodies reserves a set of body fetches for the given peer, skipping any
	// previously failed downloads. Beside the next batch of needed fetches, it also
	// returns a flag whether empty blocks were queued requiring processing.
	func (q *queue) ReserveBodies(p *peerConnection, count int) (*fetchRequest, bool, error) {
		isNoop := func(header *types.Header) bool {
			return header.TxHash == types.EmptyRootHash && header.UncleHash == types.EmptyUncleHash
		}
		q.lock.Lock()
		defer q.lock.Unlock()
	
		return q.reserveHeaders(p, count, q.blockTaskPool, q.blockTaskQueue, q.blockPendPool, q.blockDonePool, isNoop)
	}

reserveHeaders


	// reserveHeaders reserves a set of data download operations for a given peer,
	// skipping any previously failed ones. This method is a generic version used
	// by the individual special reservation functions.
	//reserveHeaders는 일부 다운로드 작업, 지정된 피어에 건너 뛰는 전에 오류를 유지합니다. 이 방법은 별도로 지정된 보존 방법이라고합니다.
	// Note, this method expects the queue lock to be already held for writing. The
	// reason the lock is not obtained in here is because the parameters already need
	// to access the queue, so they already need a lock anyway.
	//이 메소드가 호출 될 때,이이 방법에 대한 이유가없는 내부 함수에 전달 된 매개 변수를 잠 그려면, 그래서 당신이 잠금을 획득 할 때 호출하는 것입니다, 잠금을 획득 한 것으로 간주됩니다.
	func (q *queue) reserveHeaders(p *peerConnection, count int, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
		pendPool map[string]*fetchRequest, donePool map[common.Hash]struct{}, isNoop func(*types.Header) bool) (*fetchRequest, bool, error) {
		// Short circuit if the pool has been depleted, or if the peer's already
		// downloading something (sanity check not to corrupt state)
		if taskQueue.Empty() {
			return nil, false, nil
		}
		//또한 작업을 다운로드 피어가 완료되지 않은 경우.
		if _, ok := pendPool[p.id]; ok {
			return nil, false, nil
		}
		// Calculate an upper limit on the items we might fetch (i.e. throttling)
		//우리는 상한 계산을 얻을 필요가있다.
		space := len(q.resultCache) - len(donePool)
		//또한 숫자가 다운로드되는 뺄 필요가있다.
		for _, request := range pendPool {
			space -= len(request.Headers)
		}
		// Retrieve a batch of tasks, skipping previously failed ones
		send := make([]*types.Header, 0, count)
		skip := make([]*types.Header, 0)
	
		progress := false
		for proc := 0; proc < space && len(send) < count && !taskQueue.Empty(); proc++ {
			header := taskQueue.PopItem().(*types.Header)
	
			// If we're the first to request this task, initialise the result container
			index := int(header.Number.Int64() - int64(q.resultOffset))
			//인덱스 resultCache의 일부에 저장되어야하는 결과이다.
			if index >= len(q.resultCache) || index < 0 {
				common.Report("index allocation went beyond available resultCache space")
				return nil, false, errInvalidChain
			}
			if q.resultCache[index] == nil { //첫 번째 일정은 예상 일정을 여러 번이다. 그리고 아마 비어있다.
				components := 1
				if q.mode == FastSync && header.Number.Uint64() <= q.fastSyncPivot {
					//동기화가 빠르다면, 당신은 영수증 영수증뿐만 아니라 구성 요소를 다운로드해야
					components = 2
				}
				q.resultCache[index] = &fetchResult{
					Pending: components,
					Header:  header,
				}
			}
			// If this fetch task is a noop, skip this fetch operation
			if isNoop(header) {
				//블록 헤더가 트랜잭션에 포함되지 않는 경우, 액세스 존 헤더 없다
				donePool[header.Hash()] = struct{}{}
				delete(taskPool, header.Hash())
	
				space, proc = space-1, proc-1
				q.resultCache[index].Pending--
				progress = true
				continue
			}
			// Otherwise unless the peer is known not to have the data, add to the retrieve list
			//전 분명히이 데이터없이 대표 노드 해시 것이 없다.
			if p.Lacks(header.Hash()) {
				skip = append(skip, header)
			} else {
				send = append(send, header)
			}
		}
		// Merge all the skipped headers back
		for _, header := range skip {
			taskQueue.Push(header, -float32(header.Number.Uint64()))
		}
		if progress {
			// Wake WaitResults, resultCache was modified
			//공지 사항 WaitResults는 resultCache이 변경되었습니다
			q.active.Signal()
		}
		// Assemble and return the block download request
		if len(send) == 0 {
			return nil, progress, nil
		}
		request := &fetchRequest{
			Peer:	p,
			Headers: send,
			Time:	time.Now(),
		}
		pendPool[p.id] = request
	
		return request, progress, nil
	}

ReserveReceipts 거의 볼 ReserveBodys 수 있습니다. 그러나 큐가 변경됩니다.

	// ReserveReceipts reserves a set of receipt fetches for the given peer, skipping
	// any previously failed downloads. Beside the next batch of needed fetches, it
	// also returns a flag whether empty receipts were queued requiring importing.
	func (q *queue) ReserveReceipts(p *peerConnection, count int) (*fetchRequest, bool, error) {
		isNoop := func(header *types.Header) bool {
			return header.ReceiptHash == types.EmptyRootHash
		}
		q.lock.Lock()
		defer q.lock.Unlock()
	
		return q.reserveHeaders(p, count, q.receiptTaskPool, q.receiptTaskQueue, q.receiptPendPool, q.receiptDonePool, isNoop)
	}



## DeliverXXX
제공 방법은 데이터 다운로드 후 호출됩니다.

	// DeliverBodies injects a block body retrieval response into the results queue.
	// The method returns the number of blocks bodies accepted from the delivery and
	// also wakes any threads waiting for data delivery.
	//DeliverBodies 큐 결과 삽입 된 리턴 값 블록 체를 요청할
	//이 방법은 몸의 전달 될 블록의 수를 반환하고, 데이터를 기다리는 스레드를 깨워 것
	func (q *queue) DeliverBodies(id string, txLists [][]*types.Transaction, uncleLists [][]*types.Header) (int, error) {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		reconstruct := func(header *types.Header, index int, result *fetchResult) error {
			if types.DeriveSha(types.Transactions(txLists[index])) != header.TxHash || types.CalcUncleHash(uncleLists[index]) != header.UncleHash {
				return errInvalidBody
			}
			result.Transactions = txLists[index]
			result.Uncles = uncleLists[index]
			return nil
		}
		return q.deliver(id, q.blockTaskPool, q.blockTaskQueue, q.blockPendPool, q.blockDonePool, bodyReqTimer, len(txLists), reconstruct)
	}

방법을 제공
	
	func (q *queue) deliver(id string, taskPool map[common.Hash]*types.Header, taskQueue *prque.Prque,
		pendPool map[string]*fetchRequest, donePool map[common.Hash]struct{}, reqTimer metrics.Timer,
		results int, reconstruct func(header *types.Header, index int, result *fetchResult) error) (int, error) {
	
		// Short circuit if the data was never requested
		//데이터가 요청 된 적이 있는지 확인하십시오.
		request := pendPool[id]
		if request == nil {
			return 0, errNoFetchesPending
		}
		reqTimer.UpdateSince(request.Time)
		delete(pendPool, id)
	
		// If no data items were retrieved, mark them as unavailable for the origin peer
		if results == 0 {
			//경우 결과는 비어 있습니다. 그런 다음 이러한 데이터없이 피어를 식별합니다.
			for _, header := range request.Headers {
				request.Peer.MarkLacking(header.Hash())
			}
		}
		// Assemble each of the results with their headers and retrieved data parts
		var (
			accepted int
			failure  error
			useful   bool
		)
		for i, header := range request.Headers {
			// Short circuit assembly if no more fetch results are found
			if i >= results {
				break
			}
			// Reconstruct the next result if contents match up
			index := int(header.Number.Int64() - int64(q.resultOffset))
			if index >= len(q.resultCache) || index < 0 || q.resultCache[index] == nil {
				failure = errInvalidChain
				break
			}
			//수신은 구축 데이터의 함수를 호출
			if err := reconstruct(header, i, q.resultCache[index]); err != nil {
				failure = err
				break
			}
			donePool[header.Hash()] = struct{}{}
			q.resultCache[index].Pending--
			useful = true
			accepted++
	
			// Clean up a successful fetch
			//TaskPool을에서 삭제합니다. donePool 가입
			request.Headers[i] = nil
			delete(taskPool, header.Hash())
		}
		// Return all failed or missing fetches to the queue
		//작업 대기열에 가입 성공없이 모든 요청
		for _, header := range request.Headers {
			if header != nil {
				taskQueue.Push(header, -float32(header.Number.Uint64()))
			}
		}
		// Wake up WaitResults
		//결과 경우 변경 알림 WaitResults 스레드 시작이있다.
		if accepted > 0 {
			q.active.Signal()
		}
		// If none of the data was good, it's a stale delivery
		switch {
		case failure == nil || failure == errInvalidChain:
			return accepted, failure
		case useful:
			return accepted, fmt.Errorf("partial failure: %v", failure)
		default:
			return accepted, errStaleDelivery
		}
	}


## ExpireXXX and CancelXXX
### ExpireXXX
ExpireBodies 기능은 다음 함수가 만료 전화 잠금을 가져옵니다.

	// ExpireBodies checks for in flight block body requests that exceeded a timeout
	// allowance, canceling them and returning the responsible peers for penalisation.
	func (q *queue) ExpireBodies(timeout time.Duration) map[string]int {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		return q.expire(timeout, q.blockPendPool, q.blockTaskQueue, bodyTimeoutMeter)
	}

함수 만료

	// expire is the generic check that move expired tasks from a pending pool back
	// into a task pool, returning all entities caught with expired tasks.
	//만료가 일반 시험, 작업 풀 풀에서 지연된 작업을 처리됩니다 모든 캡처 만료 된 작업 개체를 반환, 다시 움직였다.

	func (q *queue) expire(timeout time.Duration, pendPool map[string]*fetchRequest, taskQueue *prque.Prque, timeoutMeter metrics.Meter) map[string]int {
		// Iterate over the expired requests and return each to the queue
		expiries := make(map[string]int)
		for id, request := range pendPool {
			if time.Since(request.Time) > timeout {
				// Update the metrics with the timeout
				timeoutMeter.Mark(1)
	
				// Return any non satisfied requests to the pool
				if request.From > 0 {
					taskQueue.Push(request.From, -float32(request.From))
				}
				for hash, index := range request.Hashes {
					taskQueue.Push(hash, float32(index))
				}
				for _, header := range request.Headers {
					taskQueue.Push(header, -float32(header.Number.Uint64()))
				}
				// Add the peer to the expiry report along the the number of failed requests
				expirations := len(request.Hashes)
				if expirations < len(request.Headers) {
					expirations = len(request.Headers)
				}
				expiries[id] = expirations
			}
		}
		// Remove the expired requests from the pending pool
		for id := range expiries {
			delete(pendPool, id)
		}
		return expiries
	}


### CancelXXX
작업을 취소 할 수 cancle하는 기능은 작업 풀을 다시 가입 할 수있는 작업을 지정되었습니다.

	// CancelBodies aborts a body fetch request, returning all pending headers to the
	// task queue.
	func (q *queue) CancelBodies(request *fetchRequest) {
		q.cancel(request, q.blockTaskQueue, q.blockPendPool)
	}

	// Cancel aborts a fetch request, returning all pending hashes to the task queue.
	func (q *queue) cancel(request *fetchRequest, taskQueue *prque.Prque, pendPool map[string]*fetchRequest) {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		if request.From > 0 {
			taskQueue.Push(request.From, -float32(request.From))
		}
		for hash, index := range request.Hashes {
			taskQueue.Push(hash, float32(index))
		}
		for _, header := range request.Headers {
			taskQueue.Push(header, -float32(header.Number.Uint64()))
		}
		delete(pendPool, request.Peer.id)
	}

## ScheduleSkeleton
예약 방법은 이미 좋은 헤더를 가져 전달됩니다. 일정 (UINT64 헤더에서 [] * types.Header). 파라미터 ScheduleSkeleton 골격의 함수 골격 및 충진 다음 요청이다. 나는 소위 골격 (192) 블록에 요청 헤더 영역을 참조하고 들어오는 모든 ScheduleSkeleton 먼저 헤더로 돌아갑니다. 예약 기능 만 ScheduleSkeleton 기능에, 당신은 또한 그 실종 헤더 영역의 스케줄링을 다운로드 할 필요가있는 동안, 큐 스케줄링 블록 본체와 영수증을 다운로드해야합니다.

	// ScheduleSkeleton adds a batch of header retrieval tasks to the queue to fill
	// up an already retrieved header skeleton.
	func (q *queue) ScheduleSkeleton(from uint64, skeleton []*types.Header) {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		// No skeleton retrieval can be in progress, fail hard if so (huge implementation bug)
		if q.headerResults != nil {
			panic("skeleton assembly already in progress")
		}
		// Shedule all the header retrieval tasks for the skeleton assembly
		//이 방법은 거짓이기 때문에 골격가 호출되지 않은 경우. 그래서 여기 수행 일부 초기화를 넣어.
		q.headerTaskPool = make(map[uint64]*types.Header)
		q.headerTaskQueue = prque.New()
		q.headerPeerMiss = make(map[string]map[uint64]struct{}) // Reset availability to correct invalid chains
		q.headerResults = make([]*types.Header, len(skeleton)*MaxHeaderFetch)
		q.headerProced = 0
		q.headerOffset = from
		q.headerContCh = make(chan bool, 1)
	
		for i, header := range skeleton {
			index := from + uint64(i*MaxHeaderFetch)
			//모든 MaxHeaderFetch 지금까지 헤더가
			q.headerTaskPool[index] = header
			q.headerTaskQueue.Push(index, -float32(index))
		}
	}

### ReserveHeaders
이 방법에서 그것은 단지 골격 모델 호출됩니다. 피어 사용 예약 된 공간 헤더 작업을 가져옵니다.
	
	// ReserveHeaders reserves a set of headers for the given peer, skipping any
	// previously failed batches.
	func (q *queue) ReserveHeaders(p *peerConnection, count int) *fetchRequest {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		// Short circuit if the peer's already downloading something (sanity check to
		// not corrupt state)
		if _, ok := q.headerPendPool[p.id]; ok {
			return nil
		}
		// Retrieve a batch of hashes, skipping previously failed ones
		//이전에 실패한 노드 스킵, 큐에서 획득.
		send, skip := uint64(0), []uint64{}
		for send == 0 && !q.headerTaskQueue.Empty() {
			from, _ := q.headerTaskQueue.Pop()
			if q.headerPeerMiss[p.id] != nil {
				if _, ok := q.headerPeerMiss[p.id][from.(uint64)]; ok {
					skip = append(skip, from.(uint64))
					continue
				}
			}
			send = from.(uint64)
		}
		// Merge all the skipped batches back
		for _, from := range skip {
			q.headerTaskQueue.Push(from, -float32(from))
		}
		// Assemble and return the block download request
		if send == 0 {
			return nil
		}
		request := &fetchRequest{
			Peer: p,
			From: send,
			Time: time.Now(),
		}
		q.headerPendPool[p.id] = request
		return request
	}


### DeliverHeaders

	
	// DeliverHeaders injects a header retrieval response into the header results
	// cache. This method either accepts all headers it received, or none of them
	// if they do not map correctly to the skeleton.
	//수신 한 모든 헤더 영역이 방법 중 모두 또는 모든 거부 (만약 위하지 뼈대 맵)
	// If the headers are accepted, the method makes an attempt to deliver the set
	// of ready headers to the processor to keep the pipeline full. However it will
	// not block to prevent stalling other pending deliveries.
	//헤더 영역이 수신되면,이 방법은 위의 headerProcCh 관에게 전달하려고합니다. 그러나이 방법은 배달을 차단하지 않습니다. 당신이 수익을 제공 할 수없는 경우에, 전달하려고합니다.
	func (q *queue) DeliverHeaders(id string, headers []*types.Header, headerProcCh chan []*types.Header) (int, error) {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		// Short circuit if the data was never requested
		request := q.headerPendPool[id]
		if request == nil {
			return 0, errNoFetchesPending
		}
		headerReqTimer.UpdateSince(request.Time)
		delete(q.headerPendPool, id)
	
		// Ensure headers can be mapped onto the skeleton chain
		target := q.headerTaskPool[request.From].Hash()
	
		accepted := len(headers) == MaxHeaderFetch
		if accepted { //먼저, 일치 할 필요성의 길이는, 그 다음 마지막으로 블록 번호의 해쉬 값을 확인하고, 해당 블록에 대한 여부.
			if headers[0].Number.Uint64() != request.From {
				log.Trace("First header broke chain ordering", "peer", id, "number", headers[0].Number, "hash", headers[0].Hash(), request.From)
				accepted = false
			} else if headers[len(headers)-1].Hash() != target {
				log.Trace("Last header broke skeleton structure ", "peer", id, "number", headers[len(headers)-1].Number, "hash", headers[len(headers)-1].Hash(), "expected", target)
				accepted = false
			}
		}
		if accepted {//하기 위해 블록의 블록 번호의 모든 부분을 확인하고, 링크가 올바른지한다.
			for i, header := range headers[1:] {
				hash := header.Hash()
				if want := request.From + 1 + uint64(i); header.Number.Uint64() != want {
					log.Warn("Header broke chain ordering", "peer", id, "number", header.Number, "hash", hash, "expected", want)
					accepted = false
					break
				}
				if headers[i].Hash() != header.ParentHash {
					log.Warn("Header broke chain ancestry", "peer", id, "number", header.Number, "hash", hash)
					accepted = false
					break
				}
			}
		}
		// If the batch of headers wasn't accepted, mark as unavailable
		if !accepted { //허용되지 않으면 피어 때문에이 작업이 마크에 실패합니다. 다음 요청이 피어에 전달되지 않습니다
			log.Trace("Skeleton filling not accepted", "peer", id, "from", request.From)
	
			miss := q.headerPeerMiss[id]
			if miss == nil {
				q.headerPeerMiss[id] = make(map[uint64]struct{})
				miss = q.headerPeerMiss[id]
			}
			miss[request.From] = struct{}{}
	
			q.headerTaskQueue.Push(request.From, -float32(request.From))
			return 0, errors.New("delivery not accepted")
		}
		// Clean up a successful fetch and try to deliver any sub-results
		copy(q.headerResults[request.From-q.headerOffset:], headers)
		delete(q.headerTaskPool, request.From)
	
		ready := 0
		for q.headerProced+ready < len(q.headerResults) && q.headerResults[q.headerProced+ready] != nil {//헤더의 도착의 계산까지 전달 될 수 headerResults 얼마나 많은 데이터를 허용한다.
			ready += MaxHeaderFetch
		}
		if ready > 0 {
			// Headers are ready for delivery, gather them and push forward (non blocking)
			process := make([]*types.Header, ready)
			copy(process, q.headerResults[q.headerProced:q.headerProced+ready])
			//게시하려고
			select {
			case headerProcCh <- process:
				log.Trace("Pre-scheduled new headers", "peer", id, "count", len(process), "from", process[0].Number)
				q.headerProced += len(process)
			default:
			}
		}
		// Check for termination and return
		if len(q.headerTaskPool) == 0 {
			//이 채널은 채널 데이터가 수신되는 경우, 모든 헤더 작업에 대한 설명이 완료되었습니다 더 중요하다.
			q.headerContCh <- false
		}
		return len(headers), nil
	}

RetrieveHeaders은 수행하지 않은 최신 일정의 경우 ScheduleSkeleton 함수가 호출되지 않습니다. 마지막 호출이 완료되면 그래서, 그것은 상태를 재설정 결과를 얻기 위해이 방법을 사용합니다.
	
	// RetrieveHeaders retrieves the header chain assemble based on the scheduled
	// skeleton.
	func (q *queue) RetrieveHeaders() ([]*types.Header, int) {
		q.lock.Lock()
		defer q.lock.Unlock()
	
		headers, proced := q.headerResults, q.headerProced
		q.headerResults, q.headerProced = nil, 0
	
		return headers, proced
	}