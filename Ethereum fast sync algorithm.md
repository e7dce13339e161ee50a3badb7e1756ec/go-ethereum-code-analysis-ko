번역 ([https://github.com/ethereum/go-ethereum/pull/1889](https://github.com/ethereum/go-ethereum/pull/1889))

This PR aggregates a lot of small modifications to core, trie, eth and other packages to collectively implement the eth/63 fast synchronization algorithm. In short, geth --fast.

제출 된 요청은 핵심, 트라이, ETH와 다른 패키지에 약간 변경을 포함하고, 공동으로 빠른 동기화 알고리즘 ETH / 63 년대를 달성하기 위해. 즉, --fast geth.

## 알고리즘 알고리즘

The goal of the the fast sync algorithm is to exchange processing power for bandwidth usage. Instead of processing the entire block-chain one link at a time, and replay all transactions that ever happened in history, fast syncing downloads the transaction receipts along the blocks, and pulls an entire recent state database. This allows a fast synced node to still retain its status an an archive node containing all historical data for user queries (and thus not influence the network's health in general), but at the same time to reassemble a recent network state at a fraction of the time it would take full block processing.

대상 빠른 동기화 알고리즘은 대역폭의 변화를 계산하는 것이다. 빠른 동기화가 체인 링크를 통해 전체 블록을 처리하지만 역사에서 발생한 모든 트랜잭션을 다시 재생되지 않으며, 이러한 블록을 따라 빠른 동기화는 거래 문서를 다운로드 한 다음 가장 가까운 정수 상태 데이터베이스를 가져옵니다. 이 지역의 전체 양을 사용, 최신 상태 변경, 블록 (일반 네트워크의 건강에 영향을주지 않는 때문에 등) 노드의 빠른 동기화가 여전히 보관 사용자 쿼리에 대한 모든 노드를 기록 데이터 포함 개최 할 수 있습니다 처리 모드 블록.

An outline of the fast sync algorithm would be:

- Similarly to classical sync, download the block headers and bodies that make up the blockchain
- Similarly to classical sync, verify the header chain's consistency (POW, total difficulty, etc)
- Instead of processing the blocks, download the transaction receipts as defined by the header
- Store the downloaded blockchain, along with the receipt chain, enabling all historical queries
- When the chain reaches a recent enough state (head - 1024 blocks), pause for state sync:
	- Retrieve the entire Merkel Patricia state trie defined by the root hash of the pivot point
	- For every account found in the trie, retrieve it's contract code and internal storage state trie
- Upon successful trie download, mark the pivot point (head - 1024 blocks) as the current head
- Import all remaining blocks (1024) by fully processing them as in the classical sync

빠른 동기화 알고리즘 요약 :

- 구성된 다운로드 영역 헤더 블록 체 블록 사슬 원래 유사 동기
- 원래의 동기화 유사하게, 헤더 영역 (POW 총 어려움)의 일관성을 검증
- 대신 처리 블록, 거래 영수증 헤더에 의해 정의 다운로드 영역입니다.
- 모든 과거 문의 가능 기억 다운로드 블록 사슬 영수증 체인
- 체인 가까운 상태 (머리 - 1,024 블록)에 도달 할 때, 일시 정지 상태 동기화 :
	- 피벗 점에 의해 정의 된 완전한 메르켈 패트리샤 트리는 상태 블록을 가져 오기
	- 각 계정에 대해 메르켈 패트리샤 트리는 내부의 계약 코드와 트리는의 중간 저장을 얻을 수 있습니다
- 메르켈 패트리샤 트리는 다운로드가 성공하면 피벗 점은 블록 헤더의 현재 영역으로 정의된다
- 그것은 완전히 (1024), 본래의 동기와 동일한 처리에 의해 모든 나머지 블록을 소개

## 분석 분석
By downloading and verifying the entire header chain, we can guarantee with all the security of the classical sync, that the hashes (receipts, state tries, etc) contained within the headers are valid. Based on those hashes, we can confidently download transaction receipts and the entire state trie afterwards. Additionally, by placing the pivoting point (where fast sync switches to block processing) a bit below the current head (1024 blocks), we can ensure that even larger chain reorganizations can be handled without the need of a new sync (as we have all the state going that many blocks back).

전체 머리와 검증 체인을 다운로드하면, 우리는 기존의 동기의 보안, 머리가 해시 (영수증, 주 및 다른 시도를) 효과가 포함되어 보장 할 수 있습니다. 이 해시를 바탕으로, 우리는 자신있게 거래 영수증과 주 전체 트리를 다운로드 할 수 있습니다. 또한, 점 (고속 동기 블록 처리로 전환) 회동하여 현재 헤더 영역 (1024) 아래에 배치되어 우리 때문에 (새로운 동기화를 필요로하지 않고, 프로세스는 더 큰 블록 사슬 재결합 될 수 있도록 할 수 있다는 것이다 우리는 모든 상태 TODO)가 있습니다.

## 고려 사항주의 사항
The historical block-processing based synchronization mechanism has two (approximately similarly costing) bottlenecks: transaction processing and PoW verification. The baseline fast sync algorithm successfully circumvents the transaction processing, skipping the need to iterate over every single state the system ever was in. However, verifying the proof of work associated with each header is still a notably CPU intensive operation.

포로 트랜잭션 처리 및 검증 : 과거 병목 기반 동기화 블록 처리를 두 개 (약 유사한 비용)을 갖는. 베이스 라인 빠른 동기화 알고리즘이 성공적으로 반복해야하는 시스템의 각 상태에 대한 필요성을 건너 뛰는, 트랜잭션을 우회. 그러나, 검증 및 증명은 작업의 각 머리는 여전히 CPU 집약적 인 작업과 연관되어 있습니다.

However, we can notice an interesting phenomenon during header verification. With a negligible probability of error, we can still guarantee the validity of the chain, only by verifying every K-th header, instead of each and every one. By selecting a single header at random out of every K headers to verify, we guarantee the validity of an N-length chain with the probability of (1/K)^(N/K) (i.e. we have 1/K chance to spot a forgery in K blocks, a verification that's repeated N/K times).

그러나, 우리가 오류로 인해 확률 헤더 동안 지역에서 흥미로운 현상을 발견 확인할 수 있습니다 것은 무시할 수있다, 우리는 여전히뿐만 아니라 머리 당은 K 헤드의 각 있는지 확인해야합니다, 체인의 효과를 보장 할 수 있습니다. 각 헤드 K에서 임의로 선택하여 검증 헤드, 우리는 확률은 블록 N (/ K (1)) ^ (N / K) (K의 사슬 길이를 위조 할 수 있도록, 우리는 1 / K 기회를 ) 위조 된 것을 발견하고, N / K 배의 선으로 확인 하였다.

Let's define the negligible probability Pn as the probability of obtaining a 256 bit SHA3 collision (i.e. the hash Ethereum is built upon): 1/2^128. To honor the Ethereum security requirements, we need to choose the minimum chain length N (below which we veriy every header) and maximum K verification batch size such as (1/K)^(N/K) <= Pn holds. Calculating this for various {N, K} pairs is pretty straighforward, a simple and lenient solution being http://play.golang.org/p/B-8sX_6Dq0.

2 ^ 128 : 우리 확률 PN 정의 256 SHA3 충돌 (이더넷 스퀘어 해시 알고리즘)을 획득하는 확률을 무시할 것이다. 안전 요건 에테르 스퀘어 준수하기 위하여, 우리는 최소의 사슬 길이가 N (우리는 각 블록 검증 전에있다)을 선택을 확인해야하는 최대 일괄 크기 K (1 / K) ^ (N / K) &lt;= PN. 다양한 {N, K}는로 계산되고, 매우 간단하고 단순하며 용액 http://play.golang.org/p/B-8sX_6Dq0 완화된다.

|N		|K		|N		|K			|N		|K			|N		|K  |
| ------|-------|-------|-----------|-------|-----------|-------|---|
|1024	|43		|1792	|91			|2560	|143		|3328	|198|
|1152	|51		|1920	|99			|2688	|152		|3456	|207|
|1280	|58		|2048	|108		|2816	|161		|3584	|217|
|1408	|66		|2176	|116		|2944	|170		|3712	|226|
|1536	|74		|2304	|128		|3072	|179		|3840	|236|
|1664	|82		|2432	|134		|3200	|189		|3968	|246|


The above table should be interpreted in such a way, that if we verify every K-th header, after N headers the probability of a forgery is smaller than the probability of an attacker producing a SHA3 collision. It also means, that if a forgery is indeed detected, the last N headers should be discarded as not safe enough. Any {N, K} pair may be chosen from the above table, and to keep the numbers reasonably looking, we chose N=2048, K=100. This will be fine tuned later after being able to observe network bandwidth/latency effects and possibly behavior on more CPU limited devices.

위의 표는이 방법으로 해석되어야한다 : 우리는 모든 확률 K 지구는 존 헤더를 확인 헤더 경우, N 헤더 영역 후, 확률은 공격자가 SHA3 충돌을 발생 단조보다 작습니다. 이 또한 위조를 발견 할 경우, N 헤드의 마지막 때문에 부적절한 보안의 폐기되어야 함을 의미한다. 어떤 {N, K} 쌍 점의 번호를 선택하기 위해 좋은 볼을 선택할 수 있습니다, 우리는 테이블에서 N = 2048, K = (100)을 선택합니다. 후속가 실행에 영향을 미칠 수있는 네트워크 대역폭 / 지연 상황에 일부 제한 CPU의 성능 비교 장치에 따라 조절 될 수있다.

Using this caveat however would mean, that the pivot point can be considered secure only after N headers have been imported after the pivot itself. To prove the pivot safe faster, we stop the "gapped verificatios" X headers before the pivot point, and verify every single header onward, including an additioanl X headers post-pivot before accepting the pivot's state. Given the above N and K numbers, we chose X=24 as a safe number.

그러나,이 기능을 사용하여 후에 N 블록을 가져온 다음 피벗 노드가 단지 안전한 것으로 간주 가져올 것을 의미한다. 더 빨리 선회의 안전을 증명하기 위해, 우리는 피벗까지 확인 나타나는 각 블록, 피벗 노드 X의 거리를 확인하는 장소에서 행동 스페이서를 중지합니다. 숫자 N과 K 위의 관점에서, 우리는 보안 디지털 X = 24으로 선택.

With this caveat calculated, the fast sync should be modified so that up to the pivoting point - X, only every K=100-th header should be verified (at random), after which all headers up to pivot point + X should be fully verified before starting state database downloading. Note: if a sync fails due to header verification the last N headers must be discarded as they cannot be trusted enough.

주의해야 할 점을 계산하여, 빠른 동기화는 피봇 포인트를 수정해야 - 다운로드 상태 데이터베이스가 완료된 후 전체 유효성 검사가 필요하다 각 블록 후 X, A는 무작위로 확인하기 위해 선택하는 모든 헤더 영역 (100), 때문에 헤더 인증 영역의 경우 실패의 N 영역의 마지막 폐기 할 필요가 헤더 때문에 동기화 그들에 대한 신뢰의 표준에 도달하지 말아야 실패했습니다.


## 결손 약점
Blockchain protocols in general (i.e. Bitcoin, Ethereum, and the others) are susceptible to Sybil attacks, where an attacker tries to completely isolate a node from the rest of the network, making it believe a false truth as to what the state of the real network is. This permits the attacker to spend certain funds in both the real network and this "fake bubble". However, the attacker can only maintain this state as long as it's feeding new valid blocks it itself is forging; and to successfully shadow the real network, it needs to do this with a chain height and difficulty close to the real network. In short, to pull off a successful Sybil attack, the attacker needs to match the network's hash rate, so it's a very expensive attack.

일반적인 블록 체인 (예 : 비트 코인, 이더넷 광장 및 기타 등)에 더 취약 마녀 공격하고, 공격자는 그래서 거짓 공격자 상태를 받아 완전히 주요 네트워크에서 격리 공격자가 될하려고합니다. 같은 자금이 실제 네트워크에 허위 네트워크에 지출하면서 공격자를 할 수 있습니다. 그러나, 이것은 공격자가 실제 블록을 제공 할뿐만 아니라, 네트워크의 실제 요구의 성공에 영향을 미치는 자신을 위조해야합니다, 우리는 블록의 높이와 난이도에 실제 네트워크에 가까운해야합니다. 간단히 말해, 성공적으로 마녀의 공격을 구현하기 위해, 공격자는 매우 고가의 공격이며, 주요 네트워크에 가까운 속도를 해시 할 필요가있다.

Compared to the classical Sybil attack, fast sync provides such an attacker with an extra ability, that of feeding a node a view of the network that's not only different from the real network, but also that might go around the EVM mechanics. The Ethereum protocol only validates state root hashes by processing all the transactions against the previous state root. But by skipping the transaction processing, we cannot prove that the state root contained within the fast sync pivot point is valid or not, so as long as an attacker can maintain a fake blockchain that's on par with the real network, it could create an invalid view of the network's state.

전통적인 마녀 공격에 비해 빠른 동기화뿐만 아니라 EVM 메커니즘을 우회하는 것도 가능 노드가 네트워크 실제 네트워크의 다른보기를 제공하지만, 공격자에 대한 추가 기능을 제공합니다. 단지 이전 상태 광장 이더넷 프로토콜은 모든 거래의 루트 해시를 처리하여 뿌리의 상태를 확인합니다. 그러나 트랜잭션을 건너 뜀으로써, 우리는 유효한, 너무 오래 공격자가 동일한 실제 네트워크 거짓 블록 체인을 유지할 수있는 한, 잘못된 네트워크 상태보기를 만들 수 있습니다 포함 빠른 동기화 상태 루트 피벗 점을 입증 할 수 있습니다.

To avoid opening up nodes to this extra attacker ability, fast sync (beside being solely opt-in) will only ever run during an initial sync (i.e. when the node's own blockchain is empty). After a node managed to successfully sync with the network, fast sync is forever disabled. This way anybody can quickly catch up with the network, but after the node caught up, the extra attack vector is plugged in. This feature permits users to safely use the fast sync flag (--fast), without having to worry about potential state root attacks happening to them in the future. As an additional safety feature, if a fast sync fails close to or after the random pivot point, fast sync is disabled as a safety precaution and the node reverts to full, block-processing based synchronization.

초기 동기화시 (현지 블록 체인 노드가 비어)에서만 실행됩니다 공격자에게 (지정) 빠른 동기화 할 수있는 기능을 열려면이 추가 노드를 방지하기 위해. 노드가 네트워크에 성공적으로 동기화 한 후, 빠른 동기화는 항상 비활성화. 누구든지 신속하게 네트워크를 잡을 수 있도록하지만, 캐치 노드 후, 추가 공격 벡터가 삽입되었다. 이 기능은 사용자가 안전하게 잠재적 인 공격에 대한 미래에 발생하는 국가의 루트를 걱정하지 않고, 빠른 동기화 플래그 (--fast) 사용할 수 있습니다. 거의 임의의 피벗 점 후 빠른 동기화 또는 실패 할 경우 추가적인 안전 장치로서, 안전 조치로서 완전히 동기화 블록 기반 프로세싱 빠른 동기화 및 복구 노드 비활성화.

## 성능 성능
To benchmark the performance of the new algorithm, four separate tests were run: full syncing from scrath on Frontier and Olympic, using both the classical sync as well as the new sync mechanism. In all scenarios there were two nodes running on a single machine: a seed node featuring a fully synced database, and a leech node with only the genesis block pulling the data. In all test scenarios the seed node had a fast-synced database (smaller, less disk contention) and both nodes were given 1GB database cache (--cache=1024).

네 개의 개별 테스트를 실행하는 새로운 알고리즘 벤치 마크의 성능을 위해 : 프론티어에서 동기화 고전뿐만 아니라 새로운 동기화 메커니즘을 사용하여, 올림픽 scrath 완전히 동기화. 모든 경우에, 컴퓨터에 두 개의 노드를 실행 : 노드는 데이터베이스 씨앗이 완전히 동기화 필요하고, 거머리 노드는 블록 데이터를 시작 당깁니다. 모든 테스트 시나리오에서 시드 노드가 데이터베이스 (작은, 적은 디스크 경합)의 빠른 동기화를 가지고, 두 개의 노드 (--cache = 1024) 1GB의 데이터베이스 캐시가 있습니다.

The machine running the tests was a Zenbook Pro, Core i7 4720HQ, 12GB RAM, 256GB m.2 SSD, Ubuntu 15.04.

테스트 시스템을 실행하는 것은 프로 Zenbook, 코어 i7 4720HQ, 12기가바이트 RAM, 256기가바이트의 M.2의 SSD, 우분투 15.04입니다.

| Dataset (blocks, states)	| Normal sync (time, db)	| Fast sync (time, db) |
| ------------------------- |:-------------------------:| ---------------------------:|
|Frontier, 357677 blocks, 42.4K states	| 12:21 mins, 1.6 GB	| 2:49 mins, 235.2 MB |
|Olympic, 837869 blocks, 10.2M states	| 4:07:55 hours, 21 GB	| 31:32 mins, 3.8 GB  |


The resulting databases contain the entire blockchain (all blocks, all uncles, all transactions), every transaction receipt and generated logs, and the entire state trie of the head 1024 blocks. This allows a fast synced node to act as a full archive node from all intents and purposes.

결과 데이터베이스는 전체 체인 블록 (모든 블록, 모든 블록, 모든 트랜잭션)하고, 생성 된 로그 거래 영수증의 각, 전체 상태 트리의 머리 (1024)이 포함되어 있습니다. 이것은 모든 의도와 목적에 대한 완전한 아카이브 노드 역할을 할 수있는 퀵 싱크 노드 수 있습니다.


## 발언 닫기 결론 발언
The fast sync algorithm requires the functionality defined by eth/63. Because of this, testing in the live network requires for at least a handful of discoverable peers to update their nodes to eth/63. On the same note, verifying that the implementation is truly correct will also entail waiting for the wider deployment of eth/63.

빠른 동기화 알고리즘은 ETH / 63로 정의 작동 할 필요가있다. 이 때문에, 현재의 네트워크 테스트는 적어도 몇 가지 다른 노드에 몇 가지가 노드 ETH / 63을 업데이트 발견 될 필요가있다. 같은 설명은, 구현 ETH / 63이 더 광범위하게 배포 기다려야 할 것이다 진정으로 올바른지 확인합니다.