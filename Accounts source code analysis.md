계정 패키지는 광장 이더넷 클라이언트 지갑 및 계정 관리를 구현합니다. 이더넷은 keyStore의 광장 지갑의 USB 모드와 지갑 두 가지를 제공합니다. 한편 또한 계정 / ABI 디렉토리에 ABI 코드 계약 이더넷 광장. ABI 프로젝트는 계정 관리와 함께 할 것 같지 않습니다. 계정 관리 인터페이스 만 여기에 일시적으로 분석. 특정 키 스토어의 구현 코드와 USB 일시적으로 부여하지.



데이터 인터페이스를 통해 계좌 번호가 다음의 구조를 정의하고

## 데이터 구조
계정

	// Account represents an Ethereum account located at a specific location defined
	// by the optional URL field.
	//계정은 데이터 20 바이트입니다. URL 필드는 선택 사항입니다.
	type Account struct {
		Address common.Address `json:"address"` // Ethereum account address derived from the key
		URL	 URL			`json:"url"`	 // Optional resource locator within a backend
	}

	const (
		HashLength	= 32
		AddressLength = 20
	)
	// Address represents the 20 byte address of an Ethereum account.
	type Address [AddressLength]byte


지갑. 지갑은 인터페이스가 가장 중요한 일이 될 것이다. 특정 지갑이 인터페이스를 구현합니다.
계층 지갑과 지갑 확신 일반 지갑이 소위.

	// Wallet represents a software or hardware wallet that might contain one or more
	// accounts (derived from the same seed).
	//하나 이상의 계정 소프트웨어 또는 하드웨어 지갑 지갑을 포함에 지갑을 의미
	type Wallet interface {
		// URL retrieves the canonical path under which this wallet is reachable. It is
		// user by upper layers to define a sorting order over all wallets from multiple
		// backends.
		//이 지갑에 액세스 할 수 있습니다 얻기 위해 사용하는 URL 경로 지정. 지갑의 백엔드 모두 사용할 상단에서 정렬하는 데 사용됩니다.
		URL() URL
	
		// Status returns a textual status to aid the user in the current state of the
		// wallet. It also returns an error indicating any failure the wallet might have
		// encountered.
		//지갑의 현재 상태를 식별하는 데 사용되는 텍스트 값을 반환합니다. 또한 지갑 발생한 오류를 식별하는 오류를 반환합니다.
		Status() (string, error)
	
		// Open initializes access to a wallet instance. It is not meant to unlock or
		// decrypt account keys, rather simply to establish a connection to hardware
		// wallets and/or to access derivation seeds.
		//지갑을 열기 액세스 인스턴스를 초기화합니다. 이 방법은 그 해제를 의미하거나 계정의 암호를 해독, 단순히 파생 시드 하드웨어 지갑 및 / 또는 액세스 연결을 설정하지 않습니다.
		// The passphrase parameter may or may not be used by the implementation of a
		// particular wallet instance. The reason there is no passwordless open method
		// is to strive towards a uniform wallet handling, oblivious to the different
		// backend providers.
		//암호 매개 변수는 일부 구현에 필요하지 않을 수 있습니다. 열기 방식이 아닌 암호 인수를 제공하지 않는 이유는 통합 된 인터페이스를 제공하는 것입니다.
		// Please note, if you open a wallet, you must close it to release any allocated
		// resources (especially important when working with hardware wallets).
		//당신이 지갑을 열면 닫을 필요가 있습니다. 그렇지 않으면, 일부 리소스가 해제되지 않을 수 있습니다. 특히 지갑 하드웨어의 사용은 특별한주의를 필요로하는 경우.
		Open(passphrase string) error
	
		// Close releases any resources held by an open wallet instance.
		//닫기 릴리스 열기 방법으로 사용되는 자원.
		Close() error
	
		// Accounts retrieves the list of signing accounts the wallet is currently aware
		// of. For hierarchical deterministic wallets, the list will not be exhaustive,
		// rather only contain the accounts explicitly pinned during account derivation.
		//계정 계정 목록에있는 지갑을 가져 오는 데 사용. 계층 지갑,이 목록은 모든 계정의 완전한 목록이 아닙니다,하지만 파생 된 계정 중 고정 명시 적으로 계정이 포함되어 있습니다.
		Accounts() []Account
	
		// Contains returns whether an account is part of this particular wallet or not.
		//이 지갑 여부 계정으로 돌아 들어 있습니다.
		Contains(account Account) bool
	
		// Derive attempts to explicitly derive a hierarchical deterministic account at
		// the specified derivation path. If requested, the derived account will be added
		// to the wallet's tracked account list.
		//명시 적으로 파생 계층 결정적 계정 유도 지정된 경로에 대한 시도를 유도. 핀에 해당하는 경우, 유도 계정 지갑 계좌 추적의 목록에 추가된다.
		Derive(path DerivationPath, pin bool) (Account, error)
	
		// SelfDerive sets a base account derivation path from which the wallet attempts
		// to discover non zero accounts and automatically add them to list of tracked
		// accounts.
		//SelfDerive 0이 아닌 지갑 계정을 찾아 자동으로 계정을 추적 목록에 추가하려고 기본 계정 수출 경로를 설정합니다.
		// Note, self derivaton will increment the last component of the specified path
		// opposed to decending into a child path to allow discovering accounts starting
		// from non zero components.
		//SelfDerive하지 아래로 하위 경로에 지정된 경로의 마지막 구성 요소를 증가합니다 어셈블리가 제로 발견 계정에서 시작 할 수 있도록 유의하십시오.
		// You can disable automatic account discovery by calling SelfDerive with a nil
		// chain state reader.
		//당신은 자동 검색을 해제하기 위해 전무 ChainStateReader 계정을 전달할 수 있습니다.
		SelfDerive(base DerivationPath, chain ethereum.ChainStateReader)
	
		// SignHash requests the wallet to sign the given hash.
		//SignHash 지갑 들어오는 해시에 대한 요청에 서명합니다.
		// It looks up the account specified either solely via its address contained within,
		// or optionally with the aid of any location metadata from the embedded URL field.
		//그것은 함유 해결할 수 (또는 대안 URL 필드에 포함 된 임의 위치 메타 데이터)을 지정 계좌를 찾아.
		// If the wallet requires additional authentication to sign the request (e.g.
		// a password to decrypt the account, or a PIN code o verify the transaction),
		// an AuthNeededError instance will be returned, containing infos for the user
		// about which fields or actions are needed. The user may retry by providing
		// the needed details via SignHashWithPassphrase, or by other means (e.g. unlock
		// the account in a keystore).
		//당신이 서명을 확인할 수있는 추가 지갑이 필요한 경우 (예를 들어, 계정 잠금을 해제하거나 거래를 확인하기 위해 PIN 코드를 요구하는 암호가 필요합니다.)
		//그것은 사용자에 대한 정보를 포함하는 오류 AuthNeededError를 반환하고, 이는 필드 또는 작업을 제공해야합니다.
		//사용자가 SignHashWithPassphrase에 의해 또는 다른 수단에 의해 서명 될 수있다 (내부의 키 스토어에 계정 잠금을 해제하기)
		SignHash(account Account, hash []byte) ([]byte, error)
	
		// SignTx requests the wallet to sign the given transaction.
		//지정된 트랜잭션에 대한 SignTx 요청 지갑 서명됩니다.
		// It looks up the account specified either solely via its address contained within,
		// or optionally with the aid of any location metadata from the embedded URL field.
		// 
		// If the wallet requires additional authentication to sign the request (e.g.
		// a password to decrypt the account, or a PIN code o verify the transaction),
		// an AuthNeededError instance will be returned, containing infos for the user
		// about which fields or actions are needed. The user may retry by providing
		// the needed details via SignTxWithPassphrase, or by other means (e.g. unlock
		// the account in a keystore).
		SignTx(account Account, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)
	
		// SignHashWithPassphrase requests the wallet to sign the given hash with the
		// given passphrase as extra authentication information.
		//주어진 암호를 사용하여 SignHashWithPassphrase 요청 지갑은 주어진 해시에 서명
		// It looks up the account specified either solely via its address contained within,
		// or optionally with the aid of any location metadata from the embedded URL field.
		SignHashWithPassphrase(account Account, passphrase string, hash []byte) ([]byte, error)
	
		// SignTxWithPassphrase requests the wallet to sign the given transaction, with the
		// given passphrase as extra authentication information.
		//주어진 암호를 사용하여 SignHashWithPassphrase 요청 지갑은 주어진 거래에 서명
		// It looks up the account specified either solely via its address contained within,
		// or optionally with the aid of any location metadata from the embedded URL field.
		SignTxWithPassphrase(account Account, passphrase string, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)
	}


백엔드 백엔드
	
	// Backend is a "wallet provider" that may contain a batch of accounts they can
	// sign transactions with and upon request, do so.
	//백엔드는 지갑 공급자입니다. 이 계정의 수를 포함 할 수있다. 그들은 그렇게 요청에 따라 거래를 로그인 할 수 있습니다.
	type Backend interface {
		// Wallets retrieves the list of wallets the backend is currently aware of.
		//지갑은 현재 지갑을 얻을 찾을 수 있습니다
		// The returned wallets are not opened by default. For software HD wallets this
		// means that no base seeds are decrypted, and for hardware wallets that no actual
		// connection is established.
		//기본값은 열리지 않습니다 지갑을 반환하는 것입니다.
		// The resulting wallet list will be sorted alphabetically based on its internal
		// URL assigned by the backend. Since wallets (especially hardware) may come and
		// go, the same wallet might appear at a different positions in the list during
		// subsequent retrievals.
		//월렛 생성 목록 후단에 할당 된 내부 URL에 따라 알파벳순으로 정렬 될 것이다. 동일한 지갑이 목록 표시에서 다른 위치 일 수있다 후속 검색 프로세스 있도록 개폐있다 지갑 (월렛 특히 하드웨어)한다.
		Wallets() []Wallet
	
		// Subscribe creates an async subscription to receive notifications when the
		// backend detects the arrival or departure of a wallet.
		//비동기를 만들 구독 백 엔드에서 지갑의 도착 또는 출발을 감지 할 때 알림을 수신에 가입.
		Subscribe(sink chan<- WalletEvent) event.Subscription
	}


## manager.go
계정 관리자는 모든 것을 포함하는 관리 도구입니다. 그리고 모든 거래에 서명 백엔드을 통신 할 수 있습니다.

데이터 구조

	// Manager is an overarching account manager that can communicate with various
	// backends for signing transactions.
	type Manager struct {
		//등록 된 모든 백엔드
		backends map[reflect.Type][]Backend // Index of backends currently registered
		//모든 가입자를 업데이트 백엔드
		updaters []event.Subscription	   // Wallet update subscriptions for all backends
		//백엔드 업데이트 가입 홈
		updates  chan WalletEvent		   // Subscription sink for backend wallet changes
		//지갑 등록 된 모든 백엔드 캐시
		wallets  []Wallet				   // Cache of all wallets from all registered backends
		//도착과 출발 알림 지갑
		feed event.Feed // Wallet feed notifying of arrivals/departures
		//종료 대기열
		quit chan chan error
		lock sync.RWMutex
	}


만들기 관리자

	
	// NewManager creates a generic account manager to sign transaction via various
	// supported backends.
	func NewManager(backends ...Backend) *Manager {
		// Subscribe to wallet notifications from all backends
		updates := make(chan WalletEvent, 4*len(backends))
	
		subs := make([]event.Subscription, len(backends))
		for i, backend := range backends {
			subs[i] = backend.Subscribe(updates)
		}
		// Retrieve the initial list of wallets from the backends and sort by URL
		var wallets []Wallet
		for _, backend := range backends {
			wallets = merge(wallets, backend.Wallets()...)
		}
		// Assemble the account manager and return
		am := &Manager{
			backends: make(map[reflect.Type][]Backend),
			updaters: subs,
			updates:  updates,
			wallets:  wallets,
			quit:	 make(chan chan error),
		}
		for _, backend := range backends {
			kind := reflect.TypeOf(backend)
			am.backends[kind] = append(am.backends[kind], backend)
		}
		go am.update()
	
		return am
	}

업데이트 방법. 그것은 goroutine이다. 모든 백엔드가 트리거를 수신 정보를 업데이트합니다. 그런 다음 피드에 전달.

	// update is the wallet event loop listening for notifications from the backends
	// and updating the cache of wallets.
	func (am *Manager) update() {
		// Close all subscriptions when the manager terminates
		defer func() {
			am.lock.Lock()
			for _, sub := range am.updaters {
				sub.Unsubscribe()
			}
			am.updaters = nil
			am.lock.Unlock()
		}()
	
		// Loop until termination
		for {
			select {
			case event := <-am.updates:
				// Wallet event arrived, update local cache
				am.lock.Lock()
				switch event.Kind {
				case WalletArrived:
					am.wallets = merge(am.wallets, event.Wallet)
				case WalletDropped:
					am.wallets = drop(am.wallets, event.Wallet)
				}
				am.lock.Unlock()
	
				// Notify any listeners of the event
				am.feed.Send(event)
	
			case errc := <-am.quit:
				// Manager terminating, return
				errc <- nil
				return
			}
		}
	}

반환 백엔드

	// Backends retrieves the backend(s) with the given type from the account manager.
	func (am *Manager) Backends(kind reflect.Type) []Backend {
		return am.backends[kind]
	}


뉴스 구독

	// Subscribe creates an async subscription to receive notifications when the
	// manager detects the arrival or departure of a wallet from any of its backends.
	func (am *Manager) Subscribe(sink chan<- WalletEvent) event.Subscription {
		return am.feed.Subscribe(sink)
	}


노드입니다. 그것은 때 계정 관리자에게 작성됩니다.

	// New creates a new P2P node, ready for protocol registration.
	func New(conf *Config) (*Node, error) {
		...
		am, ephemeralKeystore, err := makeAccountManager(conf)
		


	
	func makeAccountManager(conf *Config) (*accounts.Manager, string, error) {
		scryptN := keystore.StandardScryptN
		scryptP := keystore.StandardScryptP
		if conf.UseLightweightKDF {
			scryptN = keystore.LightScryptN
			scryptP = keystore.LightScryptP
		}
	
		var (
			keydir	string
			ephemeral string
			err	   error
		)
		switch {
		case filepath.IsAbs(conf.KeyStoreDir):
			keydir = conf.KeyStoreDir
		case conf.DataDir != "":
			if conf.KeyStoreDir == "" {
				keydir = filepath.Join(conf.DataDir, datadirDefaultKeyStore)
			} else {
				keydir, err = filepath.Abs(conf.KeyStoreDir)
			}
		case conf.KeyStoreDir != "":
			keydir, err = filepath.Abs(conf.KeyStoreDir)
		default:
			// There is no datadir.
			keydir, err = ioutil.TempDir("", "go-ethereum-keystore")
			ephemeral = keydir
		}
		if err != nil {
			return nil, "", err
		}
		if err := os.MkdirAll(keydir, 0700); err != nil {
			return nil, "", err
		}
		// Assemble the account manager and supported backends
		//키 스토어의 백엔드 만들기
		backends := []accounts.Backend{
			keystore.NewKeyStore(keydir, scryptN, scryptP),
		}
		//USB 지갑합니다. 당신은 몇 가지 추가 작업을 할 필요가있다.
		if !conf.NoUSB {
			// Start a USB hub for Ledger hardware wallets
			if ledgerhub, err := usbwallet.NewLedgerHub(); err != nil {
				log.Warn(fmt.Sprintf("Failed to start Ledger hub, disabling: %v", err))
			} else {
				backends = append(backends, ledgerhub)
			}
			// Start a USB hub for Trezor hardware wallets
			if trezorhub, err := usbwallet.NewTrezorHub(); err != nil {
				log.Warn(fmt.Sprintf("Failed to start Trezor hub, disabling: %v", err))
			} else {
				backends = append(backends, trezorhub)
			}
		}
		return accounts.NewManager(backends...), ephemeral, nil
	}
