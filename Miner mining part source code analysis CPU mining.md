## agent
에이전트는 마이닝 개체의 특정 구현입니다. 그것은 헤더의 영역 및 계산 mixhash의 넌스, 좋은 헤더가 반환 광산의 면적을 계산 받아 실행하는 과정이다.

정상적인 상황이 광산에 CPU를 사용하지 않습니다에 따라 구성 CpuAgent가, 광업 전용 GPU는 일반적으로 광산을 사용하여 수행, 광산 GPU 코드는 여기에 반영되지 않습니다.

	type CpuAgent struct {
		mu sync.Mutex
	
		workCh		chan *Work   // 광산 작업 채널을 수용
		stop		  chan struct{}
		quitCurrentOp chan struct{}
		returnCh  짱 &lt;- * 광산은 // 리턴 채널 완료된 후 결과
	
		chain  consensus.ChainReader //체인 정보에 대한 액세스를 차단
		engine consensus.Engine  // 일관성 엔진은,이 경우 엔진은 탕입니다
	
		isMining int32 // isMining indicates whether the agent is currently mining
	}
	
	func NewCpuAgent(chain consensus.ChainReader, engine consensus.Engine) *CpuAgent {
		miner := &CpuAgent{
			chain:  chain,
			engine: engine,
			stop:   make(chan struct{}, 1),
			workCh: make(chan *Work, 1),
		}
		return miner
	}

채널의 반환 값을 설정하고 외부의 전통적인 가치를 얻을 수 및 정보를 반환하기 쉬운 작업 채널을 얻을.

	func (self *CpuAgent) Work() chan<- *Work			{ return self.workCh }
	func (self *CpuAgent) SetReturnCh(ch chan<- *Result) { self.returnCh = ch }

메시지 루프를 시작하고, 이미 광산, 다음 종료를 시작한 경우, 또는 갱신이 goroutine 시작
workCh 작업에서 수용을 업데이트, 광업, 정보를 수행하거나 종료 종료에 동의합니다.
	
	func (self *CpuAgent) Start() {
		if !atomic.CompareAndSwapInt32(&self.isMining, 0, 1) {
			return // agent already started
		}
		go self.update()
	}
	
	func (self *CpuAgent) update() {
	out:
		for {
			select {
			case work := <-self.workCh:
				self.mu.Lock()
				if self.quitCurrentOp != nil {
					close(self.quitCurrentOp)
				}
				self.quitCurrentOp = make(chan struct{})
				go self.mine(work, self.quitCurrentOp)
				self.mu.Unlock()
			case <-self.stop:
				self.mu.Lock()
				if self.quitCurrentOp != nil {
					close(self.quitCurrentOp)
					self.quitCurrentOp = nil
				}
				self.mu.Unlock()
				break out
			}
		}
	}

광산, 광업, 성공하면, 일관성 엔진 광업를 호출 위 returnCh하는 메시지를 보낼 수 있습니다.
	
	func (self *CpuAgent) mine(work *Work, stop <-chan struct{}) {
		if result, err := self.engine.Seal(self.chain, work.Block, stop); result != nil {
			log.Info("Successfully sealed new block", "number", result.Number(), "hash", result.Hash())
			self.returnCh <- &Result{work, result}
		} else {
			if err != nil {
				log.Warn("Block sealing failed", "err", err)
			}
			self.returnCh <- nil
		}
	}
GetHashRate,이 함수는 현재 HashRate를 반환합니다.

	func (self *CpuAgent) GetHashRate() int64 {
		if pow, ok := self.engine.(consensus.PoW); ok {
			return int64(pow.Hashrate())
		}
		return 0
	}


## remote_agent
remote_agent는 RPC 인터페이스의 세트가 원격 함수 광부 마이닝을 가능하게 제공한다. 예를 들어, 나는 광산 기계, 광산 기계가 실행되지 않는 내부 이더넷 광장, 광산 기계, 광산이 완료되면, 계산 광산 다음, remote_agent에서 현재 첫 번째 작업을 얻을 결과, 광산의 완료를 제출 노드가 있습니다.

데이터 구조와 구성

	type RemoteAgent struct {
		mu sync.Mutex
	
		quitCh   chan struct{}
		workCh   chan *Work	  작업을 수락 //
		returnCh chan<- *Result	// 결과가 반환된다
	
		chain	   consensus.ChainReader
		engine	  consensus.Engine
		currentWork *Work// 현재 작업
		work	지도 [common.Hash] * 아직 커밋하지 않는 일 // 작업은 계산한다
	
		hashrateMu sync.RWMutex
		hashrate   map[common.Hash]hashrate  //작업 hashrate 계산
	
		running int32 // running indicates whether the agent is active. Call atomically
	}
	
	func NewRemoteAgent(chain consensus.ChainReader, engine consensus.Engine) *RemoteAgent {
		return &RemoteAgent{
			chain:	chain,
			engine:   engine,
			work:	 make(map[common.Hash]*Work),
			hashrate: make(map[common.Hash]hashrate),
		}
	}

시작 및 중지
	
	func (a *RemoteAgent) Start() {
		if !atomic.CompareAndSwapInt32(&a.running, 0, 1) {
			return
		}
		a.quitCh = make(chan struct{})
		a.workCh = make(chan *Work, 1)
		go a.loop(a.workCh, a.quitCh)
	}
	
	func (a *RemoteAgent) Stop() {
		if !atomic.CompareAndSwapInt32(&a.running, 1, 0) {
			return
		}
		close(a.quitCh)
		close(a.workCh)
	}
입력 및 출력 채널을 얻고,이를 agent.go.

	func (a *RemoteAgent) Work() chan<- *Work {
		return a.workCh
	}
	
	func (a *RemoteAgent) SetReturnCh(returnCh chan<- *Result) {
		a.returnCh = returnCh
	}

작업을 수신 할 때 루프 방법과 매우 유사한 일을 할 agent.go 내부는, 그것은 내부 currentWork 필드에 저장됩니다. 84초이 작업을 완료하지 않은 경우, 다음, 작업을 삭제 10 초 동안보고 hashrate을받지 못한 경우, 다음 트랙 / 삭제합니다.
	
	// loop monitors mining events on the work and quit channels, updating the internal
	// state of the rmeote miner until a termination is requested.
	//
	// Note, the reason the work and quit channels are passed as parameters is because
	// RemoteAgent.Start() constantly recreates these channels, so the loop code cannot
	// assume data stability in these member fields.
	func (a *RemoteAgent) loop(workCh chan *Work, quitCh chan struct{}) {
		ticker := time.NewTicker(5 * time.Second)
		defer ticker.Stop()
	
		for {
			select {
			case <-quitCh:
				return
			case work := <-workCh:
				a.mu.Lock()
				a.currentWork = work
				a.mu.Unlock()
			case <-ticker.C:
				// cleanup
				a.mu.Lock()
				for hash, work := range a.work {
					if time.Since(work.createdAt) > 7*(12*time.Second) {
						delete(a.work, hash)
					}
				}
				a.mu.Unlock()
	
				a.hashrateMu.Lock()
				for id, hashrate := range a.hashrate {
					if time.Since(hashrate.ping) > 10*time.Second {
						delete(a.hashrate, id)
					}
				}
				a.hashrateMu.Unlock()
			}
		}
	}

GetWork,이 방법은 현재 채굴 작업을 얻기 위해, 원격 광부에 의해 호출됩니다.
	
	func (a *RemoteAgent) GetWork() ([3]string, error) {
		a.mu.Lock()
		defer a.mu.Unlock()
	
		var res [3]string
	
		if a.currentWork != nil {
			block := a.currentWork.Block
	
			res[0] = block.HashNoNonce().Hex()
			seedHash := ethash.SeedHash(block.NumberU64())
			res[1] = common.BytesToHash(seedHash).Hex()
			// Calculate the "target" to be returned to the external miner
			n := big.NewInt(1)
			n.Lsh(n, 255)
			n.Div(n, block.Difficulty())
			n.Lsh(n, 1)
			res[2] = common.BytesToHash(n.Bytes()).Hex()
	
			a.work[block.HashNoNonce()] = a.currentWork
			return res, nil
		}
		return res, errors.New("No work available yet, don't panic.")
	}

SubmitWork는 원격 광부들은 광산의 결과를 제출하기 위해이 메서드를 호출합니다. 결과가 확인 된 후에 returnCh에 제출

	// SubmitWork tries to inject a pow solution into the remote agent, returning
	// whether the solution was accepted or not (not can be both a bad pow as well as
	// any other error, like no work pending).
	func (a *RemoteAgent) SubmitWork(nonce types.BlockNonce, mixDigest, hash common.Hash) bool {
		a.mu.Lock()
		defer a.mu.Unlock()
	
		// Make sure the work submitted is present
		work := a.work[hash]
		if work == nil {
			log.Info("Work submitted but none pending", "hash", hash)
			return false
		}
		// Make sure the Engine solutions is indeed valid
		result := work.Block.Header()
		result.Nonce = nonce
		result.MixDigest = mixDigest
	
		if err := a.engine.VerifySeal(a.chain, result); err != nil {
			log.Warn("Invalid proof-of-work submitted", "hash", hash, "err", err)
			return false
		}
		block := work.Block.WithSeal(result)
	
		// Solutions seems to be valid, return to the miner and notify acceptance
		a.returnCh <- &Result{work, block}
		delete(a.work, hash)
	
		return true
	}

SubmitHashrate, 해시 계산 힘을 제출

	func (a *RemoteAgent) SubmitHashrate(id common.Hash, rate uint64) {
		a.hashrateMu.Lock()
		defer a.hashrateMu.Unlock()
	
		a.hashrate[id] = hashrate{time.Now(), rate}
	}


## unconfirmed

미확인은 블록 파낸로하고 충분한 후속 블록 (5) 이후에 확인을 기다릴 같은 사용자의 로컬 마이닝 정보를 추적하는 데 사용되는 데이터 구조이며, 그 로컬 마이닝 블록이 명세서에 포함되어 볼 내부 블록 사슬.

데이터 구조
	
	// headerRetriever is used by the unconfirmed block set to verify whether a previously
	// mined block is part of the canonical chain or not.
	//headerRetriever 이전 블록 명세서 파고 체인의 일부인지를 확인하기 위해, 블록 확인되지 않은 그룹을 사용.
	type headerRetriever interface {
		// GetHeaderByNumber retrieves the canonical header associated with a block number.
		GetHeaderByNumber(number uint64) *types.Header
	}
	
	// unconfirmedBlock is a small collection of metadata about a locally mined block
	// that is placed into a unconfirmed set for canonical chain inclusion tracking.
	//unconfirmedBlock 로컬 블록을 포함하는 명세서 블록 사슬로 채취되었는지 여부를 추적 할 미확인 세트를 배치하는 로컬 메타 데이터 마이닝 블록을 작게 설정
	type unconfirmedBlock struct {
		index uint64
		hash  common.Hash
	}
	
	// unconfirmedBlocks implements a data structure to maintain locally mined blocks
	// have have not yet reached enough maturity to guarantee chain inclusion. It is
	// used by the miner to provide logs to the user when a previously mined block
	// has a high enough guarantee to not be reorged out of te canonical chain.	
	//unconfirmedBlocks 지역 광산 블록을 관리하기위한 데이터 구조를 구현하고, 이러한 블록은 아직 그들이 링크의 블록에 의해 지배되고 있음을 증명하는 신뢰의 충분한 수준에 도달하지 않았습니다. 그들이 블록을 표준 블록 체인에 포함되기 전에 그들이 파고 여부를 알 수 있도록 광부에 대한 정보를 제공하는 데 사용됩니다.
	type unconfirmedBlocks struct {
		chain  headerRetriever //블록 사슬이 현재 명세 정보 영역 헤더를 얻기 위해이 인터페이스를 확인해야 통해 표준 상태를 확인 Blockchain
		depth  uint		// 후 심도 블록 폐기 이전 블록 후에 블록 번호 이전 후 폐기
		blocks *ring.Ring  // 블록 정보를 정기적으로 정규 체인 크로스 체크 교차 검사 체인을 조절할 수 있도록 정보를 차단할 수 있도록 //
		lock   sync.RWMutex	// Protects the fields from concurrent access
	}

	// newUnconfirmedBlocks returns new data structure to track currently unconfirmed blocks.
	func newUnconfirmedBlocks(chain headerRetriever, depth uint) *unconfirmedBlocks {
		return &unconfirmedBlocks{
			chain: chain,
			depth: depth,
		}
	}

호출 할 때 광부 블록을 파고 때, 지수가 높은 블록, 추적 블록을 삽입, 해시는 블록의 해시 값입니다.
	
	
	// Insert adds a new block to the set of unconfirmed ones.
	func (set *unconfirmedBlocks) Insert(index uint64, hash common.Hash) {
		// If a new block was mined locally, shift out any old enough blocks
		//로컬 블록이 발굴되면, 블록에서 깊이를 초과했습니다
		set.Shift(index)
	
		// Create the new item as its own ring
		//작업 대기열주기.
		item := ring.New(1)
		item.Value = &unconfirmedBlock{
			index: index,
			hash:  hash,
		}
		// Set as the initial ring or append to the end
		set.lock.Lock()
		defer set.lock.Unlock()
	
		if set.blocks == nil {
			set.blocks = item
		} else {
			//순환 큐 항목에의 마지막 요소 삽입
			set.blocks.Move(-1).Link(item)
		}
		// Display a log for the user to notify of a new mined block unconfirmed
		log.Info는 ( &quot;다수&quot;인덱스 &quot;해쉬&quot;해쉬 &quot;채굴 전위 블록 🔨&quot;)
	}
변속 전달 방법보다 블록 인덱스 깊이 인덱스를 삭제하고, 그 블록 체인의 사양을 확인.
	
	// Shift drops all unconfirmed blocks from the set which exceed the unconfirmed sets depth
	// allowance, checking them against the canonical chain for inclusion or staleness
	// report.
	func (set *unconfirmedBlocks) Shift(height uint64) {
		set.lock.Lock()
		defer set.lock.Unlock()
	
		for set.blocks != nil {
			// Retrieve the next unconfirmed block and abort if too fresh
			//때문에 블록은 순차적으로 블록이다. 처음에 온 확실히 가장 오래된 블록이다.
			//그래서 당신은 치료가 끝난 경우, 블록의 시작을 확인해야 할 때마다, 그것은 내부 순환 큐에서 제거됩니다.
			next := set.blocks.Value.(*unconfirmedBlock)
			if next.index+uint64(set.depth) > height { //나이가합니다.
				break
			}
			// Block seems to exceed depth allowance, check for canonical status
			//헤더 영역의 높이를 블록 검색어
			header := set.chain.GetHeaderByNumber(next.index)
			switch {
			case header == nil:
				log.Warn("Failed to retrieve header of mined block", "number", next.index, "hash", next.hash)
			case header.Hash() == next.hash: //우리 자신의 헤더 영역 해당하는 경우
				log.Info는 ( &quot;다수&quot;next.index &quot;해쉬&quot;next.hash &quot;🔗 블록 정규 체인 도달&quot;)
			default: //그렇지 않으면, 위의 측쇄에 우리가.
				log.Info는 ( &quot;다수&quot;next.index &quot;해쉬&quot;next.hash &quot;⑂ 블록 측 포크되었다&quot;)
			}
			// Drop the block out of the ring
			//순환 큐에서 삭제
			if set.blocks.Value == set.blocks.Next().Value {
				//전류 값은 순환 큐는 하나 개의 요소는 다음의 설정이 전무 아니라는 것을 나타내는 자체 같으면
				set.blocks = nil
			} else {
				//그렇지 않으면, 마지막으로 이동 한 다음 하나를 삭제 한 다음 앞으로 이동합니다.
				set.blocks = set.blocks.Move(-1)
				set.blocks.Unlink(1)
				set.blocks = set.blocks.Move(1)
			}
		}
	}

## worker.go
그것은 내부 작업자 에이전트를 많이 포함하고 에이전트는 remote_agent 앞서 언급 포함 할 수있다. 노동자는 블록 개체를 구축하는 일을 담당합니다. 동시에 작업은 에이전트를 제공합니다.

데이터 구조 :

에이전트 인터페이스
	
	// Agent can register themself with the worker
	type Agent interface {
		Work() chan<- *Work
		SetReturnCh(chan<- *Result)
		Stop()
		Start()
		GetHashRate() int64
	}

작업 구조, 환경시 작업, 저장 및 임시 상태 정보를 모두 보유하고 있습니다.

	// Work is the workers current environment and holds
	// all of the current state information
	type Work struct {
		config *params.ChainConfig
		signer types.Signer		// 서명자
	
		state * State.StateDB // 여기에 상태 변화 상태 데이터베이스를 적용
		ancestors *set.Set   선조 세트 (삼촌 상위 유효성 검사에 사용) // 상위 집합 조상의 유효성을 확인하는 데 사용
		family	*set.Set   // 가족 수집 (삼촌의 무효 확인에 사용) 가족 세트는 무효의 조상을 확인
		uncles	*set.Set   // 삼촌은 삼촌 컬렉션입니다
		tcount	int		거래 // 텍사스 카운트 사이클이 기간을 수
	
		Block *types.Block //새로운 블록 // 새로운 블록
	
		header   *types.Header		// 헤더 영역
		txs  [* Types.Transaction // 트랜잭션
		receipts []*types.Receipt	  // 영수증
	
		createdAt time.Time		 시간을 만들기 //
	}
	
	type Result struct {  //결과
		Work  *Work
		Block *types.Block
	}
worker
	
	// worker is the main object which takes care of applying messages to the new state
	//노동자는 주 대상의 새로운 상태로 메시징 응용 프로그램에 대한 책임이 있습니다
	type worker struct {
		config *params.ChainConfig
		engine consensus.Engine
		mu sync.Mutex
		// update loop
		mux		  *event.TypeMux
		txCh		 chan core.TxPreEvent	// 채널 내부 txPool 거래를 수용하는데 사용
		txSub		event.Subscription		// 내부 가입자 거래 txPool을 적용합니다
		chainHeadCh  chan core.ChainHeadEvent채널을 수신 // 헤더 영역
		chainHeadSub event.Subscription
		chainSideCh  chan core.ChainSideEvent// 체인 블록 특정 통로로부터 제거하는 블록 사슬을받는
		chainSideSub event.Subscription
		wg		   sync.WaitGroup
	
		agents map[Agent]struct{}			// 모든 에이전트
		recv   chan *Result					// 에이전트는 채널에 결과를 보내드립니다
	
		eth	 Backend						// ETH 협정
		chain   *core.BlockChain			// 블록 사슬
		proc	core.Validator				// 블록 사슬 검사기
		chainDb ethdb.Database				// 블록 사슬 데이터베이스
	
		coinbase common.Address				// 굴착기 해결
		extra	[]byte							// 
	
		currentMu sync.Mutex
		current   *Work
	
		uncleMu		sync.Mutex
		possibleUncles map[common.Hash]*types.Block// 힘 삼촌 노드
	
		unconfirmed *unconfirmedBlocks // set of locally mined blocks pending canonicalness confirmations
	
		// atomic status counters
		mining int32
		atWork int32
	}

구조
	
	func newWorker(config *params.ChainConfig, engine consensus.Engine, coinbase common.Address, eth Backend, mux *event.TypeMux) *worker {
		worker := &worker{
			config:		 config,
			engine:		 engine,
			eth:			eth,
			mux:			mux,
			txCh:		   make(chan core.TxPreEvent, txChanSize), // 4096
			chainHeadCh:	make(chan core.ChainHeadEvent, chainHeadChanSize), // 10
			chainSideCh:	make(chan core.ChainSideEvent, chainSideChanSize), // 10
			chainDb:		eth.ChainDb(),
			recv:		   make(chan *Result, resultQueueSize), // 10
			chain:		  eth.BlockChain(),
			proc:		   eth.BlockChain().Validator(),
			possibleUncles: make(map[common.Hash]*types.Block),
			coinbase:	   coinbase,
			agents:		 make(map[Agent]struct{}),
			unconfirmed:	newUnconfirmedBlocks(eth.BlockChain(), miningLogAtDepth),
		}
		// Subscribe TxPreEvent for tx pool
		worker.txSub = eth.TxPool().SubscribeTxPreEvent(worker.txCh)
		// Subscribe events for blockchain
		worker.chainHeadSub = eth.BlockChain().SubscribeChainHeadEvent(worker.chainHeadCh)
		worker.chainSideSub = eth.BlockChain().SubscribeChainSideEvent(worker.chainSideCh)
		go worker.update()
	
		go worker.wait()
		worker.commitNewWork()
	
		return worker
	}

update
	
	func (self *worker) update() {
		defer self.txSub.Unsubscribe()
		defer self.chainHeadSub.Unsubscribe()
		defer self.chainSideSub.Unsubscribe()
	
		for {
			// A real event arrived, process interesting content
			select {
			//바로 오픈 광산 서비스를 시간의 메시지 헤더 영역을 수신 할 때 ChainHeadEvent을 처리합니다.
			case <-self.chainHeadCh:
				self.commitNewWork()
	
			//ChainSideEvent 블록이 블록 사슬 사양 아니다 수신 처리 잠재적 삼촌의 세트에 추가
			case ev := <-self.chainSideCh:
				self.uncleMu.Lock()
				self.possibleUncles[ev.Block.Hash()] = ev.Block
				self.uncleMu.Unlock()
	
			//거래 할 때 처리 TxPreEvent txPool는 내부 정보를 받았다.
			case ev := <-self.txCh:
				// Apply transaction to the pending state if we're not mining
				//어떤 광산 경우, 트랜잭션은 즉시 개방 채굴 작업을 위해, 현재 상태에 적용.
				if atomic.LoadInt32(&self.mining) == 0 {
					self.currentMu.Lock()
					acc, _ := types.Sender(self.current.signer, ev.Tx)
					txs := map[common.Address]types.Transactions{acc: {ev.Tx}}
					txset := types.NewTransactionsByPriceAndNonce(self.current.signer, txs)
	
					self.current.commitTransactions(self.mux, txset, self.chain, self.coinbase)
					self.currentMu.Unlock()
				}
	
			// System stopped
			case <-self.txSub.Err():
				return
			case <-self.chainHeadSub.Err():
				return
			case <-self.chainSideSub.Err():
				return
			}
		}
	}


새 작업을 제출 commitNewWork

	
	func (self *worker) commitNewWork() {
		self.mu.Lock()
		defer self.mu.Unlock()
		self.uncleMu.Lock()
		defer self.uncleMu.Unlock()
		self.currentMu.Lock()
		defer self.currentMu.Unlock()
	
		tstart := time.Now()
		parent := self.chain.CurrentBlock()
	
		tstamp := tstart.Unix()
		if parent.Time().Cmp(new(big.Int).SetInt64(tstamp)) >= 0 { //상황의 부모보다 더 적은 시간이 표시되지 수
			tstamp = parent.Time().Int64() + 1
		}
		// this will ensure we're not going off too far in the future
		//너무 멀리 지금 시간을 초과하지 않는 우리의 시간은 다음 시간 대기
		//사실 광산 프로그램의 경우 완전히 테스트하기 위해 구현 된이 기능을 느껴, 당신은 기다릴한다.
		if now := time.Now().Unix(); tstamp > now+1 {
			wait := time.Duration(tstamp-now) * time.Second
			log.Info("Mining too far in the future", "wait", common.PrettyDuration(wait))
			time.Sleep(wait)
		}
	
		num := parent.Number()
		header := &types.Header{
			ParentHash: parent.Hash(),
			Number:	 num.Add(num, common.Big1),
			GasLimit:   core.CalcGasLimit(parent),
			GasUsed:	new(big.Int),
			Extra:	  self.extra,
			Time:	   big.NewInt(tstamp),
		}
		// Only set the coinbase if we are mining (avoid spurious block rewards)
		//우리가이 coinbase을 설정 한 경우 마이닝하는 경우에만 (거짓 블록 보상을 피하기 위해? TODO 이해하지)
		if atomic.LoadInt32(&self.mining) == 1 {
			header.Coinbase = self.coinbase
		}
		if err := self.engine.Prepare(self.chain, header); err != nil {
			log.Error("Failed to prepare header for mining", "err", err)
			return
		}
		// If we are care about TheDAO hard-fork check whether to override the extra-data or not
		//우리가 열심히 DAO 분기점을 우려 여부에 따라 추가 데이터를 포함할지 여부를 결정합니다.
		if daoBlock := self.config.DAOForkBlock; daoBlock != nil {
			// Check whether the block is among the fork extra-override range
			//체크 블록 갈래 하드 DAO의 범위 [daoblock, daoblock 제한 +]
			limit := new(big.Int).Add(daoBlock, params.DAOForkExtraRange)
			if header.Number.Cmp(daoBlock) >= 0 && header.Number.Cmp(limit) < 0 {
				// Depending whether we support or oppose the fork, override differently
				if self.config.DAOForkSupport { //우리가 지원하는 경우 DAO는 추가 데이터의 보존을 설정
					header.Extra = common.CopyBytes(params.DAOForkBlockExtra)
				} else if bytes.Equal(header.Extra, params.DAOForkBlockExtra) {
					header.Extra = []byte{} //광부가 반대하는 경우, // 그렇지 않으면, 추가 데이터 보존이 예약 된 여분의 데이터를 사용하게하지 않는다
				}
			}
		}
		// Could potentially happen if starting to mine in an odd state.
		err := self.makeCurrent(parent, header) //현재 상태의 새로운 블록으로 설정
		if err != nil {
			log.Error("Failed to create mining context", "err", err)
			return
		}
		// Create the current work task and check any fork transitions needed
		work := self.current
		if self.config.DAOForkSupport && self.config.DAOForkBlock != nil && self.config.DAOForkBlock.Cmp(header.Number) == 0 {
			misc.ApplyDAOHardFork(work.state)  //지정 계좌로의 자금 이체 내부 DAO.
		}
		pending, err := self.eth.TxPool().Pending() //차단 된 자금을 받기
		if err != nil {
			log.Error("Failed to fetch pending transactions", "err", err)
			return
		}
		//트랜잭션을 생성합니다. 이 방법은 후속 설명
		txs := types.NewTransactionsByPriceAndNonce(self.current.signer, pending)
		// 提交交易 这个方法后续介绍
		work.commitTransactions(self.mux, txs, self.chain, self.coinbase)
	
		// compute uncles for the new block.
		var (
			uncles	[]*types.Header
			badUncles []common.Hash
		)
		for hash, uncle := range self.possibleUncles {
			if len(uncles) == 2 {
				break
			}
			if err := self.commitUncle(work, uncle.Header()); err != nil {
				log.Trace("Bad uncle found and will be removed", "hash", hash)
				log.Trace(fmt.Sprint(uncle))
	
				badUncles = append(badUncles, hash)
			} else {
				log.Debug("Committing new uncle to block", "hash", hash)
				uncles = append(uncles, uncle.Header())
			}
		}
		for _, hash := range badUncles {
			delete(self.possibleUncles, hash)
		}
		// Create the new block to seal with the consensus engine
		//작품 마무리는 블록 보수 등의 작업을 실시한다하여 새로운 블록을 생성하는 상태 주어
		if work.Block, err = self.engine.Finalize(self.chain, header, work.state, work.txs, uncles, work.receipts); err != nil {
			log.Error("Failed to finalize block for sealing", "err", err)
			return
		}
		// We only care about logging if we're actually mining.
		// 
		if atomic.LoadInt32(&self.mining) == 1 {
			log.Info("Commit new mining work", "number", work.Block.Number(), "txs", work.tcount, "uncles", len(uncles), "elapsed", common.PrettyDuration(time.Since(tstart)))
			self.unconfirmed.Shift(work.Block.NumberU64() - 1)
		}
		self.push(work)
	}

우리는 광산이없는 경우 푸시 방법은 다음 각 에이전트 a를 제공하기 위해 직접 다른 임무를 반환
	
	// push sends a new work task to currently live miner agents.
	func (self *worker) push(work *Work) {
		if atomic.LoadInt32(&self.mining) != 1 {
			return
		}
		for agent := range self.agents {
			atomic.AddInt32(&self.atWork, 1)
			if ch := agent.Work(); ch != nil {
				ch <- work
			}
		}
	}

makeCurrent가 아닌 현재의주기는 새로운 환경을 만들 수 있습니다.
	
	// makeCurrent creates a new environment for the current cycle.
	// 
	func (self *worker) makeCurrent(parent *types.Block, header *types.Header) error {
		state, err := self.chain.StateAt(parent.Root())
		if err != nil {
			return err
		}
		work := &Work{
			config:	self.config,
			signer:	types.NewEIP155Signer(self.config.ChainId),
			state:	 state,
			ancestors: set.New(),
			family:	set.New(),
			uncles:	set.New(),
			header:	header,
			createdAt: time.Now(),
		}
	
		// when 08 is processed ancestors contain 07 (quick block)
		for _, ancestor := range self.chain.GetBlocksFromHash(parent.Hash(), 7) {
			for _, uncle := range ancestor.Uncles() {
				work.family.Add(uncle.Hash())
			}
			work.family.Add(ancestor.Hash())
			work.ancestors.Add(ancestor.Hash())
		}
	
		// Keep track of transactions which return errors so they can be removed
		work.tcount = 0
		self.current = work
		return nil
	}

commitTransactions
	
	func (env *Work) commitTransactions(mux *event.TypeMux, txs *types.TransactionsByPriceAndNonce, bc *core.BlockChain, coinbase common.Address) {
		gp := new(core.GasPool).AddGas(env.header.GasLimit)
	
		var coalescedLogs []*types.Log
	
		for {
			// Retrieve the next transaction and abort if all done
			tx := txs.Peek()
			if tx == nil {
				break
			}
			// Error may be ignored here. The error has already been checked
			// during transaction acceptance is the transaction pool.
			//
			// We use the eip155 signer regardless of the current hf.
			from, _ := types.Sender(env.signer, tx)
			// Check whether the tx is replay protected. If we're not in the EIP155 hf
			// phase, start ignoring the sender until we do.
			//https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md를 참조하십시오
			//상황이 정확히 두 개의 체인과 동일하기 때문에 DAO는 사건 이후, ETC하므로, ETH 이더넷 광장 등으로 분할
			//ETH를 얻을 수 위의 거래는 그 반대의 경우도 마찬가지 위에 재현합니다. 그래서 Vitalik이 상황을 피하기 위해 EIP155을 제안했다.
			if tx.Protected() && !env.config.IsEIP155(env.header.Number) {
				log.Trace("Ignoring reply protected transaction", "hash", tx.Hash(), "eip155", env.config.EIP155Block)
	
				txs.Pop()
				continue
			}
			// Start executing the transaction
			env.state.Prepare(tx.Hash(), common.Hash{}, env.tcount)
			//거래의 실행
			err, logs := env.commitTransaction(tx, bc, coinbase, gp)
			switch err {
			case core.ErrGasLimitReached:
				// Pop the current out-of-gas transaction without shifting in the next from the account
				//모든 거래의 전체 계정 팝, 트랜잭션은 다음 사용자를 처리하지 않습니다.
				log.Trace("Gas limit exceeded for current block", "sender", from)
				txs.Pop()
	
			case core.ErrNonceTooLow:
				// New head notification data race between the transaction pool and miner, shift
				//사용자의 다음 거래로 이동
				log.Trace("Skipping transaction with low nonce", "sender", from, "nonce", tx.Nonce())
				txs.Shift()
	
			case core.ErrNonceTooHigh:
				// Reorg notification data race between the transaction pool and miner, skip account =
				//이 계정을 건너 뛰기
				log.Trace("Skipping account with hight nonce", "sender", from, "nonce", tx.Nonce())
				txs.Pop()
	
			case nil:
				// Everything ok, collect the logs and shift in the next transaction from the same account
				coalescedLogs = append(coalescedLogs, logs...)
				env.tcount++
				txs.Shift()
	
			default:
				// Strange error, discard the transaction and get the next in line (note, the
				// nonce-too-high clause will prevent us from executing in vain).
				//다른 이상한 오류는이 거래를 건너 뜁니다.
				log.Debug("Transaction failed, account skipped", "hash", tx.Hash(), "err", err)
				txs.Shift()
			}
		}
	
		if len(coalescedLogs) > 0 || env.tcount > 0 {
			// make a copy, the state caches the logs and these logs get "upgraded" from pending to mined
			// logs by filling in the block hash when the block was mined by the local miner. This can
			// cause a race condition if a log was "upgraded" before the PendingLogsEvent is processed.
			//로그 아웃 보내고, 여기에 로그온 할 필요가의 채광이 완료된 후 수정해야하기 때문에, 경합을 피하기 위해 사본을 전송됩니다.
			cpy := make([]*types.Log, len(coalescedLogs))
			for i, l := range coalescedLogs {
				cpy[i] = new(types.Log)
				*cpy[i] = *l
			}
			go func(logs []*types.Log, tcount int) {
				if len(logs) > 0 {
					mux.Post(core.PendingLogsEvent{Logs: logs})
				}
				if tcount > 0 {
					mux.Post(core.PendingStateEvent{})
				}
			}(cpy, env.tcount)
		}
	}

이 commitTransaction 실행 ApplyTransaction
	
	func (env *Work) commitTransaction(tx *types.Transaction, bc *core.BlockChain, coinbase common.Address, gp *core.GasPool) (error, []*types.Log) {
		snap := env.state.Snapshot()
	
		receipt, _, err := core.ApplyTransaction(env.config, bc, &coinbase, gp, env.state, env.header, tx, env.header.GasUsed, vm.Config{})
		if err != nil {
			env.state.RevertToSnapshot(snap)
			return err, nil
		}
		env.txs = append(env.txs, tx)
		env.receipts = append(env.receipts, receipt)
	
		return nil, receipt.Logs
	}

광산의 결과를 수신하기위한 대기 기능 및 ETH 프로토콜 중계하면서 로컬 블록 체인 물품.
	
	func (self *worker) wait() {
		for {
			mustCommitNewWork := true
			for result := range self.recv {
				atomic.AddInt32(&self.atWork, -1)
	
				if result == nil {
					continue
				}
				block := result.Block
				work := result.Work
	
				// Update the block hash in all logs since it is now available and not when the
				// receipt/log of individual transactions were created.
				for _, r := range work.receipts {
					for _, l := range r.Logs {
						l.BlockHash = block.Hash()
					}
				}
				for _, log := range work.state.Logs() {
					log.BlockHash = block.Hash()
				}
				stat, err := self.chain.WriteBlockAndState(block, work.receipts, work.state)
				if err != nil {
					log.Error("Failed writing block to chain", "err", err)
					continue
				}
				// check if canon block and write transactions
				if stat == core.CanonStatTy { //설명 체인 블록 사양에 삽입 된
					// implicit by posting ChainHeadEvent
					//이 상태 때문에,이 코드는 commitNewWork 것입니다 위의 코드의 업데이트를 트리거하는 ChainHeadEvent를 보낼 것입니다, 그래서 커밋의 필요가 없습니다.
					mustCommitNewWork = false
				}	
				// Broadcast the block and announce chain insertion event
				//브로드 캐스트 블록 및 블록 사슬 긍정 삽입 이벤트.
				self.mux.Post(core.NewMinedBlockEvent{Block: block})
				var (
					events []interface{}
					logs   = work.state.Logs()
				)
				events = append(events, core.ChainEvent{Block: block, Hash: block.Hash(), Logs: logs})
				if stat == core.CanonStatTy {
					events = append(events, core.ChainHeadEvent{Block: block})
				}
				self.chain.PostChainEvents(events, logs)
	
				// Insert the block into the set of pending ones to wait for confirmations
				//로컬 트랙 목록에 이후의 접수 상태를 볼 수 있습니다.
				self.unconfirmed.Insert(block.NumberU64(), block.Hash())
	
				if mustCommitNewWork { // TODO ? 
					self.commitNewWork()
				}
			}
		}
	}


## miner
광부 노동자, 관리 외부 이벤트에 가입, 시작을 제어하고 작업자의 중지하는 데 사용.

데이터 구조

	
	// Backend wraps all methods required for mining.
	type Backend interface {
		AccountManager() *accounts.Manager
		BlockChain() *core.BlockChain
		TxPool() *core.TxPool
		ChainDb() ethdb.Database
	}
	
	// Miner creates blocks and searches for proof-of-work values.
	type Miner struct {
		mux *event.TypeMux
	
		worker *worker
	
		coinbase common.Address
		mining   int32
		eth	  Backend
		engine   consensus.Engine
	
		canStart	int32 // can start indicates whether we can start the mining operation
		shouldStart int32 // should start indicates whether we should start after sync
	}


건설하는 CPU 에이전트가 업데이트 goroutine 광부의 시작 만들

	
	func New(eth Backend, config *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine) *Miner {
		miner := &Miner{
			eth:	  eth,
			mux:	  mux,
			engine:   engine,
			worker:   newWorker(config, engine, common.Address{}, eth, mux),
			canStart: 1,
		}
		miner.Register(NewCpuAgent(eth.BlockChain(), engine))
		go miner.update()
	
		return miner
	}

갱신 가입 다운 이벤트, 1 canStart로 설정됩니다 단지 downloader.DoneEvent 또는 downloader.FailedEvent 이벤트의 다운을 받아, 일회성는이 goroutine주기에주의를 지불하고 루프를 종료, 이것은 악의적 인 해커를 방지하는 것입니다 DOS는 비정상적인 상태를 유지, 공격
	
	// update keeps track of the downloader events. Please be aware that this is a one shot type of update loop.
	// It's entered once and as soon as `Done` or `Failed` has been broadcasted the events are unregistered and
	// the loop is exited. This to prevent a major security vuln where external parties can DOS you with blocks
	// and halt your mining operation for as long as the DOS continues.
	func (self *Miner) update() {
		events := self.mux.Subscribe(downloader.StartEvent{}, downloader.DoneEvent{}, downloader.FailedEvent{})
	out:
		for ev := range events.Chan() {
			switch ev.Data.(type) {
			case downloader.StartEvent:
				atomic.StoreInt32(&self.canStart, 0)
				if self.Mining() {
					self.Stop()
					atomic.StoreInt32(&self.shouldStart, 1)
					log.Info("Mining aborted due to sync")
				}
			case downloader.DoneEvent, downloader.FailedEvent:
				shouldStart := atomic.LoadInt32(&self.shouldStart) == 1
	
				atomic.StoreInt32(&self.canStart, 1)
				atomic.StoreInt32(&self.shouldStart, 0)
				if shouldStart {
					self.Start(self.coinbase)
				}
				// unsubscribe. we're only interested in this event once
				events.Unsubscribe()
				// stop immediately and ignore all further pending events
				break out
			}
		}
	}

Start
	
	func (self *Miner) Start(coinbase common.Address) {
		atomic.StoreInt32(&self.shouldStart, 1)  //shouldStart는 시작할지 여부입니다
		self.worker.setEtherbase(coinbase)		 
		self.coinbase = coinbase
	
		if atomic.LoadInt32(&self.canStart) == 0 {  //시작할지 여부를 canStart 수,
			log.Info("Network syncing, will start miner afterwards")
			return
		}
		atomic.StoreInt32(&self.mining, 1)
	
		log.Info("Starting mining operation")
		self.worker.start()  //시작 노동자는 광산을 시작했다
		self.worker.commitNewWork()  //새로운 광산 작업을 제출합니다.
	}