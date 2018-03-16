# 몇 가지 기본적인 도구 패키지
에서 [이더넷 광장] (https://github.com/ethereum/go-ethereum) 프로젝트에서 혼자 멀리 장 인해 단순 기능에, 너무 얇은 일부 우수한 도구 패키지의 작은 모듈 golang 생태계있다. 그러나, 이더넷 광장의 캡슐화로 인해,이 기기는 강한 독립성과 실용성, 매우 우아하다. 우리는 적어도 이더넷 광장 소스 코딩에 익숙한 도움이됩니다에 대한 몇 가지 분석을하기 위해 여기에 있습니다.
## 메트릭 (프로브)
(입 / ethdb 소스 코드 분석 .md)에서, 우리는 [goleveldb (https://github.com/syndtr/goleveldb) 프로젝트의 패키지를 참조 [분석 ethdb 소스]. ethdb 첨가 추상화 계층 goleveldb한다 :

[type Database interface](https://github.com/ethereum/go-ethereum/blob/master/ethdb/interface.go#L29)

외측지지 MemDatabase 인터페이스와 상호 교환 가능하고, 또한 프로브 도구 다음 많은 LDBDatabase [gometrics (https://github.com/rcrowley/go-metrics) 패킷에 사용하고 goroutine을 수행하기 시작할 수있다

[go db.meter(3 * time.Second)](https://github.com/ethereum/go-ethereum/blob/master/ethdb/database.go#L198)

goleveldb 지연 및 I / O 데이터 및 다른 지표를 사용하여 수집시 3초 기간. 그것은 매우 쉽게 보이지만, 문제는 우리가 그것을 사용하는 이러한 정보를 수집하는 방법은?

## 로그 (로그)
golang 로그 패키지는 저점 지점으로 구축 및 이더넷 광장 프로젝트도 예외는 없습니다. 따라서, [log15]의 도입 (https://github.com/inconshreveable/log15)를 사용하기가 불편 로그를 처리한다.


