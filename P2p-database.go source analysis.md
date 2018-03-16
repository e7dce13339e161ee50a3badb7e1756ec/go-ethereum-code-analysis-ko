P2P P2P 패키지는 일반적인 네트워크 프로토콜을 구현합니다. P2P 노드는 찾아 노드 상태를 유지 피어 연결을 설정하는 기능이 포함되어 있습니다. P2P 패키지는 일반적인 P2P 프로토콜을 구현합니다. 어떤 특정 프로토콜 (예 : 프로토콜 ETH. 위스퍼 프로토콜. 스웜 프로토콜) 특정 패킷 인터페이스 주입 P2P로 캡슐화된다. 그것은 특정 내부의 P2P 프로토콜의 달성에 포함되지 않습니다. 완료 유일한 것은 P2P 네트워크는해야한다.


## / discv5 노드 발견을 발견
현재 발견하여 패키지. discv5 최근에 개발 된 기능 또는 실험이지만, 기본적으로 일부 최적화 패키지를 발견 할 수 있습니다. 여기에 당신은 단지 코드 분석을 발견 할 것이다. 완전한 기능은 기본적인 소개를 할.


### database.go
이름에서 알 수 있듯이, 내부 문서 지속적 노드의 주요 성과 노드의 P2P 네트워크 노드 탐색 및 유지 보수가 상대적으로 다시 시작하는 시간에 시간을 보내고, 전 재 발견을 피하기 위해 때마다 상속 작업 할 수 있기 때문에. 그래서 지속적인 작업이 필요하다.

우리가 ethdb 코드와 트라이 코드를 분석하기 전에, 지속적인 작업 트라이은 leveldb를 사용합니다. 여기에 또한 leveldb을 사용했다. 그러나 P2P 메인 leveldb 인스턴스 블록 사슬의 leveldb 예는 동일하지 않다.

newNodeDB는 뷰의 파라미터에 따른 파일 기반 데이터베이스 또는 파일 기반 데이터베이스 경로를 연다.

	// newNodeDB creates a new node database for storing and retrieving infos about
	// known peers in the network. If no path is given, an in-memory, temporary
	// database is constructed.
	func newNodeDB(path string, version int, self NodeID) (*nodeDB, error) {
		if path == "" {
			return newMemoryNodeDB(self)
		}
		return newPersistentNodeDB(path, version, self)
	}
	// newMemoryNodeDB creates a new in-memory node database without a persistent
	// backend.
	func newMemoryNodeDB(self NodeID) (*nodeDB, error) {
		db, err := leveldb.Open(storage.NewMemStorage(), nil)
		if err != nil {
			return nil, err
		}
		return &nodeDB{
			lvl:  db,
			self: self,
			quit: make(chan struct{}),
		}, nil
	}
	
	// newPersistentNodeDB creates/opens a leveldb backed persistent node database,
	// also flushing its contents in case of a version mismatch.
	func newPersistentNodeDB(path string, version int, self NodeID) (*nodeDB, error) {
		opts := &opt.Options{OpenFilesCacheCapacity: 5}
		db, err := leveldb.OpenFile(path, opts)
		if _, iscorrupted := err.(*errors.ErrCorrupted); iscorrupted {
			db, err = leveldb.RecoverFile(path, nil)
		}
		if err != nil {
			return nil, err
		}
		// The nodes contained in the cache correspond to a certain protocol version.
		// Flush all nodes if the version doesn't match.
		currentVer := make([]byte, binary.MaxVarintLen64)
		currentVer = currentVer[:binary.PutVarint(currentVer, int64(version))]
		blob, err := db.Get(nodeDBVersionKey, nil)
		switch err {
		case leveldb.ErrNotFound:
			// Version not found (i.e. empty cache), insert it
			if err := db.Put(nodeDBVersionKey, currentVer, nil); err != nil {
				db.Close()
				return nil, err
			}
		case nil:
			// Version present, flush if different
			//다른 버전은 첫 번째를 다시 작성, 모든 데이터베이스 파일을 제거합니다.
			if !bytes.Equal(blob, currentVer) {
				db.Close()
				if err = os.RemoveAll(path); err != nil {
					return nil, err
				}
				return newPersistentNodeDB(path, version, self)
			}
		}
		return &nodeDB{
			lvl:  db,
			self: self,
			quit: make(chan struct{}),
		}, nil
	}


Node的存储，查询和删除

	// node retrieves a node with a given id from the database.
	func (db *nodeDB) node(id NodeID) *Node {
		blob, err := db.lvl.Get(makeKey(id, nodeDBDiscoverRoot), nil)
		if err != nil {
			return nil
		}
		node := new(Node)
		if err := rlp.DecodeBytes(blob, node); err != nil {
			log.Error("Failed to decode node RLP", "err", err)
			return nil
		}
		node.sha = crypto.Keccak256Hash(node.ID[:])
		return node
	}
	
	// updateNode inserts - potentially overwriting - a node into the peer database.
	func (db *nodeDB) updateNode(node *Node) error {
		blob, err := rlp.EncodeToBytes(node)
		if err != nil {
			return err
		}
		return db.lvl.Put(makeKey(node.ID, nodeDBDiscoverRoot), blob, nil)
	}
	
	// deleteNode deletes all information/keys associated with a node.
	func (db *nodeDB) deleteNode(id NodeID) error {
		deleter := db.lvl.NewIterator(util.BytesPrefix(makeKey(id, "")), nil)
		for deleter.Next() {
			if err := db.lvl.Delete(deleter.Key(), nil); err != nil {
				return err
			}
		}
		return nil
	}

노드 구조

	type Node struct {
		IP	   net.IP // len 4 for IPv4 or 16 for IPv6
		UDP, TCP uint16 // port numbers
		ID	   NodeID // the node's public key
		// This is a cached copy of sha3(ID) which is used for node
		// distance calculations. This is part of Node in order to make it
		// possible to write tests that need a node at a certain distance.
		// In those tests, the content of sha will not actually correspond
		// with ID.
		sha common.Hash
		// whether this node is currently being pinged in order to replace
		// it in a bucket
		contested bool
	}

노드 타임 아웃 처리


	// ensureExpirer is a small helper method ensuring that the data expiration
	// mechanism is running. If the expiration goroutine is already running, this
	// method simply returns.
	//ensureExpirer 방법은 동작 방법 expirer 보장하기 위해 사용. expirer가 이미 실행중인 경우,이 메소드는 직접 돌아갑니다.
	//이 방법의 목적은 데이터의 시작 부분에 성공적인 시작이 (폐기 씨앗 노드의 잠재적 인 유용성의 일부를 방지하기 위해) 초과 근무를 폐기 한 후 네트워크를 설정하는 것입니다.
	// The goal is to start the data evacuation only after the network successfully
	// bootstrapped itself (to prevent dumping potentially useful seed nodes). Since
	// it would require significant overhead to exactly trace the first successful
	// convergence, it's simpler to "ensure" the correct state when an appropriate
	// condition occurs (i.e. a successful bonding), and discard further events.
	func (db *nodeDB) ensureExpirer() {
		db.runner.Do(func() { go db.expirer() })
	}
	
	// expirer should be started in a go routine, and is responsible for looping ad
	// infinitum and dropping stale data from the database.
	func (db *nodeDB) expirer() {
		tick := time.Tick(nodeDBCleanupCycle)
		for {
			select {
			case <-tick:
				if err := db.expireNodes(); err != nil {
					log.Error("Failed to expire nodedb items", "err", err)
				}
	
			case <-db.quit:
				return
			}
		}
	}
	
	// expireNodes iterates over the database and deletes all nodes that have not
	// been seen (i.e. received a pong from) for some allotted time.
	//이 방법은 노드가 마지막 메시지를받은 경우,이 노드를 삭제, 지정된 값을 초과하는 모든 노드를 반복합니다.
	func (db *nodeDB) expireNodes() error {
		threshold := time.Now().Add(-nodeDBNodeExpiration)
	
		// Find discovered nodes that are older than the allowance
		it := db.lvl.NewIterator(nil, nil)
		defer it.Release()
	
		for it.Next() {
			// Skip the item if not a discovery node
			id, field := splitKey(it.Key())
			if field != nodeDBDiscoverRoot {
				continue
			}
			// Skip the node if not expired yet (and not self)
			if !bytes.Equal(id[:], db.self[:]) {
				if seen := db.lastPong(id); seen.After(threshold) {
					continue
				}
			}
			// Otherwise delete all associated information
			db.deleteNode(id)
		}
		return nil
	}


일부 상태 업데이트 기능

	// lastPing retrieves the time of the last ping packet send to a remote node,
	// requesting binding.
	func (db *nodeDB) lastPing(id NodeID) time.Time {
		return time.Unix(db.fetchInt64(makeKey(id, nodeDBDiscoverPing)), 0)
	}
	
	// updateLastPing updates the last time we tried contacting a remote node.
	func (db *nodeDB) updateLastPing(id NodeID, instance time.Time) error {
		return db.storeInt64(makeKey(id, nodeDBDiscoverPing), instance.Unix())
	}
	
	// lastPong retrieves the time of the last successful contact from remote node.
	func (db *nodeDB) lastPong(id NodeID) time.Time {
		return time.Unix(db.fetchInt64(makeKey(id, nodeDBDiscoverPong)), 0)
	}
	
	// updateLastPong updates the last time a remote node successfully contacted.
	func (db *nodeDB) updateLastPong(id NodeID, instance time.Time) error {
		return db.storeInt64(makeKey(id, nodeDBDiscoverPong), instance.Unix())
	}
	
	// findFails retrieves the number of findnode failures since bonding.
	func (db *nodeDB) findFails(id NodeID) int {
		return int(db.fetchInt64(makeKey(id, nodeDBDiscoverFindFails)))
	}
	
	// updateFindFails updates the number of findnode failures since bonding.
	func (db *nodeDB) updateFindFails(id NodeID, fails int) error {
		return db.storeInt64(makeKey(id, nodeDBDiscoverFindFails), int64(fails))
	}


적합한 종자 노드가 무작위로 내부 데이터베이스에서 선택


	// querySeeds retrieves random nodes to be used as potential seed nodes
	// for bootstrapping.
	func (db *nodeDB) querySeeds(n int, maxAge time.Duration) []*Node {
		var (
			now   = time.Now()
			nodes = make([]*Node, 0, n)
			it	= db.lvl.NewIterator(nil, nil)
			id	NodeID
		)
		defer it.Release()
	
	seek:
		for seeks := 0; len(nodes) < n && seeks < n*5; seeks++ {
			// Seek to a random entry. The first byte is incremented by a
			// random amount each time in order to increase the likelihood
			// of hitting all existing nodes in very small databases.
			ctr := id[0]
			rand.Read(id[:])
			id[0] = ctr + id[0]%16
			it.Seek(makeKey(id, nodeDBDiscoverRoot))
	
			n := nextNode(it)
			if n == nil {
				id[0] = 0
				continue seek // iterator exhausted
			}
			if n.ID == db.self {
				continue seek
			}
			if now.Sub(db.lastPong(n.ID)) > maxAge {
				continue seek
			}
			for i := range nodes {
				if nodes[i].ID == n.ID {
					continue seek // duplicate
				}
			}
			nodes = append(nodes, n)
		}
		return nodes
	}
	
	// reads the next node record from the iterator, skipping over other
	// database entries.
	func nextNode(it iterator.Iterator) *Node {
		for end := false; !end; end = !it.Next() {
			id, field := splitKey(it.Key())
			if field != nodeDBDiscoverRoot {
				continue
			}
			var n Node
			if err := rlp.DecodeBytes(it.Value(), &n); err != nil {
				log.Warn("Failed to decode node RLP", "id", id, "err", err)
				continue
			}
			return &n
		}
		return nil
	}



