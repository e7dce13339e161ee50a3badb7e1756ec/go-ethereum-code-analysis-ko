이동 - 에테 Google 오픈 소스 levelDB 키 값 파일 데이터베이스에 저장된 모든 데이터는 전체 블록 사슬에 대한 모든 데이터는 파일 크기에 의한 데이터베이스 levelDB, levelDB 지원 절삭 기능 서브 - 파일에 저장되어 있으므로 참조 데이터 블록 사슬 사실,이 문서들은 동일한 levelDB 작은 예이며, 작은 파일이다. 다음은 간단한 모양 levelDB 패키지 코드를 이동합니다.

기능의 공식 웹 사이트를 levelDB

** 특징 ** :

- 키 값은 상기 어레이의 임의의 바이트 길이이고;
- 엔트리 (즉, 기록 KV)이 사전에 기억 된 기본 키 시퀀스에 따라 물론, 개발자는 이러한 종류의 기능을 재정의 할 수있다;
- 기본 사용자 인터페이스는 제공 : (), () 일괄 가져 오기) (삭제) (넣어;
- 지원 배치 동작은 원자 동작 수행;
- 당신은 데이터를 파노라마 샷 (스냅 샷)을 생성하고, 스냅 샷의 데이터를 찾을 수 있습니다;
- 당신이 (또는 뒤로) (암시 적 반복 스냅 샷을 생성합니다)에 반복자 데이터를 통과하기 전에;
- 자동 이따위 압축 데이터;
- 휴대;

** ** 제한 :

- 비 관계형 데이터 모델 (NoSQL의)는 SQL 문을 지원하지 않습니다, 인덱스를 지원하지 않습니다;
- 특정 데이터베이스에 액세스하는 하나의 프로세스 만이 가능;
- 기본 제공되지-에 C / S 아키텍처, 개발자는 서버 LevelDB 자신의 패키지 라이브러리를 사용할 수 있습니다;


소스 디렉토리 위치를 에테 리움 / ethdb 디렉토리. 코드는 다음과 같은 세 개의 파일로 나누어, 비교적 간단하다

- database.go			  패키지 코드 levelDB
- memory_database.go	   메모리 기반 데이터베이스를 테스트하기 위해, 테스트 목적으로 만 파일로 유지 될 수 없다
- interface.go			   이 인터페이스 데이터베이스를 정의
- database_test.go		   테스트 케이스

## interface.go
SQL의 leveldb를 지원하지 않는 등, 삭제했습니다, 가져 오기, 넣고, 다음 코드는 실질적으로 데이터베이스의 기본 동작을 키 값 정의를 참조하십시오, 기본 구조는지도 내부의 데이터로 이해 될 수있다.

	package ethdb
	const IdealBatchSize = 100 * 1024
	
	// Putter wraps the database write operation supported by both batches and regular databases.
	//퍼터 인터페이스는 일괄 작업 및 정상 작동을 쓸 수있는 인터페이스를 정의
	type Putter interface {
		Put(key []byte, value []byte) error
	}
	
	// Database wraps all database operations. All methods are safe for concurrent use.
	//데이터베이스 인터페이스는 모든 데이터베이스 작업을 정의하고, 모든 메소드는 멀티 스레드로부터 안전합니다.
	type Database interface {
		Putter
		Get(key []byte) ([]byte, error)
		Has(key []byte) (bool, error)
		Delete(key []byte) error
		Close()
		NewBatch() Batch
	}
	
	// Batch is a write-only database that commits changes to its host database
	// when Write is called. Batch cannot be used concurrently.
	//일괄 운영자 인터페이스, 당신은 동시에 여러 스레드를 사용할 수 없습니다, 쓰기 메서드를 호출 할 때, 데이터베이스가 작성된 변경 사항을 적용합니다.
	type Batch interface {
		Putter
		ValueSize() int // amount of data in the batch
		Write() error
	}

## memory_database.g
이것은 본질적으로지도 메모리를 구성 캡슐화합니다. 그런 다음 자원을 보호하기 위해 멀티 스레딩에 잠금을 사용합니다.

	type MemDatabase struct {
		db   map[string][]byte
		lock sync.RWMutex
	}
	
	func NewMemDatabase() (*MemDatabase, error) {
		return &MemDatabase{
			db: make(map[string][]byte),
		}, nil
	}
	
	func (db *MemDatabase) Put(key []byte, value []byte) error {
		db.lock.Lock()
		defer db.lock.Unlock()
		db.db[string(key)] = common.CopyBytes(value)
		return nil
	}
	func (db *MemDatabase) Has(key []byte) (bool, error) {
		db.lock.RLock()
		defer db.lock.RUnlock()
	
		_, ok := db.db[string(key)]
		return ok, nil
	}

그런 다음 일괄 작업. 그것은 비교적 간단하다,보고는 이해할 것이다.
	
	
	type kv struct{ k, v []byte }
	type memBatch struct {
		db	 *MemDatabase
		writes []kv
		size   int
	}
	func (b *memBatch) Put(key, value []byte) error {
		b.writes = append(b.writes, kv{common.CopyBytes(key), common.CopyBytes(value)})
		b.size += len(value)
		return nil
	}
	func (b *memBatch) Write() error {
		b.db.lock.Lock()
		defer b.db.lock.Unlock()
	
		for _, kv := range b.writes {
			b.db.db[string(kv.k)] = kv.v
		}
		return nil
	}


##database.go
이 에테 리움 클라이언트가 사용하는 실제 코드는 levelDB 인터페이스를 캡슐화합니다.

	
	import (
		"strconv"
		"strings"
		"sync"
		"time"
	
		"github.com/ethereum/go-ethereum/log"
		"github.com/ethereum/go-ethereum/metrics"
		"github.com/syndtr/goleveldb/leveldb"
		"github.com/syndtr/goleveldb/leveldb/errors"
		"github.com/syndtr/goleveldb/leveldb/filter"
		"github.com/syndtr/goleveldb/leveldb/iterator"
		"github.com/syndtr/goleveldb/leveldb/opt"
		gometrics "github.com/rcrowley/go-metrics"
	)

使用了github.com/syndtr/goleveldb/leveldb的leveldb的封装，所以一些使用的文档可以在那里找到。可以看到，数据结构主要增加了很多的Mertrics用来记录数据库的使用情况，增加了quitChan用来处理停止时候的一些情况，这个后面会分析。如果下面代码可能有疑问的地方应该再Filter:			 filter.NewBloomFilter (10)이 일시적 우려하지 않을 수 있습니다, 이것은 그들을 무시하는 옵션의 성능을 최적화하는 데 사용 levelDB입니다.

	
	type LDBDatabase struct {
		fn string	  // filename for reporting
		db *leveldb.DB // LevelDB instance
	
		getTimer	   gometrics.Timer // Timer for measuring the database get request counts and latencies
		putTimer	   gometrics.Timer // Timer for measuring the database put request counts and latencies
		...metrics 
	
		quitLock sync.Mutex	  // Mutex protecting the quit channel access
		quitChan chan chan error // Quit channel to stop the metrics collection before closing the database
	
		log log.Logger // Contextual logger tracking the database path
	}
	
	// NewLDBDatabase returns a LevelDB wrapped object.
	func NewLDBDatabase(file string, cache int, handles int) (*LDBDatabase, error) {
		logger := log.New("database", file)
		// Ensure we have some minimal caching and file guarantees
		if cache < 16 {
			cache = 16
		}
		if handles < 16 {
			handles = 16
		}
		logger.Info("Allocated cache and file handles", "cache", cache, "handles", handles)
		// Open the db and recover any potential corruptions
		db, err := leveldb.OpenFile(file, &opt.Options{
			OpenFilesCacheCapacity: handles,
			BlockCacheCapacity:	 cache / 2 * opt.MiB,
			WriteBuffer:			cache / 4 * opt.MiB, // Two of these are used internally
			Filter:				 filter.NewBloomFilter(10),
		})
		if _, corrupted := err.(*errors.ErrCorrupted); corrupted {
			db, err = leveldb.RecoverFile(file, nil)
		}
		// (Re)check for errors and abort if opening of the db failed
		if err != nil {
			return nil, err
		}
		return &LDBDatabase{
			fn:  file,
			db:  db,
			log: logger,
		}, nil
	}


넣고 github.com/syndtr/goleveldb/leveldb 패키지 후 코드가 여러 스레드가 동시에 액세스 지원하기 때문에 다음과 같은 코드를 보호하기 위해 잠금을 사용하지 않는 것입니다, 그래서, 다음 코드를 살펴 있으며,이 메모를 취할 수 있습니다. 대부분의 코드는 매우 상세하게 설명하지 직접 전화 leveldb 패키지이있다. 더 흥미로운 장소 메트릭 코드가있다.

	// Put puts the given key / value to the queue
	func (db *LDBDatabase) Put(key []byte, value []byte) error {
		// Measure the database put latency, if requested
		if db.putTimer != nil {
			defer db.putTimer.UpdateSince(time.Now())
		}
		// Generate the data to write to disk, update the meter and write
		//value = rle.Compress(value)
	
		if db.writeMeter != nil {
			db.writeMeter.Mark(int64(len(value)))
		}
		return db.db.Put(key, value, nil)
	}
	
	func (db *LDBDatabase) Has(key []byte) (bool, error) {
		return db.db.Has(key, nil)
	}

### 통계 처리
Mertrics 내부 초기화를 많이 NewLDBDatabase을 만들고,하지 않을 때 전에이 시간의 Mertrics 전무이다. 미터 방법에 초기화 Mertrics. 외부는 Mertrics의 다양한 생성, (특히, Merter를 만드는 방법, 업 팔로우 분석됩니다 미터 항목을) 접두사 인수를 전달, 다음 quitChan을 만듭니다. 마지막으로 스레드가 방법을 db.meter 호출 시작했다.
	
	// Meter configures the database metrics collectors and
	func (db *LDBDatabase) Meter(prefix string) {
		// Short circuit metering if the metrics system is disabled
		if !metrics.Enabled {
			return
		}
		// Initialize all the metrics collector at the requested prefix
		db.getTimer = metrics.NewTimer(prefix + "user/gets")
		db.putTimer = metrics.NewTimer(prefix + "user/puts")
		db.delTimer = metrics.NewTimer(prefix + "user/dels")
		db.missMeter = metrics.NewMeter(prefix + "user/misses")
		db.readMeter = metrics.NewMeter(prefix + "user/reads")
		db.writeMeter = metrics.NewMeter(prefix + "user/writes")
		db.compTimeMeter = metrics.NewMeter(prefix + "compact/time")
		db.compReadMeter = metrics.NewMeter(prefix + "compact/input")
		db.compWriteMeter = metrics.NewMeter(prefix + "compact/output")
	
		// Create a quit channel for the periodic collector and run it
		db.quitLock.Lock()
		db.quitChan = make(chan chan error)
		db.quitLock.Unlock()
	
		go db.meter(3 * time.Second)
	}

이 방법은 매 3 초 leveldb 내부 카운터를 소요하고 측정 서브 시스템에 게시. quitChan는 종료 신호를 수신 할 때까지이 방법은 무한 루프이다.

	// meter periodically retrieves internal leveldb counters and reports them to
	// the metrics subsystem.
	// This is how a stats table look like (currently):
	//다음은 우리가 db.db.GetProperty ( &quot;leveldb.stats&quot;)를 호출 코멘트 다음 코드의 문자열이 문자열을 구문 분석 할 필요가되고 미터에 정보를 기록 반환합니다.

	//   Compactions
	//	Level |   Tables   |	Size(MB)   |	Time(sec)  |	Read(MB)   |   Write(MB)
	//   -------+------------+---------------+---------------+---------------+---------------
	//	  0   |		  0 |	   0.00000 |	   1.27969 |	   0.00000 |	  12.31098
	//	  1   |		 85 |	 109.27913 |	  28.09293 |	 213.92493 |	 214.26294
	//	  2   |		523 |	1000.37159 |	   7.26059 |	  66.86342 |	  66.77884
	//	  3   |		570 |	1113.18458 |	   0.00000 |	   0.00000 |	   0.00000
	
	func (db *LDBDatabase) meter(refresh time.Duration) {
		// Create the counters to store current and previous values
		counters := make([][]float64, 2)
		for i := 0; i < 2; i++ {
			counters[i] = make([]float64, 3)
		}
		// Iterate ad infinitum and collect the stats
		for i := 1; ; i++ {
			// Retrieve the database stats
			stats, err := db.db.GetProperty("leveldb.stats")
			if err != nil {
				db.log.Error("Failed to read database stats", "err", err)
				return
			}
			// Find the compaction table, skip the header
			lines := strings.Split(stats, "\n")
			for len(lines) > 0 && strings.TrimSpace(lines[0]) != "Compactions" {
				lines = lines[1:]
			}
			if len(lines) <= 3 {
				db.log.Error("Compaction table not found")
				return
			}
			lines = lines[3:]
	
			// Iterate over all the table rows, and accumulate the entries
			for j := 0; j < len(counters[i%2]); j++ {
				counters[i%2][j] = 0
			}
			for _, line := range lines {
				parts := strings.Split(line, "|")
				if len(parts) != 6 {
					break
				}
				for idx, counter := range parts[3:] {
					value, err := strconv.ParseFloat(strings.TrimSpace(counter), 64)
					if err != nil {
						db.log.Error("Compaction entry parsing failed", "err", err)
						return
					}
					counters[i%2][idx] += value
				}
			}
			// Update all the requested meters
			if db.compTimeMeter != nil {
				db.compTimeMeter.Mark(int64((counters[i%2][0] - counters[(i-1)%2][0]) * 1000 * 1000 * 1000))
			}
			if db.compReadMeter != nil {
				db.compReadMeter.Mark(int64((counters[i%2][1] - counters[(i-1)%2][1]) * 1024 * 1024))
			}
			if db.compWriteMeter != nil {
				db.compWriteMeter.Mark(int64((counters[i%2][2] - counters[(i-1)%2][2]) * 1024 * 1024))
			}
			// Sleep a bit, then repeat the stats collection
			select {
			case errc := <-db.quitChan:
				// Quit requesting, stop hammering the database
				errc <- nil
				return
	
			case <-time.After(refresh):
				// Timeout, gather a new set of stats
			}
		}
	}

