. 생성 블록 기원 의미 룰에 의해 형성된 동일한 작성 부에서 시작하는 블록 사슬이다. 다른 네트워크는 다른 생성, 테스트 블록 및 기본 네트워크 생성 블록의 웹을 그것은 다르다.

생성 블록이 존재하지 않는 경우,이 모듈은 수신 및 데이터베이스 기원의 초기 값을 기반으로 상기 상태의 기원은, 설정하는 내부 데이터베이스를 생성한다.

데이터 구조
	
	// Genesis specifies the header fields, state of a genesis block. It also defines hard
	// fork switch-over blocks through the chain configuration.
	//제네시스 특정 헤더 필드, 시작 블록의 상태. 그것은 하드 스위칭 블록 포크의 구성에 의해 정의된다.
	type Genesis struct {
		Config	 *params.ChainConfig `json:"config"`
		Nonce	  uint64			  `json:"nonce"`
		Timestamp  uint64			  `json:"timestamp"`
		ExtraData  []byte			  `json:"extraData"`
		GasLimit   uint64			  `json:"gasLimit"   gencodec:"required"`
		Difficulty *big.Int			`json:"difficulty" gencodec:"required"`
		Mixhash	common.Hash		 `json:"mixHash"`
		Coinbase   common.Address	  `json:"coinbase"`
		Alloc	  GenesisAlloc		`json:"alloc"	  gencodec:"required"`
	
		// These fields are used for consensus tests. Please don't use them
		// in actual genesis blocks.
		Number	 uint64	  `json:"number"`
		GasUsed	uint64	  `json:"gasUsed"`
		ParentHash common.Hash `json:"parentHash"`
	}
	
	// GenesisAlloc specifies the initial state that is part of the genesis block.
	//GenesisAlloc 블록의 시작의 초기 상태를 지정한다.
	type GenesisAlloc map[common.Address]GenesisAccount


SetupGenesisBlock,
	
	// SetupGenesisBlock writes or updates the genesis block in db.
	// 
	// The block that will be used is:
	//
	//						  genesis == nil	   genesis != nil
	//					   +------------------------------------------
	//	 db has no genesis |  main-net default  |  genesis
	//	 db has genesis	|  from DB		   |  genesis (if compatible)
	//
	// The stored chain configuration will be updated if it is compatible (i.e. does not
	// specify a fork block below the local head block). In case of a conflict, the
	// error is a *params.ConfigCompatError and the new, unwritten config is returned.
	//호환되지 않는 블록 사슬 구성 저장소) (갱신 될 경우. 충돌을 피하기 위하여, 에러를 반환하고 새로운 구성 원래 구성을 리턴한다.
	// The returned chain configuration is never nil.

	//testnet dev에 기원 또는 rinkeby 모드 경우, 전무 없습니다. mainnet 또는 개인 링크합니다. 그것은 비어
	func SetupGenesisBlock(db ethdb.Database, genesis *Genesis) (*params.ChainConfig, common.Hash, error) {
		if genesis != nil && genesis.Config == nil {
			return params.AllProtocolChanges, common.Hash{}, errGenesisNoConfig
		}
	
		// Just commit the new block if there is no stored genesis block.
		stored := GetCanonicalHash(db, 0) //해당 블록의 기원을 얻기
		if (stored == common.Hash{}) { //대부분의 시작 차단하지 않으면 geth는 여기에 입력됩니다.
			if genesis == nil { 
				//창세기는 또한 주요 네트워크를 사용하여 저장 한 후 전무 전무하고있는 경우
				//창세기 rinkeby 테스트 디바이스가 비어 있지 않은 경우, 자신의 기원을 설정합니다
				log.Info("Writing default main-net genesis block")
				genesis = DefaultGenesisBlock()
			} else { //그렇지 않으면, 구성 블록을 사용
				log.Info("Writing custom genesis block")
			}
			//데이터베이스에 기록
			block, err := genesis.Commit(db)
			return genesis.Config, block.Hash(), err
		}
	
		// Check whether the genesis block is already written.
		if genesis != nil { //창세기 블록이 존재하고있는 경우 이렇게 두 블록의 비교는 동일
			block, _ := genesis.ToBlock()
			hash := block.Hash()
			if hash != stored {
				return genesis.Config, block.Hash(), &GenesisMismatchError{stored, hash}
			}
		}
	
		// Get the existing chain configuration.
		//현재 존재 창세기 구성 블록 체인 받기
		newcfg := genesis.configOrDefault(stored)
		//블록 사슬의 현재 구성을 얻기
		storedcfg, err := GetChainConfig(db, stored)
		if err != nil {
			if err == ErrChainConfigNotFound {
				// This case happens if a genesis write was interrupted.
				log.Warn("Found genesis block without chain config")
				err = WriteChainConfig(db, stored, newcfg)
			}
			return newcfg, stored, err
		}
		// Special case: don't change the existing config of a non-mainnet chain if no new
		// config is supplied. These chains would get AllProtocolChanges (and a compat error)
		// if we just continued here.
		//특별한 경우 : 새로운 구성이 기존 구성 기본이 아닌 체인을 변경하지 마십시오.
		//우리가 여기 계속하면,이 체인은 AllProtocolChanges (와의 compat 오류)입니다.
		if genesis == nil && stored != params.MainnetGenesisHash {
			return storedcfg, stored, nil   //그것은 여기에서 종료됩니다 개인 링크 인 경우.
		}
	
		// Check config compatibility and write the config. Compatibility errors
		// are returned to the caller unless we're already at block zero.
		//우리 은행 0에 있지 않으면, 준수, 그렇지 않으면 호환성 오류를 구성을 확인합니다.
		height := GetBlockNumber(db, GetHeadHeaderHash(db))
		if height == missingNumber {
			return newcfg, stored, fmt.Errorf("missing block number for head header hash")
		}
		compatErr := storedcfg.CheckCompatible(newcfg, height)
		//데이터 블록이 기록 된 경우, 기원의 구성을 변경할 수 없다
		if compatErr != nil && height != 0 && compatErr.RewindTo != 0 {
			return newcfg, stored, compatErr
		}
		//네트워크는 여기에서 주요 종료됩니다.
		return newcfg, stored, WriteChainConfig(db, stored, newcfg)
	}


ToBlock,이 방법은 데이터의 기원 메모리 기반 데이터베이스의 사용을 사용하여 다음 블록을 생성하고 리턴.
	
	
	// ToBlock creates the block and state of a genesis specification.
	func (g *Genesis) ToBlock() (*types.Block, *state.StateDB) {
		db, _ := ethdb.NewMemDatabase()
		statedb, _ := state.New(common.Hash{}, state.NewDatabase(db))
		for addr, account := range g.Alloc {
			statedb.AddBalance(addr, account.Balance)
			statedb.SetCode(addr, account.Code)
			statedb.SetNonce(addr, account.Nonce)
			for key, value := range account.Storage {
				statedb.SetState(addr, key, value)
			}
		}
		root := statedb.IntermediateRoot(false)
		head := &types.Header{
			Number:	 new(big.Int).SetUint64(g.Number),
			Nonce:	  types.EncodeNonce(g.Nonce),
			Time:	   new(big.Int).SetUint64(g.Timestamp),
			ParentHash: g.ParentHash,
			Extra:	  g.ExtraData,
			GasLimit:   new(big.Int).SetUint64(g.GasLimit),
			GasUsed:	new(big.Int).SetUint64(g.GasUsed),
			Difficulty: g.Difficulty,
			MixDigest:  g.Mixhash,
			Coinbase:   g.Coinbase,
			Root:	   root,
		}
		if g.GasLimit == 0 {
			head.GasLimit = params.GenesisGasLimit
		}
		if g.Difficulty == nil {
			head.Difficulty = params.GenesisDifficulty
		}
		return types.NewBlock(head, nil, nil, nil), statedb
	}

방법 및 MustCommit 방법 커밋 상기 데이터베이스에 기록 된 소정 블록의 상태에있어서 창 커밋이 블록은 표준 헤더 블록 사슬 것으로 간주된다.
	
	// Commit writes the block and state of a genesis specification to the database.
	// The block is committed as the canonical head block.
	func (g *Genesis) Commit(db ethdb.Database) (*types.Block, error) {
		block, statedb := g.ToBlock()
		if block.Number().Sign() != 0 {
			return nil, fmt.Errorf("can't commit genesis block with number > 0")
		}
		if _, err := statedb.CommitTo(db, false); err != nil {
			return nil, fmt.Errorf("cannot write state: %v", err)
		}
		//어려움의 총 정도 쓰기
		if err := WriteTd(db, block.Hash(), block.NumberU64(), g.Difficulty); err != nil {
			return nil, err
		}
		//쓰기 블록
		if err := WriteBlock(db, block); err != nil {
			return nil, err
		}
		//쓰기 블록 영수증
		if err := WriteBlockReceipts(db, block.Hash(), block.NumberU64(), nil); err != nil {
			return nil, err
		}
		//headerPrefix + NUM (UINT64 큰 엔디안)를 쓰기 + numSuffix -&gt; 해시
		if err := WriteCanonicalHash(db, block.Hash(), block.NumberU64()); err != nil {
			return nil, err
		}
		//&quot;LastBlock&quot;를 쓰기 -&gt; 해시
		if err := WriteHeadBlockHash(db, block.Hash()); err != nil {
			return nil, err
		}
		//&quot;LastHeader&quot;를 쓰기 -&gt; 해시
		if err := WriteHeadHeaderHash(db, block.Hash()); err != nil {
			return nil, err
		}
		config := g.Config
		if config == nil {
			config = params.AllProtocolChanges
		}
		//에테 리움 - 설정 - 해시 쓰기 -&gt; 설정
		return block, WriteChainConfig(db, block.Hash(), config)
	}
	
	// MustCommit writes the genesis block and state to db, panicking on error.
	// The block is committed as the canonical head block.
	func (g *Genesis) MustCommit(db ethdb.Database) *types.Block {
		block, err := g.Commit(db)
		if err != nil {
			panic(err)
		}
		return block
	}

창세기는 기본의 다양한 모드로 돌아갑니다
	
	// DefaultGenesisBlock returns the Ethereum main net genesis block.
	func DefaultGenesisBlock() *Genesis {
		return &Genesis{
			Config:	 params.MainnetChainConfig,
			Nonce:	  66,
			ExtraData:  hexutil.MustDecode("0x11bbe8db4e347b4e8c937c1c8370e4b5ed33adb3db69cbdb7a38e1e50b1b82fa"),
			GasLimit:   5000,
			Difficulty: big.NewInt(17179869184),
			Alloc:	  decodePrealloc(mainnetAllocData),
		}
	}
	
	// DefaultTestnetGenesisBlock returns the Ropsten network genesis block.
	func DefaultTestnetGenesisBlock() *Genesis {
		return &Genesis{
			Config:	 params.TestnetChainConfig,
			Nonce:	  66,
			ExtraData:  hexutil.MustDecode("0x3535353535353535353535353535353535353535353535353535353535353535"),
			GasLimit:   16777216,
			Difficulty: big.NewInt(1048576),
			Alloc:	  decodePrealloc(testnetAllocData),
		}
	}
	
	// DefaultRinkebyGenesisBlock returns the Rinkeby network genesis block.
	func DefaultRinkebyGenesisBlock() *Genesis {
		return &Genesis{
			Config:	 params.RinkebyChainConfig,
			Timestamp:  1492009146,
			ExtraData:  hexutil.MustDecode("0x52657370656374206d7920617574686f7269746168207e452e436172746d616e42eb768f2244c8811c63729a21a3569731535f067ffc57839b00206d1ad20c69a1981b489f772031b279182d99e65703f0076e4812653aab85fca0f00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"),
			GasLimit:   4700000,
			Difficulty: big.NewInt(1),
			Alloc:	  decodePrealloc(rinkebyAllocData),
		}
	}
	
	// DevGenesisBlock returns the 'geth --dev' genesis block.
	func DevGenesisBlock() *Genesis {
		return &Genesis{
			Config:	 params.AllProtocolChanges,
			Nonce:	  42,
			GasLimit:   4712388,
			Difficulty: big.NewInt(131072),
			Alloc:	  decodePrealloc(devAllocData),
		}
	}
