P2P는 소스 코드와 여러 패키지를 다음

- discover	 포함 [Kademlia 프로토콜 (참조 / Kademlia 프로토콜 소개 원칙 .PDF). P2P 노드 탐색 프로토콜은 UDP 기반으로합니다.
- discv5	새로운 노드 탐색 프로토콜입니다. 또는 테스트 속성입니다. 분석은 언급하지 않았다.
- nat		코드의 네트워크 주소 변환 부분
- netutil	일부 도구
- simulations아날로그 P2P 네트워크. 분석은 언급하지 않았다.

소스 분석의 일부를 발견

- [database.go 영구 스토리지 노드 발견 (P2P-database.go 소스 분석 .md)
- [tabel.go Kademlia 프로토콜 코어 로직 (P2P-table.go 소스 분석 .md)
- [udp.go UDP 프로토콜 처리 로직 (P2P-udp.go 소스 분석 .md)
- (P2P-NAT 소스 분석 .md) 네트워크 주소 번역 nat.go]

P2P / 소스 코드 분석 부분

- (노드 간의 P2P-rlpx .md 암호화 링크) 노드 rlpx.go 간의 암호화 처리 프로토콜 링크]
- (P2P-dial.go 소스 분석 .md) 선택된 노드는 후 처리 로직 dail.go에 접속된다]
- [처리 및 프로토콜 처리 노드와 노드를 연결 peer.go (P2P-peer.go 소스 분석 .md)
- [논리 server.go에 P2P 서버 (P2P-server.go 소스 분석 .md)
