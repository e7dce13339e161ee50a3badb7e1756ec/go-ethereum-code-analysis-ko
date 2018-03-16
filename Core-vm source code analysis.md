## contract.go
계약 상태 데이터베이스 내부의 계약 이더넷 광장을 나타냅니다. 계약은 매개 변수를 호출하는 코드가 포함되어 있습니다.


구조
	
	// ContractRef is a reference to the contract's backing object
	type ContractRef interface {
		Address() common.Address
	}
	
	// AccountRef implements ContractRef.
	//
	// Account references are used during EVM initialisation and
	// it's primary use is to fetch addresses. Removing this object
	// proves difficult because of the cached jump destinations which
	// are fetched from the parent contract (i.e. the caller), which
	// is a ContractRef.
	type AccountRef common.Address
	
	// Address casts AccountRef to a Address
	func (ar AccountRef) Address() common.Address { return (common.Address)(ar) }
	
	// Contract represents an ethereum contract in the state database. It contains
	// the the contract code, calling arguments. Contract implements ContractRef
	type Contract struct {
		// CallerAddress is the result of the caller which initialised this
		// contract. However when the "call method" is delegated this value
		// needs to be initialised to that of the caller's caller.
		//CallerAddress 계약은 사람들을 초기화하는 것입니다. 대리자 경우,이 값은 발신자의 발신자로 설정됩니다.
		CallerAddress common.Address
		caller		ContractRef
		self		  ContractRef
	
		jumpdests destinations //JUMPDEST 분석 결과. JUMPDEST 명령 분석
	
		Code [] 바이트 // 코드
		CodeHash common.Hash  //해시 코드
		CodeAddr *common.Address //코드 주소
		Input	[]byte // 상원에
	
		Gas   uint64	  // 얼마나 많은 가스 계약
		value *big.Int	  
	
		Args []byte  //사용하지 않는 것
	
		DelegateCall bool  
	}

구조
	
	// NewContract returns a new contract environment for the execution of EVM.
	func NewContract(caller ContractRef, object ContractRef, value *big.Int, gas uint64) *Contract {
		c := &Contract{CallerAddress: caller.Address(), caller: caller, self: object, Args: nil}
	
		if parent, ok := caller.(*Contract); ok {
			// Reuse JUMPDEST analysis from parent context if available.
			//호출자가 계약 인 경우 계약 노트는 우리를 호출하는 것입니다. jumpdests는 jumpdests 발신자의 설정
			c.jumpdests = parent.jumpdests
		} else {
			c.jumpdests = make(destinations)
		}
	
		// Gas should be a pointer so it can safely be reduced through the run
		// This pointer will be off the state transition
		c.Gas = gas
		// ensures a value is set
		c.value = value
	
		return c
	}

호출을 위임하는 계약을 AsDelegate과 (체인 통화) 현재 계약을 반환

	// AsDelegate sets the contract to be a delegate call and returns the current
	// contract (for chaining calls)
	func (c *Contract) AsDelegate() *Contract {
		c.DelegateCall = true
		// NOTE: caller must, at all times be a contract. It should never happen
		// that caller is something other than a Contract.
		parent := c.caller.(*Contract)
		c.CallerAddress = parent.CallerAddress
		c.value = parent.value
	
		return c
	}
		
명령은 다음 홉을 가져 오는 데 사용됩니다 GetOp
	
	// GetOp returns the n'th element in the contract's byte array
	func (c *Contract) GetOp(n uint64) OpCode {
		return OpCode(c.GetByte(n))
	}
	
	// GetByte returns the n'th byte in the contract's byte array
	func (c *Contract) GetByte(n uint64) byte {
		if n < uint64(len(c.Code)) {
			return c.Code[n]
		}
	
		return 0
	}

	// Caller returns the caller of the contract.
	//
	// Caller will recursively call caller when the contract is a delegate
	// call, including that of caller's caller.
	func (c *Contract) Caller() common.Address {
		return c.CallerAddress
	}
UseGas는 가스를 사용합니다.
	
	// UseGas attempts the use gas and subtracts it and returns true on success
	func (c *Contract) UseGas(gas uint64) (ok bool) {
		if c.Gas < gas {
			return false
		}
		c.Gas -= gas
		return true
	}
	
	// Address returns the contracts address
	func (c *Contract) Address() common.Address {
		return c.self.Address()
	}
	
	// Value returns the contracts value (sent to it from it's caller)
	func (c *Contract) Value() *big.Int {
		return c.value
	}
SetCode, SetCallCode 설정 코드.

	// SetCode sets the code to the contract
	func (self *Contract) SetCode(hash common.Hash, code []byte) {
		self.Code = code
		self.CodeHash = hash
	}
	
	// SetCallCode sets the code of the contract and address of the backing data
	// object
	func (self *Contract) SetCallCode(addr *common.Address, hash common.Hash, code []byte) {
		self.Code = code
		self.CodeHash = hash
		self.CodeAddr = addr
	}


## evm.go

구조


	// Context provides the EVM with auxiliary information. Once provided
	// it shouldn't be modified.
	//컨텍스트는 EVM에 대한 추가 정보를 제공합니다. 제공 후에는 수정할 수 없습니다.
	type Context struct {
		// CanTransfer returns whether the account contains
		// sufficient ether to transfer the value
		//전송하기에 충분한 에테르가있는 경우 CanTransfer 기능은 계정을 반환
		CanTransfer CanTransferFunc
		// Transfer transfers ether from one account to the other
		//계좌 이체에 한 계정에서 다른 계정으로 이동
		Transfer TransferFunc
		// GetHash returns the hash corresponding to n
		//파라미터 N에 대응하는 해시 값을 반환 GetHash
		GetHash GetHashFunc
	
		// Message information
		//정보의 출처는 보낸 사람의 주소를 제공하는 데 사용
		Origin   common.Address // Provides information for ORIGIN
		//정보 GasPrice를 제공하기 위해
		GasPrice *big.Int	   // Provides information for GASPRICE
	
		// Block information
		Coinbase	common.Address // Provides information for COINBASE
		GasLimit	*big.Int	   // Provides information for GASLIMIT
		BlockNumber *big.Int	   // Provides information for NUMBER
		Time		*big.Int	   // Provides information for TIME
		Difficulty  *big.Int	   // Provides information for DIFFICULTY
	}
	
	// EVM is the Ethereum Virtual Machine base object and provides
	// the necessary tools to run a contract on the given state with
	// the provided context. It should be noted that any error
	// generated through any of the calls should be considered a
	// revert-state-and-consume-all-gas operation, no checks on
	// specific errors should ever be performed. The interpreter makes
	// sure that any errors generated are to be considered faulty code.
	//EVM은 광장 이더넷 가상 머신 기반 객체이며, 주어진 상태 계약을 실행하는 컨텍스트를 제공하는 데 사용하는 데 필요한 도구를 제공합니다.
	//오류의 호출이 상태를 롤백하고 모든 가스 작업을 소비하는 수정으로 간주되어야한다는 것을 주목해야한다
	//당신은 특정 오류에 대한 검사를 수행하지 않아야합니다. 통역은 생성 된 오류는 잘못된 코드로 간주되어 있는지 확인합니다.
	// The EVM should never be reused and is not thread safe.
	type EVM struct {
		// Context provides auxiliary blockchain related information
		Context
		// StateDB gives access to the underlying state
		StateDB StateDB
		// Depth is the current call stack
		//현재 호출 스택
		depth int
	
		// chainConfig contains information about the current chain
		//이 정보는 현재 블록 사슬을 포함
		chainConfig *params.ChainConfig
		// chain rules contains the chain rules for the current epoch
		chainRules params.Rules
		// virtual machine configuration options used to initialise the
		// evm.
		vmConfig Config
		// global (to this context) ethereum virtual machine
		// used throughout the execution of the tx.
		interpreter *Interpreter
		// abort is used to abort the EVM calling operations
		// NOTE: must be set atomically
		abort int32
	}

생성자
	
	// NewEVM retutrns a new EVM . The returned EVM is not thread safe and should
	// only ever be used *once*.
	func NewEVM(ctx Context, statedb StateDB, chainConfig *params.ChainConfig, vmConfig Config) *EVM {
		evm := &EVM{
			Context:	 ctx,
			StateDB:	 statedb,
			vmConfig:	vmConfig,
			chainConfig: chainConfig,
			chainRules:  chainConfig.Rules(ctx.BlockNumber),
		}
	
		evm.interpreter = NewInterpreter(evm, vmConfig)
		return evm
	}
	
	// Cancel cancels any running EVM operation. This may be called concurrently and
	// it's safe to be called multiple times.
	func (evm *EVM) Cancel() {
		atomic.StoreInt32(&evm.abort, 1)
	}


계약을 만들 만들기 새로운 계약을 만듭니다.

	
	// Create creates a new contract using code as deployment code.
	func (evm *EVM) Create(caller ContractRef, code []byte, gas uint64, value *big.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {
	
		// Depth check execution. Fail if we're trying to execute above the
		// limit.
		if evm.depth > int(params.CallCreateDepth) {
			return nil, common.Address{}, gas, ErrDepth
		}
		if !evm.CanTransfer(evm.StateDB, caller.Address(), value) {
			return nil, common.Address{}, gas, ErrInsufficientBalance
		}
		// Ensure there's no existing contract already at the designated address
		//어떤 주소를 특정 계약이 존재하지 않음을 보장하기 위하여
		nonce := evm.StateDB.GetNonce(caller.Address())
		evm.StateDB.SetNonce(caller.Address(), nonce+1)
	
		contractAddr = crypto.CreateAddress(caller.Address(), nonce)
		contractHash := evm.StateDB.GetCodeHash(contractAddr)
		if evm.StateDB.GetNonce(contractAddr) != 0 || (contractHash != (common.Hash{}) && contractHash != emptyCodeHash) { //이미있는 경우
			return nil, common.Address{}, 0, ErrContractAddressCollision
		}
		// Create a new account on the state
		snapshot := evm.StateDB.Snapshot()  //롤백 할 수있는 스냅 샷을 생성 StateDB
		evm.StateDB.CreateAccount(contractAddr) //계정 만들기
		if evm.ChainConfig().IsEIP158(evm.BlockNumber) {
			evm.StateDB.SetNonce(contractAddr, 1) //설정 넌스
		}
		evm.Transfer(evm.StateDB, caller.Address(), contractAddr, value)  //이전
	
		// initialise a new contract and set the code that is to be used by the
		// E The contract is a scoped evmironment for this execution context
		// only.
		contract := NewContract(caller, AccountRef(contractAddr), value, gas)
		contract.SetCallCode(&contractAddr, crypto.Keccak256Hash(code), code)
	
		if evm.vmConfig.NoRecursion && evm.depth > 0 {
			return nil, contractAddr, gas, nil
		}
		ret, err = run(evm, snapshot, contract, nil) //강제 계약 초기화 코드
		// check whether the max code size has been exceeded
		//한계를 초과하지 않는 생성 된 초기화 코드의 길이를 확인
		maxCodeSizeExceeded := evm.ChainConfig().IsEIP158(evm.BlockNumber) && len(ret) > params.MaxCodeSize
		// if the contract creation ran successfully and no errors were returned
		// calculate the gas required to store the code. If the code could not
		// be stored due to not enough gas set an error and let it be handled
		// by the error checking condition below.
		//계약이 성공적으로 생성되고 오류가 반환되지 않으면, 계산 코드의 가스를 저장하는 데 필요합니다. 충분하지 않은 경우 코드는 가스 오류 메모리 세트에 의해 야기 될 수 없으며, 다음과 같은 조건에 의해 검사 오류를 처리 할 수 ​​있기 때문이다.
		if err == nil && !maxCodeSizeExceeded {
			createDataGas := uint64(len(ret)) * params.CreateDataGas
			if contract.UseGas(createDataGas) {
				evm.StateDB.SetCode(contractAddr, ret)
			} else {
				err = ErrCodeStoreOutOfGas
			}
		}
	
		// When an error was returned by the EVM or when setting the creation code
		// above we revert to the snapshot and consume any gas remaining. Additionally
		// when we're in homestead this also counts for code storage gas errors.
		//우리가 롤백 변화를 오류로 돌아 오면,
		if maxCodeSizeExceeded || (err != nil && (evm.ChainConfig().IsHomestead(evm.BlockNumber) || err != ErrCodeStoreOutOfGas)) {
			evm.StateDB.RevertToSnapshot(snapshot)
			if err != errExecutionReverted {
				contract.UseGas(contract.Gas)
			}
		}
		// Assign err if contract code size exceeds the max while the err is still empty.
		if maxCodeSizeExceeded && err == nil {
			err = errMaxCodeSizeExceeded
		}
		return ret, contractAddr, contract.Gas, err
	}


통화 방법, 우리는 전송 또는 호출 명령의 내부가 여기에 실행됩니다 반면,이 계약에 계약을 호출 코드를 실행할지 여부를 지정합니다.

	
	// Call executes the contract associated with the addr with the given input as
	// parameters. It also handles any necessary value transfer required and takes
	// the necessary steps to create accounts and reverses the state in case of an
	// execution error or failed value transfer.
	
	//요지와 관련된 계약으로 주어진 입력 매개 변수를 실행 호출합니다.
	//또한 운영하고 계정을 생성하기 위해 필요한 조치를 취해야 필요한 전송을 처리합니다
	//그리고 당신은 어떤 경우에 잘못 롤백 무슨 짓을했는지.

	func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
		if evm.vmConfig.NoRecursion && evm.depth > 0 {
			return nil, gas, nil
		}
	
		// Fail if we're trying to execute above the call depth limit
		//1024 깊이까지 전화
		if evm.depth > int(params.CallCreateDepth) {
			return nil, gas, ErrDepth
		}
		// Fail if we're trying to transfer more than the available balance
		//우리가 계정에 충분한 돈이 있는지 확인합니다.
		if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
			return nil, gas, ErrInsufficientBalance
		}
	
		var (
			to	   = AccountRef(addr)
			snapshot = evm.StateDB.Snapshot()
		)
		if !evm.StateDB.Exist(addr) { //지정된 주소가 존재 확인
			//주소가 존재하지 않는 경우, 네이티브 이동 계약, 계약의 기본 이동 있는지 여부를 확인합니다
			//내부 contracts.go 파일
			precompiles := PrecompiledContractsHomestead
			if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
				precompiles = PrecompiledContractsByzantium
			}
			if precompiles[addr] == nil && evm.ChainConfig().IsEIP158(evm.BlockNumber) && value.Sign() == 0 {
				//그것은 계약 주소를 지정하지 않고, 값이 0은 정상으로 돌아되며,이 호출은 가스를 소비하지 않는 경우
				return nil, gas, nil
			}
			//요지는 로컬 상태를 만들기위한 책임이있다
			evm.StateDB.CreateAccount(addr)
		}
		//전송 실행
		evm.Transfer(evm.StateDB, caller.Address(), to.Address(), value)
	
		// initialise a new contract and set the code that is to be used by the
		// E The contract is a scoped environment for this execution context
		// only.
		contract := NewContract(caller, to, value, gas)
		contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))
	
		ret, err = run(evm, snapshot, contract, input)
		// When an error was returned by the EVM or when setting the creation code
		// above we revert to the snapshot and consume any gas remaining. Additionally
		// when we're in homestead this also counts for code storage gas errors.
		if err != nil {
			evm.StateDB.RevertToSnapshot(snapshot)
			if err != errExecutionReverted { 
				//오류가 되돌리기 명령에 의해 트리거되는 경우, ICO는 일반적인 제한이나 금융 제약의 번호를 설정하기 때문에
				//우리가 구입하면 펌핑 많은 돈을 선도, 이러한 제약을 트리거 할 가능성이있다. 이 시간
				//그것은 낮은 GasPrice 및 GasLimit을 설정할 수 없습니다. 속도를 빠르게하기 때문에.
				//그래서 모든 남아있는 가스를 사용하지 않는,하지만 코드 실행 가스를 사용합니다
				//또는 GasLimit * GasPrice 돈을 펌핑 될 것입니다, 그들은 많은 수 있습니다.
				contract.UseGas(contract.Gas)
			}
		}
		return ret, contract.Gas, err
	}


CallCode, DelegateCall 및 StaticCall은,이 세 가지 기능은 외부에서 호출 할 수 없습니다 나머지 세 가지 기능은 단지 연산 코드에 의해 트리거 될 수 있습니다.


CallCode

	// CallCode differs from Call in the sense that it executes the given address'
	// code with the caller as context.
	//다른 전화 CallCode는 코드 지정된 주소에 발신자의 컨텍스트를 사용하여 수행되는 것입니다.
	
	func (evm *EVM) CallCode(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
		if evm.vmConfig.NoRecursion && evm.depth > 0 {
			return nil, gas, nil
		}
	
		// Fail if we're trying to execute above the call depth limit
		if evm.depth > int(params.CallCreateDepth) {
			return nil, gas, ErrDepth
		}
		// Fail if we're trying to transfer more than the available balance
		if !evm.CanTransfer(evm.StateDB, caller.Address(), value) {
			return nil, gas, ErrInsufficientBalance
		}
	
		var (
			snapshot = evm.StateDB.Snapshot()
			to   = AccountRef은 (caller.Address는 ()) //이 발신자의 주소를 해결하기 위해 수정 될 수있는 대부분의 다른 장소와 행동없이 전송입니다
		)
		// initialise a new contract and set the code that is to be used by the
		// E The contract is a scoped evmironment for this execution context
		// only.
		contract := NewContract(caller, to, value, gas)
		contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))
	
		ret, err = run(evm, snapshot, contract, input)
		if err != nil {
			evm.StateDB.RevertToSnapshot(snapshot)
			if err != errExecutionReverted {
				contract.UseGas(contract.Gas)
			}
		}
		return ret, contract.Gas, err
	}

DelegateCall

	// DelegateCall differs from CallCode in the sense that it executes the given address'
	// code with the caller as context and the caller is set to the caller of the caller.
	//DelegateCall CallCode 다른 장소와 발신자가 발신자의 호출로 제공되는
	func (evm *EVM) DelegateCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
		if evm.vmConfig.NoRecursion && evm.depth > 0 {
			return nil, gas, nil
		}
		// Fail if we're trying to execute above the call depth limit
		if evm.depth > int(params.CallCreateDepth) {
			return nil, gas, ErrDepth
		}
	
		var (
			snapshot = evm.StateDB.Snapshot()
			to	   = AccountRef(caller.Address()) 
		)
	
		// Initialise a new contract and make initialise the delegate values
		//()로 식별 AsDelete
		contract := NewContract(caller, to, nil, gas).AsDelegate() 
		contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))
	
		ret, err = run(evm, snapshot, contract, input)
		if err != nil {
			evm.StateDB.RevertToSnapshot(snapshot)
			if err != errExecutionReverted {
				contract.UseGas(contract.Gas)
			}
		}
		return ret, contract.Gas, err
	}
	
	// StaticCall executes the contract associated with the addr with the given input
	// as parameters while disallowing any modifications to the state during the call.
	// Opcodes that attempt to perform such modifications will result in exceptions
	// instead of performing the modifications.
	//상태를 수정하는 작업을 수행 할 수 없습니다 StaticCall,
	
	func (evm *EVM) StaticCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
		if evm.vmConfig.NoRecursion && evm.depth > 0 {
			return nil, gas, nil
		}
		// Fail if we're trying to execute above the call depth limit
		if evm.depth > int(params.CallCreateDepth) {
			return nil, gas, ErrDepth
		}
		// Make sure the readonly is only set if we aren't in readonly yet
		// this makes also sure that the readonly flag isn't removed for
		// child calls.
		if !evm.interpreter.readOnly {
			evm.interpreter.readOnly = true
			defer func() { evm.interpreter.readOnly = false }()
		}
	
		var (
			to	   = AccountRef(addr)
			snapshot = evm.StateDB.Snapshot()
		)
		// Initialise a new contract and set the code that is to be used by the
		// EVM. The contract is a scoped environment for this execution context
		// only.
		contract := NewContract(caller, to, new(big.Int), gas)
		contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))
	
		// When an error was returned by the EVM or when setting the creation code
		// above we revert to the snapshot and consume any gas remaining. Additionally
		// when we're in Homestead this also counts for code storage gas errors.
		ret, err = run(evm, snapshot, contract, input)
		if err != nil {
			evm.StateDB.RevertToSnapshot(snapshot)
			if err != errExecutionReverted {
				contract.UseGas(contract.Gas)
			}
		}
		return ret, contract.Gas, err
	}
