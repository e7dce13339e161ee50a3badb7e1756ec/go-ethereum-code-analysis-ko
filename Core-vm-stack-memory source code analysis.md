VM은 stack.go 내부에 사용되는 가상 머신의 스택으로 스택 객체. 메모리를 사용하는 가상 머신 내부의 메모리 오브젝트를 나타낸다.

## stack
또한, 비교적 간단 저장소 스택 1024 big.Int의 고정 길이의 배열을 사용하는 것이다.

구조

	// stack is an object for basic stack operations. Items popped to the stack are
	// expected to be changed and modified. stack does not take care of adding newly
	// initialised objects.
	type Stack struct {
		data []*big.Int
	}
	
	func newstack() *Stack {
		return &Stack{data: make([]*big.Int, 0, 1024)}
	}

푸시 작업

	func (st *Stack) push(d *big.Int) { //대부분의 끝에 추가
		// NOTE push limit (1024) is checked in baseCheck
		//stackItem := new(big.Int).Set(d)
		//st.data = append(st.data, stackItem)
		st.data = append(st.data, d)
	}
	func (st *Stack) pushN(ds ...*big.Int) {
		st.data = append(st.data, ds...)
	}

팝업 작업


	func (st *Stack) pop() (ret *big.Int) { //대부분의 끝에서 제거.
		ret = st.data[len(st.data)-1]
		st.data = st.data[:len(st.data)-1]
		return
	}
조작 요소의 과거 값이 조작?
	
	FUNC (세인트 * 스택) 스왑 (N 개의 INT) {스택의 상단과 스택에서 N 요소의 값으로부터 스위칭 소자.
		st.data[st.len()-n], st.data[st.len()-1] = st.data[st.len()-1], st.data[st.len()-n]
	}

스택의 상부에 지정된 위치의 값으로서 복사 작업 DUP

	func (st *Stack) dup(pool *intPool, n int) {
		st.push(pool.get().Set(st.data[st.len()-n]))
	}

. 동작 들여다 상단 요소를 들여다

	func (st *Stack) peek() *big.Int {
		return st.data[st.len()-1]
	}
지정된 위치의 요소에서 뒤로 슬쩍

	// Back returns the n'th item in stack
	func (st *Stack) Back(n int) *big.Int {
		return st.data[st.len()-n-1]
	}

적층 요소 보장 번호가 동일한 N보다 클 필요하다.

	func (st *Stack) require(n int) error {
		if st.len() < n {
			return fmt.Errorf("stack underflow (%d <=> %d)", len(st.data), n)
		}
		return nil
	}

## intpool
아주 간단합니다. 256 big.int 풀의 크기는, bit.Int의 분포를 가속
	
	var checkVal = big.NewInt(-42)
	
	const poolLimit = 256
	
	// intPool is a pool of big integers that
	// can be reused for all big.Int operations.
	type intPool struct {
		pool *Stack
	}
	
	func newIntPool() *intPool {
		return &intPool{pool: newstack()}
	}
	
	func (p *intPool) get() *big.Int {
		if p.pool.len() > 0 {
			return p.pool.pop()
		}
		return new(big.Int)
	}
	func (p *intPool) put(is ...*big.Int) {
		if len(p.pool.data) > poolLimit {
			return
		}
	
		for _, i := range is {
			// verifyPool is a build flag. Pool verification makes sure the integrity
			// of the integer pool by comparing values to a default value.
			if verifyPool {
				i.Set(checkVal)
			}
	
			p.pool.push(i)
		}
	}

## memory

구조, 메모리 저장 바이트이다]. lastGasCost의 [기록이있다.
	
	type Memory struct {
		store	   []byte
		lastGasCost uint64
	}
	
	func NewMemory() *Memory {
		return &Memory{}
	}

크기 조정을 사용하여 공간을 할당하는 최초의 필요성을 사용하여

	// Resize resizes the memory to size
	func (m *Memory) Resize(size uint64) {
		if uint64(m.Len()) < size {
			m.store = append(m.store, make([]byte, size-uint64(m.Len()))...)
		}
	}

그런 다음 설정 값을 설정하는 데 사용

	// Set sets offset + size to value
	func (m *Memory) Set(offset, size uint64, value []byte) {
		// length of store may never be less than offset + size.
		// The store should be resized PRIOR to setting the memory
		if size > uint64(len(m.store)) {
			panic("INVALID memory: store empty")
		}
	
		// It's possible the offset is greater than 0 and size equals 0. This is because
		// the calcMemSize (common.go) could potentially return 0 when size is zero (NO-OP)
		if size > 0 {
			copy(m.store[offset:offset+size], value)
		}
	}
, 사본을 얻을 것입니다에 값을 취득 포인터를 얻는 것입니다.
	
	// Get returns offset + size as a new slice
	func (self *Memory) Get(offset, size int64) (cpy []byte) {
		if size == 0 {
			return nil
		}
	
		if len(self.store) > int(offset) {
			cpy = make([]byte, size)
			copy(cpy, self.store[offset:offset+size])
	
			return
		}
	
		return
	}
	
	// GetPtr returns the offset + size
	func (self *Memory) GetPtr(offset, size int64) []byte {
		if size == 0 {
			return nil
		}
	
		if len(self.store) > int(offset) {
			return self.store[offset : offset+size]
		}
	
		return nil
	}


## stack_table.go 내부 일부 추가 도움말 기능

	
	func makeStackFunc(pop, push int) stackValidationFunc {
		return func(stack *Stack) error {
			if err := stack.require(pop); err != nil {
				return err
			}
	
			if stack.len()+push-pop > int(params.StackLimit) {
				return fmt.Errorf("stack limit reached %d (%d)", stack.len(), params.StackLimit)
			}
			return nil
		}
	}
	
	func makeDupStackFunc(n int) stackValidationFunc {
		return makeStackFunc(n, n+1)
	}
	
	func makeSwapStackFunc(n int) stackValidationFunc {
		return makeStackFunc(n, n)
	}


