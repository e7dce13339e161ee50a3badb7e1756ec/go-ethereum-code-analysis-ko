Hashimoto :I/O bound proof of work


Abstract: Using a cryptographic hash function not as a proofofwork by itself, but
rather as a generator of pointers to a shared data set, allows for an I/O bound
proof of work. This method of proof of work is difficult to optimize via ASIC
design, and difficult to outsource to nodes without the full data set. The name is
based on the three operations which comprise the algorithm: hash, shift, and
modulo.

초록 : 증거로 작동하지 않는 암호화 해시 함수 자체를 사용하여,
그러나 공유 데이터 집합 생성기에 대한 포인터를 허용하는 I / O 바운드로
증거를 작업 할 수 있습니다. 작업이 방법은 ASIC 설계 최적화하기 어려운 입증, 상황의 설정 완전한 데이터가없는 경우에 노드를 아웃소싱하기가 어렵습니다. 이름은 세 가지 동작을 기반 알고리즘을 구성한다 : 해시 시프트 및
모드.


The need for proofs which are difficult to outsource and optimize

워크로드 수요는 아웃소싱 및 최적화를 증명하기 어렵습니다

A common challenge in cryptocurrency development is maintaining decentralization ofthe
network. The use ofproofofwork to achieve decentralized consensus has been most notably
demonstrated by Bitcoin, which uses partial collisions with zero ofsha256, similar to hashcash. As
비트 코인의 인기가 성장하고, 전용 하드웨어 (현재 응용 프로그램 특정 집적 회로, 또는
ASIC들)을 신속 hashbased proofofwork 함수를 반복 제조되었다. 최근 프로젝트
similar to Bitcoin often use different algorithms for proofofwork, and often with the goal ofASIC
저항. 이러한 비트 코인 등의 알고리즘의 경우, 방어 인자 ofASICs 해당 상품을 의미
computer hardware can no longer be effectively used, potentially limiting adoption.

챌린지 암호화 금전적 개발 중심의 네트워크 구조를 유지하는 방법이다. 증거 비트 코인의 작업 부하를 달성하기 위해 SHA256 해시 퍼즐을 사용하기 때문에 일관성을 분산. 비트 코인 인기 특수 하드웨어 (응용 프로그램 특정 집적 회로의 전류, 또는 ASIC을) 신속하게 증거의 작업 방법 해시 함수를 기반으로 실행하는 데 사용되었다. 다른 유사한 비트 코인 새 프로젝트 작업은 일반적으로 알고리즘을 증명하는 데 사용, 종종 ASIC를 방지 목표입니다. 같은 비트 코인과 같은 알고리즘 같은 성능 향상을위한 ASIC는 일반 상업용 컴퓨터 하드웨어가 더 이상 효과적인 사용을 제한 할 수 없다 사용하는 것을 의미한다.

Proofofwork 또한 &quot;외주 &#39;또는 전용 머신 (a&quot;마이너 &quot;)에 의해 수행 될 수있다
검증되고 정확히 어떤 지식없이.이 비트 코인의 &quot;광산 풀&quot;의 경우는. 그것은 또한 종종
beneficial for a proofofwork algorithm to be difficult to outsource, in order to promote decentralization
and encourage all nodes participating in the proofofwork process to also verify transactions. With these
goals in mind, we present Hashimoto, an I/O bound proofofwork algorithm we believe to be resistant to
both ASIC design and outsourcing.

일은 똑같이 아웃소싱 증명 또는 작업을 수행하는 전용 시스템 (광산 기계) 내용의 확인을 위해이 기계는 명확하지 않습니다 것을 보여 주었다. &quot;광산 풀&quot;비트 코인 일반적인 경우입니다. 아웃소싱 작업의 양이 입증되면 알고리즘은 분산 촉진하기 어렵다
또한 트랜잭션을 확인하는 인증 과정에 참여하는 모든 노드를 권장합니다. 이 목표를 달성하기 위해, 우리는 또한 외부에서 조달하기 어려운 hashimoti, 내가 / O 대역폭, 우리는이 알고리즘이 ASIC에 저항 할 수 있다고 생각 워크로드를 기반으로 검증 된 알고리즘을 설계하지만.

Initial attempts at "ASIC resistance" involved changing Bitcoin's sha256 algorithm for a different,
more memory intensive algorithm, Percival's "scrypt" password based key derivation function1. Many
implementations set the scrypt arguments to low memory requirements, defeating much ofthe purpose of
the key derivation algorithm. While changing to a new algorithm, coupled with the relative obscurity of the
지연 허용 다양한 scryptbased cryptocurrencies는 scrypt 최적화 아식스 사용할 수 있습니다.
Similar attempts at variations or multiple heterogeneous hash functions can at best only delay ASIC
implementations.

&quot;ASIC 저항&quot;초기 시도 SHA256 알고리즘은 상이한 메모리 집약적 인 알고리즘의 퍼시발 &quot;scrypt&quot;패스워드 기반 키 유도 함수로, 현재 비트를 변경하는 단계를 포함한다. 많은 구현 스크립트 매개 변수는 크게 목적으로 키 유도 알고리즘을 훼손 낮은 메모리 요구 사항으로 설정됩니다. 새로운 알고리즘을 플러스 할 수있는 다양한 전환하는 동시에 지연이 발생할 수 있습니다 암호화 통화 상대 무명 scrypt 기반하지만, ASIC는 사용할 수 있습니다 최적화 scrypt. 유사 변경하거나 여러 이기종 해시 함수는 ASIC 구현을 지연시킬 수보십시오.

Leveraging shared data sets to create I/O bound proofs

는 I를 사용하여 설정 공유 데이터를 생성 / O 바운드 증명하기

	"A supercomputer is a device for turning compute-bound problems into I/O-bound  problems."
	-Ken Batcher


	&quot;슈퍼이 장치의 제한된 I / O 제약 문제로 계산된다.&quot;
	Ken Batcher

Instead, an algorithm will have little room to be sped up by new hardware if it acts in a way that commodity computer systems are already optimized for.

상품의 컴퓨터 시스템에서 실행되는 알고리즘이 방법을 최적화되었습니다 반대로, 알고리즘은 많은 공간이 새로운 하드웨어를 가속화 할 수 없습니다.

Since I/O bounds are what decades ofcomputing research has gone towards solving, it's unlikely that the relatively small motivation ofmining a few coins would be able to advance the state ofthe art in cache hierarchies. In the case that advances are made, they will be likely to impact the entire industry of computer hardware.

는 I / O 제한이 문제를 해결 한 수십 년 연구에 대해 계산되기 때문에, 일부 암호화 상대적으로 작은 금전적 인센티브가 예술적 수준의 캐시 계층을 올리는 것은 불가능하다 파다. 진보의 경우, 전체 컴퓨터 하드웨어 산업에 영향을 미칠 수 있습니다.

우연히, 상호 데이터 합의의 큰 세트가 ofcryptocurrency 현재 구현에 참여하는 모든 노드는, 실제로이 &quot;blockchain은&quot;모두 장점 ofspecialized 하드웨어를 제한하고,을 가지고 작업 노드를 필요로 할 수 있습니다이 큰 데이터 세트를 사용하여 통화 적힌 기초입니다. 전체 데이터 세트.

다행스럽게도, 현재 암호화 통화의 구현에 참여하는 모든 노드는 상호 합의 된 데이터의 큰 숫자를 가지고, 사실, &quot;블록 체인은&quot;기본 통화입니다. 이 큰 데이터 세트는 전용 하드웨어의 두 장점을 제한 할뿐만 아니라 작업자 노드가 전체 데이터 세트를 가질 수 있습니다.

하시모토는 offBitcoin의 proofofwork2을 기반으로합니다. 비트 코인의 경우, 하시모토와 같이, 성공적인
proofsatisfies the following inequality:

하시모토는 비트 코인 증거의 작업 부하를 기반으로합니다. 다음 부등식을 만족하는 성공적인 증거로 비트 코인, 하시모토의 경우 :

	hash_output < target

For bitcoin, the hash_output is determined by

토큰 비트에서 hash_output은 다음에 의해 결정된다.

	hash_output = sha256(prev_hash, merkle_root, nonce)

prev_hash 이전 블록의 해시이며 변경할 수없는 경우. merkle_root는 블록에 포함되는 트랜잭션에 기초하여, 각각의 노드에 대해 상이 할 것이다. 넌 스는 신속 hash_outputs 계산하고 부등식을 만족하지 않기 때문에 증가한다. 따라서 proofis SHA256 기능, 속도 ofsha256을 증가 또는 병렬 처리의 병목 현상은 ASIC을 매우 효율적으로 할 수있는 일이다.

prev_hash 이전 블록의 해시 값이며, 변경할 수 없다. merkle_root 트랜잭션 블록에 기초하여 생성되고, 각각의 노드에 대해 상이 할 것이다. 의이 넌스의 값을 수정하여 위의 불평등을 할 수 있습니다. 그래야 전체 작업은 SHA256 병목 방법을 시연하고, 크게 ASIC에 의해 계산 속도 SHA256을 늘리거나 병렬로 실행할 수 있습니다.

Hashimoto uses this hash output as a starting point, which is used to generated inputs for a second hash function. We call the original hash hash_output_A, and the final result of the prooffinal_output.

하시모토 해시 함수의 제 2 입력을 생성하기위한 시작 지점으로 사용 hash_output. 우리는 원래 해시 hash_output_A에 대한 최종 결과는 prooffinal_output입니다 호출합니다.

Hash_output_A can be used to select many transactions from the shared blockchain, which are then used as inputs to the second hash. Instead of organizing transactions into blocks, for this purpose it is simpler to organize all transactions sequentially. For example, the 47th transaction of the 815th block might be termed transaction 141,918. We will use 64 transactions, though higher and lower numbers could work, with different access properties. We define the following functions:

다음 두 번째 해시에 대한 입력으로 사용되는 공유 된 블록 사슬에서 거래를 복수 선택할 수 hash_output_a. 쉽게 모든 거래의 조직의 순서는 오히려 이것에 대한 목적을 거래보다는 블록으로있다 조직. 예를 들어, 815 블록 거래의 첫 번째 47 거래 141,918를 호출 할 수 있습니다. 높고 낮은 번호가 서로 다른 속성에 액세스 작업 할 수 있지만, 우리는 64 트랜잭션을 사용합니다. 우리는 다음과 같은 기능을 정의한다 :

-. 난스 64bit를 새로운 넌스은 각 시도 생성됩니다.
- get_txid(T) return the txid (a hash ofa transaction) of transaction number T from block B.
- block_height the current height ofthe block chain, which increases at each new block

-. 난스 64bit를 새로운 넌스 값을 생성합니다 각 시도.
- get_txid (T)는 트랜잭션 번호 블록 B가 트랜잭션의 ID를 얻는다
- block_height 블록 높이

Hashimoto chooses transactions by doing the following:

하시모토는 다음의 알고리즘에 의해 거래를 선택합니다 :

	hash_output_A = sha256(prev_hash, merkle_root, nonce)
	for i = 0 to 63 do
		shifted_A = hash_output_A >> i
		transaction = shifted_A mod total_transactions
		txid[i] = get_txid(transaction) << i
	end for
	TxID 삽입 txid_mix = [0] ⊕ TxID 삽입 [1] ... ⊕ TxID 삽입 [63]
	final_output = txid_mix ⊕ (논스 &lt;&lt; 192)

The target is then compared with final_output, and smaller values are accepted as proofs.

목표보다 작은 final_output 경우에는 허용됩니다.


