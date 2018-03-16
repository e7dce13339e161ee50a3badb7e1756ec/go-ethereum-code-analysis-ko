jumptable. 데이터 구조의 [256] 동작이다. 각 인덱스 처리 로직 기능에 사용하는 가스 소비량 스택 인증 방법, 메모리 크기에 대응하는 명령들을 저장하는 데 사용되는 명령어 연산에 대응한다.
## jumptable

명령에 필요한 데이터 구조 연산 격납 기능.

	type operation struct {
		//OP는 기능을 수행하도록 동작 함수
		execute executionFunc
		//gasCost는 가스 기능입니다 및 실행 가스 소비 기능에 필요한 가스를 반환
		gasCost gasFunc
		//validateStack가 동작 검증 함수 스택 크기 스택 (크기)을 유효화
		validateStack stackValidationFunc
		//memorySize는 필요한 메모리 크기의 작업에 필요한 메모리 크기를 반환
		memorySize memorySizeFunc
	
		halts   bool //동작은 상기 실행을 중지할지 여부를 나타내는 shoult의 동작은 상기 실행을 중지할지 여부를 나타내는
		jumps   bool //프로그램 카운터는 증가 여부를 나타내는 프로그램 카운터를 증가시키지할지 여부를 나타내는
		writes  bool //이 상태를 변경하는 조작이 변형 동작 상태인지 여부를 판정 여부를 판정한다
		valid   bool //검색된 동작은 유효한 공지인지 표시하는 동작을 검색할지 여부를 나타내는 유효 공지
		reverts bool //동작 상태는 (암시 적으로 중지) 여부 회복 동작 상태 (암시 적 중지) 복귀 판정 여부를 판정한다
		returns bool //opertions가 복귀 조작 내용 데이터 여부를 결정하기 위해 반환 데이터 콘텐츠를 설정할지 여부를 판정한다
	}

명령어 세트는 다음과 같은 세 가지 명령, 이더넷 광장의 세 가지 다른 버전의 설정 정의합니다

var (
	frontierInstructionSet  = NewFrontierInstructionSet()
	homesteadInstructionSet = NewHomesteadInstructionSet()
	byzantiumInstructionSet = NewByzantiumInstructionSet()
)
첫 번째 호출 NewByzantiumInstructionSet 비잔틴 버전은 그들 만의 독특한 지침 .STATICCALL, RETURNDATASIZE, RETURNDATACOPY, 되돌리기를 추가, 이전 명령의 버전을 만들 NewHomesteadInstructionSet
	
	// NewByzantiumInstructionSet returns the frontier, homestead and
	// byzantium instructions.
	func NewByzantiumInstructionSet() [256]operation {
		// instructions that can be executed during the homestead phase.
		instructionSet := NewHomesteadInstructionSet()
		instructionSet[STATICCALL] = operation{
			execute:	   opStaticCall,
			gasCost:	   gasStaticCall,
			validateStack: makeStackFunc(6, 1),
			memorySize:	memoryStaticCall,
			valid:		 true,
			returns:	   true,
		}
		instructionSet[RETURNDATASIZE] = operation{
			execute:	   opReturnDataSize,
			gasCost:	   constGasFunc(GasQuickStep),
			validateStack: makeStackFunc(0, 1),
			valid:		 true,
		}
		instructionSet[RETURNDATACOPY] = operation{
			execute:	   opReturnDataCopy,
			gasCost:	   gasReturnDataCopy,
			validateStack: makeStackFunc(3, 0),
			memorySize:	memoryReturnDataCopy,
			valid:		 true,
		}
		instructionSet[REVERT] = operation{
			execute:	   opRevert,
			gasCost:	   gasRevert,
			validateStack: makeStackFunc(2, 0),
			memorySize:	memoryRevert,
			valid:		 true,
			reverts:	   true,
			returns:	   true,
		}
		return instructionSet
	}

NewHomesteadInstructionSet

	// NewHomesteadInstructionSet returns the frontier and homestead
	// instructions that can be executed during the homestead phase.
	func NewHomesteadInstructionSet() [256]operation {
		instructionSet := NewFrontierInstructionSet()
		instructionSet[DELEGATECALL] = operation{
			execute:	   opDelegateCall,
			gasCost:	   gasDelegateCall,
			validateStack: makeStackFunc(6, 1),
			memorySize:	memoryDelegateCall,
			valid:		 true,
			returns:	   true,
		}
		return instructionSet
	}



## instruction.go 
많은 지침 때문에 기능의 조합이 복잡 할 수 있지만 단지. 예를 몇 가지 이름을, 목록까지가 아니라 단일 명령어, 그것은 매우 직관적이다.

	func opPc(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
		stack.push(evm.interpreter.intPool.get().SetUint64(*pc))
		return nil, nil
	}
	
	func opMsize(pc *uint64, evm *EVM, contract *Contract, memory *Memory, stack *Stack) ([]byte, error) {
		stack.push(evm.interpreter.intPool.get().SetInt64(int64(memory.Len())))
		return nil, nil
	}



## gas_table.go
gas_table 다양한 지침은 가스 소비 기능을 반환
이 기능은 오류 값이 기본적으로 만 errGasUintOverflow 정수 오버 플로우입니다 반환합니다.

	func gasBalance(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
		return gt.Balance, nil
	}
	
	func gasExtCodeSize(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
		return gt.ExtcodeSize, nil
	}
	
	func gasSLoad(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
		return gt.SLoad, nil
	}
	
	func gasExp(gt params.GasTable, evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
		expByteLen := uint64((stack.data[stack.len()-2].BitLen() + 7) / 8)
	
		var (
			gas	  = expByteLen * gt.ExpByte // no overflow check required. Max is 256 * ExpByte gas
			overflow bool
		)
		if gas, overflow = math.SafeAdd(gas, GasSlowStep); overflow {
			return 0, errGasUintOverflow
		}
		return gas, nil
	}

## interpreter.go 인터프리터

데이터 구조
	
	// Config are the configuration options for the Interpreter
	type Config struct {
		// Debug enabled debugging Interpreter options
		Debug bool
		// EnableJit enabled the JIT VM
		EnableJit bool
		// ForceJit forces the JIT VM
		ForceJit bool
		// Tracer is the op code logger
		Tracer Tracer
		// NoRecursion disabled Interpreter call, callcode,
		// delegate call and create.
		NoRecursion bool
		// Disable gas metering
		DisableGasMetering bool
		// Enable recording of SHA3/keccak preimages
		EnablePreimageRecording bool
		// JumpTable contains the EVM instruction table. This
		// may be left uninitialised and will be set to the default
		// table.
		JumpTable [256]operation
	}
	
	// Interpreter is used to run Ethereum based contracts and will utilise the
	// passed evmironment to query external sources for state information.
	// The Interpreter will run the byte code VM or JIT VM based on the passed
	// configuration.
	type Interpreter struct {
		evm	  *EVM
		cfg	  Config
		gasTable params.GasTable   //이 작업 가스 가격의 번호를 식별
		intPool  *intPool
	
		readOnly   bool   // Whether to throw on stateful modifications
		returnData []byte //함수의 연속 재사용 마지막 반환 값의 마지막 호출의 반환 데이터
	}

생성자
	
	// NewInterpreter returns a new instance of the Interpreter.
	func NewInterpreter(evm *EVM, cfg Config) *Interpreter {
		// We use the STOP instruction whether to see
		// the jump table was initialised. If it was not
		// we'll set the default jump table.
		//초기화되지 않은 경우 테스트 JumpTable와 STOP 명령이 이미 초기화 된,, 디폴트 값으로 설정
		if !cfg.JumpTable[STOP].valid { 
			switch {
			case evm.ChainConfig().IsByzantium(evm.BlockNumber):
				cfg.JumpTable = byzantiumInstructionSet
			case evm.ChainConfig().IsHomestead(evm.BlockNumber):
				cfg.JumpTable = homesteadInstructionSet
			default:
				cfg.JumpTable = frontierInstructionSet
			}
		}
	
		return &Interpreter{
			evm:	  evm,
			cfg:	  cfg,
			gasTable: evm.ChainConfig().GasTable(evm.BlockNumber),
			intPool:  newIntPool(),
		}
	}


두 방법 enforceRestrictions있어서 총과 실행 방법의 인터프리터.


	
	func (in *Interpreter) enforceRestrictions(op OpCode, operation operation, stack *Stack) error {
		if in.evm.chainRules.IsByzantium {
			if in.readOnly {
				// If the interpreter is operating in readonly mode, make sure no
				// state-modifying operation is performed. The 3rd stack item
				// for a call operation is the value. Transferring value from one
				// account to the others means the state is modified and should also
				// return with an error.
				if operation.writes || (op == CALL && stack.Back(2).BitLen() > 0) {
					return errWriteProtection
				}
			}
		}
		return nil
	}
	
	// Run loops and evaluates the contract's code with the given input data and returns
	// the return byte-slice and an error if one occurred.
	//오류가 발생하면 계약 파라미터로 지정된 루프 실행 코드, 리턴 리턴 바이트 세그먼트와, 에러가 반환된다.
	// It's important to note that any errors returned by the interpreter should be
	// considered a revert-and-consume-all-gas operation. No error specific checks
	// should be handled to reduce complexity and errors further down the in.
	//어떤 오류가 모든 가스를 소비하는 인터프리터에 의해 반환되는 것이 중요합니다. 복잡성을 줄이기 위해, 특별한 오류 처리 과정이 없습니다.
	func (in *Interpreter) Run(snapshot int, contract *Contract, input []byte) (ret []byte, err error) {
		// Increment the call depth which is restricted to 1024
		in.evm.depth++
		defer func() { in.evm.depth-- }()
	
		// Reset the previous call's return data. It's unimportant to preserve the old buffer
		// as every returning call will return new data anyway.
		in.returnData = nil
	
		// Don't bother with the execution if there's no code.
		if len(contract.Code) == 0 {
			return nil, nil
		}
	
		codehash := contract.CodeHash // codehash is used when doing jump dest caching
		if codehash == (common.Hash{}) {
			codehash = crypto.Keccak256Hash(contract.Code)
		}
	
		var (
			op	OpCode		// current opcode
			mem   = NewMemory() // bound memory
			stack = newstack()  // local stack
			// For optimisation reason we're using uint64 as the program counter.
			// It's theoretically possible to go above 2^64. The YP defines the PC
			// to be uint256. Practically much less so feasible.
			pc   = uint64(0) // program counter
			cost uint64
			// copies used by tracer
			stackCopy = newstack() // stackCopy needed for Tracer since stack is mutated by 63/64 gas rule 
			pcCopy uint64 // needed for the deferred Tracer
			gasCopy uint64 // for Tracer to log gas remaining before execution
			logged bool // deferred Tracer should ignore already logged steps
		)
		contract.Input = input
	
		defer func() {
			if err != nil && !logged && in.cfg.Debug {
				in.cfg.Tracer.CaptureState(in.evm, pcCopy, op, gasCopy, cost, mem, stackCopy, contract, in.evm.depth, err)
			}
		}()
	
		// The Interpreter main run loop (contextual). This loop runs until either an
		// explicit STOP, RETURN or SELFDESTRUCT is executed, an error occurred during
		// the execution of one of the operations or until the done flag is set by the
		// parent context.
		//이 STOP, RETURN을 만날 때까지 인터프리터의 메인 루프는 자체 폐기 명령이 실행되거나 오류가 발생하거나 완료 플래그 부모 컨텍스트를 설정합니다.
		for atomic.LoadInt32(&in.evm.abort) == 0 {
			// Get the memory location of pc
			//실행되는 다음 명령어입니다
			op = contract.GetOp(pc)
	
			if in.cfg.Debug {
				logged = false
				pcCopy = uint64(pc)
				gasCopy = uint64(contract.Gas)
				stackCopy = newstack()
				for _, val := range stack.data {
					stackCopy.push(val)
				}
			}
	
			// get the operation from the jump table matching the opcode
			//JumpTable하여 해당 작업을 얻으려면
			operation := in.cfg.JumpTable[op]
			//검사는 실행할 수없는 명령은 아래의 읽기 전용 모드를 쓴다
			//모드가 읽기 전용으로 설정되어 StaticCall 경우
			if err := in.enforceRestrictions(op, operation, stack); err != nil {
				return nil, err
			}
	
			// if the op is invalid abort the process and return an error
			if !operation.valid { //명령이 불법 확인
				return nil, fmt.Errorf("invalid opcode 0x%x", int(op))
			}
	
			// validate the stack and make sure there enough stack items available
			// to perform the operation
			//충분한 스택 공간이 있는지 확인합니다. 밀고 터지는 포함
			if err := operation.validateStack(stack); err != nil {
				return nil, err
			}
	
			var memorySize uint64
			// calculate the new memory size and expand the memory to fit
			// the operation
			if operation.memorySize != nil { //메모리 사용 요금을 계산할 때
				memSize, overflow := bigUint64(operation.memorySize(stack))
				if overflow {
					return nil, errGasUintOverflow
				}
				// memory is expanded in words of 32 bytes. Gas
				// is also calculated in words.
				if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
					return nil, errGasUintOverflow
				}
			}
	
			if !in.cfg.DisableGasMetering { //이 매개 변수는 로컬 아날로그 실행될 때, 소비 또는 가스 거래를 실행 확인 결과를 리턴받을 수없는 경우에 유용
				// consume the gas and return an error if not enough gas is available.
				// cost is explicitly set so that the capture state defer method cas get the proper cost
				//하지 않을 경우 비용 계산 및 가스의 사용은, 그것은 OutOfGas 오류를 반환합니다.
				cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
				if err != nil || !contract.UseGas(cost) {
					return nil, ErrOutOfGas
				}
			}
			if memorySize > 0 { //확장 된 메모리 영역
				mem.Resize(memorySize)
			}
	
			if in.cfg.Debug {
				in.cfg.Tracer.CaptureState(in.evm, pc, op, gasCopy, cost, mem, stackCopy, contract, in.evm.depth, err)
				logged = true
			}
	
			// execute the operation
			//명령을 실행
			res, err := operation.execute(&pc, in.evm, contract, mem, stack)
			// verifyPool is a build flag. Pool verification makes sure the integrity
			// of the integer pool by comparing values to a default value.
			if verifyPool {
				verifyIntegerPool(in.intPool)
			}
			// if the operation clears the return data (e.g. it has returning data)
			// set the last return to the result of the operation.
			if operation.returns { //반환 값이있는 경우, 반환 값을 설정합니다. 그 마지막 만의 복귀 효과가 있습니다.
				in.returnData = res
			}
	
			switch {
			case err != nil:
				return nil, err
			case operation.reverts:
				return res, errExecutionReverted
			case operation.halts:
				return res, nil
			case !operation.jumps:
				pc++
			}
		}
		return nil, nil
	}
