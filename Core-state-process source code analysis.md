## StateTransition
상태 전이 모델



	/*
	The State Transitioning Model
	상태 전이 모델
	A state transition is a change made when a transaction is applied to the current world state
	상태 전이는 현재 세계 상태와 트랜잭션을 수행하고, 현재의 세계 상태를 변경하는 것을 의미한다
	The state transitioning model does all all the necessary work to work out a valid new state root.
	상태 전이는 새로운 유효한 상태 루트를 생산하는 데 필요한 모든 일을했다
	1) 논스 처리 난스 처리
	2) Pre pay gas 선불 가스
	수신자가 비어있는 경우 수신자가 \ 0 * 32 인 경우 3) 새로운 상태 개체를 만든 다음 새로운 상태 객체를 생성
	4) 값 전송 전송
	== If contract creation ==
	  4A) 데이터 입력 트랜잭션 데이터가 실행하려고 실행하려고
	  4B) 유효한 경우, 새로운 상태 객체의 연산 코드의 결과로서, 새로운 상태 객체 유효한 경우에 대한 코드로 결과를 사용
	== end ==
	5) 스크립트 실행 섹션 실행 스크립트 섹션
	6) 새로운 상태 루트를 도출 새로운 상태 루트를 유도
	*/
	type StateTransition struct {
		gp	 * GasPool은 블록 내부에 사용 가스를 추적하는 데 사용 //
		msg		Message		// Message Call
		gas		uint64
		gasPrice   *big.Int	// 가스 가격
		initialGas *big.Int	가스 // 시작
		value	  *big.Int	// 전송 값
		data	   []byte	// 입력 데이터
		state	  vm.StateDB	// StateDB
		evm		*vm.EVM	// 가상 머신
	}
	
	// Message represents a message sent to a contract.
	type Message interface {
		From() common.Address
		//FromFrontier() (common.Address, error)
		To() *common.Address	// 
	
		GasPrice() *big.Int  //GasPrice의 메시지
		Gas() *big.Int	// GasLimit의 메시지
		Value() *big.Int
	
		Nonce() uint64
		CheckNonce() bool
		Data() []byte
	}


구조
	
	// NewStateTransition initialises and returns a new state transition object.
	func NewStateTransition(evm *vm.EVM, msg Message, gp *GasPool) *StateTransition {
		return &StateTransition{
			gp:		 gp,
			evm:		evm,
			msg:		msg,
			gasPrice:   msg.GasPrice(),
			initialGas: new(big.Int),
			value:	  msg.Value(),
			data:	   msg.Data(),
			state:	  evm.StateDB,
		}
	}


실행 메시지
	
	// ApplyMessage computes the new state by applying the given message
	// against the old state within the environment.
	//ApplyMessage는 주어진 상태 및 메시지의 응용 프로그램에 의해 새로운 상태를 생성
	// ApplyMessage returns the bytes returned by any EVM execution (if it took place),
	// the gas used (which includes gas refunds) and an error if it failed. An error always
	// indicates a core error meaning that the message would always fail for that particular
	// state and would never be accepted within a block.
	//ApplyMessage 반환은 (발생한 경우), 모든 EVM에 의해 수행 반환 바이트
	//실패하면 가스의 사용 (환불 포함 가스), 오류가 반환됩니다. 오류가 항상 핵심 오류를 나타냅니다,
	//뉴스는이 특정 상태에 대해 항상 실패하고 한 블록에 허용되지 않습니다 것을 의미합니다.
	func ApplyMessage(evm *vm.EVM, msg Message, gp *GasPool) ([]byte, *big.Int, bool, error) {
		st := NewStateTransition(evm, msg, gp)
	
		ret, _, gasUsed, failed, err := st.TransitionDb()
		return ret, gasUsed, failed, err
	}

TransitionDb
	
	// TransitionDb will transition the state by applying the current message and returning the result
	// including the required gas for the operation as well as the used gas. It returns an error if it
	// failed. An error indicates a consensus issue.
	// TransitionDb 
	func (st *StateTransition) TransitionDb() (ret []byte, requiredGas, usedGas *big.Int, failed bool, err error) {
		if err = st.preCheck(); err != nil {
			return
		}
		msg := st.msg
		sender := st.from() // err checked in preCheck
	
		homestead := st.evm.ChainConfig().IsHomestead(st.evm.BlockNumber)
		contractCreation := msg.To() == nil //msg.To는 nil을 만들 수있는 계약으로 간주하는 경우
	
		// Pay intrinsic gas
		// TODO convert to uint64
		//가스 G0의 시작을 계산
		intrinsicGas := IntrinsicGas(st.data, contractCreation, homestead)
		if intrinsicGas.BitLen() > 64 {
			return nil, nil, nil, false, vm.ErrOutOfGas
		}
		if err = st.useGas(intrinsicGas.Uint64()); err != nil {
			return nil, nil, nil, false, err
		}
	
		var (
			evm = st.evm
			// vm errors do not effect consensus and are therefor
			// not assigned to err, except for insufficient balance
			// error.
			vmerr error
		)
		if contractCreation { //계약이 생성되면, 방법 EVM 만들기 전화
			ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
		} else {
			// Increment the nonce for the next transaction
			//경우 메서드가 호출됩니다. 이어서 논스 송신기의 제 1 세트.
			st.state.SetNonce(sender.Address(), st.state.GetNonce(sender.Address())+1)
			ret, st.gas, vmerr = evm.Call(sender, st.to().Address(), st.data, st.gas, st.value)
		}
		if vmerr != nil {
			log.Debug("VM returned with error", "err", vmerr)
			// The only possible consensus-error would be if there wasn't
			// sufficient balance to make the transfer happen. The first
			// balance transfer may never fail.
			if vmerr == vm.ErrInsufficientBalance {
				return nil, nil, nil, false, vmerr
			}
		}
		requiredGas = new(big.Int).Set(st.gasUsed()) //가스 수를 사용하는 계산
	
		st.refundGas()  //환불은 위의 가스 증가 st.gas를 계산됩니다. 그래서 광부 리베이트 후입니다 얻을 수
		st.state.AddBalance(st.evm.Coinbase, new(big.Int).Mul(st.gasUsed(), st.gasPrice)) //광부는 수익을 증가시킵니다.
		//차이 requiredGas의 GasUsed 및 환불, 최대의 환급이 없습니다.
		//ApplyMessage 호출 직접 폐기 requiredGas, 세금 반환의 설명은 위를 참조하십시오.
		return ret, requiredGas, st.gasUsed(), vmerr != nil, err
	}

G0은 옐로우 북 (Yellow Book)의 상세한 설명에서의 계산
노란색 책, 그리고 contractCreation &amp;&amp; 농가 {igas.SetUint64 (params.TxGasContractCreation)의 경우 일부에 대한 액세스에 있기 때문에 Gtxcreate + Gtransaction = TxGasContractCreation

	func IntrinsicGas(data []byte, contractCreation, homestead bool) *big.Int {
		igas := new(big.Int)
		if contractCreation && homestead {
			igas.SetUint64(params.TxGasContractCreation)
		} else {
			igas.SetUint64(params.TxGas)
		}
		if len(data) > 0 {
			var nz int64
			for _, byt := range data {
				if byt != 0 {
					nz++
				}
			}
			m := big.NewInt(nz)
			m.Mul(m, new(big.Int).SetUint64(params.TxDataNonZeroGas))
			igas.Add(igas, m)
			m.SetInt64(int64(len(data)) - nz)
			m.Mul(m, new(big.Int).SetUint64(params.TxDataZeroGas))
			igas.Add(igas, m)
		}
		return igas
	}


실행 전에 확인

	func (st *StateTransition) preCheck() error {
		msg := st.msg
		sender := st.from()
	
		// Make sure this transaction's nonce is correct
		if msg.CheckNonce() {
			nonce := st.state.GetNonce(sender.Address())
			//현재 지역의 요구와 같은 상태의 비표 난스 MSG 이상이 동기화되지 않습니다.
			if nonce < msg.Nonce() {
				return ErrNonceTooHigh
			} else if nonce > msg.Nonce() {
				return ErrNonceTooLow
			}
		}
		return st.buyGas()
	}

buyGas, 당신의 GasLimit * GasPrice 돈을 공제하는 우선 사전 공제 가스를 달성했다. 그런 다음 계산 된 환불 일부 완성 된 상태에 따라.

	func (st *StateTransition) buyGas() error {
		mgas := st.msg.Gas()
		if mgas.BitLen() > 64 {
			return vm.ErrOutOfGas
		}
	
		mgval := new(big.Int).Mul(mgas, st.gasPrice)
	
		var (
			state  = st.state
			sender = st.from()
		)
		if state.GetBalance(sender.Address()).Cmp(mgval) < 0 {
			return errInsufficientBalanceForGas
		}
		if err := st.gp.SubGas(mgas); err != nil { //, 블록 gaspool 마이너스 내부에서 블록은 GasLimit의 사용에 의해 제한 가스 전체 블록이기 때문이다.
			return err
		}
		st.gas += mgas.Uint64()
	
		st.initialGas.Set(mgas)
		state.SubBalance(sender.Address(), mgval)
		//GasLimit * GasPrice 내부 계정에서 차감
		return nil
	}
		

이러한 빈 저장 계정으로 명령의 부담을 줄일 수있는 몇 가지 블록 체인을 실행하는 모든 사람들을위한 세금 환급, 세금 인센티브는. 자살 또는 실행 계정을 취소 명령.
	
	func (st *StateTransition) refundGas() {
		// Return eth for remaining gas to the sender account,
		// exchanged at the original rate.
		sender := st.from() // err already checked
		remaining := new(big.Int).Mul(new(big.Int).SetUint64(st.gas), st.gasPrice)
		//먼저, 사용자는 다시 가스로 남아 있습니다.
		st.state.AddBalance(sender.Address(), remaining)
	
		// Apply refund counter, capped to half of the used gas.
		//그리고 총 세액은 사용자의 가스 1/2의 총 사용을 초과하지 않습니다.
		uhalf := remaining.Div(st.gasUsed(), common.Big2)
		refund := math.BigMin(uhalf, st.state.GetRefund())
		st.gas += refund.Uint64()
		//리베이트 금액은 사용자의 계정에 추가됩니다.
		st.state.AddBalance(sender.Address(), refund.Mul(refund, st.gasPrice))
	
		// Also return remaining gas to the block gas counter so it is
		// available for the next transaction.
		//양성자는 가스 공간 포인트 gaspool뿐만 아니라 다음 트랜잭션에 다시 돈을 환불합니다.
		st.gp.AddGas(new(big.Int).SetUint64(st.gas))
	}


## StateProcessor
StateTransition은 하나 개의 트랜잭션을 처리하는 데 사용됩니다. 그래서 StateProcessor는 블록 레벨의 트랜잭션을 처리하는 데 사용됩니다.

건축 및 건설
	
	// StateProcessor is a basic Processor, which takes care of transitioning
	// state from one point to another.
	//
	// StateProcessor implements Processor.
	type StateProcessor struct {
		config *params.ChainConfig // Chain configuration options
		bc	 *BlockChain		 // Canonical block chain
		engine consensus.Engine	// Consensus engine used for block rewards
	}
	
	// NewStateProcessor initialises a new StateProcessor.
	func NewStateProcessor(config *params.ChainConfig, bc *BlockChain, engine consensus.Engine) *StateProcessor {
		return &StateProcessor{
			config: config,
			bc:	 bc,
			engine: engine,
		}
	}


프로세스는이 방법 blockchain 호출됩니다.
	
	// Process processes the state changes according to the Ethereum rules by running
	// the transaction messages using the statedb and applying any rewards to both
	// the processor (coinbase) and any included uncles.
	//프로세스 상태뿐만 아니라 보상 컵을 변경하거나 다른 삼촌은 이더넷 광장 거래 정보를 실행하는 규칙에 따라 statedb 노드.
	// Process returns the receipts and logs accumulated during the process and
	// returns the amount of gas that was used in the process. If any of the
	// transactions failed to execute due to insufficient gas it will return an error.
	//누적 수익률 영수증을 처리하고 구현 프로세스를 기록, 그 과정에서 사용되는 가스를 반환합니다. 어떤 트랜잭션이 실패로 인한 가스의 부족, 오류가 리턴됩니다 인해합니다.
	func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (types.Receipts, []*types.Log, *big.Int, error) {
		var (
			receipts	 types.Receipts
			totalUsedGas = big.NewInt(0)
			header	   = block.Header()
			allLogs	  []*types.Log
			gp		   = new(GasPool).AddGas(block.GasLimit())
		)
		// Mutate the the block and state according to any hard-fork specs
		//하드 분기 프로세스 DAO 이벤트
		if p.config.DAOForkSupport && p.config.DAOForkBlock != nil && p.config.DAOForkBlock.Cmp(block.Number()) == 0 {
			misc.ApplyDAOHardFork(statedb)
		}
		// Iterate over and process the individual transactions
		for i, tx := range block.Transactions() {
			statedb.Prepare(tx.Hash(), block.Hash(), i)
			receipt, _, err := ApplyTransaction(p.config, p.bc, nil, gp, statedb, header, tx, totalUsedGas, cfg)
			if err != nil {
				return nil, nil, nil, err
			}
			receipts = append(receipts, receipt)
			allLogs = append(allLogs, receipt.Logs...)
		}
		// Finalize the block, applying any consensus engine specific extras (e.g. block rewards)
		p.engine.Finalize(p.bc, header, statedb, block.Transactions(), block.Uncles(), receipts)
		//총 수신 확인은 가스 사용 및 전무를 기록
		return receipts, allLogs, totalUsedGas, nil
	}

ApplyTransaction
	
	// ApplyTransaction attempts to apply a transaction to the given state database
	// and uses the input parameters for its environment. It returns the receipt
	// for the transaction, gas used and an error if the transaction failed,
	// indicating the block was invalid.
	ApplyTransaction는 주어진 상태 데이터베이스에 트랜잭션을 적용 시도하고 환경에 대한 입력 매개 변수를 사용합니다.
	//트랜잭션이 블록이 무효임을 나타내는, 실패하면 그것은, 트랜잭션, 가스 및 잘못된 사용의 영수증을 반환합니다.
	
	func ApplyTransaction(config *params.ChainConfig, bc *BlockChain, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *big.Int, cfg vm.Config) (*types.Receipt, *big.Int, error) {
		//메시지에 거래를 변환
		//여기에 메시지를 실제로 보낸 사람이 보낸되어 있는지 확인하는 방법입니다. TODO
		msg, err := tx.AsMessage(types.MakeSigner(config, header.Number))
		if err != nil {
			return nil, nil, err
		}
		// Create a new context to be used in the EVM environment
		//각 트랜잭션은 새로운 가상 머신 환경을 만듭니다.
		context := NewEVMContext(msg, header, bc, author)
		// Create a new environment which holds all relevant information
		// about the transaction and calling mechanisms.
		vmenv := vm.NewEVM(context, statedb, config, cfg)
		// Apply the transaction to the current state (included in the env)
		_, gas, failed, err := ApplyMessage(vmenv, msg, gp)
		if err != nil {
			return nil, nil, err
		}
	
		// Update the state with pending changes
		//중간 상태 결정
		var root []byte
		if config.IsByzantium(header.Number) {
			statedb.Finalise(true)
		} else {
			root = statedb.IntermediateRoot(config.IsEIP158(header.Number)).Bytes()
		}
		usedGas.Add(usedGas, gas)
	
		// Create a new receipt for the transaction, storing the intermediate root and gas used by the tx
		// based on the eip phase, we're passing wether the root touch-delete accounts.
		//중간 상태 루트를 저장하는 데 사용 영수증을 작성, 가스 거래의 사용
		receipt := types.NewReceipt(root, failed, usedGas)
		receipt.TxHash = tx.Hash()
		receipt.GasUsed = new(big.Int).Set(gas)
		// if the transaction created a contract, store the creation address in the receipt.
		//당신이 거래 계약서를 작성하는 경우. 그래서 우리는 내부의 영수증에 저장된 주소를 만들 수 있습니다.
		if msg.To() == nil {
			receipt.ContractAddress = crypto.CreateAddress(vmenv.Context.Origin, tx.Nonce())
		}
	
		// Set the receipt logs and create a bloom for filtering
		receipt.Logs = statedb.GetLogs(tx.Hash())
		receipt.Bloom = types.CreateBloom(types.Receipts{receipt})
		//모든 로그를 취득하고, 블룸 필터 로그를 생성합니다.
		return receipt, gas, err
	}
