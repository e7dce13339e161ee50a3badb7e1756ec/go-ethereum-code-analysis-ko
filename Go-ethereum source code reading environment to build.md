
## 이동 - 에테 리움 소스 분석
이 에테 리움을 진행하기 때문에 광장 이더넷 클라이언트는 위의 GitHub의에서 분석하고 이에 대한 소스 코드의 가장 널리 사용되는, 이후의 분석이다. 그럼 난 창 10 64 비트 환경을 사용하고 있습니다.

### 이동 에테 리움 디버깅 환경 구축
첫째, 설치 패키지 GO 웹 사이트 벽 때문에, 설치로 이동 다운로드, 그래서 다음 주소에서 다운로드합니다.

	https://studygolang.com/dl/golang/go1.9.1.windows-amd64.msi

내가 설정 (PATH 환경 변수에 \로 이동 \ bin 디렉토리, 다음 GOPATH 환경 변수를 추가, 코드 경로 당신의 GO 언어 다운로드 GOPATH의 값을 설정 : 설치 한 후, 환경 변수는 C로 설정 C : \의 GOPATH)

![image](https://raw.githubusercontent.com/wugang33/go-ethereum-code-analysis/master/picture/go_env_1.png)

망할 놈의 설치 도구는 자식 도구를 설치 지원해야하는 언어가 자동으로 GitHub의 이눔 도구에서 코드를 다운로드 이동 네트워크의 튜토리얼을 참조하시기 바랍니다

명령 줄 도구를 다운로드 이동 - 에테 리움 코드를 엽니 다
	
	go get github.com/ethereum/go-ethereum

명령이 성공하면, 코드는 다음 디렉토리에 다운로드됩니다, % GOPATH % \ SRC \ github.com \ 에테 리움 \ 이동 - 에테 리움
구현 프로세스를 표시하는 경우

	# github.com/ethereum/go-ethereum/crypto/secp256k1
	exec: "gcc": executable file not found in %PATH%

당신은 우리가 다운로드 다음 주소에서 설치, GCC 도구를 설치해야합니다

	http://tdm-gcc.tdragon.net/download

다음으로, IDE 도구를 설치합니다. 나는 JetBrains의 IDE가 고글란 트 섬이다 사용하고 있습니다. 다음 주소에서 다운로드 할 수 있습니다

	https://download.jetbrains.com/go/gogland-173.2696.28.exe

설치가 완료되면 IDE에서 파일을 엽니 다 -.&gt; 열기 -&gt; 선택 GOPATH \ SRC \ github.com \ 에테 리움 \ 이동 - 에테 리움 디렉토리 열립니다.

그런 다음. 이동 - 에테 리움 / RLP / decode_test.go을 열 완료 설정 환경을 대신하여, 성공적으로 실행하는 경우, 편집 상자 오른쪽에서 실행을 선택합니다.

![image](https://raw.githubusercontent.com/wugang33/go-ethereum-code-analysis/master/picture/go_env_2.png)

### 아마도 도입 에테 리움 카탈로그를 이동
조직 구조 이동 - 에테 리움 프로젝트는 기본적으로 각 디렉토리에 대한 간단한 구조에 따라 기능 모듈 디렉토리에 따라 구분되며, 패키지가 된 GO 언어의 각 디렉토리, 나는 자바 패키지 내부에 유사해야 이해 의미한다.


	accounts		이더넷은 계정 관리 워크샵의 높은 수준을 달성
	bmt		메르켈 총리는 이진 트리를 달성
	build		주로 컴파일하고 일부 스크립트와 구성을 구축
	cmd		도구 명령 줄과는 소개 다음, 명령 줄 도구를 많이 갈라
		/abigen		Source code generator to convert Ethereum contract definitions into easy to use, compile-time type-safe Go packages
		/bootnode만 시작 노드 네트워크 검색을 실현하기
		/evm	광장 이더넷 가상 머신 툴, 코드 디버깅 환경의 분리에 의해, 구성을 제공하는 데 사용
		/faucet		
		/geth	이더넷 광장 명령 행 클라이언트, 가장 중요한 도구
		/p2psim	API를 HTTP를 시뮬레이션 할 수있는 도구를 제공합니다
		/puppeth새로운 이더넷 네트워크 마법사 광장을 만들기
		/rlpdump RLP는 포맷 된 출력 데이터를 제공한다
		/swarm	네트워크 액세스 포인트 떼
		/util	그것은 몇 가지 일반적인 도구를 제공합니다
		/wnode	이것은 간단한 위스퍼 노드입니다. 이는 부트 스트랩 노드 독립적으로 사용될 수있다. 또한, 다른 테스트 및 진단 목적으로 사용할 수 있습니다.
	common		그것은 몇 가지 일반적인 도구를 제공합니다
	compression		Package rle implements the run-length encoding used for Ethereum data.
	consensus	이더넷은 ethhash, 파벌 (증명 권한)으로, 일부 합의 알고리즘 광장을 제공합니다
	console		콘솔 클래스
	contracts	
	core		코어 데이터 구조와 알고리즘 스퀘어 에테르 (가상 기계 상태 블록 사슬의 블룸 필터)
	crypto		암호화 및 해시 알고리즘,
	eth		광장은 이더넷 프로토콜을 실현
	ethclient	RPC는 이더넷 클라이언트 광장을 제공합니다
	ethdb		(실제로 메모리와 데이터베이스를 테스트하기 위해 사용 leveldb 포함) ETH 데이터베이스
	ethstats	네트워크 상태보고를 제공
	event		실시간 이벤트 처리
	les		경량 프로토콜은 이더넷 광장의 부분 집합을 구현
	light		구현 이더넷 광장, 경량 클라이언트에 대한 주문형 검색 기능을 제공합니다
	log		그것은 인간 친화적 인 로그 정보를 모두 제공
	metrics		디스크 카운터 제공
	miner		광장 블록은 이더넷과 채광을 제공하기 위해 만들어졌습니다
	mobile		일부 모바일 단말기 사용 warpper
	node		이더넷 노드 광장의 다양한 형태
	p2p		이더넷 네트워크 프로토콜의 P2P 광장
	rlp		에테르 광장 직렬화
	rpc		원격 메소드 호출
	swarm		메뚜기 네트워크 처리
	tests		테스트
	trie		이더넷 광장 중요한 데이터 구조의 패키지 트라이는는 Merkle 패트리샤 시도 구현합니다.
	whisper		이 프로토콜 속삭임 노드를 제공합니다.

당신은 코드 이더넷 광장의 양이 계속 커지고있다 볼 수 있지만, 거친 모습은, 코드 구조는 매우 좋다. 나는 분석을위한 몇 가지 상대적으로 독립적 인 모듈로 시작합니다. 그런 다음, 내부 코드에 대한 심층 분석. 초점은 옐로우 북 내부의 P2P 네트워크 모듈과 관련이없는 초점을 맞출 수 있습니다.
