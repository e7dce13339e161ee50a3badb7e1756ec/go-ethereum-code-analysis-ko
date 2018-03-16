P2P 네트워크 디스커버리 프로토콜은 네트워크 노드 발견 과정을 Kademlia 프로토콜을 사용한다. 노드와 노드 업데이트를 찾습니다. Kademlia 프로토콜은 네트워크 통신을위한 UDP 프로토콜을 사용합니다.

코드의이 부분은 당신이 Kademlia 프로토콜이 무엇인지 봐 내부 Kademlia 프로토콜 개요 참조를 살펴 것이 좋습니다 읽어보십시오.

첫째, 데이터 구조 봐. 네 네트워크 전송 패킷 각각 핑 퐁 findnode 이웃 (UDP 패킷 프로토콜은 프로토콜 기반 전송 패킷이다). 다음 네 패킷 형식을 정의합니다.


	// RPC packet types
	const (
		pingPacket = iota + 1 // zero is 'reserved'
		pongPacket
		findnodePacket
		neighborsPacket
	)
	// RPC request structures
	type (
		ping struct {
			Version	uint		 // 프로토콜 버전
			From, To   rpcEndpoint	대상 IP 주소의 // 소스 IP 주소
			Expiration uint64		// 타임 아웃
			// Ignore additional fields (for forward compatibility).
			//그것은 무시 필드가 될 수 있습니다. 이전 버전과의 호환성을 위해
			Rest []rlp.RawValue `rlp:"tail"`
		}
	
		// pong is the reply to ping.
		//패킷을 핑 (ping)에 응답
		pong struct {
			// This field should mirror the UDP envelope address
			// of the ping packet, which provides a way to discover the
			// the external address (after NAT).
			//대상 IP 주소
			To rpcEndpoint
			//설명이 패키지는 탁구 패킷에 대한 응답이다. 핑 패킷은 해시 값을 포함
			ReplyTok   []byte // This contains the hash of the ping packet.
			//절대 시간 시간 제한을 패키지로 제공된다. 수신 된 패킷은 시간이 시간을 초과하면, 패킷을 초과 여겨진다.
			Expiration uint64 // Absolute timestamp at which the packet becomes invalid.
			// Ignore additional fields (for forward compatibility).
			Rest []rlp.RawValue `rlp:"tail"`
		}
		//findnode은 최근 대상 노드에서 쿼리하는 데 사용됩니다
		// findnode is a query for nodes close to the given target.
		findnode struct {
			//대상 노드
			Target	 NodeID // doesn't need to be an actual public key
			Expiration uint64
			// Ignore additional fields (for forward compatibility).
			Rest []rlp.RawValue `rlp:"tail"`
		}
	
		// reply to findnode
		//findnode 응답
		neighbors struct {
			//비교적 가까운 거리 타겟 노드 값.
			Nodes	  []rpcNode
			Expiration uint64
			// Ignore additional fields (for forward compatibility).
			Rest []rlp.RawValue `rlp:"tail"`
		}
	
		rpcNode struct {
			IP  net.IP // len 4 for IPv4 or 16 for IPv6
			UDP uint16 // for discovery protocol
			TCP uint16 // for RLPx protocol
			ID  NodeID
		}
	
		rpcEndpoint struct {
			IP  net.IP // len 4 for IPv4 or 16 for IPv6
			UDP uint16 // for discovery protocol
			TCP uint16 // for RLPx protocol
		}
	)


두 종류의 인터페이스를 정의 패킷 타입은 패킷 할당의 네 가지 유형의 인터페이스가 서로 다른 방법을 처리해야한다. CONN 인터페이스는 UDP를 연결하는 기능을 정의한다.


	type packet interface {
		handle(t *udp, from *net.UDPAddr, fromID NodeID, mac []byte) error
		name() string
	}
	
	type conn interface {
		ReadFromUDP(b []byte) (n int, addr *net.UDPAddr, err error)
		WriteToUDP(b []byte, addr *net.UDPAddr) (n int, err error)
		Close() error
		LocalAddr() net.Addr
	}


중요한 UDP 구조는, 마지막 필드의 * 표는 익명의 필드 안으로 들어가 있음을 유의하십시오. 즉, UDP를 직접 익명 필드 테이블 메서드를 호출 할 수 있습니다.


	// udp implements the RPC protocol.
	type udp struct {
		conn		conn				// 네트워크 연결
		netrestrict *netutil.Netlist
		priv		*ecdsa.PrivateKey	// 개인 키는, 당신의 ID는이 오는에 의해 생성됩니다.
		ourEndpoint rpcEndpoint
	
		addpending chan *pending		// 대기중인를 적용하는 데 사용
		gotreply   chan reply			// 큐는 응답을 얻기 위해
	
		closing chan struct{}			대기열에 // 닫기
		nat	 nat.Interface				
	
		*Table
	}



출원 및 응답 구조. 이 둘 사이의 통신을위한 구조는 사용자의 일상적인 내부 구조를 이동합니다.


	// pending represents a pending reply.
	// some implementations of the protocol wish to send more than one
	// reply packet to findnode. in general, any neighbors packet cannot
	// be matched up with a specific findnode packet.
	// our implementation handles this by storing a callback function for
	// each pending reply. incoming packets from a node are dispatched
	// to all the callback functions for that node.
	//구조를 나타냅니다중인 것은 응답을 기다리고 있습니다
	//각각의 콜백을 저장하려면 우리는 보류 응답하여이를 수행. 모든 패킷을 하나 개의 노드는 대응하는 콜백 최상위 노드에 할당에서.
	type pending struct {
		// these fields must match in the reply.
		from  NodeID
		ptype byte
	
		// time when the request must complete
		deadline time.Time
	
		// callback is called when a matching reply arrives. if it returns
		// true, the callback is removed from the pending reply queue.
		// if it returns false, the reply is considered incomplete and
		// the callback will be invoked again for the next matching reply.
		//경우 반환 값은 true입니다. 이어서 내부 콜백 큐로부터 제거 될 것이다. false를 반환하는 경우가 완료되지 않은 응답을 다음 회신을 기다릴 것입니다 것으로 생각된다.
		callback func(resp interface{}) (done bool)
	
		// errc receives nil when the callback indicates completion or an
		// error if no further reply is received within the timeout.
		errc chan<- error
	}
	
	type reply struct {
		from  NodeID
		ptype byte
		data  interface{}
		// loop indicates whether there was
		// a matching request by sending on this channel.
		//이 채널의 상단에 메시지를 전송함으로써 일치하는 요청을 나타냅니다.
		matched chan<- bool
	}


UDP의 창조

	// ListenUDP returns a new table that listens for UDP packets on laddr.
	func ListenUDP(priv *ecdsa.PrivateKey, laddr string, natm nat.Interface, nodeDBPath string, netrestrict *netutil.Netlist) (*Table, error) {
		addr, err := net.ResolveUDPAddr("udp", laddr)
		if err != nil {
			return nil, err
		}
		conn, err := net.ListenUDP("udp", addr)
		if err != nil {
			return nil, err
		}
		tab, _, err := newUDP(priv, conn, natm, nodeDBPath, netrestrict)
		if err != nil {
			return nil, err
		}
		log.Info("UDP listener up", "self", tab.self)
		return tab, nil
	}
	
	func newUDP(priv *ecdsa.PrivateKey, c conn, natm nat.Interface, nodeDBPath string, netrestrict *netutil.Netlist) (*Table, *udp, error) {
		udp := &udp{
			conn:		c,
			priv:		priv,
			netrestrict: netrestrict,
			closing:	 make(chan struct{}),
			gotreply:	make(chan reply),
			addpending:  make(chan *pending),
		}
		realaddr := c.LocalAddr().(*net.UDPAddr)
		if natm != nil {   //NATM은 NAT 매핑은 외부 주소를 얻기 위해 사용
			if !realaddr.IP.IsLoopback() {  //주소는 로컬 루프백 주소 인 경우
				go nat.Map(natm, udp.closing, "udp", realaddr.Port, realaddr.Port, "ethereum discovery")
			}
			// TODO: react to external IP changes over time.
			if ext, err := natm.ExternalIP(); err == nil {
				realaddr = &net.UDPAddr{IP: ext, Port: realaddr.Port}
			}
		}
		// TODO: separate TCP port
		udp.ourEndpoint = makeEndpoint(realaddr, uint16(realaddr.Port))
		//후속 테이블을 만듭니다 소개합니다. 이 클래스에서 구현 Kademlia 주요 논리.
		tab, err := newTable(udp, PubkeyID(&priv.PublicKey), realaddr, nodeDBPath)
		if err != nil {
			return nil, nil, err
		}
		udp.Table = tab   //익명 필드 할당
		
		go udp.loop()		//go routine 
		go udp.readLoop()네트워크에 대한 데이터를 읽을 //.
		return udp.Table, udp, nil
	}

이전 보류에 대해 이야기 보류 치료의 핑 방법은 응답을 기다리고 있습니다. 여기가 응답을 기다리는 분석하는 코드를 통해입니다.

addpending 위해 보류 보류 구조를 송신하는 방법. 그 다음, 프로세스는 기다리는 메시지를 수신한다.

	// ping sends a ping message to the given node and waits for a reply.
	func (t *udp) ping(toid NodeID, toaddr *net.UDPAddr) error {
		// TODO: maybe check for ReplyTo field in callback to measure RTT
		errc := t.pending(toid, pongPacket, func(interface{}) bool { return true })
		t.send(toaddr, pingPacket, &ping{
			Version:	Version,
			From:	   t.ourEndpoint,
			To:		 makeEndpoint(toaddr, 0), // TODO: maybe use known TCP port from DB
			Expiration: uint64(time.Now().Add(expiration).Unix()),
		})
		return <-errc
	}
	// pending adds a reply callback to the pending reply queue.
	// see the documentation of type pending for a detailed explanation.
	func (t *udp) pending(id NodeID, ptype byte, callback func(interface{}) bool) <-chan error {
		ch := make(chan error, 1)
		p := &pending{from: id, ptype: ptype, callback: callback, errc: ch}
		select {
		case t.addpending <- p:
			// loop will handle it
		case <-t.closing:
			ch <- errClosed
		}
		return ch
	}

메시지 처리를 Addpending. 시간 이전에 생성 된 UDP는 newUDP 방법이라고했다. 어떤 두 goroutine을 시작했다. 상기 루프 ()는 보류중인 메시지를 처리하는 데 사용된다.


	// loop runs in its own goroutine. it keeps track of
	// the refresh timer and the pending reply queue.
	func (t *udp) loop() {
		var (
			plist		= list.New()
			timeout	  = time.NewTimer(0)
			nextTimeout  *pending // head of plist when timeout was last reset
			contTimeouts = 0	  // number of continuous timeouts to do NTP checks
			ntpWarnTime  = time.Unix(0, 0)
		)
		<-timeout.C // ignore first timeout
		defer timeout.Stop()
	
		resetTimeout := func() {  
			//이 방법의 주요 기능은 메시지 큐 필요한 제한 시간 대기가 있는지 확인하는 것입니다. (있는 경우). 그때
			//첫 번째 웨이크 타임 아웃 시간 제한 설정.
			if plist.Front() == nil || nextTimeout == plist.Front().Value {
				return
			}
			// Start the timer so it fires when the next pending reply has expired.
			now := time.Now()
			for el := plist.Front(); el != nil; el = el.Next() {
				nextTimeout = el.Value.(*pending)
				if dist := nextTimeout.deadline.Sub(now); dist < 2*respTimeout {
					timeout.Reset(dist)
					return
				}
				// Remove pending replies whose deadline is too far in the
				// future. These can occur if the system clock jumped
				// backwards after the deadline was assigned.
				//미래에까지 마감의 메시지가, 다음 직접 제한 시간을 설정하면, 다음 제거.
				//이러한 상황은 시스템 시간을 변경하는 경우 발생할 수 있으며 치료하지 않으면 너무 오래 막혀으로 이어질 수 있습니다.
				nextTimeout.errc <- errClockWarp
				plist.Remove(el)
			}
			nextTimeout = nil
			timeout.Stop()
		}
	
		for {
			resetTimeout()  //가공 첫번째 타임 아웃.
	
			select {
			case <-t.closing:  //닫기 정보를받을 수 있습니다. 모든 시간 제한 큐 혼잡
				for el := plist.Front(); el != nil; el = el.Next() {
					el.Value.(*pending).errc <- errClosed
				}
				return
	
			case p := <-t.addpending:  //보류 기한 세트 추가
				p.deadline = time.Now().Add(respTimeout)
				plist.PushBack(p)
	
			case r := <-t.gotreply:  //보류 일치에 대한 응답을 수신
				var matched bool
				for el := plist.Front(); el != nil; el = el.Next() {
					p := el.Value.(*pending)
					if p.from == r.from && p.ptype == r.ptype { //당신 같은 사람에서 온 경우. 그리고 같은 유형
						matched = true
						// Remove the matcher if its callback indicates
						// that all replies have been received. This is
						// required for packet types that expect multiple
						// reply packets.
						if p.callback(r.data) { //경우 콜백 반환 값은 true입니다. 설명 출원이 완료되었습니다. 무기 호에 P.errc 쓰기. 출원 완료.
							p.errc <- nil
							plist.Remove(el)
						}
						// Reset the continuous timeout counter (time drift detection)
						contTimeouts = 0
					}
				}
				r.matched <- matched //일치하는 답변 쓰기
	
			case now := <-timeout.C:   //정보 처리 시간 제한
				nextTimeout = nil
	
				// Notify and remove callbacks whose deadline is in the past.
				for el := plist.Front(); el != nil; el = el.Next() {
					p := el.Value.(*pending)
					if now.After(p.deadline) || now.Equal(p.deadline) { //시간 제한 및 제거 정보 쓰기 초과하는 경우
						p.errc <- errTimeout
						plist.Remove(el)
						contTimeouts++
					}
				}
				// If we've accumulated too many timeouts, do an NTP time sync check
				if contTimeouts > ntpFailureThreshold {
					//연속 시간 제한 여러 번합니다. 시간이 동기화되지 않은 경우 다음을 참조하십시오. 동기화 및 NTP 서버입니다.
					if time.Since(ntpWarnTime) >= ntpWarningCooldown {
						ntpWarnTime = time.Now()
						go checkClockDrift()
					}
					contTimeouts = 0
				}
			}
		}
	}

위 보류 처리를 볼 수 있습니다. 그러나 루프는 ()도 gotreply의 종류를 접근. 이건 정말 readLoop입니다 ()이 goroutine이 발생.

	// readLoop runs in its own goroutine. it handles incoming UDP packets.
	func (t *udp) readLoop() {
		defer t.conn.Close()
		// Discovery packets are defined to be no larger than 1280 bytes.
		// Packets larger than this size will be cut at the end and treated
		// as invalid because their hash won't match.
		buf := make([]byte, 1280)
		for {
			nbytes, from, err := t.conn.ReadFromUDP(buf)
			if netutil.IsTemporaryError(err) {
				// Ignore temporary read errors.
				log.Debug("Temporary UDP read error", "err", err)
				continue
			} else if err != nil {
				// Shut down the loop for permament errors.
				log.Debug("UDP read error", "err", err)
				return
			}
			t.handlePacket(from, buf[:nbytes])
		}
	}

	func (t *udp) handlePacket(from *net.UDPAddr, buf []byte) error {
		packet, fromID, hash, err := decodePacket(buf)
		if err != nil {
			log.Debug("Bad discv4 packet", "addr", from, "err", err)
			return err
		}
		err = packet.handle(t, from, fromID, hash)
		log.Trace("<< "+packet.name(), "addr", from, "err", err)
		return err
	}
	
	func (req *ping) handle(t *udp, from *net.UDPAddr, fromID NodeID, mac []byte) error {
		if expired(req.Expiration) {
			return errExpired
		}
		t.send(from, pongPacket, &pong{
			To:		 makeEndpoint(from, req.From.TCP),
			ReplyTok:   mac,
			Expiration: uint64(time.Now().Add(expiration).Unix()),
		})
		if !t.handleReply(fromID, pingPacket, req) {
			// Note: we're ignoring the provided IP address right now
			go t.bond(true, fromID, from, req.From.TCP)
		}
		return nil
	}
	
	func (t *udp) handleReply(from NodeID, ptype byte, req packet) bool {
		matched := make(chan bool, 1)
		select {
		case t.gotreply <- reply{from, ptype, req, matched}:
			// loop will handle it
			return <-matched
		case <-t.closing:
			return false
		}
	}


위에서 실질적 UDP의 처리 흐름을 설명한다. 다음은 UDP에서 처리의 주요 사업이다. UDP 메인 송신 요청, 해당 요청 사람들은이 개 응답을 생성합니다 두 개의 요청에 해당하는 두 개의 전송받을 수 있습니다.

ping 요청, 당신은 탁구 응답에 대한 요청을 볼 수 있습니다. 그리고 돌아갑니다.

	// ping sends a ping message to the given node and waits for a reply.
	func (t *udp) ping(toid NodeID, toaddr *net.UDPAddr) error {
		// TODO: maybe check for ReplyTo field in callback to measure RTT
		errc := t.pending(toid, pongPacket, func(interface{}) bool { return true })
		t.send(toaddr, pingPacket, &ping{
			Version:	Version,
			From:	   t.ourEndpoint,
			To:		 makeEndpoint(toaddr, 0), // TODO: maybe use known TCP port from DB
			Expiration: uint64(time.Now().Add(expiration).Unix()),
		})
		return <-errc
	}

대답은 해당 ping 요청에 일치하지 않는 경우 탁구, 탁구 답변. 그런 다음 errUnsolicitedReply 예외를 반환합니다.

	func (req *pong) handle(t *udp, from *net.UDPAddr, fromID NodeID, mac []byte) error {
		if expired(req.Expiration) {
			return errExpired
		}
		if !t.handleReply(fromID, pongPacket, req) {
			return errUnsolicitedReply
		}
		return nil
	}

findnode 요청, 다음, findnode 요청을 보내 응답 노드 (k) 이웃을 기다립니다.

	// findnode sends a findnode request to the given node and waits until
	// the node has sent up to k neighbors.
	func (t *udp) findnode(toid NodeID, toaddr *net.UDPAddr, target NodeID) ([]*Node, error) {
		nodes := make([]*Node, 0, bucketSize)
		nreceived := 0
		errc := t.pending(toid, neighborsPacket, func(r interface{}) bool {
			reply := r.(*neighbors)
			for _, rn := range reply.Nodes {
				nreceived++
				n, err := t.nodeFromRPC(toaddr, rn)
				if err != nil {
					log.Trace("Invalid neighbor node received", "ip", rn.IP, "addr", toaddr, "err", err)
					continue
				}
				nodes = append(nodes, n)
			}
			return nreceived >= bucketSize
		})
		t.send(toaddr, findnodePacket, &findnode{
			Target:	 target,
			Expiration: uint64(time.Now().Add(expiration).Unix()),
		})
		err := <-errc
		return nodes, err
	}

이웃은 매우 간단 반응했다. 큐로 전송 된 응답 Gotreply. 일치는 findnode 요청을 찾을 수없는 경우. 반환 errUnsolicitedReply 오류

	func (req *neighbors) handle(t *udp, from *net.UDPAddr, fromID NodeID, mac []byte) error {
		if expired(req.Expiration) {
			return errExpired
		}
		if !t.handleReply(fromID, neighborsPacket, req) {
			return errUnsolicitedReply
		}
		return nil
	}



탁구 답을 보내는 다른 노드에서 보낸 ping 요청을 받았습니다. 보류에 일치가없는 경우 (결과가 없습니다 자신의 당사자가 요청 설명). 이 방법은 채권 노드 자신의 버킷 캐시를 추가라고합니다. (원리의이 부분은 내부 table.go에서 자세히 설명한다)

	func (req *ping) handle(t *udp, from *net.UDPAddr, fromID NodeID, mac []byte) error {
		if expired(req.Expiration) {
			return errExpired
		}
		t.send(from, pongPacket, &pong{
			To:		 makeEndpoint(from, req.From.TCP),
			ReplyTok:   mac,
			Expiration: uint64(time.Now().Add(expiration).Unix()),
		})
		if !t.handleReply(fromID, pingPacket, req) {
			// Note: we're ignoring the provided IP address right now
			go t.bond(true, fromID, from, req.From.TCP)
		}
		return nil
	}

Findnode 다른 사람이 보낸 요청을 받았습니다. 이 요청은 노드 k는 다시 원하는 목표 거리와 가까운 전송된다. 다음 참조 카탈로그 PDF 문서 알고리즘을 참조하십시오.

	
	func (req *findnode) handle(t *udp, from *net.UDPAddr, fromID NodeID, mac []byte) error {
		if expired(req.Expiration) {
			return errExpired
		}
		if t.db.node(fromID) == nil {
			// No bond exists, we don't process the packet. This prevents
			// an attack vector where the discovery protocol could be used
			// to amplify traffic in a DDOS attack. A malicious actor
			// would send a findnode request with the IP address and UDP
			// port of the target as the source address. The recipient of
			// the findnode packet would then send a neighbors packet
			// (which is a much bigger packet than findnode) to the victim.
			return errUnknownNode
		}
		target := crypto.Keccak256Hash(req.Target[:])
		t.mutex.Lock()
		//BucketSize 비슷한 거리와 목표 노드를 얻을. 이 방법은 table.go 안에 구현된다. 자세한 사항은 따를 것이다
		closest := t.closest(target, bucketSize).entries
		t.mutex.Unlock()
	
		p := neighbors{Expiration: uint64(time.Now().Add(expiration).Unix())}
		// Send neighbors in chunks with at most maxNeighbors per packet
		// to stay below the 1280 byte limit.
		for i, n := range closest {
			if netutil.CheckRelayIP(from.IP, n.IP) != nil {
				continue
			}
			p.Nodes = append(p.Nodes, nodeToRPC(n))
			if len(p.Nodes) == maxNeighbors || i == len(closest)-1 {
				t.send(from, neighborsPacket, &p)
				p.Nodes = p.Nodes[:0]
			}
		}
		return nil
	}


### UDP 정보 암호화 및 보안 문제
데이터가 평문으로 전송하지만, 데이터 무결성을 보장하기 위해 및 위조 할되지 않도록 때문에 민감한 데이터를 전송하지 않은 것을의 동의를 발견, 그래서 패킷 헤더 플러스 디지털 서명.

	
	func encodePacket(priv *ecdsa.PrivateKey, ptype byte, req interface{}) ([]byte, error) {
		b := new(bytes.Buffer)
		b.Write(headSpace)
		b.WriteByte(ptype)
		if err := rlp.Encode(b, req); err != nil {
			log.Error("Can't encode discv4 packet", "err", err)
			return nil, err
		}
		packet := b.Bytes()
		sig, err := crypto.Sign(crypto.Keccak256(packet[headSize:]), priv)
		if err != nil {
			log.Error("Can't sign discv4 packet", "err", err)
			return nil, err
		}
		copy(packet[macSize:], sig)
		// add the hash to the front. Note: this doesn't protect the
		// packet in any way. Our public key will be part of this hash in
		// The future.
		copy(packet, crypto.Keccak256(packet[macSize:]))
		return packet, nil
	}

	func decodePacket(buf []byte) (packet, NodeID, []byte, error) {
		if len(buf) < headSize+1 {
			return nil, NodeID{}, nil, errPacketTooSmall
		}
		hash, sig, sigdata := buf[:macSize], buf[macSize:headSize], buf[headSize:]
		shouldhash := crypto.Keccak256(buf[macSize:])
		if !bytes.Equal(hash, shouldhash) {
			return nil, NodeID{}, nil, errBadHash
		}
		fromID, err := recoverNodeID(crypto.Keccak256(buf[headSize:]), sig)
		if err != nil {
			return nil, NodeID{}, hash, err
		}
		var req packet
		switch ptype := sigdata[0]; ptype {
		case pingPacket:
			req = new(ping)
		case pongPacket:
			req = new(pong)
		case findnodePacket:
			req = new(findnode)
		case neighborsPacket:
			req = new(neighbors)
		default:
			return nil, fromID, hash, fmt.Errorf("unknown type: %d", ptype)
		}
		s := rlp.NewStream(bytes.NewReader(sigdata[1:]), 0)
		err = s.Decode(req)
		return req, fromID, hash, err
	}