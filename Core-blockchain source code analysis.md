보기 테스트 케이스의 관점에서, 점의 주요 기능은 다음과 같은 점을 blockchain.

1. import.
2. GetLastBlock 기능.
3. 만약 가장 어려운 사양 블록 체인 중 하나로서 선택 될 수있는 블록 사슬 복수.
4. BadHashes 수동으로 blocks.go 내부의 금지 블록 해시 값을 받아들입니다.
금지 및 활성 상태를 입력 할 때 5. 새로운 구성 BadHashes 경우. 그런 다음 블록이 자동으로 시작된다.
6. 잘못된 넌스이 거부됩니다.
7. 지원 빠른 가져 오기.
8. 조명 효과 상기 전체 처리 블록 헤더와 동일한 처리 영역 대 고속 대.

Blockchain 주 기능 검증 블록 삽입 쿼리 상태를 포함하는 상태 블록 사슬을 유지하는 것을 볼 수있다.

용어 :

표준 체인 블록은 무엇입니까

블록의 생성하기 때문에, 어떤 우리의 데이터베이스에 기록하는 짧은 시간에 분기, 실제로 나무의 블록을 가질 수있다. 우리는 우리가 생각하는 경로의 어려움의 가장 높은 총 학위 중 하나가 규범이라고 생각 블록 사슬.이 있지만 따라서, 많은 블록이 형성 될 수있는 블록 사슬, 블록 사슬하지만 표준화되지.

데이터베이스 구조 :

	해시 값과 지구 블록 블록 헤더의 해시 값은 무엇과 동일합니다. 소위 블록의 해시 값은 실제로 헤더 값의 블록입니다.
	// key -> value
	//+ 연결을 나타냅니다

	&quot;LastHeader&quot;을 사용하여 최신 헤더 HeaderChain 영역
	&quot;LastBlock는&quot;최신 블록 헤더 BlockChain 영역을 사용하는
	"LastFast"헤더 영역의 최신 빠른 동기화

	&quot;H&quot;+ NUM + &quot;N&quot;-&gt; 헤더의 높이 해시의 해시 값과 사양을 저장하는 블록 사슬 영역
	
	&quot;H&quot;+ NUM + 해시 -&gt; 헤더 높이 + 해시 값 -&gt; 헤더 영역
	
	&quot;H&quot;+ NUM + 해시 + &quot;t&quot;-&gt; TD 높이 + 해시 값 -&gt; 총 난이도
	
	&quot;H&quot;+ 해시 -&gt; NUM 블록 체 해시 -&gt; 높이
	
	&quot;B&quot;+ NUM + 해시 -&gt; 블록 체 높이 + 해시 값 -&gt; 블록 체
	
	&quot;R&quot;+ NUM + 해시 -&gt; 블록 영수증 높이 + 해시 값 -&gt; 차단 입고
	
	"l" + hash -> transaction/receipt lookup metadata

키 | 값 | 설명 | 삽입 | 삭제 |
---- | --- |---------------|-----------|-----------|
&quot;LastHeader&quot;| 헤드 쇄 대체 블록 형제 블록 사슬 업데이트 또는 분지 된 | 블록이 블록 사슬 헤드의 최신 규격으로 간주된다 | 해시 | 최신 영역 헤더 HeaderChain 사용 그것
&quot;LastBlock&quot;| 해시 | 최근 헤더 BlockChain 영역 사용 | 현재 블록 헤드 명세서의 체인에서 최신으로 간주되는 블록 | 갱신되면 첫번째 블록 사슬 또는 분지 쇄 대체 블록 형제 그것
블록이 현재 블록 사슬 최신 스펙 향하는 것으로 간주 될 때 |가 | &quot;LastFast는&quot;| 해시 | 최근 헤더 BlockChain 영역 사용 업데이트되었습니다 첫번째 블록 사슬 또는 분지 쇄 블록 형제를 교체 할 때 그것
명세서 체인의 블록이 때 블록 | | 영역 블록이 표준화되어 있지 않은 경우 HeaderChain를 사용하여 표준 높이와 블록 사슬의 헤더 영역의 해시 값을 저장 | 해시 | &quot;H&quot;+ NUM + &quot;N&quot; 블록 사슬
&quot;H&quot;+ NUM + 해시 + &quot;t&quot;| TD | 총 어려움 | WriteBlockAndState 검증 및 (표준화 여부) 블록을 완료 한 후에 실행 | SetHead 메소드 호출. 이 방법은, 하나는 현재 블록 체인 badhashs이 포함되어 있습니다, 두 경우 모두에서 호출 badhashs의 시작에서 모든 블록을 제거해야 할 것, 2. 기원을 재설정 할 필요가 블록 오류의 현재 상태입니다.
WriteBlockAndState 후되면 전체 블록의 검증 및 구현 | | &quot;H&quot;해시 + | NUM | HeaderChain SetHead에서 사용되는 블록의 높이라고 저서
&quot;B&quot;+ NUM + 해시 | 블록 체는 | WriteBlockAndState 또는 InsertReceiptChain | | SetHead 데이터 블록은, 책이 삭제된다.
&quot;R&quot;+ NUM + 해시 | 블록 영수증 | 블록 영수증 | WriteBlockAndState 또는 InsertReceiptChain | 같은 책.
&quot;L&quot;+ txHash | {해시 NUM, TxIndex | 트랜잭션 해시 블록 거래에서 찾을 수있다 | 표준 블록 사슬 블록에 추가 할 때 | 블록 블록은 표준 체인으로부터 제거 될 때



데이터 구조

	
	// BlockChain represents the canonical chain given a database with a genesis
	// block. The Blockchain manages chain imports, reverts, chain reorganisations.
	//Blockchain이 체인 블록 생성하여 지정된 데이터베이스를 포함하는, 표준 체인을 나타낸다. Blockchain 삽입 체인 관리, 감소, 재구성 동작.
	// Importing blocks in to the block chain happens according to the set of rules
	// defined by the two stage Validator. Processing of blocks is done using the
	// Processor which processes the included transaction. The validation of the state
	// is done in the second part of the Validator. Failing results in aborting of
	// the import.
	//일련의 규칙을 통해 블록을 삽입하면 지정된 단계 두 검증자를 지정합니다.
	//블록 거래. 확인 상태를 처리하기 위해 사용되는 프로세서는 검사기의 두 번째 단계이다. 오류 중지의 삽입을 야기한다.
	// The BlockChain also helps in returning blocks from **any** chain included
	// in the database as well as blocks that represents the canonical chain. It's
	// important to note that GetBlock can return any block and does not need to be
	// included in the canonical one where as GetBlockByNumber always represents the
	// canonical chain.
	//현재 GetBlock 규격이 블록 사슬의 임의의 블록을 반환하지 않을 수 있음에 유의
	//그러나 GetBlockByNumber는 항상 현재 사양의 블록 체인 블록을 반환합니다.
	type BlockChain struct {
		config *params.ChainConfig // chain & network configuration
	
		hc			*HeaderChain	//는 블록 사슬의 헤더 영역을 포함
		chainDb	   ethdb.Database// 기본 데이터베이스
		rmLogsFeed	event.Feed	// 다음 많은 메시지 알림의 구성 요소는
		chainFeed	 event.Feed
		chainSideFeed event.Feed
		chainHeadFeed event.Feed
		logsFeed	  event.Feed
		scope		 event.SubscriptionScope
		genesisBlock  *types.Block	// 생성 블록
	
		mu	  sync.RWMutex // global mutex for locking chain operations
		chainmu sync.RWMutex // blockchain insertion lock
		procmu  sync.RWMutex // block processor lock
	
		checkpoint	   int		  // checkpoint counts towards the new checkpoint
		currentBlock * Types.Block // 현재 헤더의 블록 사슬 영역의 현재 헤드
		currentFastBlock *types.Block //고속 동기 체인의 현재 헤드 (블록 사슬 이상일 수있다!) 현재 빠른 동기 헤더 영역.
	
		stateCache   state.Database // State database to reuse between imports (contains state cache)
		bodyCache	*lru.Cache	 // Cache for the most recent block bodies
		bodyRLPCache *lru.Cache	 // Cache for the most recent block bodies in RLP encoded format
		blockCache   *lru.Cache	 // Cache for the most recent entire blocks
		futureBlocks *lru.Cache // 미래 블록은 여전히 ​​블록 저장 위치에 삽입되지 않은 후 처리 블록을 추가한다.
	
		quit	chan struct{} // blockchain quit channel
		running int32		 // running must be called atomically
		// procInterrupt must be atomically called
		procInterrupt int32		  // interrupt signaler for block processing
		wg			sync.WaitGroup // chain processing wait group for shutting down
	
		engine	consensus.Engine// 일관성 엔진
		processor Processor //블록 프로세서 인터페이스 블록 // 프로세서 인터페이스
		validator Validator //블록 및 상태 검증 인터페이스 // 블록 상태 확인 인터페이스
		vmConfig  vm.Config //가상 컴퓨터 구성
	
		badBlocks *lru.Cache //나쁜 블록 캐시 오류 블록 캐시.
	}
	


기본 인증 스퀘어 에테르 및 프로세서를 초기화하는 구성, NewBlockChain 가능한 정보 데이터베이스가 좋은 초기화 블록 사슬 내부 구성된다 (검사기 프로세서)

	
	// NewBlockChain returns a fully initialised block chain using information
	// available in the database. It initialises the default Ethereum Validator and
	// Processor.
	func NewBlockChain(chainDb ethdb.Database, config *params.ChainConfig, engine consensus.Engine, vmConfig vm.Config) (*BlockChain, error) {
		bodyCache, _ := lru.New(bodyCacheLimit)
		bodyRLPCache, _ := lru.New(bodyCacheLimit)
		blockCache, _ := lru.New(blockCacheLimit)
		futureBlocks, _ := lru.New(maxFutureBlocks)
		badBlocks, _ := lru.New(badBlockLimit)
	
		bc := &BlockChain{
			config:	   config,
			chainDb:	  chainDb,
			stateCache:   state.NewDatabase(chainDb),
			quit:		 make(chan struct{}),
			bodyCache:	bodyCache,
			bodyRLPCache: bodyRLPCache,
			blockCache:   blockCache,
			futureBlocks: futureBlocks,
			engine:	   engine,
			vmConfig:	 vmConfig,
			badBlocks:	badBlocks,
		}
		bc.SetValidator(NewBlockValidator(config, bc, engine))
		bc.SetProcessor(NewStateProcessor(config, bc, engine))
	
		var err error
		bc.hc, err = NewHeaderChain(chainDb, config, engine, bc.getProcInterrupt)
		if err != nil {
			return nil, err
		}
		bc.genesisBlock = bc.GetBlockByNumber(0)  //블록 생성을 가져 오기
		if bc.genesisBlock == nil {
			return nil, ErrNoGenesis
		}
		if err := bc.loadLastState(); err != nil { //최신로드
			return nil, err
		}
		// Check the current state of the block hashes and make sure that we do not have any of the bad blocks in our chain
		//현재의 상태를 확인하고 우리의 블록 사슬의 꼭대기에 그 어떤 불법적 인 블록을 확인하지 않습니다.
		//하드 분기에 사용되는 몇 가지 수동 구성 블록 해시 값을 BadHashes.
		for hash := range BadHashes {
			if header := bc.GetHeaderByHash(hash); header != nil {
				// get the canonical block corresponding to the offending header's number
				//이 지역은 우리의 사양 헤더 블록 체인에 참이면 우리가 하나의 헤더 영역의 높이로 롤백해야하는, 헤더 영역 위의 같은 높이의 표준화 된 블록 체인을 얻기 -
				headerByNumber := bc.GetHeaderByNumber(header.Number.Uint64())
				// make sure the headerByNumber (if present) is in our current canonical chain
				if headerByNumber != nil && headerByNumber.Hash() == header.Hash() {
					log.Error("Found bad hash, rewinding chain", "number", header.Number, "hash", header.ParentHash)
					bc.SetHead(header.Number.Uint64() - 1)
					log.Error("Chain rewind was successful, resuming normal operation")
				}
			}
		}
		// Take ownership of this particular state
		go bc.update()
		return bc, nil
	}

loadLastState, 우리는 블록 체인의 최신 상태를 알고있는 데이터베이스 내부에로드.이 방법을 사용하면 잠금을 얻을 수 있다고 가정합니다.
	
	// loadLastState loads the last known chain state from the database. This method
	// assumes that the chain manager mutex is held.
	func (bc *BlockChain) loadLastState() error {
		// Restore the last known head block
		//해시 우리는 최신 블록을 알고 돌려줍니다
		head := GetHeadBlockHash(bc.chainDb)
		if head == (common.Hash{}) { //빈받은 후. 데이터베이스가 손상 그래서 생각합니다. 그런 다음 생성 블록을 차단하는 체인을 설정합니다.
			// Corrupt or empty database, init from scratch
			log.Warn("Empty database, resetting chain")
			return bc.Reset()
		}
		// Make sure the entire head block is available
		//blockHash을 따르면 블록을 볼 수 있습니다
		currentBlock := bc.GetBlockByHash(head)
		if currentBlock == nil {
			// Corrupt or empty database, init from scratch
			log.Warn("Head block missing, resetting chain", "hash", head)
			return bc.Reset()
		}
		// Make sure the state associated with the block is available
		//세계 상태와이 블록의 확인이 정확합니다.
		if _, err := state.New(currentBlock.Root(), bc.stateCache); err != nil {
			// Dangling block without a state associated, init from scratch
			log.Warn("Head state missing, resetting chain", "number", currentBlock.Number(), "hash", currentBlock.Hash())
			return bc.Reset()
		}
		// Everything seems to be fine, set as the head block
		bc.currentBlock = currentBlock
	
		// Restore the last known head header
		//헤더 영역의 최신 해시를 구합니다
		currentHeader := bc.currentBlock.Header()
		if head := GetHeadHeaderHash(bc.chainDb); head != (common.Hash{}) {
			if header := bc.GetHeaderByHash(head); header != nil {
				currentHeader = header
			}
		}
		//헤더 체인 현재 구역 헤더로 설정.
		bc.hc.SetCurrentHeader(currentHeader)
	
		// Restore the last known head fast block
		bc.currentFastBlock = bc.currentBlock
		if head := GetHeadFastBlockHash(bc.chainDb); head != (common.Hash{}) {
			if block := bc.GetBlockByHash(head); block != nil {
				bc.currentFastBlock = block
			}
		}
	
		//인쇄 로그에 대한 사용자의 상태 로그를 실행합니다.
		headerTd := bc.GetTd(currentHeader.Hash(), currentHeader.Number.Uint64())
		blockTd := bc.GetTd(bc.currentBlock.Hash(), bc.currentBlock.NumberU64())
		fastTd := bc.GetTd(bc.currentFastBlock.Hash(), bc.currentFastBlock.NumberU64())
	
		log.Info("Loaded most recent local header", "number", currentHeader.Number, "hash", currentHeader.Hash(), "td", headerTd)
		log.Info("Loaded most recent local full block", "number", bc.currentBlock.Number(), "hash", bc.currentBlock.Hash(), "td", blockTd)
		log.Info("Loaded most recent local fast block", "number", bc.currentFastBlock.Number(), "hash", bc.currentFastBlock.Hash(), "td", fastTd)
	
		return nil
	}

goroutine 업데이트 과정은 매우 간단합니다. 타이머 처리 미래의 블록.
	
	func (bc *BlockChain) update() {
		futureTimer := time.Tick(5 * time.Second)
		for {
			select {
			case <-futureTimer:
				bc.procFutureBlocks()
			case <-bc.quit:
				return
			}
		}
	}

초기화 방법의 블록 사슬 재설정.
	
	// Reset purges the entire blockchain, restoring it to its genesis state.
	func (bc *BlockChain) Reset() error {
		return bc.ResetWithGenesisBlock(bc.genesisBlock)
	}
	
	// ResetWithGenesisBlock purges the entire blockchain, restoring it to the
	// specified genesis state.
	func (bc *BlockChain) ResetWithGenesisBlock(genesis *types.Block) error {
		// Dump the entire block chain and purge the caches
		//전체 블록 체인 덤프하고 캐시를 지우
		if err := bc.SetHead(0); err != nil {
			return err
		}
		bc.mu.Lock()
		defer bc.mu.Unlock()
	
		// Prepare the genesis block and reinitialise the chain
		//블록 생성 사슬을 재 초기화하는 블록을 사용
		if err := bc.hc.WriteTd(genesis.Hash(), genesis.NumberU64(), genesis.Difficulty()); err != nil {
			log.Crit("Failed to write genesis block TD", "err", err)
		}
		if err := WriteBlock(bc.chainDb, genesis); err != nil {
			log.Crit("Failed to write genesis block", "err", err)
		}
		bc.genesisBlock = genesis
		bc.insert(bc.genesisBlock)
		bc.currentBlock = bc.genesisBlock
		bc.hc.SetGenesis(bc.genesisBlock.Header())
		bc.hc.SetCurrentHeader(bc.genesisBlock.Header())
		bc.currentFastBlock = bc.genesisBlock
	
		return nil
	}

새로운 머리를 롤백 지역 체인을 SetHead. 새로운 헤더에 주어진 모든 컨텐츠 삭제됩니다, 새로운 헤더가 설정됩니다. 블록은 (고속 동기화 후 비 압축 노드) 손실 된 경우, 상기 헤드는 또한 되감기 할 수있다.
	
	// SetHead rewinds the local chain to a new head. In the case of headers, everything
	// above the new head will be deleted and the new one set. In the case of blocks
	// though, the head may be further rewound if block bodies are missing (non-archive
	// nodes after a fast sync).
	func (bc *BlockChain) SetHead(head uint64) error {
		log.Warn("Rewinding blockchain", "target", head)
	
		bc.mu.Lock()
		defer bc.mu.Unlock()
	
		// Rewind the header chain, deleting all block bodies until then
		delFn := func(hash common.Hash, num uint64) {
			DeleteBody(bc.chainDb, hash, num)
		}
		bc.hc.SetHead(head, delFn)
		currentHeader := bc.hc.CurrentHeader()
	
		// Clear out any stale content from the caches
		bc.bodyCache.Purge()
		bc.bodyRLPCache.Purge()
		bc.blockCache.Purge()
		bc.futureBlocks.Purge()
	
		// Rewind the block chain, ensuring we don't end up with a stateless head block
		if bc.currentBlock != nil && currentHeader.Number.Uint64() < bc.currentBlock.NumberU64() {
			bc.currentBlock = bc.GetBlock(currentHeader.Hash(), currentHeader.Number.Uint64())
		}
		if bc.currentBlock != nil {
			if _, err := state.New(bc.currentBlock.Root(), bc.stateCache); err != nil {
				// Rewound state missing, rolled back to before pivot, reset to genesis
				bc.currentBlock = nil
			}
		}
		// Rewind the fast block in a simpleton way to the target head
		if bc.currentFastBlock != nil && currentHeader.Number.Uint64() < bc.currentFastBlock.NumberU64() {
			bc.currentFastBlock = bc.GetBlock(currentHeader.Hash(), currentHeader.Number.Uint64())
		}
		// If either blocks reached nil, reset to the genesis state
		if bc.currentBlock == nil {
			bc.currentBlock = bc.genesisBlock
		}
		if bc.currentFastBlock == nil {
			bc.currentFastBlock = bc.genesisBlock
		}
		if err := WriteHeadBlockHash(bc.chainDb, bc.currentBlock.Hash()); err != nil {
			log.Crit("Failed to reset head full block", "err", err)
		}
		if err := WriteHeadFastBlockHash(bc.chainDb, bc.currentFastBlock.Hash()); err != nil {
			log.Crit("Failed to reset head fast block", "err", err)
		}
		return bc.loadLastState()
	}

InsertChain는 체인의 명세서에 삽입되어 소정의 블록의 블록을 삽입하는 블록 사슬 체인 시도 삽입 또는 포크를 생성한다. 에러가 발생하면 에러가 발생하면, 인덱스 특정 에러 메시지를 반환한다.
	
	// InsertChain attempts to insert the given batch of blocks in to the canonical
	// chain or, otherwise, create a fork. If an error is returned it will return
	// the index number of the failing block as well an error describing what went
	// wrong.
	//
	// After insertion is done, all accumulated events will be fired.
	//삽입이 완료되면, 모든 축적 된 이벤트가 트리거됩니다.
	func (bc *BlockChain) InsertChain(chain types.Blocks) (int, error) {
		n, events, logs, err := bc.insertChain(chain)
		bc.PostChainEvents(events, logs)
		return n, err
	}

insertChain 방법으로 인해 핸들을 잠금 해제 연기를 사용할 필요가 있으므로 별도의 방법으로서,이 방법. 블록 사슬이 삽입 수행하고, 이벤트 정보를 수집한다.

	// insertChain will execute the actual chain insertion and event aggregation. The
	// only reason this method exists as a separate one is to make locking cleaner
	// with deferred statements.
	func (bc *BlockChain) insertChain(chain types.Blocks) (int, []interface{}, []*types.Log, error) {
		// Do a sanity check that the provided chain is actually ordered and linked
		//사운드 체크를 수행, 체인이 실제로 질서와 상호 링크입니다 제공
		for i := 1; i < len(chain); i++ {
			if chain[i].NumberU64() != chain[i-1].NumberU64()+1 || chain[i].ParentHash() != chain[i-1].Hash() {
				// Chain broke ancestry, log a messge (programming error) and skip insertion
				log.Error("Non contiguous block insert", "number", chain[i].Number(), "hash", chain[i].Hash(),
					"parent", chain[i].ParentHash(), "prevnumber", chain[i-1].Number(), "prevhash", chain[i-1].Hash())
	
				NIL로는 무기 호, fmt.Errorf (0 복귀 &quot;비 연속 인서트 : 아이템 % (D)가 # % d에 인 [% X를 ..., 상품 % D는 # % d에있다 [%의 X ... (부모 [%의 X ...])&quot; I-1, 체인 [I-1] .NumberU64 ()
					chain[i-1].Hash().Bytes()[:4], i, chain[i].NumberU64(), chain[i].Hash().Bytes()[:4], chain[i].ParentHash().Bytes()[:4])
			}
		}
		// Pre-checks passed, start the full block imports
		bc.wg.Add(1)
		defer bc.wg.Done()
	
		bc.chainmu.Lock()
		defer bc.chainmu.Unlock()
	
		// A queued approach to delivering events. This is generally
		// faster than direct delivery and requires much less mutex
		// acquiring.
		var (
			stats		 = insertStats{startTime: mclock.Now()}
			events		= make([]interface{}, 0, len(chain))
			lastCanon	 *types.Block
			coalescedLogs []*types.Log
		)
		// Start the parallel header verifier
		headers := make([]*types.Header, len(chain))
		seals := make([]bool, len(chain))
	
		for i, block := range chain {
			headers[i] = block.Header()
			seals[i] = true
		}
		//엔진 영역 헤더의 일관성이 유효한지 확인하기 위해 호출합니다.
		abort, results := bc.engine.VerifyHeaders(bc, headers, seals)
		defer close(abort)
	
		// Iterate over the blocks and insert when the verifier permits
		for i, block := range chain {
			// If the chain is terminating, stop processing blocks
			if atomic.LoadInt32(&bc.procInterrupt) == 1 {
				log.Debug("Premature abort during blocks processing")
				break
			}
			// If the header is a banned one, straight out abort
			//경우 헤더 영역을 금지합니다.
			if BadHashes[block.Hash()] {
				bc.reportBlock(block, nil, ErrBlacklistedHash)
				return i, events, coalescedLogs, ErrBlacklistedHash
			}
			// Wait for the block's verification to complete
			bstart := time.Now()
	
			err := <-results
			if err == nil { //오류가없는 경우. 검증 기관
				err = bc.Validator().ValidateBody(block)
			}
			if err != nil {
				if err == ErrKnownBlock { //블록이 직접 삽입 된 경우 계속
					stats.ignored++
					continue
				}
	
				if err == consensus.ErrFutureBlock { 
					// Allow up to MaxFuture second in the future blocks. If this limit
					// is exceeded the chain is discarded and processed at a later time
					// if given.
					//이 블록의 미래, 시간의 블록 지금부터 매우 긴없는 경우. 그런 다음 저장합니다.
					max := big.NewInt(time.Now().Unix() + maxTimeFutureBlocks)
					if block.Time().Cmp(max) > 0 {
						return i, events, coalescedLogs, fmt.Errorf("future block: %v > %v", block.Time(), max)
					}
					bc.futureBlocks.Add(block.Hash(), block)
					stats.queued++
					continue
				}
	
				만약 ERR == 블록이 미래의 블록이 블록의 조상 조상 포함 발견되지 않는 경우가 bc.futureBlocks.Contains consensus.ErrUnknownAncestor &amp;&amp; (block.ParentHash는 ()) {하고 또한 장래에 저장된
					bc.futureBlocks.Add(block.Hash(), block)
					stats.queued++
					continue
				}
	
				bc.reportBlock(block, nil, err)
				return i, events, coalescedLogs, err
			}
			// Create a new statedb using the parent block and report an
			// error if it fails.
			var parent *types.Block
			if i == 0 {
				parent = bc.GetBlock(block.ParentHash(), block.NumberU64()-1)
			} else {
				parent = chain[i-1]
			}
			state, err := state.New(parent.Root(), bc.stateCache)
			if err != nil {
				return i, events, coalescedLogs, err
			}
			// Process block using the parent state as reference point.
			//블록을 처리하면 거래, 영수증, 로그 및 기타 정보를 생성합니다.
			//사실 그것은 state_processor.go 프로세스 방법 내에서 호출합니다.
			receipts, logs, usedGas, err := bc.processor.Process(block, state, bc.vmConfig)
			if err != nil {
				bc.reportBlock(block, receipts, err)
				return i, events, coalescedLogs, err
			}
			// Validate the state using the default validator
			//차 검증은 법적 상태를 확인 여부
			err = bc.Validator().ValidateState(block, parent, state, receipts, usedGas)
			if err != nil {
				bc.reportBlock(block, receipts, err)
				return i, events, coalescedLogs, err
			}
			// Write the block to the chain and get the status		
			//블록과 상태를 작성합니다.
			status, err := bc.WriteBlockAndState(block, receipts, state)
			if err != nil {
				return i, events, coalescedLogs, err
			}
			switch status {
			case CanonStatTy:  //새로운 블록을 삽입합니다.
				log.Debug("Inserted new block", "number", block.Number(), "hash", block.Hash(), "uncles", len(block.Uncles()),
					"txs", len(block.Transactions()), "gas", block.GasUsed(), "elapsed", common.PrettyDuration(time.Since(bstart)))
	
				coalescedLogs = append(coalescedLogs, logs...)
				blockInsertTimer.UpdateSince(bstart)
				events = append(events, ChainEvent{block, block.Hash(), logs})
				lastCanon = block
	
			case SideStatTy:  //갈래 블록 삽입
				log.Debug("Inserted forked block", "number", block.Number(), "hash", block.Hash(), "diff", block.Difficulty(), "elapsed",
					common.PrettyDuration(time.Since(bstart)), "txs", len(block.Transactions()), "gas", block.GasUsed(), "uncles", len(block.Uncles()))
	
				blockInsertTimer.UpdateSince(bstart)
				events = append(events, ChainSideEvent{block})
			}
			stats.processed++
			stats.usedGas += usedGas.Uint64()
			stats.report(chain, i)
		}
		// Append a single chain head event if we've progressed the chain
		//우리는 새로운 지역 헤더 및 lastCanon와 같은 최신의 헤더 영역을 생성하는 경우
		//그래서 우리는 새로운 ChainHeadEvent을 발표
		if lastCanon != nil && bc.LastBlockHash() == lastCanon.Hash() {
			events = append(events, ChainHeadEvent{lastCanon})
		}
		return 0, events, coalescedLogs, nil
	}

WriteBlockAndState 블록 기록부 체인.
	
	// WriteBlock writes the block to the chain.
	func (bc *BlockChain) WriteBlockAndState(block *types.Block, receipts []*types.Receipt, state *state.StateDB) (status WriteStatus, err error) {
		bc.wg.Add(1)
		defer bc.wg.Done()
	
		// Calculate the total difficulty of the block
		//블록의 총 어려움 계산 삽입되는
		ptd := bc.GetTd(block.ParentHash(), block.NumberU64()-1)
		if ptd == nil {
			return NonStatTy, consensus.ErrUnknownAncestor
		}
		// Make sure no inconsistent state is leaked during insertion
		//있는지 확인 삽입시에는 일관성이없는 상태 누설하지
		bc.mu.Lock()
		defer bc.mu.Unlock()
		//상기 현재 블록의 전체 블록 사슬을 계산하는 어려움.
		localTd := bc.GetTd(bc.currentBlock.Hash(), bc.currentBlock.NumberU64())
		//난이도 새로운 총 블록 사슬을 계산
		externTd := new(big.Int).Add(block.Difficulty(), ptd)
	
		// Irrelevant of the canonical status, write the block itself to the database
		//데이터베이스에 기록는 관계 블록 사양. 작성하지 않는 상태는 전체 높이와 대응하는 블록의 어려움 해시.
		if err := bc.hc.WriteTd(block.Hash(), block.NumberU64(), externTd); err != nil {
			return NonStatTy, err
		}
		// Write other block data using a batch.
		batch := bc.chainDb.NewBatch()
		if err := WriteBlock(batch, block); err != nil { //쓰기 블록
			return NonStatTy, err
		}
		if _, err := state.CommitTo(batch, bc.config.IsEIP158(block.Number())); err != nil {  //Commit
			return NonStatTy, err
		}
		if err := WriteBlockReceipts(batch, block.Hash(), block.NumberU64(), receipts); err != nil { //쓰기 블록 영수증
			return NonStatTy, err
		}
	
		// If the total difficulty is higher than our known, add it to the canonical chain
		// Second clause in the if statement reduces the vulnerability to selfish mining.
		// Please refer to http://www.cs.cornell.edu/~ie53/publications/btcProcFC.pdf
		//새로운 블록이 현재 블록의 전반적인 어려움보다 높은 경우,이 블록은 블록을 규격에 설정되어 있습니다.
		//제 식 ((externTd.Cmp (localTd) == 0 &amp;&amp; mrand.Float64 () &lt;0.5))
		//위해서는 이기적인 광산의 가능성을 줄일 수 있습니다.
		if externTd.Cmp(localTd) > 0 || (externTd.Cmp(localTd) == 0 && mrand.Float64() < 0.5) {
			// Reorganise the chain if the parent is not the head block
			//이 블록의 상위 블록이 현재 블록이 아닌 경우, 분기점이 있음을 나타냅니다. 재조직이 블록 체인을 재구성 호출해야합니다.
			if block.ParentHash() != bc.currentBlock.Hash() {
				if err := bc.reorg(bc.currentBlock, block); err != nil {
					return NonStatTy, err
				}
			}
			// Write the positional metadata for transaction and receipt lookups
			// "l" + txHash -> {blockHash,blockNum,txIndex}
			//트랜잭션을 기반으로 트랜잭션의 블록을 대응 해당 해시 값을 찾을 수 있습니다.
			if err := WriteTxLookupEntries(batch, block); err != nil {
				return NonStatTy, err
			}
			// Write hash preimages
			//해시 (Keccak-256) -이 기능에 대응&gt; 데이터를 검사하는 데 사용된다. 당신이 dev에 모드를 설정하면,
			//또는 vmdebug 매개 변수, 당신이 경우에 SHA3 명령은 프리 이미지를 추가됩니다
			if err := WritePreimages(bc.chainDb, block.NumberU64(), state.Preimages()); err != nil {
				return NonStatTy, err
			}
			status = CanonStatTy
		} else {
			status = SideStatTy
		}
		if err := batch.Write(); err != nil {
			return NonStatTy, err
		}
	
		// Set new head.
		if status == CanonStatTy {
			bc.insert(block)
		}
		bc.futureBlocks.Remove(block.Hash())
		return status, nil
	}


새로운 연쇄의 경우 reorgs 방법은 로컬 체인을 조절하는 블록 사슬 체인에 새로운 블록을 교체 할 필요없이, 전체 전체 어려움 난이도 로컬 체인보다 크다.
	
	// reorgs takes two blocks, an old chain and a new chain and will reconstruct the blocks and inserts them
	// to be part of the new canonical chain and accumulates potential missing transactions and post an
	// event about them
	//reorgs 매개 변수로 두 블록을 수용 한 이전 블록 사슬 새로운 체인 블록이며,이 방법은 그 삽입 할
	//하기 표준 블록 사슬을 재 구축. 그리고 축적 된 잠재적 인 거래를 잃고 이벤트로 외출에 게시합니다.
	func (bc *BlockChain) reorg(oldBlock, newBlock *types.Block) error {
		var (
			newChain	types.Blocks
			oldChain	types.Blocks
			commonBlock *types.Block
			deletedTxs  types.Transactions
			deletedLogs []*types.Log
			// collectLogs collects the logs that were generated during the
			// processing of the block that corresponds with the given hash.
			// These logs are later announced as deleted.
			//collectLogs 우리가 (실제로 데이터베이스에서 삭제되지 않습니다) 제거, 이러한 로그가 선언됩니다 나중에 생성 한 로그 정보를 수집합니다.
			collectLogs = func(h common.Hash) {
				// Coalesce logs and set 'Removed'.
				receipts := GetBlockReceipts(bc.chainDb, h, bc.hc.GetBlockNumber(h))
				for _, receipt := range receipts {
					for _, log := range receipt.Logs {
						del := *log
						del.Removed = true
						deletedLogs = append(deletedLogs, &del)
					}
				}
			}
		)
	
		// first reduce whoever is higher bound
		if oldBlock.NumberU64() > newBlock.NumberU64() {
			//기존의 체인이 새 체인보다 높은 경우 기존 체인을 줄일 수 있습니다. 그런 다음 필요가 이전 체인을 줄이기 위해, 그리고 새로운 체인만큼 높다
			for ; oldBlock != nil && oldBlock.NumberU64() != newBlock.NumberU64(); oldBlock = bc.GetBlock(oldBlock.ParentHash(), oldBlock.NumberU64()-1) {
				oldChain = append(oldChain, oldBlock)
				deletedTxs = append(deletedTxs, oldBlock.Transactions()...)
	
				collectLogs(oldBlock.Hash())
			}
		} else {
			// reduce new chain and append new chain blocks for inserting later on
			//새로운 체인은 기존의 체인보다 높은 경우, 새 체인을 줄일 수 있습니다.
			for ; newBlock != nil && newBlock.NumberU64() != oldBlock.NumberU64(); newBlock = bc.GetBlock(newBlock.ParentHash(), newBlock.NumberU64()-1) {
				newChain = append(newChain, newBlock)
			}
		}
		if oldBlock == nil {
			return fmt.Errorf("Invalid old chain")
		}
		if newBlock == nil {
			return fmt.Errorf("Invalid new chain")
		}
	
		for { //for 루프는 공통 조상을 찾을 필요가있다.
			if oldBlock.Hash() == newBlock.Hash() {
				commonBlock = oldBlock
				break
			}
	
			oldChain = append(oldChain, oldBlock)
			newChain = append(newChain, newBlock)
			deletedTxs = append(deletedTxs, oldBlock.Transactions()...)
			collectLogs(oldBlock.Hash())
	
			oldBlock, newBlock = bc.GetBlock(oldBlock.ParentHash(), oldBlock.NumberU64()-1), bc.GetBlock(newBlock.ParentHash(), newBlock.NumberU64()-1)
			if oldBlock == nil {
				return fmt.Errorf("Invalid old chain")
			}
			if newBlock == nil {
				return fmt.Errorf("Invalid new chain")
			}
		}
		// Ensure the user sees large reorgs
		if len(oldChain) > 0 && len(newChain) > 0 {
			logFn := log.Debug
			if len(oldChain) > 63 {
				logFn = log.Warn
			}
			logFn("Chain split detected", "number", commonBlock.Number(), "hash", commonBlock.Hash(),
				"drop", len(oldChain), "dropfrom", oldChain[0].Hash(), "add", len(newChain), "addfrom", newChain[0].Hash())
		} else {
			log.Error("Impossible reorg, please file an issue", "oldnum", oldBlock.Number(), "oldhash", oldBlock.Hash(), "newnum", newBlock.Number(), "newhash", newBlock.Hash())
		}
		var addedTxs types.Transactions
		// insert blocks. Order does not matter. Last block will be written in ImportChain itself which creates the new head properly
		for _, block := range newChain {
			// insert the block in the canonical way, re-writing history
			//삽입 블록 업데이트 기록 열쇠 고리 블록 사양
			bc.insert(block)
			// write lookup entries for hash based transaction/receipt searches
			//서면 질의 정보 거래.
			if err := WriteTxLookupEntries(bc.chainDb, block); err != nil {
				return err
			}
			addedTxs = append(addedTxs, block.Transactions()...)
		}
	
		// calculate the difference between deleted and added transactions
		diff := types.TxDifference(deletedTxs, addedTxs)
		// When transactions get deleted from the database that means the
		// receipts that were created in the fork must also be deleted
		//삭제 그 거래는 쿼리 정보를 삭제해야합니다.
		//여기 삭제 될 그 블록, 헤더 영역, 영수증 및 기타 정보를 제거하지 않습니다.
		for _, tx := range diff {
			DeleteTxLookupEntry(bc.chainDb, tx.Hash())
		}
		if len(deletedLogs) > 0 { //메시지 알림 보내기
			go bc.rmLogsFeed.Send(RemovedLogsEvent{deletedLogs})
		}
		if len(oldChain) > 0 {
			go func() {
				for _, block := range oldChain { //메시지 알림을 보냅니다.
					bc.chainSideFeed.Send(ChainSideEvent{Block: block})
				}
			}()
		}
	
		return nil
	}

