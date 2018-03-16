geth 우리의 주요 이동 - 에테 리움 명령 줄 도구입니다. 또한 우리의 다양한 네트워크 액세스 포인트 (메인 순 기본 네트워크 테스트 및 네트워크 테스트-NET 개인 네트워크). 그것은 전체 노드 노드 경량 모드 또는 모드를 실행을 지원합니다. 다른 프로그램은 광장 JSON의 RPC 호출을 노출 이더넷을 통해 네트워크에 액세스 할 수 있습니다.

당신이 어떤 명령 실행을 입력하지 않으면 직접 geth. 기본 노드는 전체 노드 모델을 시작합니다. 주요 네트워크에 연결됨. 우리는 이러한 구성 요소와 관련된 주요 프로세스를보기 시작 무엇.


## 주요 기능 cmd를 / geth / main.go의 시작
참조 a의 주요 기능은 직접적으로까지 실행합니다. 조금 보이기 시작하면 무지 강요했다. 언어 뒤에 이동 하나는 main () 함수이며, 두 개의 기본 기능이 있습니다 발견했다. 하나는 초기화 () 함수입니다. 언어는 자동으로 모든 패키지 초기화의 첫 번째 () 함수를 호출하는 특정 순서로 이동합니다. 그리고 주 () 함수를 호출합니다.

	func main() {
		if err := app.Run(os.Args); err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
	}
	

main.go 초기화 기능
응용 프로그램은 노사정 gopkg.in/urfave/cli.v1 패키지의 예입니다. 이 삼자 패키지의 사용은 일반적으로 먼저 응용 프로그램 객체를 생성하는 것입니다. 행동 코드를 통해 구성 응용 프로그램 객체는 콜백을 제공합니다. 실행하면 다음 직접 내부의 주요 기능의 라인에 app.Run (os.Args)를 실행합니다.

	import (
		...
		"gopkg.in/urfave/cli.v1"
	)

	var (

		app = utils.NewApp(gitCommit, "the go-ethereum command line interface")
		// flags that configure the node
		nodeFlags = []cli.Flag{
			utils.IdentityFlag,
			utils.UnlockedAccountFlag,
			utils.PasswordFileFlag,
			utils.BootnodesFlag,
			...
		}
	
		rpcFlags = []cli.Flag{
			utils.RPCEnabledFlag,
			utils.RPCListenAddrFlag,
			...
		}
	
		whisperFlags = []cli.Flag{
			utils.WhisperEnabledFlag,
			...
		}
	)
	func init() {
		// Initialize the CLI app and start Geth
		//사용자가 다른 하위 명령을 입력하지 않을 경우 조치 필드를 나타냅니다,이 기능이 필드를 지적 호출합니다.
		app.Action = geth
		app.HideVersion = true // we have a command to print the version
		app.Copyright = "Copyright 2013-2017 The go-ethereum Authors"
		//명령의 모든 하위 명령이 지원됩니다
		app.Commands = []cli.Command{
			// See chaincmd.go:
			initCommand,
			importCommand,
			exportCommand,
			removedbCommand,
			dumpCommand,
			// See monitorcmd.go:
			monitorCommand,
			// See accountcmd.go:
			accountCommand,
			walletCommand,
			// See consolecmd.go:
			consoleCommand,
			attachCommand,
			javascriptCommand,
			// See misccmd.go:
			makecacheCommand,
			makedagCommand,
			versionCommand,
			bugCommand,
			licenseCommand,
			// See config.go
			dumpConfigCommand,
		}
		sort.Sort(cli.CommandsByName(app.Commands))
		//옵션을 모두 해결할 수
		app.Flags = append(app.Flags, nodeFlags...)
		app.Flags = append(app.Flags, rpcFlags...)
		app.Flags = append(app.Flags, consoleFlags...)
		app.Flags = append(app.Flags, debug.Flags...)
		app.Flags = append(app.Flags, whisperFlags...)
	
		app.Before = func(ctx *cli.Context) error {
			runtime.GOMAXPROCS(runtime.NumCPU())
			if err := debug.Setup(ctx); err != nil {
				return err
			}
			// Start system runtime metrics collection
			go metrics.CollectProcessMetrics(3 * time.Second)
	
			utils.SetupNetwork(ctx)
			return nil
		}
	
		app.After = func(ctx *cli.Context) error {
			debug.Exit()
			console.Stdin.Close() // Resets terminal mode.
			return nil
		}
	}

우리가 어떤 매개 변수를 입력하지 않으면 자동으로 geth 메소드를 호출합니다.

	// geth is the main entry point into the system if no special subcommand is ran.
	// It creates a default node based on the command line arguments and runs it in
	// blocking mode, waiting for it to be shut down.
	//특정 하위 명령을 지정하지 않으면 geth는 정문 시스템입니다.
	//그것은 제공된 매개 변수를 기반으로 기본 노드를 작성합니다. 그리고 노드가 종료 될 때까지 대기,이 노드 블록 모드를 실행합니다.
	func geth(ctx *cli.Context) error {
		node := makeFullNode(ctx)
		startNode(ctx, node)
		node.Wait()
		return nil
	}

makeFullNode 함수
	
	func makeFullNode(ctx *cli.Context) *node.Node {
		//명령 줄 인수를 기반으로 노드와 특별한 구성을 만들려면
		stack, cfg := makeConfigNode(ctx)
		//위의이 노드에 ETH 서비스 등록. ETH 서비스는 기본 서비스 이더넷 광장입니다. 광장 이더넷 기능의 제공 업체입니다.
		utils.RegisterEthService(stack, &cfg.Eth)
	
		// Whisper must be explicitly enabled by specifying at least 1 whisper flag or in dev mode
		//속삭임 암호 통신 기능을위한 새로운 모듈이다. 우리는 수 있도록 명시 적으로 매개 변수를 제공해야합니다, 또는 개발 모드에 있습니다.
		shhEnabled := enableWhisper(ctx)
		shhAutoEnabled := !ctx.GlobalIsSet(utils.WhisperEnabledFlag.Name) && ctx.GlobalIsSet(utils.DevModeFlag.Name)
		if shhEnabled || shhAutoEnabled {
			if ctx.GlobalIsSet(utils.WhisperMaxMessageSizeFlag.Name) {
				cfg.Shh.MaxMessageSize = uint32(ctx.Int(utils.WhisperMaxMessageSizeFlag.Name))
			}
			if ctx.GlobalIsSet(utils.WhisperMinPOWFlag.Name) {
				cfg.Shh.MinimumAcceptedPOW = ctx.Float64(utils.WhisperMinPOWFlag.Name)
			}
			//쉬 등록 서비스
			utils.RegisterShhService(stack, &cfg.Shh)
		}
	
		// Add the Ethereum Stats daemon if requested.
		if cfg.Ethstats.URL != "" {
			//광장 이더넷 등록 현황 서비스. 기본값은 활성화되지 않습니다.
			utils.RegisterEthStatsService(stack, cfg.Ethstats.URL)
		}
	
		// Add the release oracle service so it boots along with node.
		//릴리스 오라클 서비스는 클라이언트 버전이 서비스의 최신 버전입니다 볼하는 데 사용됩니다.
		//당신은 업데이트해야합니다. 그런 다음 로그를 인쇄하여 버전을 업데이트하라는 메시지가 표시됩니다.
		//릴리스는 계약의 지능적인 양식을 통해 실행하는 것입니다. 후속 구체적으로이 서비스를 논의 될 것이다.
		if err := stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
			config := release.Config{
				Oracle: relOracle,
				Major:  uint32(params.VersionMajor),
				Minor:  uint32(params.VersionMinor),
				Patch:  uint32(params.VersionPatch),
			}
			commit, _ := hex.DecodeString(gitCommit)
			copy(config.Commit[:], commit)
			return release.NewReleaseService(ctx, config)
		}); err != nil {
			utils.Fatalf("Failed to register the Geth release oracle service: %v", err)
		}
		return stack
	}

makeConfigNode. 이 기능은 주로 구성 파일 및 플래그를 통해 전체 시스템 구성을 실행 생성합니다.
	
	func makeConfigNode(ctx *cli.Context) (*node.Node, gethConfig) {
		// Load defaults.
		cfg := gethConfig{
			Eth:  eth.DefaultConfig,
			Shh:  whisper.DefaultConfig,
			Node: defaultNodeConfig(),
		}
	
		// Load config file.
		if file := ctx.GlobalString(configFileFlag.Name); file != "" {
			if err := loadConfig(file, &cfg); err != nil {
				utils.Fatalf("%v", err)
			}
		}
	
		// Apply flags.
		utils.SetNodeConfig(ctx, &cfg.Node)
		stack, err := node.New(&cfg.Node)
		if err != nil {
			utils.Fatalf("Failed to create the protocol stack: %v", err)
		}
		utils.SetEthConfig(ctx, stack, &cfg.Eth)
		if ctx.GlobalIsSet(utils.EthStatsURLFlag.Name) {
			cfg.Ethstats.URL = ctx.GlobalString(utils.EthStatsURLFlag.Name)
		}
	
		utils.SetShhConfig(ctx, stack, &cfg.Shh)
	
		return stack, cfg
	}

RegisterEthService

	// RegisterEthService adds an Ethereum client to the stack.
	func RegisterEthService(stack *node.Node, cfg *eth.Config) {
		var err error
		//동기화 모드는 경량 동기 모드 인 경우. 그래서 경량 클라이언트를 시작합니다.
		if cfg.SyncMode == downloader.LightSync {
			err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
				return les.New(ctx, cfg)
			})
		} else {
			//그렇지 않으면 전체 노드를 시작합니다
			err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
				fullNode, err := eth.New(ctx, cfg)
				if fullNode != nil && cfg.LightServ > 0 {
					//기본 크기는 0 LightServ가 시작되지 않은 것입니다 LesServer
					//LesServer 서비스를 제공하기 위해 경량 노드입니다.
					ls, _ := les.NewLesServer(fullNode, cfg)
					fullNode.AddLesServer(ls)
				}
				return fullNode, err
			})
		}
		if err != nil {
			Fatalf("Failed to register the Ethereum service: %v", err)
		}
	}


startNode

	// startNode boots up the system node and all registered protocols, after which
	// it unlocks any requested accounts, and starts the RPC/IPC interfaces and the
	// miner.
	func startNode(ctx *cli.Context, stack *node.Node) {
		// Start up the node itself
		utils.StartNode(stack)
	
		// Unlock any account specifically requested
		ks := stack.AccountManager().Backends(keystore.KeyStoreType)[0].(*keystore.KeyStore)
	
		passwords := utils.MakePasswordList(ctx)
		unlocks := strings.Split(ctx.GlobalString(utils.UnlockedAccountFlag.Name), ",")
		for i, account := range unlocks {
			if trimmed := strings.TrimSpace(account); trimmed != "" {
				unlockAccount(ctx, ks, trimmed, i, passwords)
			}
		}
		// Register wallet event handlers to open and auto-derive wallets
		events := make(chan accounts.WalletEvent, 16)
		stack.AccountManager().Subscribe(events)
	
		go func() {
			// Create an chain state reader for self-derivation
			rpcClient, err := stack.Attach()
			if err != nil {
				utils.Fatalf("Failed to attach to self: %v", err)
			}
			stateReader := ethclient.NewClient(rpcClient)
	
			// Open any wallets already attached
			for _, wallet := range stack.AccountManager().Wallets() {
				if err := wallet.Open(""); err != nil {
					log.Warn("Failed to open wallet", "url", wallet.URL(), "err", err)
				}
			}
			// Listen for wallet event till termination
			for event := range events {
				switch event.Kind {
				case accounts.WalletArrived:
					if err := event.Wallet.Open(""); err != nil {
						log.Warn("New wallet appeared, failed to open", "url", event.Wallet.URL(), "err", err)
					}
				case accounts.WalletOpened:
					status, _ := event.Wallet.Status()
					log.Info("New wallet appeared", "url", event.Wallet.URL(), "status", status)
	
					if event.Wallet.URL().Scheme == "ledger" {
						event.Wallet.SelfDerive(accounts.DefaultLedgerBaseDerivationPath, stateReader)
					} else {
						event.Wallet.SelfDerive(accounts.DefaultBaseDerivationPath, stateReader)
					}
	
				case accounts.WalletDropped:
					log.Info("Old wallet dropped", "url", event.Wallet.URL())
					event.Wallet.Close()
				}
			}
		}()
		// Start auxiliary services if enabled
		if ctx.GlobalBool(utils.MiningEnabledFlag.Name) {
			// Mining only makes sense if a full Ethereum node is running
			var ethereum *eth.Ethereum
			if err := stack.Service(&ethereum); err != nil {
				utils.Fatalf("ethereum service not running: %v", err)
			}
			// Use a reduced number of threads if requested
			if threads := ctx.GlobalInt(utils.MinerThreadsFlag.Name); threads > 0 {
				type threaded interface {
					SetThreads(threads int)
				}
				if th, ok := ethereum.Engine().(threaded); ok {
					th.SetThreads(threads)
				}
			}
			// Set the gas price to the limits from the CLI and start mining
			ethereum.TxPool().SetGasPrice(utils.GlobalBig(ctx, utils.GasPriceFlag.Name))
			if err := ethereum.StartMining(true); err != nil {
				utils.Fatalf("Failed to start mining: %v", err)
			}
		}
	}

요약 :

실제로 매개 변수를 구문 분석하는 전체 프로세스를 시작합니다. 그런 다음 생성하고 노드를 시작합니다. 서비스는 그 노드에 주입. 모든 이더넷 관련 기능과 광장은 서비스의 형태로 구현된다.


모든 등록에 추가하여 서비스를 열면 이동합니다. 이 시간은 시스템 goroutine 무엇 켜집니다. 다음은 요약을 할 수 있습니다.


goroutine의 현재의 모든 주민은 다음의 몇 가지가 있습니다. 주로 P2P 서비스 관련. RPC 및 관련 서비스를 제공합니다.

![image](picture/geth_1.png)

