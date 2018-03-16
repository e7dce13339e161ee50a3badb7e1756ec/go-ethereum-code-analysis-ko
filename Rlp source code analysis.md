RLP는 재귀 길이 접두사 속기이다. 제곱 법 이더넷의 시퀀스이며, 이더넷 스퀘어 모든 개체는 RLP 방법의 순서에 바이트 배열을 사용한다. 여기에 내가 RLP 방법을 이해 공식화 한 다음 코드를 통해 실제 구현을 분석 할 수있는 옐로우 북 (Yellow Book)으로 시작하고 싶습니다.

## 옐로우 북의 공식적인 정의
우리는 T. 세트를 정의 T는 다음 식에 의해 정의된다

![image](picture/rlp_1.png)

O 모든 세트 바이트 위의 그림은 다음 B (예 분기 노드 구조 또는 트리 노드 비 리프 노드로) 모든 가능한 바이트 배열, L은 트리 구조의 단일 노드 이상을 나타내고를 나타냄 , T는 바이트 배열하고 트리 구조의 모든 조합을 나타낸다.

우리는 서브 함수는 두 개의 구조 (L 또는 B) 상술 한 모두 처리되고, 하위 기능을 정의하는 두 개의 RLP 기능을 사용한다.

![image](picture/rlp_2.png)

어레이 B 바이트의 모든 유형. 우리는 다음과 같은 처리 규칙을 정의합니다.

- 바이트 어레이는 하나의 바이트 및 128 미만의 바이트 크기를 포함하는 경우, 데이터가 처리되지 않고, 처리 결과는 원래의 데이터 인
- 배열의 길이보다 56 바이트 인 경우, 상기 프로세싱 결과는 앞 원판 데이터 플러스 (128+ 바이트의 데이터 길이) 프리픽스와 동일하다.
-하지 위의 두 가지 경우 경우, 처리 결과는 다음 데이터 길이를 더한 빅 엔디안 표현 전에 원래 원시 데이터 같고, 앞에 추가 (길이 + 원본 데이터 (183)의 빅 엔디안 표현)

언어의 다음 정형화 사용하는 표현

![image](picture/rlp_3.png)

** 일부 수학 기호 설명 **

- || X || x는 추구의 길이를 나타냅니다
- ... (A) (B, C) (D, E) = (A, B, C, D, E)의 동작 CONCAT 즉, 문자열의 가산 동작을 나타낸다. &quot;여보세요&quot;+ &quot;세계&quot;= &quot;안녕하세요!&quot;
- BE (x)의 기능을 실제로 최고의 빅 엔디안 모드 0을 제거한다. 0x1234 성형 등의 4 바이트는 프로세스 00,001,234 BE 함수가 실제로 여분의 12 34 00 제거한 후에 시작 빅 엔디안 모드를 반환 나타내는.
- ^ 기호는 표현과 의미.
- &quot;세&quot;와 동일한 의미를 대신 등호의 형태로

다른 모든 종류의 (나무)의 경우, 우리는 다음과 같은 처리 규칙을 정의

먼저, 내부의 트리 구조의 각 요소에 대한 RLP 프로세싱을 사용하고이 결과 CONCAT 연결.

- 커넥터의 바이트 길이 (56)보다 작은 경우, 우리가 연결된 결과 전면에 플러스 (+ 길이 커넥터 (192) 이후), 최종 결과의 조성물.
- 커넥터의 바이트 길이 (56)와 동일한보다 큰 경우, 우리는 연결 길이 후 다음 앞에 추가 (길이 + 247 빅 엔디안 모드를 패턴의 길이에 연결되는 제 플러스 대단 결과 앞에 접속되고 )

다음은 더 명확하게 규정 된 식으로 표시됩니다 발견 정형화 된 언어의 사용.

![image](picture/rlp_4.png)
그 RLP 데이터 구조가 반복적으로 처리 될 수 있도록 상기, S (x)는 차례로이 메소드를 호출 RLP 얻는 과정에서, 재귀 적 정의를 알 수있다.


RLP 스칼라 데이터를 처리하는 경우, RLP은 양의 정수를 처리하는데 사용될 수. 단지 빅 엔디안 모드 처리를 처리 할 수있는 RLP 정수 후. 이는 x가 정수이면, 처음 사용이 (00 초에 제거) 가장 간단한 빅 엔디안 모드로 X를 변환하는 (x)의 함수, BE (X)의 한 다음 그 결과를 인코딩 바이트 배열로 취급되는 것을 의미한다.

수식도에 의해 표현되는 경우.

![image](picture/rlp_5.png)

때 RLP 데이터를 분석. 그냥 정수 데이터를 분석해야하는 경우 특별한 상황이 선으로 처리를 필요로, 이번에는 주요 00이 시간을 만났다.


** ** 요약

RLP 두 가지 유형의 데이터 바이트의 한 배열에서의 클래스 유사한 데이터 구조의 조합과 같은 모든 데이터. 나는 이것이 기본 데이터 구조의 모든 유형이 포함되어 있음을 이해합니다. 구조체보다 더와 예를 들어. 그것은 목록에있는 필드의 다른 종류를 많이 볼 수있다


### RLP 소스 분석
RLP 소스는 주로 세 개의 파일로 나누어, 많은 아니다

	decode.go		디코더는, 데이터가 RLP 데이터 구조를 이동 디코딩
	decode_tail_test.go	테스트 코드 디코더
	decode_test.go			解码器测试代码
	doc.go			문서 코드
	encode.go		인코더 GO의 데이터 구조는 바이트의 배열로 연재
	encode_test.go		인코더 테스트
	encode_example_test.go
	raw.go			RLP 데이터를 디코딩되지 않은
	raw_test.go
	typecache.go		버퍼 유형 버퍼 형 기록 형 -&gt; (코더 | 디코더) 함량.


#### 유형 typecache.go에 따라 대응하는 인코더 및 디코더를 찾는 방법
C ++, 자바 또는 다른 오버 지원 언어, 예를 들면, 또한 일반적인 디스패치 함수를 통해 달성 될 수 분포의 종류에 대한 다양한 유형의 방법으로 동일한 기능 명을 달성하기 위해 재정의 될 수있다.
	
	string encode(int);
	string encode(long);
	string encode(struct test*)

그러나 GO 언어 자체에 과부하가 지원하지 않으며, 어떤 제네릭은, 그래서 당신은 당신의 자신의 현실의 기능을 할당 할 필요가 없습니다. typecache.go 신속하게 그 기능의 인코더 및 디코더 기능을 찾기 위해 자신의 유형을 통해이 목표를 달성하는 것입니다.

우리는 먼저 핵심 데이터 구조를 보면

	var (
		typeCacheMutex sync.RWMutex			  멀티 스레드 보호에 사용하는 경우 잠금, 읽기 - 쓰기 //지도를 typeCache
		typeCache  = 확인 (맵 [typekey * 소속 카테고리) // 코어 데이터 구조 타입은 저장 -&gt; 코덱 기능
	)
	type typeinfo struct { //저장된 인코더 및 디코더 기능
		decoder
		writer
	}
	type typekey struct {
		reflect.Type
		// the key must include the struct tags because they
		// might generate a different decoder.
		tags
	}

typeCache 맵을 코어 데이터 구조 인 것을 알 수 있고, 키 맵의 종류, 값 대응 인코더와 디코더이다.

여기에 사용자가 인코더 및 디코더 기능에 액세스하는 방법입니다

	
	func cachedTypeInfo(typ reflect.Type, tags tags) (*typeinfo, error) {
		typeCacheMutex.RLock()	// 보호하기 위해 읽기 잠금을 추가,
		info := typeCache[typekey{typ, tags}]
		typeCacheMutex.RUnlock()
		if info != nil { //성공하면, 다음 반환 정보를 얻을 수 있습니다
			return info, nil
		}
		// not in the cache, need to generate info for this type.
		typeCacheMutex.Lock()  //그렇지 않으면, 생성 기능을 잠금 작성하고 반환은 다중 스레드 환경에서, 여러 스레드가 동시에이 곳으로 전화를 할 수 있습니다 점에 유의 cachedTypeInfo1를 호출, 그래서 당신은 cachedTypeInfo1를 입력 할 때 방법은 판단 할 필요가 먼저 다른 스레드 된 것을 여부 성공적으로 만들기.
		defer typeCacheMutex.Unlock()
		return cachedTypeInfo1(typ, tags)
	}
	
	func cachedTypeInfo1(typ reflect.Type, tags tags) (*typeinfo, error) {
		key := typekey{typ, tags}
		info := typeCache[key]
		if info != nil {
			//다른 스레드가 성공적으로 생성 한 후, 우리는 정보와 수익에 직접 액세스 할 수
			return info, nil
		}
		// put a dummmy value into the cache before generating.
		// if the generator tries to lookup itself, it will get
		// the dummy value and won't call itself recursively.
		//첫 번째 장소 위치의이 유형을 채우기 위해 가치를 창출하기 위해, 당신은 죽은 정의 데이터 형식의 재귀 루프 형성의 일부가 발생하지 않습니다
		typeCache[key] = new(typeinfo)
		info, err := genTypeInfo(typ, tags)
		if err != nil {
			// remove the dummy value if the generator fails
			delete(typeCache, key)
			return nil, err
		}
		*typeCache[key] = *info
		return typeCache[key], err
	}

genTypeInfo 코덱 기능의 종류에 대응하여 생성된다.

	func genTypeInfo(typ reflect.Type, tags tags) (info *typeinfo, err error) {
		info = new(typeinfo)
		if info.decoder, err = makeDecoder(typ, tags); err != nil {
			return nil, err
		}
		if info.writer, err = makeWriter(typ, tags); err != nil {
			return nil, err
		}
		return info, nil
	}

처리 로직 및 처리 로직 makeWriter 대략 비슷한 MakeDecoder, 여기 난 단지, makeWriter 처리 로직을 게시

	// makeWriter creates a writer function for the given type.
	func makeWriter(typ reflect.Type, ts tags) (writer, error) {
		kind := typ.Kind()
		switch {
		case typ == rawValueType:
			return writeRawValue, nil
		case typ.Implements(encoderInterface):
			return writeEncoder, nil
		case kind != reflect.Ptr && reflect.PtrTo(typ).Implements(encoderInterface):
			return writeEncoderNoPtr, nil
		case kind == reflect.Interface:
			return writeInterface, nil
		case typ.AssignableTo(reflect.PtrTo(bigInt)):
			return writeBigIntPtr, nil
		case typ.AssignableTo(bigInt):
			return writeBigIntNoPtr, nil
		case isUint(kind):
			return writeUint, nil
		case kind == reflect.Bool:
			return writeBool, nil
		case kind == reflect.String:
			return writeString, nil
		case kind == reflect.Slice && isByte(typ.Elem()):
			return writeBytes, nil
		case kind == reflect.Array && isByte(typ.Elem()):
			return writeByteArray, nil
		case kind == reflect.Slice || kind == reflect.Array:
			return makeSliceWriter(typ, ts)
		case kind == reflect.Struct:
			return makeStructWriter(typ)
		case kind == reflect.Ptr:
			return makePtrWriter(typ)
		default:
			return nil, fmt.Errorf("rlp: type %v is not RLP-serializable", typ)
		}
	}

이 스위치 케이스는, 처리의 종류에 따라 서로 다른 기능들을 할당 할 수있는 것을 알 수있다. 프로세싱 로직은 매우 간단하다. 간단한 유형의 노란색 설명 위의에 따라 책을 처리하기 위해 간단합니다. 그러나, 과정의 유형의 구조는 매우 흥미했지만, 위의 상세 처리 로직이 부분은 오렌지 북에서 찾을 수 없습니다.

	type field struct {
		index int
		info  *typeinfo
	}
	func makeStructWriter(typ reflect.Type) (writer, error) {
		fields, err := structFields(typ)
		if err != nil {
			return nil, err
		}
		writer := func(val reflect.Value, w *encbuf) error {
			lh := w.list()
			for _, f := range fields {
				//F는 소속 ​​카테고리가 포인터 f.info, 실제로 현장 인코더 메소드를 호출하고, 필드 구조이다.
				if err := f.info.writer(val.Field(f.index), w); err != nil {
					return err
				}
			}
			w.listEnd(lh)
			return nil
		}
		return writer, nil
	}

이 함수는 부호화 구조 structFields 모든 필드 법을 수득 인코더를 정의하고, 다음 방법을 반환 인코더 메소드를 호출하는 각각의 모든 필드를 통해.
	
	func structFields(typ reflect.Type) (fields []field, err error) {
		for i := 0; i < typ.NumField(); i++ {
			if f := typ.Field(i); f.PkgPath == "" { // exported
				tags, err := parseStructTag(typ, i)
				if err != nil {
					return nil, err
				}
				if tags.ignored {
					continue
				}
				info, err := cachedTypeInfo1(f.Type, tags)
				if err != nil {
					return nil, err
				}
				fields = append(fields, field{i, info})
			}
		}
		return fields, nil
	}

structFields 기능은 모든 필드를 순환하고 각 필드에 대해 cachedTypeInfo1를 호출합니다. 우리는이 과정에 재귀 호출입니다 볼 수 있습니다. 위의 코드, f.PkgPath == &quot;는&quot;이 판결은 모든 수출 필드, 필드는 명령의 대문자로 시작 내보낼 소위 필드이므로주의 할 필요가있다.


#### 인코더 encode.go
빈 문자열은 최초의 우주 각각 목록 및 0x80으로 0xc0과 정의. 값 0 성형에 대응하는 값을 0x80이며, 참고. 위의 정의에 옐로우 북이 볼 수 없습니다. 그런 다음 또 다른 유형에 대한 인터페이스 유형이 EncodeRLP를 구현 정의

	var (
		// Common encoded values.
		// These are useful when implementing EncodeRLP.
		EmptyString = []byte{0x80}
		EmptyList   = []byte{0xC0}
	)
	
	// Encoder is implemented by types that require custom
	// encoding rules or want to encode private fields.
	type Encoder interface {
		// EncodeRLP should write the RLP encoding of its receiver to w.
		// If the implementation is a pointer method, it may also be
		// called for nil pointers.
		//
		// Implementations should generate valid RLP. The data written is
		// not verified at the moment, but a future version might. It is
		// recommended to write only a single value but writing multiple
		// values or no value at all is also permitted.
		EncodeRLP(io.Writer) error
	}

그런 다음, 방법의 대부분이 직접 호출하는 가장 중요한 방법 중 하나 EncodeRLP이 방법의 인코딩 방법을 정의합니다. 이 방법은 먼저 encbuf 객체를 얻었다. 그런 다음 객체의 인코딩 메서드를 호출합니다. 인코딩 방법은, 처음에는 반사 형 인코더를 취득하여, 피사체의 반사 형 획득, 인코더는 라이터 메소드를 호출한다. 이것은 단지 서로 연결 typecache 위에 온다.
	
	func Encode(w io.Writer, val interface{}) error {
		if outer, ok := w.(*encbuf); ok {
			// Encode was called by some type's EncodeRLP.
			// Avoid copying by writing to the outer encbuf directly.
			return outer.encode(val)
		}
		eb := encbufPool.Get().(*encbuf)
		defer encbufPool.Put(eb)
		eb.reset()
		if err := eb.encode(val); err != nil {
			return err
		}
		return eb.toWriter(w)
	}
	func (w *encbuf) encode(val interface{}) error {
		rval := reflect.ValueOf(val)
		ti, err := cachedTypeInfo(rval.Type(), tags{})
		if err != nil {
			return err
		}
		return ti.writer(rval, w)
	}

##### encbuf 소개
encbuf 인코딩 버퍼 (내 생각)의 약어입니다. encbuf는 인코딩 방법에 표시하고, 많은 작가 방법. 이름이 암시 하듯이,이 역할은 인코딩 프로세스의 버퍼로서 작용한다. 여기에 정의 encbuf에서 첫 모습입니다.

	type encbuf struct {
		str	 []byte	  // string data, contains everything except list headers
		lheads  []*listhead // all list headers
		lhsize  int		 // sum of sizes of all encoded list headers
		sizebuf []byte	  // 9-byte auxiliary buffer for uint encoding
	}
	
	type listhead struct {
		offset int // index of this header in string data
		size   int // total size of encoded data (including list headers)
	}

당신은 코멘트에서, STR 필드 목록의 머리뿐만 아니라, 내용이 모두 포함되어 볼 수 있습니다. lheads 필드에 레코드 목록의 머리. 보조 버퍼는 인코딩 UINT를 처리하기위한 크기 9 바이트이며 sizebuf lhsize 필드 길이 lheads를 기록한다. listhead 두 필드 오프셋리스트 STR 필드, 사이즈 필드리스트의 선두를 포함하는 부호화 된 데이터의 전체 길이를 기록하는 장소에 기록 된 데이터 필드로 구성된다. 다음 표를 참조하십시오.

![image](picture/rlp_6.png)	

광고에 대한 STR 필드에 직접 충전 같은 문자열, 플라스틱, BOOL 데이터 유형 등의 통상의 형태, 내용. 그러나 구조를 처리하는 유형은 특별한 방법이 필요합니다. 상기의 방법은 이전 makeStructWriter 볼 수 있습니다.
	
	func makeStructWriter(typ reflect.Type) (writer, error) {
			fields, err := structFields(typ)
			...
			writer := func(val reflect.Value, w *encbuf) error {
				lh := w.list()
				for _, f := range fields {
					if err := f.info.writer(val.Field(f.index), w); err != nil {
						return err
					}
				}
				w.listEnd(lh)
			}
		}

이 특별한 데이터 구조의 가공 방법의 실시, 상기 코드는, 첫 번째 전화가 w.list ()에있어서, 처리가 완료된 후 다음 listEnd (LH) 메소드를 호출하는 것을 알 수있다. 이 방법을 사용하는 이유는 우리가 노란색 책을 불러 구조물의 길이 (구조에 따라 때문에 치료의 머리를 결정하기의 필요성, 치료 구조의 시작 부분에 있습니다 얼마나 오래 치료 후 구조의 길이를 알 수 없다는 것입니다 다음) 몸을 취급, 그래서 우리는 사전 처리에 STR 좋은 위치를 기록하고하면 데이터가 STR 시간 구조 치료 길이 후 알 수를 증가 봐 처리 한 후, 각 필드를 처리하기 시작한다.
	
	func (w *encbuf) list() *listhead {
		lh := &listhead{offset: len(w.str), size: w.lhsize}
		w.lheads = append(w.lheads, lh)
		return lh
	}
	
	func (w *encbuf) listEnd(lh *listhead) {
		lh.size = w.size() - lh.offset - lh.size//lh.size w.size 목록) 큐 헤드 (점유되기 시작 시간을 기록 STR 플러스 lhsize의 길이를 반환
		if lh.size < 56 {
			w.lhsize += 1 // length encoded into kind tag
		} else {
			w.lhsize += 1 + intsize(uint64(lh.size))
		}
	}
	func (w *encbuf) size() int {
		return len(w.str) + w.lhsize
	}

그럼 최종 처리 로직 encbuf을 볼 수 listhead는 전체 RLP 데이터로 조립 처리 될

	func (w *encbuf) toBytes() []byte {
		out := make([]byte, w.size())
		strpos := 0
		pos := 0
		for _, head := range w.lheads {
			// write string data before header
			n := copy(out[pos:], w.str[strpos:head.offset])
			pos += n
			strpos += n
			// write the header
			enc := head.encode(out[pos:])
			pos += len(enc)
		}
		// copy string data after the last list header
		copy(out[pos:], w.str[strpos:])
		return out
	}


##### 소개 작가
나머지 과정은 실제로 비교적 간단하다. 노란 책 바늘 내부에 채워진 다른 데이터 encbuf 각각 투입된다.

	func writeBool(val reflect.Value, w *encbuf) error {
		if val.Bool() {
			w.str = append(w.str, 0x01)
		} else {
			w.str = append(w.str, 0x80)
		}
		return nil
	}
	func writeString(val reflect.Value, w *encbuf) error {
		s := val.String()
		if len(s) == 1 && s[0] <= 0x7f {
			// fits single byte, no string header
			w.str = append(w.str, s[0])
		} else {
			w.encodeStringHeader(len(s))
			w.str = append(w.str, s...)
		}
		return nil
	}


#### 디코더 decode.go
일반 과정과 인코더 디코더는 거의 인코더의 일반 프로세스, 우리는 디코더의 일반적인 과정을 알 수 이해합니다.

	func (s *Stream) Decode(val interface{}) error {
		if val == nil {
			return errDecodeIntoNil
		}
		rval := reflect.ValueOf(val)
		rtyp := rval.Type()
		if rtyp.Kind() != reflect.Ptr {
			return errNoPointer
		}
		if rval.IsNil() {
			return errDecodeIntoNil
		}
		info, err := cachedTypeInfo(rtyp.Elem(), tags{})
		if err != nil {
			return err
		}
		err = info.decoder(s, rval.Elem())
		if decErr, ok := err.(*decodeError); ok && len(decErr.ctx) > 0 {
			// add decode target type to error so context has more meaning
			decErr.ctx = append(decErr.ctx, fmt.Sprint("(", rtyp.Elem(), ")"))
		}
		return err
	}

	func makeDecoder(typ reflect.Type, tags tags) (dec decoder, err error) {
		kind := typ.Kind()
		switch {
		case typ == rawValueType:
			return decodeRawValue, nil
		case typ.Implements(decoderInterface):
			return decodeDecoder, nil
		case kind != reflect.Ptr && reflect.PtrTo(typ).Implements(decoderInterface):
			return decodeDecoderNoPtr, nil
		case typ.AssignableTo(reflect.PtrTo(bigInt)):
			return decodeBigInt, nil
		case typ.AssignableTo(bigInt):
			return decodeBigIntNoPtr, nil
		case isUint(kind):
			return decodeUint, nil
		case kind == reflect.Bool:
			return decodeBool, nil
		case kind == reflect.String:
			return decodeString, nil
		case kind == reflect.Slice || kind == reflect.Array:
			return makeListDecoder(typ, tags)
		case kind == reflect.Struct:
			return makeStructDecoder(typ)
		case kind == reflect.Ptr:
			if tags.nilOK {
				return makeOptionalPtrDecoder(typ)
			}
			return makePtrDecoder(typ)
		case kind == reflect.Interface:
			return decodeInterface, nil
		default:
			return nil, fmt.Errorf("rlp: type %v is not RLP-serializable", typ)
		}
	}

또한 복호화 처리의 종류에 의해 복호 처리의 구체적인 구성을 확인. 거의 프로세스를 인코딩, 무엇보다도 모든 필드 structFields 복호해야 얻고, 각 필드를 복호한다. 거의 목록 ()와 ListEnd ()하지만, 프로세스 및 여기에 인코딩 과정의 인코딩 처리 작업으로는 다음 장에서 상세하게 설명하지 동일합니다.

	func makeStructDecoder(typ reflect.Type) (decoder, error) {
		fields, err := structFields(typ)
		if err != nil {
			return nil, err
		}
		dec := func(s *Stream, val reflect.Value) (err error) {
			if _, err := s.List(); err != nil {
				return wrapStreamError(err, typ)
			}
			for _, f := range fields {
				err := f.info.decoder(s, val.Field(f.index))
				if err == EOL {
					return &decodeError{msg: "too few elements", typ: typ}
				} else if err != nil {
					return addErrorContext(err, "."+typ.Field(f.index).Name)
				}
			}
			return wrapStreamError(s.ListEnd(), typ)
		}
		return dec, nil
	}

서로 다른 길이의 문자열이 다른 방법을 인코딩하기 때문에, 우리는 우리가 현재 s.Kind (해결 형) 방법을 얻을 필요가 문자열의 접두사의 다른 유형을 얻을 수 있습니다, 다음과 같은 문자열 디코딩 프로세스를 참조하고, 길이는 바이트 입력하면, 그 값은 직접 바이트 반환 등을 입력 한 다음 문자열 지정된 길이의 값을 읽어 경우 반환합니다. 이 종류의 () 메서드를 사용하는 방법이다.

	func (s *Stream) Bytes() ([]byte, error) {
		kind, size, err := s.Kind()
		if err != nil {
			return nil, err
		}
		switch kind {
		case Byte:
			s.kind = -1 // rearm Kind
			return []byte{s.byteval}, nil
		case String:
			b := make([]byte, size)
			if err = s.readFull(b); err != nil {
				return nil, err
			}
			if size == 1 && b[0] < 128 {
				return nil, ErrCanonSize
			}
			return b, nil
		default:
			return nil, ErrExpectedString
		}
	}

##### 스트림 구조 분석
다른 구성은 거의 인코더 코드와 디코더이지만, 특별한 구조가 인코더에 있지 있습니다. 즉 스트림입니다.
이 클래스는 RLP를 디코딩하는 보조 유동 방식으로 판독하는데 사용된다. 우리는 이전 복호 과정이 실질적으로 획득 된 제 타입 객체의 길이가 길이 및 데이터 종류에 따라 종류 () 메소드에 의해 디코딩 된 후 디코딩 될 언급된다. 그래서 우리는 필드 구조를 다루는 어떻게 그것의 데이터 구조는 우리가 구조를 먼저 호출 s.List () 메소드를 처리 할 때, 기억하고 다음) (각 필드, 마지막 호출 s.EndList를 디코딩 방법. 이러한 두 가지 방법의 내부에 팁의이 두 가지 방법을 살펴 보자.

	type Stream struct {
		r ByteReader
		// number of bytes remaining to be read from r.
		remaining uint64
		limited   bool
		// auxiliary buffer for integer decoding
		uintbuf []byte
		kind	Kind   // kind of value ahead
		size	uint64 // size of value ahead
		byteval byte   // value of single byte in type tag
		kinderr error  // error from last readKind
		stack   []listpos
	}
	type listpos struct{ pos, size uint64 }

스트림의 목록 방법, 시간의 메소드 호출 목록. 유형 오류를 다음과 일치 던져하지 않는 경우의이 종류와 길이를 얻을 수있는 종류의 방법을 부르 자, 우리는 listpos 객체는 객체가 핵심이다 스택으로 푸시됩니다했습니다. 이 개체의 현재 레코드의 POS 필드에서는이 목록 데이터의 바이트를 읽기 때문에 시간의 시작 개체가 데이터를 읽고 얼마나 많은 바이트의 총을 필요로 목록을 확실히 0 크기 필드 레코드입니다. 나는 각 후속 필드 다룰 때,이 분야의 POS, POS 비교 과정의 가치를 높이는 몇 가지 바이트 각을 읽을 수 있도록 결국 필드와 필드의 크기가 동일하지 않을 경우, 다음 예외를 동일하게 발생하는 .

	func (s *Stream) List() (size uint64, err error) {
		kind, size, err := s.Kind()
		if err != nil {
			return 0, err
		}
		if kind != List {
			return 0, ErrExpectedList
		}
		s.stack = append(s.stack, listpos{0, size})
		s.kind = -1
		s.size = 0
		return size, nil
	}

ListEnd 방법의 스트림 데이터의 수를 현재 데이터를 읽을 경우, 예외가 발생, 선언 된 길이 크기 POS 동일하지 않고, 현재의 스택이 비어 있지 않은 경우 다음 스택 팝 동작으로, 다음 스택 스택 POS의 상단에 추가 처리 된 데이터의 현재 길이 (이 경우를 처리 할 수는 - 구조의 필드 구조, 이러한 순환 구조)

	func (s *Stream) ListEnd() error {
		if len(s.stack) == 0 {
			return errNotInList
		}
		tos := s.stack[len(s.stack)-1]
		if tos.pos != tos.size {
			return errNotAtEOL
		}
		s.stack = s.stack[:len(s.stack)-1] // pop
		if len(s.stack) > 0 {
			s.stack[len(s.stack)-1].pos += tos.size
		}
		s.kind = -1
		s.size = 0
		return nil
	}


