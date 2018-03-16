## nonceHeap
nonceHeap heap.Interface는 데이터 구조, 힙을 구현하는 데 사용되는 데이터 구조를 구현합니다. 문서 heap.Interface 설명에서 기본 구현은 최소한의 힙입니다.

h를 배열 한 배열의 데이터는 다음 조건을 만족하는 경우이다. 그런 다음 최소 힙 시간입니다.

	!h.Less(j, i) for 0 <= i < h.Len() and 2*i+1 <= j <= 2*i+2 and j < h.Len()
	//어레이가 완전한 바이너리 트리로 간주되어, 첫번째 요소는 루트, 두 번째 및 세 번째 요소 인 두 가지의 루트
	//이것은 차례로 전 루트는 두 가지가 * 2 다음이다 그래서 만약 아래로 밀려 난 + 2, 2 * 내가 + 2.
	//최소 힙은 그 두 가지보다 더 할 수없는 뿌리로 정의됩니다. 코드 상술과 같이 정의된다.
	heap.Interface 정의
	
	우리는 다음과 같은 인터페이스가 힙 힙 구조로 구현 될 수있는 몇 가지 방법을 사용할 수 있습니다 만날 수있는 데이터 구조를 정의해야합니다.
	type Interface interface {
		sort.Interface
		Push(x interface{}) //소자 렌 ()로 추가 x가 최종 증가 X
		Pop() interface{}   //제거하고 요소를 반환 렌 () - 1 제거하고 마지막 요소를 반환
	}

nonceHeap 코드 분석.

	// nonceHeap is a heap.Interface implementation over 64bit unsigned integers for
	// retrieving sorted transactions from the possibly gapped future queue.
	type nonceHeap []uint64
	
	func (h nonceHeap) Len() int		   { return len(h) }
	func (h nonceHeap) Less(i, j int) bool { return h[i] < h[j] }
	func (h nonceHeap) Swap(i, j int)	  { h[i], h[j] = h[j], h[i] }
	
	func (h *nonceHeap) Push(x interface{}) {
		*h = append(*h, x.(uint64))
	}
	
	func (h *nonceHeap) Pop() interface{} {
		old := *h
		n := len(old)
		x := old[n-1]
		*h = old[0 : n-1]
		return x
	}


## txSortedMap

txSortedMap는, 스토리지는 아래의 계정으로 모든 거래입니다.

구조

	// txSortedMap is a nonce->transaction hash map with a heap based index to allow
	// iterating over the contents in a nonce-incrementing way.
	//txSortedMap는이, 힙 인덱스 nonce-&gt; 해시 맵 거래입니다
	//그것은 넌스 점진적으로 반복적 인 내용을 수 있습니다.
	
	type Transactions []*Transaction 

	type txSortedMap struct {
		items map[uint64]*types.Transaction // Hash map storing the transaction data
		index *nonceHeap					// Heap of nonces of all the stored transactions (non-strict mode)
		cache types.Transactions		// 이미 캐시에 정렬 된 거래의 캐시는 거래를 분류하고있다.
	}

넣고 가져 오기, 맵에 삽입 된 거래에 넣어 지정된 넌스를 획득하기위한 트랜잭션을 가져옵니다.
	
	// Get retrieves the current transactions associated with the given nonce.
	func (m *txSortedMap) Get(nonce uint64) *types.Transaction {
		return m.items[nonce]
	}
	
	// Put inserts a new transaction into the map, also updating the map's nonce
	// index. If a transaction already exists with the same nonce, it's overwritten.
	//새로운 트랜잭션이 맵에 삽입 넣어,지도 동시에 넌스 인덱스를 업데이트됩니다. 트랜잭션이 이미 존재하는 경우, 그것을 적용했습니다. 동시에 캐시 된 데이터가 삭제됩니다.
	func (m *txSortedMap) Put(tx *types.Transaction) {
		nonce := tx.Nonce()
		if m.items[nonce] == nil {
			heap.Push(m.index, nonce)
		}
		m.items[nonce], m.cache = tx, nil
	}

앞으로 모든 거래의 비표가 임계 값보다 작 삭제하는 데 사용됩니다. 그런 다음 제거 된 모든 트랜잭션을 반환합니다.
	
	// Forward removes all transactions from the map with a nonce lower than the
	// provided threshold. Every removed transaction is returned for any post-removal
	// maintenance.
	func (m *txSortedMap) Forward(threshold uint64) types.Transactions {
		var removed types.Transactions
	
		// Pop off heap items until the threshold is reached
		for m.index.Len() > 0 && (*m.index)[0] < threshold {
			nonce := heap.Pop(m.index).(uint64)
			removed = append(removed, m.items[nonce])
			delete(m.items, nonce)
		}
		// If we had a cached order, shift the front
		//캐시는 거래를 정렬됩니다.
		if m.cache != nil {
			m.cache = m.cache[len(removed):]
		}
		return removed
	}
	
필터, 모두가 필터 기능이 참 호출 트랜잭션을 반환하고 그 거래를 반환하게 제거합니다.
			
	// Filter iterates over the list of transactions and removes all of them for which
	// the specified function evaluates to true.
	func (m *txSortedMap) Filter(filter func(*types.Transaction) bool) types.Transactions {
		var removed types.Transactions
	
		// Collect all the transactions to filter out
		for nonce, tx := range m.items {
			if filter(tx) {
				removed = append(removed, tx)
				delete(m.items, nonce)
			}
		}
		// If transactions were removed, the heap and cache are ruined
		//트랜잭션은 삭제 및 힙 캐시가 파괴 된 경우
		if len(removed) > 0 {
			*m.index = make([]uint64, 0, len(m.items))
			for nonce := range m.items {
				*m.index = append(*m.index, nonce)
			}
			//우리는 힙을 다시 할 필요가
			heap.Init(m.index)
			//전무로 설정 캐시
			m.cache = nil
		}
		return removed
	}

캡 내부 항목의 수, 한계 이상의 모든 거래의 반환을 제한합니다.
	
	// Cap places a hard limit on the number of items, returning all transactions
	// exceeding that limit.
	//캡 내부 항목의 수, 한계 이상의 모든 거래의 반환을 제한합니다.
	func (m *txSortedMap) Cap(threshold int) types.Transactions {
		// Short circuit if the number of items is under the limit
		if len(m.items) <= threshold {
			return nil
		}
		// Otherwise gather and drop the highest nonce'd transactions
		var drops types.Transactions
	
		sort.Sort(*m.index) //대량 주문 작은에서 꼬리에서 제거됩니다.
		for size := len(m.items); size > threshold; size-- {
			drops = append(drops, m.items[(*m.index)[size-1]])
			delete(m.items, (*m.index)[size-1])
		}
		*m.index = (*m.index)[:threshold]
		//재건 힙
		heap.Init(m.index)
	
		// If we had a cache, shift the back
		if m.cache != nil {
			m.cache = m.cache[:len(m.cache)-len(drops)]
		}
		return drops
	}

Remove
	
	// Remove deletes a transaction from the maintained map, returning whether the
	// transaction was found.
	// 
	func (m *txSortedMap) Remove(nonce uint64) bool {
		// Short circuit if no transaction is present
		_, ok := m.items[nonce]
		if !ok {
			return false
		}
		// Otherwise delete the transaction and fix the heap index
		for i := 0; i < m.index.Len(); i++ {
			if (*m.index)[i] == nonce {
				heap.Remove(m.index, i)
				break
			}
		}
		delete(m.items, nonce)
		m.cache = nil
	
		return true
	}

Ready函数
	
	// Ready retrieves a sequentially increasing list of transactions starting at the
	// provided nonce that is ready for processing. The returned transactions will be
	// removed from the list.
	//지정된 넌스, 연속 거래의 처음부터 복귀 할 준비가되었습니다. 공정 반환은 삭제됩니다.
	// Note, all transactions with nonces lower than start will also be returned to
	// prevent getting into and invalid state. This is not something that should ever
	// happen but better to be self correcting than failing!
	//항목 및 비활성 상태를 방지하기 위해, 시작보다 낮은 넌스 모든 거래는 반환됩니다 유의하시기 바랍니다 있습니다.
	//이 문제가 발생할 수 있지만, 자기 보정 실패하지 말아야 것이 아니다!
	func (m *txSortedMap) Ready(start uint64) types.Transactions {
		// Short circuit if no transactions are available
		if m.index.Len() == 0 || (*m.index)[0] > start {
			return nil
		}
		// Otherwise start accumulating incremental transactions
		var ready types.Transactions
		//작은 시작부터, 하나 개 증가 하나,
		for next := (*m.index)[0]; m.index.Len() > 0 && (*m.index)[0] == next; next++ {
			ready = append(ready, m.items[next])
			delete(m.items, next)
			heap.Pop(m.index)
		}
		m.cache = nil
	
		return ready
	}

평평하게, 트랜잭션 기반 넌스 종류의 목록을 반환합니다. 캐시와 캐시, 내부 필드를 반복적으로 수정하지 않고 사용할 수 있습니다.
	
	// Len returns the length of the transaction map.
	func (m *txSortedMap) Len() int {
		return len(m.items)
	}
	
	// Flatten creates a nonce-sorted slice of transactions based on the loosely
	// sorted internal representation. The result of the sorting is cached in case
	// it's requested again before any modifications are made to the contents.
	func (m *txSortedMap) Flatten() types.Transactions {
		// If the sorting was not cached yet, create and cache it
		if m.cache == nil {
			m.cache = make(types.Transactions, 0, len(m.items))
			for _, tx := range m.items {
				m.cache = append(m.cache, tx)
			}
			sort.Sort(types.TxByNonce(m.cache))
		}
		// Copy the cache to prevent accidental modifications
		txs := make(types.Transactions, len(m.cache))
		copy(txs, m.cache)
		return txs
	}

## txList
txList는 비표 종류에 따라, 동일한 계정 거래의 목록에 속한다. 실행 지속적인 거래를 저장하는 데 사용할 수 있습니다. 비 연속 거래의 경우, 몇 가지 작은 다른 행동이있다.

구조
	
	// txList is a "list" of transactions belonging to an account, sorted by account
	// nonce. The same type can be used both for storing contiguous transactions for
	// the executable/pending queue; and for storing gapped transactions for the non-
	// executable/future queue, with minor behavioral changes.
	type txList struct {
		strict bool	 // 난스를 엄격하게 지속 여부를 논 스가 여부 엄격하게 연속 또는 불연속
		txs* TxSortedMap // 힙 거래 해시 맵 힙 지수에 따라 거래의 정렬 된 해시 맵 색인
	
		costcap *big.Int //가장 높은 값 GasPrice * GasLimit 내부의 모든 거래의 비용이 가장 높은 거래 가격 (균형을 초과하는 경우에만 재설정)
		gascap  *big.Int //가장 높은 지출 거래의 가장 높은 값 가스 제한, 내부 GasPrice 모든 거래 (블록 제한을 초과하는 경우에만 재설정)
	}
비표가 있는지 여부 같은으로 주어진 무역 거래를 반환 겹칩니다.

	// Overlaps returns whether the transaction specified has the same nonce as one
	// already contained within the list.
	// 
	func (l *txList) Overlaps(tx *types.Transaction) bool {
		return l.txs.Get(tx.Nonce()) != nil
	}
새로운 거래 이전 거래의 값이 일정 비율의 priceBump을 GasPrice보다 높은 것으로 경우, 이러한 작업을 수행 할 추가, 그것은 이전 거래를 대체합니다.
	
	// Add tries to insert a new transaction into the list, returning whether the
	// transaction was accepted, and if yes, any previous transaction it replaced.
	//받은 경우 모든 거래는 대체 될 전에 트랜잭션 반환, 수신 여부, 새 트랜잭션을 삽입하는 시도를 추가합니다.
	// If the new transaction is accepted into the list, the lists' cost and gas
	// thresholds are also potentially updated.
	//새 트랜잭션이 수신 될 경우, 총 비용은 업데이트 및 가스 제한됩니다.
	func (l *txList) Add(tx *types.Transaction, priceBump uint64) (bool, *types.Transaction) {
		// If there's an older better transaction, abort
		//이전 무역 존재하는 경우. 그리고 이전보다 어느 정도 높은 새로운 거래의 가격. 그럼 대체했다.
		old := l.txs.Get(tx.Nonce())
		if old != nil {
			threshold := new(big.Int).Div(new(big.Int).Mul(old.GasPrice(), big.NewInt(100+int64(priceBump))), big.NewInt(100))
			if threshold.Cmp(tx.GasPrice()) >= 0 {
				return false, nil
			}
		}
		// Otherwise overwrite the old transaction with the current one
		l.txs.Put(tx)
		if cost := tx.Cost(); l.costcap.Cmp(cost) < 0 {
			l.costcap = cost
		}
		if gas := tx.Gas(); l.gascap.Cmp(gas) < 0 {
			l.gascap = gas
		}
		return true, old
	}
앞으로 모든 거래의 특정 값보다 작은 비표를 삭제합니다.

	// Forward removes all transactions from the list with a nonce lower than the
	// provided threshold. Every removed transaction is returned for any post-removal
	// maintenance.
	func (l *txList) Forward(threshold uint64) types.Transactions {
		return l.txs.Forward(threshold)
	}

Filter,
	
	// Filter removes all transactions from the list with a cost or gas limit higher
	// than the provided thresholds. Every removed transaction is returned for any
	// post-removal maintenance. Strict-mode invalidated transactions are also
	// returned.
	//필터 값을 제공 또는 거래의 gasLimit 비용보다 모두 높은 제거합니다. 공정 반환은 추가 처리를 위해 제거. 엄격 모드에서 같은 유효하지 않은 트랜잭션이 모두 반환됩니다.
	// 
	// This method uses the cached costcap and gascap to quickly decide if there's even
	// a point in calculating all the costs or if the balance covers all. If the threshold
	// is lower than the costgas cap, the caps will be reset to a new high after removing
	// the newly invalidated transactions.
	//이 방법은 신속하게 모든 트랜잭션을 통과해야하는지 여부를 결정하기 위해 캐시 costcap 및 gascap를 사용합니다. 한도는 캐시 costcap 및 gascap보다 작은 경우, 불법 거래의 제거는 값 costcap 및 gascap를 업데이트 한 후 후.

	func (l *txList) Filter(costLimit, gasLimit *big.Int) (types.Transactions, types.Transactions) {
		// If all transactions are below the threshold, short circuit
		//모든 거래는 한계보다 작은 경우, 직접 돌아갑니다.
		if l.costcap.Cmp(costLimit) <= 0 && l.gascap.Cmp(gasLimit) <= 0 {
			return nil, nil
		}
		l.costcap = new(big.Int).Set(costLimit) // Lower the caps to the thresholds
		l.gascap = new(big.Int).Set(gasLimit)
	
		// Filter out all the transactions above the account's funds
		removed := l.txs.Filter(func(tx *types.Transaction) bool { return tx.Cost().Cmp(costLimit) > 0 || tx.Gas().Cmp(gasLimit) > 0 })
	
		// If the list was strict, filter anything above the lowest nonce
		var invalids types.Transactions
	
		if l.strict && len(removed) > 0 {
			//최소보다 큰 모든 넌스은 비표 트랜잭션 작업이 유효했다 제거됩니다.
			//엄격 모드에서는이 거래도 제거.
			lowest := uint64(math.MaxUint64)
			for _, tx := range removed {
				if nonce := tx.Nonce(); lowest > nonce {
					lowest = nonce
				}
			}
			invalids = l.txs.Filter(func(tx *types.Transaction) bool { return tx.Nonce() > lowest })
		}
		return removed, invalids
	}

반환 캡 기능은 트랜잭션의 수를 초과. 트랜잭션의 수는 임계 값의 제거 및 복귀 후 다음 트랜잭션을 초과하는 경우.
	
	// Cap places a hard limit on the number of items, returning all transactions
	// exceeding that limit.
	func (l *txList) Cap(threshold int) types.Transactions {
		return l.txs.Cap(threshold)
	}

제거 엄격 모드뿐만 아니라, 주어진 거래보다 모든 넌스 난스 더 삭제하면 주어진 논스 거래를 제거하고 돌아갑니다.
	
	// Remove deletes a transaction from the maintained list, returning whether the
	// transaction was found, and also returning any transaction invalidated due to
	// the deletion (strict mode only).
	func (l *txList) Remove(tx *types.Transaction) (bool, types.Transactions) {
		// Remove the transaction from the set
		nonce := tx.Nonce()
		if removed := l.txs.Remove(nonce); !removed {
			return false, nil
		}
		// In strict mode, filter out non-executable transactions
		if l.strict {
			return true, l.txs.Filter(func(tx *types.Transaction) bool { return tx.Nonce() > nonce })
		}
		return true, nil
	}

준비, 렌, 빈, 직접 txSortedMap의 해당 메소드를 호출 평평.

	// Ready retrieves a sequentially increasing list of transactions starting at the
	// provided nonce that is ready for processing. The returned transactions will be
	// removed from the list.
	//
	// Note, all transactions with nonces lower than start will also be returned to
	// prevent getting into and invalid state. This is not something that should ever
	// happen but better to be self correcting than failing!
	func (l *txList) Ready(start uint64) types.Transactions {
		return l.txs.Ready(start)
	}

	// Len returns the length of the transaction list.
	func (l *txList) Len() int {
		return l.txs.Len()
	}

	// Empty returns whether the list of transactions is empty or not.
	func (l *txList) Empty() bool {
		return l.Len() == 0
	}

	// Flatten creates a nonce-sorted slice of transactions based on the loosely
	// sorted internal representation. The result of the sorting is cached in case
	// it's requested again before any modifications are made to the contents.
	func (l *txList) Flatten() types.Transactions {
		return l.txs.Flatten()
	}


## priceHeap
priceHeap 힙을 구축하는 가격의 크기에 따라, 최소 힙입니다.
	
	// priceHeap is a heap.Interface implementation over transactions for retrieving
	// price-sorted transactions to discard when the pool fills up.
	type priceHeap []*types.Transaction
	
	func (h priceHeap) Len() int		   { return len(h) }
	func (h priceHeap) Less(i, j int) bool { return h[i].GasPrice().Cmp(h[j].GasPrice()) < 0 }
	func (h priceHeap) Swap(i, j int)	  { h[i], h[j] = h[j], h[i] }
	
	func (h *priceHeap) Push(x interface{}) {
		*h = append(*h, x.(*types.Transaction))
	}
	
	func (h *priceHeap) Pop() interface{} {
		old := *h
		n := len(old)
		x := old[n-1]
		*h = old[0 : n-1]
		return x
	}


## txPricedList
데이터 구조와 구축이 txPricedList 가격은 정렬 힙 (heap)을 기반으로, 점진적으로 가격에 따라 거래를 처리 할 수 ​​있습니다.

	
	// txPricedList is a price-sorted heap to allow operating on transactions pool
	// contents in a price-incrementing way.
	type txPricedList struct {
		all모든 거래의지도에 대한 포인터 모든 거래의 맵 * 맵 [common.Hash] * types.Transaction // 포인터
		items  *priceHeap						  // Heap of prices of all the stored transactions
		stales int								 // Number of stale price points to (re-heap trigger)
	}
	
	// newTxPricedList creates a new price-sorted transaction heap.
	func newTxPricedList(all *map[common.Hash]*types.Transaction) *txPricedList {
		return &txPricedList{
			all:   all,
			items: new(priceHeap),
		}
	}

Put

	// Put inserts a new transaction into the heap.
	func (l *txPricedList) Put(tx *types.Transaction) {
		heap.Push(l.items, tx)
	}

Removed

	// Removed notifies the prices transaction list that an old transaction dropped
	// from the pool. The list will just keep a counter of stale objects and update
	// the heap if a large enough ratio of transactions go stale.
	//txPricedList을 통지 제거 오래된 무역이 삭제됩니다있다. TxPricedList 힙 정보를 업데이트 할시기를 결정하기 위해 카운터를 사용합니다.
	func (l *txPricedList) Removed() {
		// Bump the stale counter, but exit if still too low (< 25%)
		l.stales++
		if l.stales <= len(*l.items)/4 {
			return
		}
		// Seems we've reached a critical number of stale transactions, reheap
		reheap := make(priceHeap, 0, len(*l.all))
	
		l.stales, l.items = 0, &reheap
		for _, tx := range *l.all {
			*l.items = append(*l.items, tx)
		}
		heap.Init(l.items)
	}

캡은 주어진 임계 값 아래의 모든 거래 가격을 찾는 데 사용. 가격표 및 반환에서 그들을 삭제합니다.
	
	// Cap finds all the transactions below the given price threshold, drops them
	// from the priced list and returs them for further removal from the entire pool.
	func (l *txPricedList) Cap(threshold *big.Int, local *accountSet) types.Transactions {
		drop := make(types.Transactions, 0, 128) // Remote underpriced transactions to drop
		save := make(types.Transactions, 0, 64)  // Local underpriced transactions to keep
	
		for len(*l.items) > 0 {
			// Discard stale transactions if found during cleanup
			tx := heap.Pop(l.items).(*types.Transaction)
			if _, ok := (*l.all)[tx.Hash()]; !ok {
				//당신이 삭제 발견되면 업데이트가 카운터 상태
				l.stales--
				continue
			}
			// Stop the discards if we've reached the threshold
			if tx.GasPrice().Cmp(threshold) >= 0 {
				//가격이 임계 값보다 작은 경우, 종료
				save = append(save, tx)
				break
			}
			// Non stale transaction found, discard unless local
			if local.containsTx(tx) {  //거래는 지방 제거하지 않습니다
				save = append(save, tx)
			} else {
				drop = append(drop, tx)
			}
		}
		for _, tx := range save {
			heap.Push(l.items, tx)
		}
		return drop
	}


이 매겨져, 가장 저렴한 거래 내부 txPricedList보다 저렴를 송신하는 것은 현재 동일하거나 저렴 있는지 확인합니다.
	
	// Underpriced checks whether a transaction is cheaper than (or as cheap as) the
	// lowest priced transaction currently being tracked.
	func (l *txPricedList) Underpriced(tx *types.Transaction, local *accountSet) bool {
		// Local transactions cannot be underpriced
		if local.containsTx(tx) {
			return false
		}
		// Discard stale price points if found at the heap start
		for len(*l.items) > 0 {
			head := []*types.Transaction(*l.items)[0]
			if _, ok := (*l.all)[head.Hash()]; !ok {
				l.stales--
				heap.Pop(l.items)
				continue
			}
			break
		}
		// Check if the transaction is underpriced or not
		if len(*l.items) == 0 {
			log.Error("Pricing query for empty pool") // This cannot happen, print to catch programming errors
			return false
		}
		cheapest := []*types.Transaction(*l.items)[0]
		return cheapest.GasPrice().Cmp(tx.GasPrice()) >= 0
	}

, 폐기 가장 저렴한 거래를 현재 목록 및 반환에서 제거 이들의 특정 번호를 찾을 수 있습니다.
	
	// Discard finds a number of most underpriced transactions, removes them from the
	// priced list and returns them for further removal from the entire pool.
	func (l *txPricedList) Discard(count int, local *accountSet) types.Transactions {
		drop := make(types.Transactions, 0, count) // Remote underpriced transactions to drop
		save := make(types.Transactions, 0, 64)	// Local underpriced transactions to keep
	
		for len(*l.items) > 0 && count > 0 {
			// Discard stale transactions if found during cleanup
			tx := heap.Pop(l.items).(*types.Transaction)
			if _, ok := (*l.all)[tx.Hash()]; !ok {
				l.stales--
				continue
			}
			// Non stale transaction found, discard unless local
			if local.containsTx(tx) {
				save = append(save, tx)
			} else {
				drop = append(drop, tx)
				count--
			}
		}
		for _, tx := range save {
			heap.Push(l.items, tx)
		}
		return drop
	}


## accountSet
accountSet 개체의 수집 및 처리 계정의 서명이다.
	
	// accountSet is simply a set of addresses to check for existence, and a signer
	// capable of deriving addresses from transactions.
	type accountSet struct {
		accounts map[common.Address]struct{}
		signer   types.Signer
	}
	
	// newAccountSet creates a new address set with an associated signer for sender
	// derivations.
	func newAccountSet(signer types.Signer) *accountSet {
		return &accountSet{
			accounts: make(map[common.Address]struct{}),
			signer:   signer,
		}
	}
	
	// contains checks if a given address is contained within the set.
	func (as *accountSet) contains(addr common.Address) bool {
		_, exist := as.accounts[addr]
		return exist
	}
	
	// containsTx checks if the sender of a given tx is within the set. If the sender
	// cannot be derived, this method returns false.
	//containsTx은 컬렉션 내의 여부를 확인하는 발신자에게 주어진 송신기 (TX). 보낸 사람이 계산 할 수없는 경우,이 메소드는 false를 돌려줍니다.
	func (as *accountSet) containsTx(tx *types.Transaction) bool {
		if addr, err := types.Sender(as.signer, tx); err == nil {
			return as.contains(addr)
		}
		return false
	}
	
	// add inserts a new address into the set to track.
	func (as *accountSet) add(addr common.Address) {
		as.accounts[addr] = struct{}{}
	}


## txJournal

txJournal 트랜잭션의 목적은 트랜잭션이 실행되지 있도록 로컬 만들어 저장 노드가 다시 시작된 후에 계속해서 실행되는 원형 로그 트랜잭션이다.
구조
	

	// txJournal is a rotating log of transactions with the aim of storing locally
	// created transactions to allow non-executed ones to survive node restarts.
	type txJournal struct {
		path   string	 거래에 트랜잭션을 저장하기 // 파일 시스템 경로를 저장하는 파일 시스템 경로로 사용됩니다.
		writer io.WriteCloser //출력 스트림은 새로운 트랜잭션을 작성하기위한 출력 스트림에 새로운 트랜잭션을 작성합니다.
	}

newTxJournal 새 트랜잭션 로그를 생성 할 수 있습니다.

	// newTxJournal creates a new transaction journal to
	func newTxJournal(path string) *txJournal {
		return &txJournal{
			path: path,
		}
	}

load方法从磁盘解析交易,然后调用add回调方法.
	
	// load parses a transaction journal dump from disk, loading its contents into
	// the specified pool.
	func (journal *txJournal) load(add func(*types.Transaction) error) error {
		// Skip the parsing if the journal file doens't exist at all
		if _, err := os.Stat(journal.path); os.IsNotExist(err) {
			return nil
		}
		// Open the journal for loading any past transactions
		input, err := os.Open(journal.path)
		if err != nil {
			return err
		}
		defer input.Close()
	
		// Inject all transactions from the journal into the pool
		stream := rlp.NewStream(input, 0)
		total, dropped := 0, 0
	
		var failure error
		for {
			// Parse the next transaction and terminate on error
			tx := new(types.Transaction)
			if err = stream.Decode(tx); err != nil {
				if err != io.EOF {
					failure = err
				}
				break
			}
			// Import the transaction and bump the appropriate progress counters
			total++
			if err = add(tx); err != nil {
				log.Debug("Failed to add journaled transaction", "err", err)
				dropped++
				continue
			}
		}
		log.Info("Loaded local transaction journal", "transactions", total, "dropped", dropped)
	
		return failure
	}
방법을 삽입, rlp.Encode 쓰기 작가를 호출
	
	// insert adds the specified transaction to the local disk journal.
	func (journal *txJournal) insert(tx *types.Transaction) error {
		if journal.writer == nil {
			return errNoActiveJournal
		}
		if err := rlp.Encode(journal.writer, tx); err != nil {
			return err
		}
		return nil
	}

현재 거래 구덩이를 기반으로 트랜잭션을 재생하는 방법을 회전,

	// rotate regenerates the transaction journal based on the current contents of
	// the transaction pool.
	func (journal *txJournal) rotate(all map[common.Address]types.Transactions) error {
		// Close the current journal (if any is open)
		if journal.writer != nil {
			if err := journal.writer.Close(); err != nil {
				return err
			}
			journal.writer = nil
		}
		// Generate a new journal with the contents of the current pool
		replacement, err := os.OpenFile(journal.path+".new", os.O_WRONLY|os.O_CREATE|os.O_TRUNC, 0755)
		if err != nil {
			return err
		}
		journaled := 0
		for _, txs := range all {
			for _, tx := range txs {
				if err = rlp.Encode(replacement, tx); err != nil {
					replacement.Close()
					return err
				}
			}
			journaled += len(txs)
		}
		replacement.Close()
	
		// Replace the live journal with the newly generated one
		if err = os.Rename(journal.path+".new", journal.path); err != nil {
			return err
		}
		sink, err := os.OpenFile(journal.path, os.O_WRONLY|os.O_APPEND, 0755)
		if err != nil {
			return err
		}
		journal.writer = sink
		log.Info("Regenerated local transaction journal", "transactions", journaled, "accounts", len(all))
	
		return nil
	}

close

	// close flushes the transaction journal contents to disk and closes the file.
	func (journal *txJournal) close() error {
		var err error
	
		if journal.writer != nil {
			err = journal.writer.Close()
			journal.writer = nil
		}
		return err
	}
