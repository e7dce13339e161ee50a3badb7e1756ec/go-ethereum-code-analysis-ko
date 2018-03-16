RLPx 암호화 (RLPx 암호화)

소개 노드 탐색 프로토콜을 발견하기 전에 데이터 캐리어가 중요하지 않기 때문에, 그것은 실질적으로 일반 텍스트로 전송됩니다.

각 노드는 노드 탐색 들어, UDP 포트가 같은 두 개의 포트를 열어야합니다, TCP 포트는 데이터 트래픽을 전송하는 데 사용됩니다. 포트의 TCP 및 UDP 포트 번호는 동일합니다. 그래서 오랫동안 UDP 포트가 발견, 그것은 해당 포트에 TCP 연결이 될 것을 의미합니다.

RLPx 계약 암호화 프로세스의 TCP 연결을 정의합니다.

RLPx은 (전달 완전 보안) 간단한 용어를 사용했다. 두 개의 링크 위에 생성 된 임의의 개인 키를 생성, 공개 키는 임의의 개인 키를 얻을. 양측이 자신의 개인 키와 상대방의 공개 키를 통해 임의 동일한 공유 키 (공유 비밀)을 생성 할 수 있도록 다음 양측은, 자신의 공개 키를 교환한다. 대칭 암호화 알고리즘과 공유 키를 사용하여 후속 통신. 이다. 개인 키의 일일 하나가 손상되어있는 경우 (통신이 실행의 실종 후, 무작위로 생성 된 키 때문에) 이전 통신에 누출이 안전 후에는 단지 메시지의 보안에 영향을 미칠 것입니다.


## 앞으로 보안 (위키 백과에서 인용)
앞으로 보안 또는 앞으로 비밀 (영어 : 앞으로 비밀, 약어 : FS)를, 또한 때때로 보안 [1] 퍼펙트 (: 전달 완전 보안, 약어 : 영어 PFS)라고, 암호화는 보안 통신 프로토콜입니다 부동산, 마스터 키 장기간 사용을 참조하면 누설 지난 세션 키 누출로 이어질하지 않습니다. 과거는 미래의 노출을 수행에서 [2] 앞으로 암호 또는 키 위협에 대한 통신 보안을 보호 할 수 있습니다. 시스템 보안에 대한 프론트가있는 경우 [3], 시스템이 활성 공격이 경우에도 실수로 어떤 점에서 유출 된 비밀번호 나 키, 통신, 그것은 여전히 ​​안전 영향을받지 않습니다되어 과거에 완료 될 경우 다음 사항을 확인 할 수 있습니다 그래서.

### 디피 - 헬만 키 교환
디피 - 헬만 키 교환 (영어 : 디피 - 헬만 키 교환, 약식 DH)는 보안 프로토콜입니다. 이것은 전혀 다른 정보가 안전하지 않은 채널에서 생성되지 않는 중요한 전제 조건에서 당을 허용한다. 이 통신은 후속 통신에 대칭 키와 콘텐츠 키를 암호화 할 수있다. 공개 키 교환의 개념은 첫째 랄프 머클 (랄프 C.는 Merkle), 그리고 윗 필드 디피 (베일리 윗 필드 디피)와 마틴 헬맨 (마틴 에드워드 헬맨)에 의해이 키 교환 방식에 의해 제안되었다 1976 년에 출판. 헬만 - - 잉크 g 키 교환 (: 디피 - 헬만-는 Merkle 키 교환 영어) 마틴 Hellman 키 교환이 방법은 디피를 호출 할 필요가 주장했다.

- 디피 - 헬만 키 교환 동의어은 다음과 같습니다 :
- 디피 - 헬만 키 동의
- 디피 - 헬만 키 생성
- 지수 키 교환
- 디피 - 헬만 프로토콜

디피 있지만 - Hellman 키 교환 자체가 (인증 없음) 키 교환 프로토콜 익명이며, 많은 인증 프로토콜의 기초이며, 완전한 모델은 짧은 전방 전송 계층 보안을 제공하는 데 사용됩니다 보안.

#### 기술
디피 - 공통 채널을 통해 Hellman 교환 정보는 공유 비밀 (공유 비밀) 공공 안전 통신 채널을 만들 수 있습니다.
(알고리즘의 수학 부분 포함) 프로세스의 다음 설명 :
![image](picture/rlpx_1.png)

가장 단순한, 소수 (p)의 프로토콜과 정수의 곱셈 군의 사용을 제안하는 첫 번째 모듈로 N 원시적 루트 g. 다음 공연이 알고리즘은, 녹색 기밀 정보, 굵은 빨간색에 비밀 정보를 나타냅니다 :
![image](picture/rlpx_2.png)
![image](picture/rlpx_3.png)

## P2P / rlpx.go 소스 해석
이 파일은 RLPx 링크 프로토콜을 구현합니다.

다음과 같이 링크 연락처 일반적인 프로세스는 다음과 같습니다

1. doEncHandshake (),이 방법에 의해 키 교환을 완료하기 위해 암호화 된 채널 흐름을 만들. 실패하면, 다음 링크가 닫힙니다.
이따위 암호화 및 다른 동작들을 지원할지 여부를, 예컨대 쌍방의 프로토콜 버전과 기능간에 협상이 협정 2. doProtoHandshake () 메소드.


처리 링크이 두 후 설립 된 경우에도 마찬가지입니다. TCP는 스트리밍 프로토콜이기 때문이다. 모든 프로토콜은 프레임 RLPx을 정의합니다. 모든 데이터는 하나 rlpxFrame 하나로서 이해 될 수있다. 읽기 및 쓰기가 rlpx rlpxFrameRW 객체에 의해 처리된다.

### doEncHandshake
링크 개시제를 개시라고합니다. 링크는 수동적 인 수신자 수신기가 될 것입니다. 양쪽 모드에서의 처리의 흐름이 다르다. 핸드 쉐이크의 완료 후. 잠깐 생성하는. 대칭 암호화 키를 얻기 위해 이해 될 수있다. 그럼 newRLPXFrameRW 프레임 리더 만든다. 암호화 된 채널을 생성하는 방법.


	func (t *rlpx) doEncHandshake(prv *ecdsa.PrivateKey, dial *discover.Node) (discover.NodeID, error) {
		var (
			sec secrets
			err error
		)
		if dial == nil {
			sec, err = receiverEncHandshake(t.fd, prv, nil)
		} else {
			sec, err = initiatorEncHandshake(t.fd, prv, dial.ID, nil)
		}
		if err != nil {
			return discover.NodeID{}, err
		}
		t.wmu.Lock()
		t.rw = newRLPXFrameRW(t.fd, sec)
		t.wmu.Unlock()
		return sec.RemoteID, nil
	}

initiatorEncHandshake 먼저 링크의 개시의 작동을 확인합니다. 먼저 makeAuthMsg authMsg에 의해 만들어진. 그리고 네트워크를 통해 피어에 송신. 그런 다음 readHandshakeMsg을 통해 피어의 응답을 읽습니다. 마지막 호출은 공유 비밀 비밀을 만들 수 있습니다.

	// initiatorEncHandshake negotiates a session token on conn.
	// it should be called on the dialing side of the connection.
	//
	// prv is the local client's private key.
	func initiatorEncHandshake(conn io.ReadWriter, prv *ecdsa.PrivateKey, remoteID discover.NodeID, token []byte) (s secrets, err error) {
		h := &encHandshake{initiator: true, remoteID: remoteID}
		authMsg, err := h.makeAuthMsg(prv, token)
		if err != nil {
			return s, err
		}
		authPacket, err := sealEIP8(authMsg, h)
		if err != nil {
			return s, err
		}
		if _, err = conn.Write(authPacket); err != nil {
			return s, err
		}
	
		authRespMsg := new(authRespV4)
		authRespPacket, err := readHandshakeMsg(authRespMsg, encAuthRespLen, prv, conn)
		if err != nil {
			return s, err
		}
		if err := h.handleAuthResp(authRespMsg); err != nil {
			return s, err
		}
		return h.secrets(authPacket, authRespPacket)
	}

makeAuthMsg. 이 방법의 악수 메시지 개시를 작성합니다. 먼저, 상기 단말기의 공개 키는 종료 ID에 의해 획득 될 수있다. 연결의 공공 측면에 사람들이 알고 시작에 대한 그래서. 사람을 연결하기 위해하지만, 대중의 끝은 모르는 것이다.

	// makeAuthMsg creates the initiator handshake message.
	func (h *encHandshake) makeAuthMsg(prv *ecdsa.PrivateKey, token []byte) (*authMsgV4, error) {
		rpub, err := h.remoteID.Pubkey()
		if err != nil {
			return nil, fmt.Errorf("bad remoteID: %v", err)
		}
		h.remotePub = ecies.ImportECDSAPublic(rpub)
		// Generate random initiator nonce.
		//재생 공격을 방지하기 위해, 임의의 초기 값을 생성? 다중 연결을 방지하기 위해 또는 키를 추측하여?
		h.initNonce = make([]byte, shaLen)
		if _, err := rand.Read(h.initNonce); err != nil {
			return nil, err
		}
		// Generate random keypair to for ECDH.
		//임의의 비밀 키를 생성합니다
		h.randomPrivKey, err = ecies.GenerateKey(rand.Reader, crypto.S256(), nil)
		if err != nil {
			return nil, err
		}
	
		// Sign known message: static-shared-secret ^ nonce
		//이 장소는 정적 공유 비밀로 사용되어야한다. 자신의 개인 키와 공유 암호에 의해 생성의 다른 공개 키를 사용하여.
		token, err = h.staticSharedSecret(prv)
		if err != nil {
			return nil, err
		}
		//여기에서 나는 initNonce를 암호화하는 데 사용되는 공유 비밀을 이해합니다.
		signed := xor(token, h.initNonce)
		//임의의 개인 키는 메시지를 암호화합니다.
		signature, err := crypto.Sign(signed, h.randomPrivKey.ExportECDSA())
		if err != nil {
			return nil, err
		}
	
		msg := new(authMsgV4)
		copy(msg.Signature[:], signature)
		//여기에 발신자의 공개 키를 상대방에게 알려줍니다. 그래서 다른 하나는 자신의 개인 키를 사용하고 공개 키는 정적 공유 암호를 생성 할 수 있습니다.
		copy(msg.InitiatorPubkey[:], crypto.FromECDSAPub(&prv.PublicKey)[1:])
		copy(msg.Nonce[:], h.initNonce)
		msg.Version = 4
		return msg, nil
	}

	// staticSharedSecret returns the static shared secret, the result
	// of key agreement between the local and remote static node key.
	func (h *encHandshake) staticSharedSecret(prv *ecdsa.PrivateKey) ([]byte, error) {
		return ecies.ImportECDSA(prv).GenerateShared(h.remotePub, sskLen, sskLen)
	}

MSG의 RLP 부호화 팩 방법 sealEIP8 방법. 일부 데이터를 입력합니다. 그런 다음 데이터를 암호화하는 상대방의 공개 키를 사용합니다. 이것은 단지 개인 키 정보의 조각이 다른 한쪽을 복호화 할 수 있다는 것을 의미한다.

	func sealEIP8(msg interface{}, h *encHandshake) ([]byte, error) {
		buf := new(bytes.Buffer)
		if err := rlp.Encode(buf, msg); err != nil {
			return nil, err
		}
		// pad with random amount of data. the amount needs to be at least 100 bytes to make
		// the message distinguishable from pre-EIP-8 handshakes.
		pad := padSpace[:mrand.Intn(len(padSpace)-100)+100]
		buf.Write(pad)
		prefix := make([]byte, 2)
		binary.BigEndian.PutUint16(prefix, uint16(buf.Len()+eciesOverhead))
	
		enc, err := ecies.Encrypt(rand.Reader, h.remotePub, buf.Bytes(), nil, prefix)
		return append(prefix, enc...), err
	}

이 방법을 readHandshakeMsg하는 두 곳에서 호출됩니다. 하나는 initiatorEncHandshake입니다. 하나는 receiverEncHandshake입니다. 이 방법은 비교적 간단하다. 첫째, 포맷 디코딩을 사용해보십시오. 하지에 서로에 대한합니다. 그것은 호환성 설정해야합니다. 기본적으로 디코딩하기 위해 자신의 개인 키를 사용하여 다음 통화 RLP 구조로 디코딩. 구조는 가장 중요한 공공 임의 피어 authRespV4, 설명한다. 양측은 자신의 개인 키와 임의 종료 공개 키에 의해 같은 공유 암호를 얻을 수 있습니다. 그리고이 공유 비밀을 제 3 자 범위입니다.

	
	// RLPx v4 handshake response (defined in EIP-8).
	type authRespV4 struct {
		RandomPubkey [pubLen]byte
		Nonce		[shaLen]byte
		Version	  uint
	
		// Ignore additional fields (forward-compatibility)
		Rest []rlp.RawValue `rlp:"tail"`
	}


	func readHandshakeMsg(msg plainDecoder, plainSize int, prv *ecdsa.PrivateKey, r io.Reader) ([]byte, error) {
		buf := make([]byte, plainSize)
		if _, err := io.ReadFull(r, buf); err != nil {
			return buf, err
		}
		// Attempt decoding pre-EIP-8 "plain" format.
		key := ecies.ImportECDSA(prv)
		if dec, err := key.Decrypt(rand.Reader, buf, nil, nil); err == nil {
			msg.decodePlain(dec)
			return buf, nil
		}
		// Could be EIP-8 format, try that.
		prefix := buf[:2]
		size := binary.BigEndian.Uint16(prefix)
		if size < uint16(plainSize) {
			return buf, fmt.Errorf("size underflow, need at least %d bytes", plainSize)
		}
		buf = append(buf, make([]byte, size-uint16(plainSize)+2)...)
		if _, err := io.ReadFull(r, buf[plainSize:]); err != nil {
			return buf, err
		}
		dec, err := key.Decrypt(rand.Reader, buf[2:], nil, prefix)
		if err != nil {
			return buf, err
		}
		// Can't use rlp.DecodeBytes here because it rejects
		// trailing data (forward-compatibility).
		s := rlp.NewStream(bytes.NewReader(dec), 0)
		return buf, s.Decode(msg)
	}



handleAuthResp이 방법은 매우 간단합니다.

	func (h *encHandshake) handleAuthResp(msg *authRespV4) (err error) {
		h.respNonce = msg.Nonce[:]
		h.remoteRandomPub, err = importPublicKey(msg.RandomPubkey[:])
		return err
	}

악수 한 후 호출 할 마지막 비밀 기능이 완료됩니다. 공유 비밀을 생성 할 수있는 피어의 자신의 임의 개인 및 공개 키으로 공유 비밀 (이는 현재 링크에 존재하는) 순간이다. 어느 날 개인 키가 손상 그래서 때. 전에 소식은 안전했다.

	// secrets is called after the handshake is completed.
	// It extracts the connection secrets from the handshake values.
	func (h *encHandshake) secrets(auth, authResp []byte) (secrets, error) {
		ecdheSecret, err := h.randomPrivKey.GenerateShared(h.remoteRandomPub, sskLen, sskLen)
		if err != nil {
			return secrets{}, err
		}
	
		// derive base secrets from ephemeral key agreement
		sharedSecret := crypto.Keccak256(ecdheSecret, crypto.Keccak256(h.respNonce, h.initNonce))
		aesSecret := crypto.Keccak256(ecdheSecret, sharedSecret)
		//사실이 MAC은 ecdheSecret이 공유 비밀을 보호합니다. respNonce과 세 값 initNonce
		s := secrets{
			RemoteID: h.remoteID,
			AES:	  aesSecret,
			MAC:	  crypto.Keccak256(ecdheSecret, aesSecret),
		}
	
		// setup sha3 instances for the MACs
		mac1 := sha3.NewKeccak256()
		mac1.Write(xor(s.MAC, h.respNonce))
		mac1.Write(auth)
		mac2 := sha3.NewKeccak256()
		mac2.Write(xor(s.MAC, h.initNonce))
		mac2.Write(authResp)
		//수신 된 MAC 값을 확인하는 각 패킷은 계산의 결과를 만족한다. 하지 않으면 것은 문제가 있음을 나타냅니다.
		if h.initiator {
			s.EgressMAC, s.IngressMAC = mac1, mac2
		} else {
			s.EgressMAC, s.IngressMAC = mac2, mac1
		}
	
		return s, nil
	}


요약 receiverEncHandshake 실질적으로 동일한 기능 및 initiatorEncHandshake. 그러나 순서는 다소 다르다.
	
	// receiverEncHandshake negotiates a session token on conn.
	// it should be called on the listening side of the connection.
	//
	// prv is the local client's private key.
	// token is the token from a previous session with this node.
	func receiverEncHandshake(conn io.ReadWriter, prv *ecdsa.PrivateKey, token []byte) (s secrets, err error) {
		authMsg := new(authMsgV4)
		authPacket, err := readHandshakeMsg(authMsg, encAuthMsgLen, prv, conn)
		if err != nil {
			return s, err
		}
		h := new(encHandshake)
		if err := h.handleAuthMsg(authMsg, prv); err != nil {
			return s, err
		}
	
		authRespMsg, err := h.makeAuthResp()
		if err != nil {
			return s, err
		}
		var authRespPacket []byte
		if authMsg.gotPlain {
			authRespPacket, err = authRespMsg.sealPlain(h)
		} else {
			authRespPacket, err = sealEIP8(authRespMsg, h)
		}
		if err != nil {
			return s, err
		}
		if _, err = conn.Write(authRespPacket); err != nil {
			return s, err
		}
		return h.secrets(authPacket, authRespPacket)
	}

### doProtocolHandshake
이 방법은 비교적 간단하고, 암호화 채널이 생성되어있다. 우리는 여기서 단지 암호화를 사용하고 종료 할 것인지 여부를 뭐 이따위로 합의 참조하십시오.

	// doEncHandshake runs the protocol handshake using authenticated
	// messages. the protocol handshake is the first authenticated message
	// and also verifies whether the encryption handshake 'worked' and the
	// remote side actually provided the right public key.
	func (t *rlpx) doProtoHandshake(our *protoHandshake) (their *protoHandshake, err error) {
		// Writing our handshake happens concurrently, we prefer
		// returning the handshake read error. If the remote side
		// disconnects us early with a valid reason, we should return it
		// as the error so it can be tracked elsewhere.
		werr := make(chan error, 1)
		go func() { werr <- Send(t.rw, handshakeMsg, our) }()
		if their, err = readProtocolHandshake(t.rw, our); err != nil {
			<-werr // make sure the write terminates too
			return nil, err
		}
		if err := <-werr; err != nil {
			return nil, fmt.Errorf("write error: %v", err)
		}
		// If the protocol version supports Snappy encoding, upgrade immediately
		t.rw.snappy = their.Version >= snappyProtocolVersion
	
		return their, nil
	}


### 데이터 rlpxFrameRW 프레이밍
데이터는 주로 rlpxFrameRW 클래스에 의해 수행 프레임.


	// rlpxFrameRW implements a simplified version of RLPx framing.
	// chunked messages are not supported and all headers are equal to
	// zeroHeader.
	//
	// rlpxFrameRW is not safe for concurrent use from multiple goroutines.
	type rlpxFrameRW struct {
		conn io.ReadWriter
		enc  cipher.Stream
		dec  cipher.Stream
	
		macCipher  cipher.Block
		egressMAC  hash.Hash
		ingressMAC hash.Hash
	
		snappy bool
	}

후 우리는 두 방향 핸드 셰이크를 완료했습니다. 전화 newRLPXFrameRW 방법은 개체를 만듭니다.

	t.rw = newRLPXFrameRW(t.fd, sec)

그리고하면 ReadMsg WriteMsg 방법을 제공한다. 직접 호출 rlpxFrameRW하면 ReadMsg 및 WriteMsg 이러한 두 가지 방법


	func (t *rlpx) ReadMsg() (Msg, error) {
		t.rmu.Lock()
		defer t.rmu.Unlock()
		t.fd.SetReadDeadline(time.Now().Add(frameReadTimeout))
		return t.rw.ReadMsg()
	}
	func (t *rlpx) WriteMsg(msg Msg) error {
		t.wmu.Lock()
		defer t.wmu.Unlock()
		t.fd.SetWriteDeadline(time.Now().Add(frameWriteTimeout))
		return t.rw.WriteMsg(msg)
	}

WriteMsg

	func (rw *rlpxFrameRW) WriteMsg(msg Msg) error {
		ptype, _ := rlp.EncodeToBytes(msg.Code)
	
		// if snappy is enabled, compress message now
		if rw.snappy {
			if msg.Size > maxUint24 {
				return errPlainMessageTooLarge
			}
			payload, _ := ioutil.ReadAll(msg.Payload)
			payload = snappy.Encode(nil, payload)
	
			msg.Payload = bytes.NewReader(payload)
			msg.Size = uint32(len(payload))
		}
		// write header
		headbuf := make([]byte, 32)
		fsize := uint32(len(ptype)) + msg.Size
		if fsize > maxUint24 {
			return errors.New("message size overflows uint24")
		}
		putInt24(fsize, headbuf) // TODO: check overflow
		copy(headbuf[3:], zeroHeader)
		rw.enc.XORKeyStream(headbuf[:16], headbuf[:16]) // first half is now encrypted
	
		// write header MAC
		copy(headbuf[16:], updateMAC(rw.egressMAC, rw.macCipher, headbuf[:16]))
		if _, err := rw.conn.Write(headbuf); err != nil {
			return err
		}
	
		// write encrypted frame, updating the egress MAC hash with
		// the data written to conn.
		tee := cipher.StreamWriter{S: rw.enc, W: io.MultiWriter(rw.conn, rw.egressMAC)}
		if _, err := tee.Write(ptype); err != nil {
			return err
		}
		if _, err := io.Copy(tee, msg.Payload); err != nil {
			return err
		}
		if padding := fsize % 16; padding > 0 {
			if _, err := tee.Write(zero16[:16-padding]); err != nil {
				return err
			}
		}
	
		// write frame MAC. egress MAC hash is up to date because
		// frame content was written to it as well.
		fmacseed := rw.egressMAC.Sum(nil)
		mac := updateMAC(rw.egressMAC, rw.macCipher, fmacseed)
		_, err := rw.conn.Write(mac)
		return err
	}

ReadMsg

	func (rw *rlpxFrameRW) ReadMsg() (msg Msg, err error) {
		// read the header
		headbuf := make([]byte, 32)
		if _, err := io.ReadFull(rw.conn, headbuf); err != nil {
			return msg, err
		}
		// verify header mac
		shouldMAC := updateMAC(rw.ingressMAC, rw.macCipher, headbuf[:16])
		if !hmac.Equal(shouldMAC, headbuf[16:]) {
			return msg, errors.New("bad header MAC")
		}
		rw.dec.XORKeyStream(headbuf[:16], headbuf[:16]) // first half is now decrypted
		fsize := readInt24(headbuf)
		// ignore protocol type for now
	
		// read the frame content
		var rsize = fsize // frame size rounded up to 16 byte boundary
		if padding := fsize % 16; padding > 0 {
			rsize += 16 - padding
		}
		framebuf := make([]byte, rsize)
		if _, err := io.ReadFull(rw.conn, framebuf); err != nil {
			return msg, err
		}
	
		// read and validate frame MAC. we can re-use headbuf for that.
		rw.ingressMAC.Write(framebuf)
		fmacseed := rw.ingressMAC.Sum(nil)
		if _, err := io.ReadFull(rw.conn, headbuf[:16]); err != nil {
			return msg, err
		}
		shouldMAC = updateMAC(rw.ingressMAC, rw.macCipher, fmacseed)
		if !hmac.Equal(shouldMAC, headbuf[:16]) {
			return msg, errors.New("bad frame MAC")
		}
	
		// decrypt frame content
		rw.dec.XORKeyStream(framebuf, framebuf)
	
		// decode message code
		content := bytes.NewReader(framebuf[:fsize])
		if err := rlp.Decode(content, &msg.Code); err != nil {
			return msg, err
		}
		msg.Size = uint32(content.Len())
		msg.Payload = content
	
		// if snappy is enabled, verify and decompress message
		if rw.snappy {
			payload, err := ioutil.ReadAll(msg.Payload)
			if err != nil {
				return msg, err
			}
			size, err := snappy.DecodedLen(payload)
			if err != nil {
				return msg, err
			}
			if size > int(maxUint24) {
				return msg, errPlainMessageTooLarge
			}
			payload, err = snappy.Decode(nil, payload)
			if err != nil {
				return msg, err
			}
			msg.Size, msg.Payload = uint32(size), bytes.NewReader(payload)
		}
		return msg, nil
	}

프레임 구조

	  normal = not chunked
	  chunked-0 = First frame of a multi-frame packet
	  chunked-n = Subsequent frames for multi-frame packet
	  || is concatenate
	  ^ is xor
	
	Single-frame packet:
	header || header-mac || frame || frame-mac
	
	Multi-frame packet:
	header || header-mac || frame-0 ||
	[ header || header-mac || frame-n || ... || ]
	header || header-mac || frame-last || frame-mac
	
	header: frame-size || header-data || padding
	frame-size: 3-byte integer size of frame, big endian encoded (excludes padding)
	header-data:
		normal: rlp.list(protocol-type[, context-id])
		chunked-0: rlp.list(protocol-type, context-id, total-packet-size)
		chunked-n: rlp.list(protocol-type, context-id)
		values:
			protocol-type: < 2**16
			context-id: < 2**16 (optional for normal frames)
			total-packet-size: < 2**32
	padding: zero-fill to 16-byte boundary
	
	header-mac: right128 of egress-mac.update(aes(mac-secret,egress-mac) ^ header-ciphertext).digest
	
	frame:
		normal: rlp(packet-type) [|| rlp(packet-data)] || padding
		chunked-0: rlp(packet-type) || rlp(packet-data...)
		chunked-n: rlp(...packet-data) || padding
	padding: zero-fill to 16-byte boundary (only necessary for last frame)
	
	frame-mac: right128 of egress-mac.update(aes(mac-secret,egress-mac) ^ right128(egress-mac.update(frame-ciphertext).digest))
	
	egress-mac: h256, continuously updated with egress-bytes*
	ingress-mac: h256, continuously updated with ingress-bytes*


때문에 여기에 대한 분석은 매우 철저하지 그래서 나는 매우 익숙하지 오전 암호화 및 복호화 알고리즘. 그것은 단지 과정의 대략적인 분석이다. 확인되지 않은 많은 세부 사항이있다.