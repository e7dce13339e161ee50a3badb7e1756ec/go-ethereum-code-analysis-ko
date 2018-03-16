내부의 P2P 코드에서. 좋은 네트워크 링크를 만들 대표 피어. 단일 링크에서 여러 프로토콜을 실행 할 수있다. 예를 들어, 이더넷 프로토콜 광장 (ETH). 계약의 떼. 또는 위스퍼 계약.

피어 구조

	type protoRW struct {
		Protocol
		in	 chan Msg		// receices read messages
		closed <-chan struct{} // receives when peer is shutting down
		wstart <-chan struct{} // receives when write may start
		werr   chan<- error	// for write results
		offset uint64
		w	  MsgWriter
	}

	// Protocol represents a P2P subprotocol implementation.
	type Protocol struct {
		// Name should contain the official protocol name,
		// often a three-letter word.
		Name string
	
		// Version should contain the version number of the protocol.
		Version uint
	
		// Length should contain the number of message codes used
		// by the protocol.
		Length uint64
	
		// Run is called in a new groutine when the protocol has been
		// negotiated with a peer. It should read and write messages from
		// rw. The Payload for each message must be fully consumed.
		//
		// The peer connection is closed when Start returns. It should return
		// any protocol-level error (such as an I/O error) that is
		// encountered.
		Run func(peer *Peer, rw MsgReadWriter) error
	
		// NodeInfo is an optional helper method to retrieve protocol specific metadata
		// about the host node.
		NodeInfo func() interface{}
	
		// PeerInfo is an optional helper method to retrieve protocol specific metadata
		// about a certain peer in the network. If an info retrieval function is set,
		// but returns nil, it is assumed that the protocol handshake is still running.
		PeerInfo func(id discover.NodeID) interface{}
	}

	// Peer represents a connected remote node.
	type Peer struct {
		rw	  *conn
		running map[string]*protoRW   //동작 계약
		log	 log.Logger
		created mclock.AbsTime
	
		wg	   sync.WaitGroup
		protoErr chan error
		closed   chan struct{}
		disc	 chan DiscReason
	
		// events receives message send / receive events if set
		events *event.Feed
	}

피어 생성, 현재의 경기에 따라이 발견 피어 지원 protomap

	func newPeer(conn *conn, protocols []Protocol) *Peer {
		protomap := matchProtocols(protocols, conn.caps, conn)
		p := &Peer{
			rw:	   conn,
			running:  protomap,
			created:  mclock.Now(),
			disc:	 make(chan DiscReason),
			protoErr: make(chan error, len(protomap)+1), // protocols + pingLoop
			closed:   make(chan struct{}),
			log:	  log.New("id", conn.id, "conn", conn.flags),
		}
		return p
	}

피어의 시작은,이 개 goroutine 스레드를 시작했다. 하나는 읽습니다. 하나는 ping을하는 것입니다.

	func (p *Peer) run() (remoteRequested bool, err error) {
		var (
			writeStart = make(chan struct{}, 1)  //때 파이프 라인 제어에 쓸 수 있습니다.
			writeErr   = make(chan error, 1)
			readErr	= make(chan error, 1)
			reason	 DiscReason // sent to the peer
		)
		p.wg.Add(2)
		go p.readLoop(readErr)
		go p.pingLoop()
	
		// Start all protocol handlers.
		writeStart <- struct{}{}
		//모든 프로토콜을 시작합니다.
		p.startProtocols(writeStart, writeErr)
	
		// Wait for an error or disconnect.
	loop:
		for {
			select {
			case err = <-writeErr:
				// A write finished. Allow the next write to start if
				// there was no error.
				if err != nil {
					reason = DiscNetworkError
					break loop
				}
				writeStart <- struct{}{}
			case err = <-readErr:
				if r, ok := err.(DiscReason); ok {
					remoteRequested = true
					reason = r
				} else {
					reason = DiscNetworkError
				}
				break loop
			case err = <-p.protoErr:
				reason = discReasonForError(err)
				break loop
			case err = <-p.disc:
				break loop
			}
		}
	
		close(p.closed)
		p.rw.close(reason)
		p.wg.Wait()
		return remoteRequested, err
	}

모든 프로토콜을 통해 루프 startProtocols 방법.

	func (p *Peer) startProtocols(writeStart <-chan struct{}, writeErr chan<- error) {
		p.wg.Add(len(p.running))
		for _, proto := range p.running {
			proto := proto
			proto.closed = p.closed
			proto.wstart = writeStart
			proto.werr = writeErr
			var rw MsgReadWriter = proto
			if p.events != nil {
				rw = newMsgEventer(rw, p.events, p.ID(), proto.Name)
			}
			p.log.Trace(fmt.Sprintf("Starting protocol %s/%d", proto.Name, proto.Version))
			//여기에서 우리는 각 프로토콜에 대한 goroutine을 열 같다. 그 실행 방법을 호출합니다.
			go func() {
				//proto.Run (p, RW)이 방법은 무한 루프이어야한다. 당신이 돌아 경우 오류가 발생 보여줍니다.
				err := proto.Run(p, rw)
				if err == nil {
					p.log.Trace(fmt.Sprintf("Protocol %s/%d returned", proto.Name, proto.Version))
					err = errProtocolReturned
				} else if err != io.EOF {
					p.log.Trace(fmt.Sprintf("Protocol %s/%d failed", proto.Name, proto.Version), "err", err)
				}
				p.protoErr <- err
				p.wg.Done()
			}()
		}
	}


돌아가서 readLoop 방법을 확인합니다. 이 방법은 무한 루프입니다. 전화가 메시지를 읽을 p.rw (실제로는 메시지 오브젝트의 종류에 따라.이어서 해당 프레임 처리 후, 상기 프로토콜 유형은 메시지가 내부에서 실행되는 경우 개체, 이전에 언급 frameRLPx RW 타입. 다음 상기 해당 프로토콜에 proto.in 큐를 보내.

	
	func (p *Peer) readLoop(errc chan<- error) {
		defer p.wg.Done()
		for {
			msg, err := p.rw.ReadMsg()
			if err != nil {
				errc <- err
				return
			}
			msg.ReceivedAt = time.Now()
			if err = p.handle(msg); err != nil {
				errc <- err
				return
			}
		}
	}
	func (p *Peer) handle(msg Msg) error {
		switch {
		case msg.Code == pingMsg:
			msg.Discard()
			go SendItems(p.rw, pongMsg)
		case msg.Code == discMsg:
			var reason [1]DiscReason
			// This is the last message. We don't need to discard or
			// check errors because, the connection will be closed after it.
			rlp.Decode(msg.Payload, &reason)
			return reason[0]
		case msg.Code < baseProtocolLength:
			// ignore other base protocol messages
			return msg.Discard()
		default:
			// it's a subprotocol message
			proto, err := p.getProto(msg.Code)
			if err != nil {
				return fmt.Errorf("msg code out of range: %v", msg.Code)
			}
			select {
			case proto.in <- msg:
				return nil
			case <-p.closed:
				return io.EOF
			}
		}
		return nil
	}

pingLoop에보기. 이 방법은 매우 간단합니다. 그것은 그것의 피어에 pingMsg 메시지를 전송 타이밍이다.
	
	func (p *Peer) pingLoop() {
		ping := time.NewTimer(pingInterval)
		defer p.wg.Done()
		defer ping.Stop()
		for {
			select {
			case <-ping.C:
				if err := SendItems(p.rw, pingMsg); err != nil {
					p.protoErr <- err
					return
				}
				ping.Reset(pingInterval)
			case <-p.closed:
				return
			}
		}
	}

마지막으로, protoRW에서 읽기를 살펴보고 방법을 쓰기. 당신은 읽기를 볼 때까지 블록을 작성할 수 있습니다.
	
	func (rw *protoRW) WriteMsg(msg Msg) (err error) {
		if msg.Code >= rw.Length {
			return newPeerError(errInvalidMsgCode, "not handled")
		}
		msg.Code += rw.offset
		select {
		case <-rw.wstart:  //피사체가 실행에 쓸 수까지. 이 제어 그것의 여러 스레드를 위해 그것을이다.
			err = rw.w.WriteMsg(msg)
			// Report write status back to Peer.run. It will initiate
			// shutdown if the error is non-nil and unblock the next write
			// otherwise. The calling protocol code should exit for errors
			// as well but we don't want to rely on that.
			rw.werr <- err
		case <-rw.closed:
			err = fmt.Errorf("shutting down")
		}
		return err
	}
	
	func (rw *protoRW) ReadMsg() (Msg, error) {
		select {
		case msg := <-rw.in:
			msg.Code -= rw.offset
			return msg, nil
		case <-rw.closed:
			return Msg{}, io.EOF
		}
	}
