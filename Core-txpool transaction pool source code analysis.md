txpool 주로 말단부와 지역을 가진 서면 블록을 제출 한 현재의 트랜잭션 (transaction)를 기다리는 저장하는 데 사용.

거래의 두 가지 유형이 있습니다 txpool,
1. 제출 만 실행할 수 있습니다 (예를 들어, 비표가 너무 높은 경우) 내부의 대기 큐에서 실행할 수 없습니다.
2. 실행을 기다리고 안쪽에 대기, 실행 기다리고입니다.

관점에서 Txpool 테스트 케이스는 txpool 주요 특징은 다음과 같은 점이다.

1. 자금 부족, 불충분 한 가스를 포함하는 트랜잭션 검증 기능은, 논스이 너무 낮 가치 값은 법적, 음수가 될 수 없습니다.
2. 능력은 난스에게 현지 거래의 현재 계정 상태 이상을 캐시합니다. 큐 필드에 저장된. 트랜잭션이 보류 필드에 저장되어 실행될 수있는 경우
3. 같은 난스 동일한 사용자 트랜잭션은 GasPrice 그 최대 저장됩니다. 다른 삽입 실패.
계정이 돈이없는 4. 경우, 큐에 대기하고 해당 공정 계정이 삭제됩니다.
5. 계정 잔액이 트랜잭션의 임계 수보다 작은 경우, 해당 공정이 삭제, 내부에 계류중인 이동에서 효과적인 공정 큐. 에서가 방송된다.
txPool는 일부 지원 6. (트랜잭션의 여분의 가격의 비율은 동일 논스이다) PriceLimit (최저 GasPrice 제한을 제거) PriceBump AccountSlots가 GlobalSlots (최대 국제 출원 큐) (각 계정 슬롯의 최소값을 보류)을 한정 AccountQueue는 (큐의 큐에 최대 대기 시간) GlobalQueue (전체 최대 큐잉) 수명 (최소 슬롯 각 계정 큐잉)
우선 GasPrice 7. 제한된 자원에 따라 교체.
8. 공정 사용 지역의 저널 기능이 디스크에 저장됩니다, 재시작 한 후 다시 가져옵니다. 원격 거래는하지 않습니다.




데이터 구조
	
	// TxPool contains all currently known transactions. Transactions
	// enter the pool when they are received from the network or submitted
	// locally. They exit the pool when they are included in the blockchain.
	//TxPool는 트랜잭션의 현재 지식은, 현재의 네트워크 트랜잭션을 수신하거나 TxPool에 추가 지역 박람회에 의해 제출 된이 포함되어 있습니다.
	//들이 제거되면 블록이 시간 체인에 추가되었다.
	// The pool separates processable transactions (which can be applied to the
	// current state) and future transactions. Transactions move between those
	// two states over time as they are received and processed.
	//TxPool는 실행 거래로 향후 거래 (현재의 상태에 적용 할 수 있습니다). 두 상태 전환 사이에 무역,
	type TxPool struct {
		config	   TxPoolConfig
		chainconfig  *params.ChainConfig
		chain		blockChain
		gasPrice	 *big.Int		 // 가장 낮은 한계 GasPrice
		txFeed	   event.Feed		  // txFeed에 의해 TxPool 메시지를 구독
		scope		event.SubscriptionScope
		chainHeadCh  chan ChainHeadEvent  //지역이 여기에 통지 할 때 새로운 헤더가 생성 될 때 메시지의 헤더 영역에 가입
		chainHeadSub event.Subscription   //가입자 메시지 헤더 영역.
		signer	   types.Signer	  // 트랜잭션 서명 프로세스를 캡슐화합니다.
		mu		   sync.RWMutex
	
		currentState  *state.StateDB	  // Current state in the blockchain head
		pendingState  *state.ManagedState // Pending state tracking virtual nonces
		currentMaxGas *big.Int		// 거래에 대한 현재의 가스 제한은 현재 GasLimit의 상한 거래되고 캡
	
		locals  *accountSet //지역 무역 규칙의 추방 면제 evicion 규칙에서 exepmt하는 로컬 트랜잭션 세트
		journal *txJournal  //디스크에 백업 할 로컬 트랜잭션의 저널은 디스크 현지 박람회에 기록됩니다
	
		pending map[common.Address]*txList	 // 현재의 모든 거래의 모든 현재 처리 가능한 트랜잭션을 처리 할 수 ​​있습니다
		queue   map[common.Address]*txList	 거래 // 대기하지만 비 처리 가능한 거래는 현재 처리되지
		beats   map[common.Address]time.Time   마지막으로 각 알려진 계정 알려진 모든 계정의 하트 비트 정보 // 마지막 하트 비트
		all 조회는 모든 거래를 찾을 수 있도록 [common.Hash] * types.Transaction // 모든 거래를 매핑
		priced  *txPricedList				  // 모든 거래는 가격 거래에 의해 가격 정렬 기준으로 정렬
	
		wg sync.WaitGroup // for shutdown sync
	
		homestead bool  //홈 버전
	}



건설
	
	
	// NewTxPool creates a new transaction pool to gather, sort and filter inbound
	// trnsactions from the network.
	func NewTxPool(config TxPoolConfig, chainconfig *params.ChainConfig, chain blockChain) *TxPool {
		// Sanitize the input to ensure no vulnerable gas prices are set
		config = (&config).sanitize()
	
		// Create the transaction pool with its initial settings
		pool := &TxPool{
			config:	  config,
			chainconfig: chainconfig,
			chain:	   chain,
			signer:	  types.NewEIP155Signer(chainconfig.ChainId),
			pending:	 make(map[common.Address]*txList),
			queue:	   make(map[common.Address]*txList),
			beats:	   make(map[common.Address]time.Time),
			all:		 make(map[common.Hash]*types.Transaction),
			chainHeadCh: make(chan ChainHeadEvent, chainHeadChanSize),
			gasPrice:	new(big.Int).SetUint64(config.PriceLimit),
		}
		pool.locals = newAccountSet(pool.signer)
		pool.priced = newTxPricedList(&pool.all)
		pool.reset(nil, chain.CurrentBlock().Header())
	
		// If local transactions and journaling is enabled, load from disk
		//로컬 트랜잭션이 가능하고, 저널 디렉토리의 구성이 비어 있지 않은 경우, 지정된 디렉토리에서 로그를로드합니다.
		//이전 트랜잭션이 실패했을 수 있기 때문에 add 메소드를 호출 한 후 다음 로그받은 거래 기록을 누른 후. 트랜잭션 로그를 회전 할 수 있습니다.
		// 
		if !config.NoLocals && config.Journal != "" {
			pool.journal = newTxJournal(config.Journal)
	
			if err := pool.journal.load(pool.AddLocal); err != nil {
				log.Warn("Failed to load transaction journal", "err", err)
			}
			if err := pool.journal.rotate(pool.local()); err != nil {
				log.Warn("Failed to rotate transaction journal", "err", err)
			}
		}
		//블록 체인에서 blockchain의 이벤트는 이벤트에 가입 구독합니다.
		pool.chainHeadSub = pool.chain.SubscribeChainHeadEvent(pool.chainHeadCh)
	
		// Start the event loop and return
		pool.wg.Add(1)
		go pool.loop()
	
		return pool
	}

리셋 방법의 현 상태는 블록 사슬을 검색하고 블록 사슬 상태의 현재 풀 트랜잭션의 내용이 유효한지 확인하기 위해. 주요 기능은 다음과 같습니다 :

1. 때문에 헤더 영역의 교체, 그래서 원래 블록 거래의 일부 때문에 헤더 영역 및 무효의 교체는 트랜잭션이 부분은 새로운 블록을 삽입 기다리고 내부 txPool에 가입해야
2. 새의 currentState 및 pendingState 생성
3. 상태 변화 때문에. 상기 큐에 계류중인 트랜잭션의 일부가 안쪽으로 이동
4. 상태 변화 때문에 대기열 안에 계류중인 트랜잭션의 내부로 옮겼다.

코드를 재설정

	// reset retrieves the current state of the blockchain and ensures the content
	// of the transaction pool is valid with regard to the chain state.
	func (pool *TxPool) reset(oldHead, newHead *types.Header) {
		// If we're reorging an old state, reinject all dropped transactions
		var reinject types.Transactions
	
		if oldHead != nil && oldHead.Hash() != newHead.ParentHash {
			// If the reorg is too deep, avoid doing it (will happen during fast sync)
			oldNum := oldHead.Number.Uint64()
			newNum := newHead.Number.Uint64()
	
			if depth := uint64(math.Abs(float64(oldNum) - float64(newNum))); depth > 64 { //이전과 머리 사이의 격차의 새로운 머리 경우, 재구성을 취소
				log.Warn("Skipping deep transaction reorg", "depth", depth)
			} else {
				// Reorg seems shallow enough to pull in all transactions into memory
				var discarded, included types.Transactions
	
				var (
					rem = pool.chain.GetBlock(oldHead.Hash(), oldHead.Number.Uint64())
					add = pool.chain.GetBlock(newHead.Hash(), newHead.Number.Uint64())
				)
				//기존의 높이가 새보다 큰 경우. 당신은 모든 자세한 내용을 삭제해야합니다.
				for rem.NumberU64() > add.NumberU64() {
					discarded = append(discarded, rem.Transactions()...)
					if rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil {
						log.Error("Unrooted old chain seen by tx pool", "block", oldHead.Number, "hash", oldHead.Hash())
						return
					}
				}
				//새로운 높이가 이전보다 큰 경우가 증가 될 필요가있다.
				for add.NumberU64() > rem.NumberU64() {
					included = append(included, add.Transactions()...)
					if add = pool.chain.GetBlock(add.ParentHash(), add.NumberU64()-1); add == nil {
						log.Error("Unrooted new chain seen by tx pool", "block", newHead.Number, "hash", newHead.Hash())
						return
					}
				}
				//, 다시 볼 필요가 해시 다른 경우 같은 높이., 그들은 같은 해시 루트 노드를 찾을 수있다.
				for rem.Hash() != add.Hash() {
					discarded = append(discarded, rem.Transactions()...)
					if rem = pool.chain.GetBlock(rem.ParentHash(), rem.NumberU64()-1); rem == nil {
						log.Error("Unrooted old chain seen by tx pool", "block", oldHead.Number, "hash", oldHead.Hash())
						return
					}
					included = append(included, add.Transactions()...)
					if add = pool.chain.GetBlock(add.ParentHash(), add.NumberU64()-1); add == nil {
						log.Error("Unrooted new chain seen by tx pool", "block", newHead.Number, "hash", newHead.Hash())
						return
					}
				}
				//모두를 폐기하는 것은이 안에 찾을 수 있습니다, 그러나의 값은 포함되지 않습니다.
				//이러한 거래는 기다릴 필요에 따라 내부의 풀에 다시 삽입합니다.
				reinject = types.TxDifference(discarded, included)
			}
		}
		// Initialize the internal state to the current head
		if newHead == nil {
			newHead = pool.chain.CurrentBlock().Header() // Special case during testing
		}
		statedb, err := pool.chain.StateAt(newHead.Root)
		if err != nil {
			log.Error("Failed to reset txpool state", "err", err)
			return
		}
		pool.currentState = statedb
		pool.pendingState = state.ManageState(statedb)
		pool.currentMaxGas = newHead.GasLimit
	
		// Inject any transactions discarded due to reorgs
		log.Debug("Reinjecting stale transactions", "count", len(reinject))
		pool.addTxsLocked(reinject, false)
	
		// validate the pool of pending transactions, this will remove
		// any transactions that have been included in the block or
		// have been invalidated because of another transaction (e.g.
		// higher gas price)
		//트랜잭션 내부 트랜잭션 풀 아직 확인 트랜잭션 안에 존재하는 모든 블록 사슬을 제거하거나 불가능하기 때문에 트랜잭션 결과 다른 트랜잭션 (예컨대 높은 gasPrice 등)
		//내부 큐에 대기중인 트랜잭션의 일부를 다운 그레이드 다운 그레이드 강등.
		pool.demoteUnexecutables()
	
		// Update all accounts to the latest known pending nonce
		//비표 넌스중인 큐에 따라 모든 계정을 업데이트
		for addr, list := range pool.pending {
			txs := list.Flatten() // Heavy but will be cached and is needed by the miner anyway
			pool.pendingState.SetNonce(addr, txs[len(txs)-1].Nonce()+1)
		}
		// Check the queue and move transactions over to the pending if possible
		// or remove those that have become invalid
		//대기열을 확인하고 대기중인 가능한 트랜잭션을 많이 이동하거나 그 거래를 삭제 실패
		//업그레이드를 촉진
		pool.promoteExecutables(nil)
	}
addTx 
	
	// addTx enqueues a single transaction into the pool if it is valid.
	func (pool *TxPool) addTx(tx *types.Transaction, local bool) error {
		pool.mu.Lock()
		defer pool.mu.Unlock()
	
		// Try to inject the transaction and update any state
		replace, err := pool.add(tx, local)
		if err != nil {
			return err
		}
		// If we added a new transaction, run promotion checks and return
		if !replace {
			from, _ := types.Sender(pool.signer, tx) // already validated
			pool.promoteExecutables([]common.Address{from})
		}
		return nil
	}

addTxsLocked
	
	// addTxsLocked attempts to queue a batch of transactions if they are valid,
	// whilst assuming the transaction pool lock is already held.
	//잠금 획득 된 것으로 가정 할 때이 기능을, 큐 큐에 효과적으로 대처 호출 할 addTxsLocked
	func (pool *TxPool) addTxsLocked(txs []*types.Transaction, local bool) error {
		// Add the batch of transaction, tracking the accepted ones
		dirty := make(map[common.Address]struct{})
		for _, tx := range txs {
			if replace, err := pool.add(tx, local); err == nil {
				if !replace { //교체하지 않을 경우 대체하기위한 것입니다 교체, 그 상태 업데이트, 그것은 프로세스의 다음 단계가 될 수 있습니다.
					from, _ := types.Sender(pool.signer, tx) // already validated
					dirty[from] = struct{}{}
				}
			}
		}
		// Only reprocess the internal state if something was actually added
		if len(dirty) > 0 {
			addrs := make([]common.Address, 0, len(dirty))
			for addr, _ := range dirty {
				addrs = append(addrs, addr)
			}	
			//주소를 전달하면, 수정한다
			pool.promoteExecutables(addrs)
		}
		return nil
	}
demoteUnexecutables 처리 트랜잭션을 보류 중이거나 이미에서 잘못된을 제거, 다른 박람회는 미래 대기열에 집행로 이동됩니다.
	
	// demoteUnexecutables removes invalid and processed transactions from the pools
	// executable/pending queue and any subsequent transactions that become unexecutable
	// are moved back into the future queue.
	func (pool *TxPool) demoteUnexecutables() {
		// Iterate over all accounts and demote any non-executable transactions
		for addr, list := range pool.pending {
			nonce := pool.currentState.GetNonce(addr)
	
			// Drop all transactions that are deemed too old (low nonce)
			//모든 주소 현재의 트랜잭션 (transaction)보다 넌스 및 pool.all에서 제거 삭제합니다.
			for _, tx := range list.Forward(nonce) {
				hash := tx.Hash()
				log.Trace("Removed old pending transaction", "hash", hash)
				delete(pool.all, hash)
				pool.priced.Removed()
			}
			// Drop all transactions that are too costly (low balance or out of gas), and queue any invalids back for later
			//모든 너무 비싸 거래를 삭제합니다. 사용자의 밸런스가 충분하지 않을 수 있습니다. 또는 가스 중
			drops, invalids := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
			for _, tx := range drops {
				hash := tx.Hash()
				log.Trace("Removed unpayable pending transaction", "hash", hash)
				delete(pool.all, hash)
				pool.priced.Removed()
				pendingNofundsCounter.Inc(1)
			}
			for _, tx := range invalids {
				hash := tx.Hash()
				log.Trace("Demoting pending transaction", "hash", hash)
				pool.enqueueTx(hash, tx)
			}
			// If there's a gap in front, warn (should never happen) and postpone all transactions
			//구멍 (넌스 비어)가있는 경우, 당신은 미래 큐에 모든 트랜잭션을 넣어해야합니다.
			//필터 걸었다 병자가 처리되기 때문에이 단계는 정말 일이 안된다. 거래의 병자는 그, 빈 없다 존재하지 않아야합니다.
			if list.Len() > 0 && list.txs.Get(nonce) == nil {
				for _, tx := range list.Cap(0) {
					hash := tx.Hash()
					log.Error("Demoting invalidated transaction", "hash", hash)
					pool.enqueueTx(hash, tx)
				}
			}
			// Delete the entire queue entry if it became empty.
			if list.Empty() { 
				delete(pool.pending, addr)
				delete(pool.beats, addr)
			}
		}
	}

새로운 계약에 enqueueTx 미래 큐에 삽입됩니다. 이 방법을 사용하면 잠금 풀을 획득 한 것으로 가정합니다.
	
	// enqueueTx inserts a new transaction into the non-executable transaction queue.
	//
	// Note, this method assumes the pool lock is held!
	func (pool *TxPool) enqueueTx(hash common.Hash, tx *types.Transaction) (bool, error) {
		// Try to insert the transaction into the future queue
		from, _ := types.Sender(pool.signer, tx) // already validated
		if pool.queue[from] == nil {
			pool.queue[from] = newTxList(false)
		}
		inserted, old := pool.queue[from].Add(tx, pool.config.PriceBump)
		if !inserted {
			// An older transaction was better, discard this
			queuedDiscardCounter.Inc(1)
			return false, ErrReplaceUnderpriced
		}
		// Discard any previous transaction and mark this
		if old != nil {
			delete(pool.all, old.Hash())
			pool.priced.Removed()
			queuedReplaceCounter.Inc(1)
		}
		pool.all[hash] = tx
		pool.priced.Put(tx)
		return old != nil, nil
	}

promoteExecutables 거래에 삽입하는 방법은 보류 큐 미래 큐에서 수행 될 수되었다. 이 과정을 통해 잘못된 거래 모두 삭제됩니다 (넌스이 너무 낮 불충분 균형).
	
	// promoteExecutables moves transactions that have become processable from the
	// future queue to the set of pending transactions. During this process, all
	// invalidated transactions (low nonce, low balance) are deleted.
	func (pool *TxPool) promoteExecutables(accounts []common.Address) {
		// Gather all the accounts potentially needing updates
		//상점 귀하의 계정을 업데이트 할 수있는 모든 잠재적 인 필요를 차지한다. 계정이 전무에 전달하면, 모든 알려진 계정을 나타내는.
		if accounts == nil {
			accounts = make([]common.Address, 0, len(pool.queue))
			for addr, _ := range pool.queue {
				accounts = append(accounts, addr)
			}
		}
		// Iterate over all accounts and promote any executable transactions
		for _, addr := range accounts {
			list := pool.queue[addr]
			if list == nil {
				continue // Just in case someone calls with a non existing account
			}
			// Drop all transactions that are deemed too old (low nonce)
			//비표 모두 너무 낮게 거래되고 삭제
			for _, tx := range list.Forward(pool.currentState.GetNonce(addr)) {
				hash := tx.Hash()
				log.Trace("Removed old queued transaction", "hash", hash)
				delete(pool.all, hash)
				pool.priced.Removed()
			}
			// Drop all transactions that are too costly (low balance or out of gas)
			//불충분 한 균형 트랜잭션을 모두 삭제합니다.
			drops, _ := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
			for _, tx := range drops {
				hash := tx.Hash()
				log.Trace("Removed unpayable queued transaction", "hash", hash)
				delete(pool.all, hash)
				pool.priced.Removed()
				queuedNofundsCounter.Inc(1)
			}
			// Gather all executable transactions and promote them
			//모든 트랜잭션이 실행 될 수 있으며, promoteTx이 보류 조인 받기
			for _, tx := range list.Ready(pool.pendingState.GetNonce(addr)) {
				hash := tx.Hash()
				log.Trace("Promoting queued transaction", "hash", hash)
				pool.promoteTx(addr, hash, tx)
			}
			// Drop all transactions over the allowed limit
			//한도를 초과하는 모든 트랜잭션을 삭제합니다.
			if !pool.locals.contains(addr) {
				for _, tx := range list.Cap(int(pool.config.AccountQueue)) {
					hash := tx.Hash()
					delete(pool.all, hash)
					pool.priced.Removed()
					queuedRateLimitCounter.Inc(1)
					log.Trace("Removed cap-exceeding queued transaction", "hash", hash)
				}
			}
			// Delete the entire queue entry if it became empty.
			if list.Empty() {
				delete(pool.queue, addr)
			}
		}
		// If the pending limit is overflown, start equalizing allowances
		pending := uint64(0)
		for _, list := range pool.pending {
			pending += uint64(list.Len())
		}
		//출원의 총 수는 구성된 시스템을 초과하는 경우.
		if pending > pool.config.GlobalSlots {
			
			pendingBeforeCap := pending
			// Assemble a spam order to penalize large transactors first
			spammers := prque.New()
			for addr, list := range pool.pending {
				// Only evict transactions from high rollers
				//먼저 계정 레코드의 최소 값보다 큰 모든 AccountSlots의 트랜잭션 내부에서 이러한 계정의 일부를 제거하는 것입니다.
				//스패머가 가장 큰에서 작은에 거래 번호를 기준으로 정렬됩니다 우선 순위 큐, 유의하십시오.
				if !pool.locals.contains(addr) && uint64(list.Len()) > pool.config.AccountSlots {
					spammers.Push(addr, float32(list.Len()))
				}
			}
			// Gradually drop transactions from offenders
			offenders := []common.Address{}
			for pending > pool.config.GlobalSlots && !spammers.Empty() {
				/*	
				범죄자 큐 모의 거래 계좌 변경 횟수 봐.
					第一次循环   [10]주기의 끝 [10]
					제 2 사이클 [10, 10] 루프 단부 [9,9]
					세번째 사이클 [9,9, 7 사이클이 종료 [7, 7, 7]
					네번째 사이클 [7, 7, 7, 2 사이클 종료 [2, 2, 2, 2]
				*/
				// Retrieve the next offender if not local address
				offender, _ := spammers.Pop()
				offenders = append(offenders, offender.(common.Address))
	
				// Equalize balances until all the same or below threshold
				if len(offenders) > 1 { //이 개 거래 계정의 가장 큰 번호가이주기, 범죄자 큐에 처음
					// Calculate the equalization threshold for all current offenders
					//거래의 수는 계정에 가입하기 위해 마지막으로 할 때 임계 비용 시간
					threshold := pool.pending[offender.(common.Address)].Len()
	
					// Iteratively reduce all offenders until below limit or threshold reached
					//보류까지 효과적인 탐색하거나 거래 마지막 거래의 개수와 동일한 개수의 제 역수
					for pending > pool.config.GlobalSlots && pool.pending[offenders[len(offenders)-2]].Len() > threshold {
						//마지막 계정을 제외한 모든 계정을 순회, 거래 1을 뺀 자신의 번호입니다.
						for i := 0; i < len(offenders)-1; i++ {
							list := pool.pending[offenders[i]]
							for _, tx := range list.Cap(list.Len() - 1) {
								// Drop the transaction from the global pools too
								hash := tx.Hash()
								delete(pool.all, hash)
								pool.priced.Removed()
	
								// Update the account nonce to the dropped transaction
								if nonce := tx.Nonce(); pool.pendingState.GetNonce(offenders[i]) > nonce {
									pool.pendingState.SetNonce(offenders[i], nonce)
								}
								log.Trace("Removed fairness-exceeding pending transaction", "hash", hash)
							}
							pending--
						}
					}
				}
			}
			// If still above threshold, reduce to limit or min allowance
			//위의 사이클 후 모든 AccountSlots 계정을 통해 트랜잭션의 수는 전에 최소한되고있다.
			//여전히 임계 값을 초과하는 경우, 내부 범죄자에서 삭제마다 계속됩니다.
			if pending > pool.config.GlobalSlots && len(offenders) > 0 {
				for pending > pool.config.GlobalSlots && uint64(pool.pending[offenders[len(offenders)-1]].Len()) > pool.config.AccountSlots {
					for _, addr := range offenders {
						list := pool.pending[addr]
						for _, tx := range list.Cap(list.Len() - 1) {
							// Drop the transaction from the global pools too
							hash := tx.Hash()
							delete(pool.all, hash)
							pool.priced.Removed()
	
							// Update the account nonce to the dropped transaction
							if nonce := tx.Nonce(); pool.pendingState.GetNonce(addr) > nonce {
								pool.pendingState.SetNonce(addr, nonce)
							}
							log.Trace("Removed fairness-exceeding pending transaction", "hash", hash)
						}
						pending--
					}
				}
			}
			pendingRateLimitCounter.Inc(int64(pendingBeforeCap - pending))
		}  //end if pending > pool.config.GlobalSlots {
		// If we've queued more transactions than the hard limit, drop oldest ones
		//우리는 보류 제한 미래 큐 제한을 처리하기 위해 다음의 필요성을 다룬다.
		queued := uint64(0)
		for _, list := range pool.queue {
			queued += uint64(list.Len())
		}
		if queued > pool.config.GlobalQueue {
			// Sort all accounts with queued transactions by heartbeat
			addresses := make(addresssByHeartbeat, 0, len(pool.queue))
			for addr := range pool.queue {
				if !pool.locals.contains(addr) { // don't drop locals
					addresses = append(addresses, addressByHeartbeat{addr, pool.beats[addr]})
				}
			}
			sort.Sort(addresses)
	
			// Drop transactions until the total is below the limit or only locals remain
			//새로운 마음입니다 전면에 다시에서 더 삭제됩니다.
			for drop := queued - pool.config.GlobalQueue; drop > 0 && len(addresses) > 0; {
				addr := addresses[len(addresses)-1]
				list := pool.queue[addr.address]
	
				addresses = addresses[:len(addresses)-1]
	
				// Drop all transactions if they are less than the overflow
				if size := uint64(list.Len()); size <= drop {
					for _, tx := range list.Flatten() {
						pool.removeTx(tx.Hash())
					}
					drop -= size
					queuedRateLimitCounter.Inc(int64(size))
					continue
				}
				// Otherwise drop only last few transactions
				txs := list.Flatten()
				for i := len(txs) - 1; i >= 0 && drop > 0; i-- {
					pool.removeTx(txs[i].Hash())
					drop--
					queuedRateLimitCounter.Inc(1)
				}
			}
		}
	}

promoteTx는 보류 대기열에 추가 거래로.이 방법을 사용하면 잠금을 얻을 수 있다고 가정합니다.

	// promoteTx adds a transaction to the pending (processable) list of transactions.
	//
	// Note, this method assumes the pool lock is held!
	func (pool *TxPool) promoteTx(addr common.Address, hash common.Hash, tx *types.Transaction) {
		// Try to insert the transaction into the pending queue
		if pool.pending[addr] == nil {
			pool.pending[addr] = newTxList(true)
		}
		list := pool.pending[addr]
	
		inserted, old := list.Add(tx, pool.config.PriceBump)
		if !inserted { //당신이 대체 할 수없는 경우, 이미 오래된 거래. 삭제.
			// An older transaction was better, discard this
			delete(pool.all, hash)
			pool.priced.Removed()
	
			pendingDiscardCounter.Inc(1)
			return
		}
		// Otherwise discard any previous transaction and mark this
		if old != nil { 
			delete(pool.all, old.Hash())
			pool.priced.Removed()
	
			pendingReplaceCounter.Inc(1)
		}
		// Failsafe to work around direct pending inserts (tests)
		if pool.all[hash] == nil {
			pool.all[hash] = tx
			pool.priced.Put(tx)
		}
		// Set the potentially new pending nonce and notify any subsystems of the new tx
		//거래는 큐에 추가하고, 내부적으로 ETH 계약의 모든 가입자, 가입자에게 메시지를 보냅니다. 메시지와 네트워크에 의해 방송 뉴스를 수신한다.
		pool.beats[addr] = time.Now()
		pool.pendingState.SetNonce(addr, tx.Nonce()+1)
		go pool.txFeed.Send(TxPreEvent{tx})
	}
	

removeTx는 트랜잭션을 삭제하고, 이후의 모든 거래는 미래 대기열로 이동

	
	// removeTx removes a single transaction from the queue, moving all subsequent
	// transactions back to the future queue.
	func (pool *TxPool) removeTx(hash common.Hash) {
		// Fetch the transaction we wish to delete
		tx, ok := pool.all[hash]
		if !ok {
			return
		}
		addr, _ := types.Sender(pool.signer, tx) // already validated during insertion
	
		// Remove it from the list of known transactions
		delete(pool.all, hash)
		pool.priced.Removed()
	
		// Remove the transaction from the pending lists and reset the account nonce
		//거래는 보류에서 제거되고 트랜잭션이 미래의 큐에 무효가되기 때문에 트랜잭션을 삭제
		//그리고 상태를 업데이트 pendingState
		if pending := pool.pending[addr]; pending != nil {
			if removed, invalids := pending.Remove(tx); removed {
				// If no more transactions are left, remove the list
				if pending.Empty() {
					delete(pool.pending, addr)
					delete(pool.beats, addr)
				} else {
					// Otherwise postpone any invalidated transactions
					for _, tx := range invalids {
						pool.enqueueTx(tx.Hash(), tx)
					}
				}
				// Update the account nonce if needed
				if nonce := tx.Nonce(); pool.pendingState.GetNonce(addr) > nonce {
					pool.pendingState.SetNonce(addr, nonce)
				}
				return
			}
		}
		// Transaction is in the future queue
		//거래는 미래의 대기열에서 삭제됩니다.
		if future := pool.queue[addr]; future != nil {
			future.Remove(tx)
			if future.Empty() {
				delete(pool.queue, addr)
			}
		}
	}



루프는 goroutine입니다 txPool. 또한 메인 이벤트 루프, 대기 및 블록 체인 외부 이벤트, 각종 보고서 및 거래 퇴거에 응답.
	
	// loop is the transaction pool's main event loop, waiting for and reacting to
	// outside blockchain events as well as for various reporting and transaction
	// eviction events.
	func (pool *TxPool) loop() {
		defer pool.wg.Done()
	
		// Start the stats reporting and transaction eviction tickers
		var prevPending, prevQueued, prevStales int
	
		report := time.NewTicker(statsReportInterval)
		defer report.Stop()
	
		evict := time.NewTicker(evictionInterval)
		defer evict.Stop()
	
		journal := time.NewTicker(pool.config.Rejournal)
		defer journal.Stop()
	
		// Track the previous head headers for transaction reorgs
		head := pool.chain.CurrentBlock()
	
		// Keep waiting for and reacting to the various events
		for {
			select {
			// Handle ChainHeadEvent
			//이벤트의 헤더 영역 듣기, 새로운 영역 헤더에 도착.
			//리셋 메서드를 호출
			case ev := <-pool.chainHeadCh:
				if ev.Block != nil {
					pool.mu.Lock()
					if pool.chainconfig.IsHomestead(ev.Block.Number()) {
						pool.homestead = true
					}
					pool.reset(head.Header(), ev.Block.Header())
					head = ev.Block
	
					pool.mu.Unlock()
				}
			// Be unsubscribed due to system stopped
			case <-pool.chainHeadSub.Err():
				return
	
			//틱 보고서를보고 처리 통계는 일부 로그를 인쇄하는 것입니다
			case <-report.C:
				pool.mu.RLock()
				pending, queued := pool.stats()
				stales := pool.priced.stales
				pool.mu.RUnlock()
	
				if pending != prevPending || queued != prevQueued || stales != prevStales {
					log.Debug("Transaction pool status report", "executable", pending, "queued", queued, "stales", stales)
					prevPending, prevQueued, prevStales = pending, queued, stales
				}
	
			// Handle inactive account transaction eviction
			//가공 거래 정보 시간 제한,
			case <-evict.C:
				pool.mu.Lock()
				for addr := range pool.queue {
					// Skip local transactions from the eviction mechanism
					if pool.locals.contains(addr) {
						continue
					}
					// Any non-locals old enough should be removed
					if time.Since(pool.beats[addr]) > pool.config.Lifetime {
						for _, tx := range pool.queue[addr].Flatten() {
							pool.removeTx(tx.Hash())
						}
					}
				}
				pool.mu.Unlock()
	
			//정보 쓰기 트랜잭션 로그 타이밍 로컬 트랜잭션 저널 회전 ​​처리를 처리합니다.
			case <-journal.C:
				if pool.journal != nil {
					pool.mu.Lock()
					if err := pool.journal.rotate(pool.local()); err != nil {
						log.Warn("Failed to rotate local tx journal", "err", err)
					}
					pool.mu.Unlock()
				}
			}
		}
	}


, 방법을 추가 트랜잭션을 확인하고 미래를 큐에 삽입합니다. 거래는 거래의 현재 존재 반환하기 전에 다음 트랜잭션을 교체하는 경우, 그래서 외부 방법을 촉진하기 위해 호출하지 않습니다. 새로운 계약이 추가 된 경우 지역으로 분류하여, 다음은 계정을 보내는 화이트리스트, 제거하기 때문에 가격 제한 또는 기타 제한으로하지 않습니다이 거래와 관련된 계정을 입력합니다.
	
	// add validates a transaction and inserts it into the non-executable queue for
	// later pending promotion and execution. If the transaction is a replacement for
	// an already pending or queued one, it overwrites the previous and returns this
	// so outer code doesn't uselessly call promote.
	//
	// If a newly added transaction is marked as local, its sending account will be
	// whitelisted, preventing any associated transaction from being dropped out of
	// the pool due to pricing constraints.
	func (pool *TxPool) add(tx *types.Transaction, local bool) (bool, error) {
		// If the transaction is already known, discard it
		hash := tx.Hash()
		if pool.all[hash] != nil {
			log.Trace("Discarding already known transaction", "hash", hash)
			return false, fmt.Errorf("known transaction: %x", hash)
		}
		// If the transaction fails basic validation, discard it
		//기본 인증 거래 후 폐기 할 수없는 경우
		if err := pool.validateTx(tx, local); err != nil {
			log.Trace("Discarding invalid transaction", "hash", hash, "err", err)
			invalidTxCounter.Inc(1)
			return false, err
		}
		// If the transaction pool is full, discard underpriced transactions
		//트랜잭션 풀이 가득합니다. 그래서 싸구려 거래를 삭제합니다.
		if uint64(len(pool.all)) >= pool.config.GlobalSlots+pool.config.GlobalQueue {
			// If the new transaction is underpriced, don't accept it
			//새로운 거래 자체가 저렴합니다. 그래서 수신하지 않습니다
			if pool.priced.Underpriced(tx, pool.locals) {
				log.Trace("Discarding underpriced transaction", "hash", hash, "price", tx.GasPrice())
				underpricedTxCounter.Inc(1)
				return false, ErrUnderpriced
			}
			// New transaction is better than our worse ones, make room for it
			//그렇지 않으면, 그에게 양자 공간을 낮은 값을 삭제합니다.
			drop := pool.priced.Discard(len(pool.all)-int(pool.config.GlobalSlots+pool.config.GlobalQueue-1), pool.locals)
			for _, tx := range drop {
				log.Trace("Discarding freshly underpriced transaction", "hash", tx.Hash(), "price", tx.GasPrice())
				underpricedTxCounter.Inc(1)
				pool.removeTx(tx.Hash())
			}
		}
		// If the transaction is replacing an already pending one, do directly
		from, _ := types.Sender(pool.signer, tx) // already validated
		if list := pool.pending[from]; list != nil && list.Overlaps(tx) {
			// Nonce already pending, check if required price bump is met
			//트랜잭션이 논스 이미 큐에 대기중인 해당하는 경우 생산이 대체 할 수있는 경우, 다음을 참조하십시오.
			inserted, old := list.Add(tx, pool.config.PriceBump)
			if !inserted {
				pendingDiscardCounter.Inc(1)
				return false, ErrReplaceUnderpriced
			}
			// New transaction is better, replace old one
			if old != nil {
				delete(pool.all, old.Hash())
				pool.priced.Removed()
				pendingReplaceCounter.Inc(1)
			}
			pool.all[tx.Hash()] = tx
			pool.priced.Put(tx)
			pool.journalTx(from, tx)
	
			log.Trace("Pooled new executable transaction", "hash", hash, "from", from, "to", tx.To())
			return old != nil, nil
		}
		// New transaction isn't replacing a pending one, push into queue
		//새로운 트랜잭션은 다음 보류중인 트랜잭션의 내부의 대체 내부 큐를 futuren 그를 밀어 수 없습니다.
		replace, err := pool.enqueueTx(hash, tx)
		if err != nil {
			return false, err
		}
		// Mark local addresses and journal local transactions
		if local {
			pool.locals.add(from)
		}
		//로컬 트랜잭션은 journalTx에 기록됩니다
		pool.journalTx(from, tx)
	
		log.Trace("Pooled new future transaction", "hash", hash, "from", from, "to", tx.To())
		return replace, nil
	}


validateTx 트랜잭션의 유효성을 확인하기 위해 일관성 규칙을 사용하고, 사용 횟수 발견 로컬 노드를 제한 할 수 있습니다.

	// validateTx checks whether a transaction is valid according to the consensus
	// rules and adheres to some heuristic limits of the local node (price and size).
	func (pool *TxPool) validateTx(tx *types.Transaction, local bool) error {
		// Heuristic limit, reject transactions over 32KB to prevent DOS attacks
		if tx.Size() > 32*1024 {
			return ErrOversizedData
		}
		// Transactions can't be negative. This may never happen using RLP decoded
		// transactions but may occur if you create a transaction using the RPC.
		if tx.Value().Sign() < 0 {
			return ErrNegativeValue
		}
		// Ensure the transaction doesn't exceed the current block limit gas.
		if pool.currentMaxGas.Cmp(tx.Gas()) < 0 {
			return ErrGasLimit
		}
		// Make sure the transaction is signed properly
		//트랜잭션이 제대로 체결되어 있는지 확인합니다.
		from, err := types.Sender(pool.signer, tx)
		if err != nil {
			return ErrInvalidSender
		}
		// Drop non-local transactions under our own minimal accepted gas price
		local = local || pool.locals.contains(from) // account may be local even if the transaction arrived from the network
		//트랜잭션이, 우리의 설정 아래 지역이 아니며, GasPrice 경우 수신하지.
		if !local && pool.gasPrice.Cmp(tx.GasPrice()) > 0 {
			return ErrUnderpriced
		}
		// Ensure the transaction adheres to nonce ordering
		//트랜잭션이 순서 논스을 준수하는지 확인
		if pool.currentState.GetNonce(from) > tx.Nonce() {
			return ErrNonceTooLow
		}
		// Transactor should have enough funds to cover the costs
		// cost == V + GP * GL
		//사용자가 비용을 지불하기에 충분한 잔액이 있는지 확인합니다.
		if pool.currentState.GetBalance(from).Cmp(tx.Cost()) < 0 {
			return ErrInsufficientFunds
		}
		intrGas := IntrinsicGas(tx.Data(), tx.To() == nil, pool.homestead)
		//트랜잭션이 계약을 만들거나 전화를하는 경우 충분한 초기 가스가있는 경우. 다음을 참조하십시오.
		if tx.Gas().Cmp(intrGas) < 0 {
			return ErrIntrinsicGas
		}
		return nil
	}
