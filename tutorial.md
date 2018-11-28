# SDK 어플리케이션 튜토리얼

이것은 SDK의 구조, 기본 컨셉, 프로세스, 기능을 빌드하는 방법을 배웁니다. 이 예제는 코스모스 SDK를 이용하여 얼마나 빠르고 쉽게 여러분들만의 블록체인을 만들 수 있는지 보여줍니다.

이 튜토리얼을 마치고 나면 여러분들은 문자열로 맵핑(map[string]string)된 `nameservice` 어플리케이션을 가지게 될겁니다. 이것은 전통적인 DNS시스템 모델(map[domain]zonefile)같은 Namecoin, ENS 또는 Handshake와 비슷합니다. 사용자는 사용되지 않은 이름을 살수 있고 그들의 이름을 판매/교환할 수 있을것입니다.

이 프로젝트의 최종코드는 이 directory에 있습니다. 그러나 이 매뉴얼을 직접 따라해보고 프로젝트를 스스로 빌드하는것이 최고입니다.



## 필수사항

* `golang` > 1.11 설치
* `$GOPATH` 환경변수 설정
* 여러분들의 블록체인을 만들고 싶다는 욕망



## 튜토리얼

이 튜토리얼을 통해 여러분들은 다음과 같은 프로젝트 구성을 가져야 합니다.

```
./nameservice
├── Gopkg.toml
├── Makefile
├── app.go
├── cmd
│   ├── nameservicecli
│   │   └── main.go
│   └── nameserviced
│       └── main.go
└── x
    └── nameservice
        ├── client
        │   └── cli
        │       ├── query.go
        │       └── tx.go
        ├── codec.go
        ├── handler.go
        ├── keeper.go
        ├── msgs.go
        └── querier.go
```

새로운 git 저장소를 만들면서 프로젝트를 시작합니다.

```bash
mkdir -p $GOPATH/src/github.com/{{ .Username }}/nameservice
cd $GOPATH/src/github.com/{{ .Username }}/nameservice
git init

```

첫 번째 단계는 여러분들의 어플리케이션을 설명합니다. 만약 여러분들이 바로 코딩하고 싶다면 첫 번째 단계는 건너뛰고 두 번째 단계부터 시작하십시오.



### 튜토리얼 파트

1. 어플리케이션 설명.
2. `./app.go`에서 어플리케이션 구현 실행.
3. `keeper`에서 모듈 구현
4. `Msgs`와 `Handlers`를 통해 상태 트랜잭션 정의
  * SetName
  * BuyName
5. `Queriers`에서 상태머신 조회
6. `sdk.Codec`을 사용하여 인코딩된 타입 등록
7. 모듈과 상호작용하는 CLI 생성
8. 최종적으로 만들어진 어플리케이션 import
9. 어플리케이션을 위한 `nameserviced`와 `nameservicecli` 엔트리포인트(entry point == 연결지점?) 생성
10. `dep`을 이용하여 의존성 관리 설정



---



# 1. 어플리케이션 설명



## 어플리케이션 목표

생성된 여러분들의 어플리케이션 목표는 사용자가 이름을 구입하고  설정합니다. 이름을 받은 계정은 최고 입찰자일 것 입니다. 이번 섹션에서 이러한 간단한 요구 사항이 어플리케이션 설계로 어떻게 전환되는지 알아봅니다.

블록체인 어플리케이션은 [replicated deterministic state machine](https://en.wikipedia.org/wiki/State_machine_replication)(결정된 상태를 공유한 머신)입니다. 개발자는 상태머신(상태가 무엇인지, 상태 변환 트리거하는 시작 상태 및 메시지), 그리고 [*Tendermint*](https://tendermint.com/docs/introduction/introduction.html)(텐더민트)는 네트워크를 통해 복제처리를 합니다.

> Tendermint는 블록체인의 네트워킹과 합의 계층을 처리하는 어플리케이션 엔진입니다. Tendermint는 트랜잭션 바이트를 전파하고 처리합니다. Tendermint Core는 BFT 알고리즘으로 트랜잭션 순서에 대한 합의를 처리합니다. Tendermint를 더 알고싶으면 [here](https://tendermint.com/docs/introduction/introduction.html)클릭하세요

[Cosmos SDK](https://github.com/cosmos/cosmos-sdk/)은 상태머신을 빌드하는것을 돕기위해 디자인되었습니다. SDK는 모듈을 조합하여 빌드된 어플레케이션을 만드는 블록체인을 모듈화시킨 프레임워크입니다. 각각의 모듈은 메시지/트랜잭션 프로세서 내장, SDK는 각 메시지를 해당 모듈로 라우팅할 책임을 가지고 있습니다.

nameservice 어플리케이션은 다음과 같은 기능을 필요합니다.

* `auth`:  이 모듈은 어카운트와 수수료를 정의하고 어플리케이션에서 다른 기능들을 접근할 수 있도록 한다
* `bank`: 이 모듈은 토큰과 토큰 잔액을 생성하고 관리할 수 있습니다.
* `nameservice`: 이 모듈은 아직 존재하지 않습니다. `nameservice` 어플리케이션의 핵심 로직을 처리합니다. 이것은 어플리케이션을 구축하기 위해 작업해야 하는 주요 소프트웨어입니다.

> 여러분들은 유효성 검사 설정 변경을 처리할 모듈이 없는지 궁금할 수 있습니다. 실제로, Tendermint는 블록체인에 추가할 다음 유효한 트랜잭션을 블록에 포함에 대하여 합의 도출을 위해 유효성을 설정합니다. 만약 유효성 설정을 변경하지 않는다면, 유효성은 `genesis.json` 파일과 같아질것입니다. 이 어플리케이션도 마찬가지입니다. 만약 어플리케이션의 유효성 설정을 바꾸길 원한다면, SDK의  [staking module](https://github.com/cosmos/cosmos-sdk/tree/develop/x/stake) 을 사용하거나 직접 작성할 수 있습니다.



이제 어플리케이션의 주요부분인 상태와 메시지의 타입에 대해 보겠습니다.



## 상태

상태는 주어진 순간 어플리케이션 상태를 보여준다. 그것은 토큰이 각각의 어카운트에서 얼마큼 있는지 알 수 있고 각각의 이름의 owner와 price, 그리고 값을 보여줍니다.

토큰과 계정 상태는  `auth`와 `bank` 모듈에 정의되어 있습니다. 이것은 해당 부분에 신경쓰지 않아도 되는것을 의미합니다. `nameservice` 모듈과 관련된 상태 부분을 정의하면 됩니다.

SDK에서 모든것은 `multistore`라 불리는 곳에 저장됩니다. 멀티스토어 인에 키/값을 생성할 수 있습니다. 어플리케이션을 위해 다음과 같이 저장해야 합니다.



* 이름과 `value` 매핑된 데이터를 `멀티스토어`에 저장하여 `namestore` 생성.
* 이름과 `owner` 매핑된 데이터를 `멀티스토어`에 저장하여 `namestore` 생성.
* 이름과 `price` 매핑된 데이터를 `멀티스토어`에 저장하여 `namestore` 생성.



## 메시지

메시지는 트랜잭션 안에 포함되어있습니다. 그것은 상태 트랜잭션을 트리거 합니다. 각각의 모듈은 메시지 리스트와 어떻게 핸들링 할지 정의되어 있습니다. 어플리케이션 개발을 위해 다음과 같이 기능을 만들어야 합니다.

* `MsgSetName`: 이 메시지는 `NameStore`에 이름 소유자는 이름을 설정할 수 있습니다.
* `MsgBuyName`: 이 메시시지는 계정을 통해 이름을 구입하고 `ownerStore`의 소유자가 될 수 있도록 합니다.

Tendermint에 트랜잭션(블록에 포함된)이 도착할 때, 그것은 [ABCI](https://github.com/tendermint/tendermint/tree/master/abci)를 사용하여 통과되고 메시지를 가져와 디코딩합니다. 그 메시지는 해당 라우터로 라우팅되고 처리기에 정의된 로직에 따라 처리됩니다. 만약 상태 업데이트가 필요하다면, `handler` 는 업데이트를 위해 `keeper`를  호출합니다. 여러분들은 이 튜토리얼의 다음스탭에 대해서 더 배우고 싶을것입니다,



# 2. ./app.go 에서 어플리케이션 구현 실행.



## 어플리케이션 시작

새로운 파일(`./app.go`)을 생성하여 시작합니다. 이 파일은 어플리케이션에서 상태머신(deterministic state-machine)의 핵심입니다.

`app.go` 파일을 보면, 트랜잭션을 받을 때 수행하는 작업을 정의합니다. 먼저, 올바른 순서로 트랜잭션 수신할 수 있어야 합니다. 이것은 [Tendermint consensus engine](https://github.com/tendermint/tendermint)의 역할입니다.

먼저 의존성 파일을 가져옵니다.

```go
package app

import (
    "github.com/tendermint/tendermint/libs/log"
    "github.com/cosmos/cosmos-sdk/x/auth"

	 bam "github.com/cosmos/cosmos-sdk/baseapp"
	 dbm "github.com/tendermint/tendermint/libs/db"
)
```

임포트된 각각의 패키지, 모듈에 대한 문서로 링크되어있습니다.

* `log`: tendermint의 로거.
* `auto`: 코스모스 SDK를 위한 `auth` 모듈
* `dbm`: tendermint 데이터베이스를 사용하기 위한 코드.
* `baseapp`: 아래내용 참조

패키지 몇개는 tendermint의 패키지 입니다. tendermint는 [ABCI](https://github.com/tendermint/tendermint/tree/master/abci)라고 불리는 상호작용을 통한 네트워크로 부터 발생된 트랜잭션을 보냅니다. 만약 설치된 블록체인 아키텍처를 본다면 그것은 다음과 같을 것 입니다.

```
+---------------------+
|                     |
|     Application     |
|                     |
+--------+---+--------+
         ^   |
         |   | ABCI
         |   v
+--------+---+--------+
|                     |
|                     |
|     Tendermint      |
|                     |
|                     |
+---------------------+
```

다행히도 ABCI 인터페이스를 구현할 필요 없습니다. 코스모스 SDK에서 [`baseapp`](https://godoc.org/github.com/cosmos/cosmos-sdk/baseapp)의 형태로 구현을 제공합니다.



다음은 `baseapp`이 무엇을 하는지 보여줍니다.

* 텐더민트 합의 엔진에서 받은 트랜잭션을 디코딩 합니다.
* 트랜잭션에서 메시지를 추출하고 기본적인 검사를 수행합니다.
* 메시지가 처리될 수 있도록 적절한 모듈로 전달합니다. `bashapp`은 사용하려는 특정 모듈에 대한 지식이 없습니다. 이 튜토리얼 뒷 부분에서 볼 수 있듯이 이러한 모듈은 app.go에 선언하는 것은 사용자의 일 입니다.
* ABCI 메시지가 [`DeliverTx`](https://tendermint.com/docs/spec/abci/abci.html#delivertx) 되면 커밋([`CheckTx`](https://tendermint.com/docs/spec/abci/abci.html#checktx)  변경이 지속되지 않음)
* 각 블록의 시작과 끝 부분에 로직 실행을 정의할 수 잇는 두 메시지인 [`Beginblock`](https://tendermint.com/docs/spec/abci/abci.html#beginblock)과 [`Endblock`](https://tendermint.com/docs/spec/abci/abci.html#endblock) 설정을 도와줌. 각 모듈은 BeginBlock과 EndBlock 하위 로직 구현하며, 앱의 역할은 모든 것을 함께 통합하는 것입니다.(응용프로그램에서는 이러한 메시지를 사용하지 않습니다.)
* 상태 초기화를 도와줍니다.
* 조회를 도와줍니다.

지금 여러분들은 새로운 사용자 정의된 타입의 `nameserviceApp` 생성이 필요합니다. 이 타입은 `baseapp`에 포함(embed)되는것은 (다른 언어의 상속과 유사한 golang 에서의 embedding)`baseapp`의 메소드 접근할 수 있습니다.

```go
const (
    appName = "nameservice"
)

type nameserviceApp struct {
    *bam.BaseApp
}
```

어플리케이션을 위해 간단한 생성자를 추가합니다.

```go
func NewnameserviceApp(logger log.Logger, db dbm.DB) *nameserviceApp {

    // First define the top level codec that will be shared by the different modules
    cdc := MakeCodec()

    // BaseApp handles interactions with Tendermint through the ABCI protocol
    bApp := bam.NewBaseApp(appName, logger, db, auth.DefaultTxDecoder(cdc))

    var app = &nameserviceApp{
        BaseApp: bApp,
        cdc:     cdc,
    }

    return app
}
```

멋집니다! 여러분들은 어플피케이션의 기본 골격을 가지고 있습니다. 하지만 아직 기능이 많이 부족합니다.

`baseapp`은 라우트, 어플리케이션에서 사용할 상호작용을 알지 못합니다. 어플리케이션에서 기본적인 역할은 라우트들을 정의합니다. 그 중 다른 역할은 상태 초기화를 정의합니다. 이 두가지 형태의 모듈을 어플리케이션에 추가해야 합니다.

[application design](https://github.com/cosmos/sdk-application-tutorial/blob/master/tutorial/app-design.md) 파트에 나와있는것 처럼, nameservice를 위해 `auth`, `bank` 그리고 `nameservice`3개의 모듈이 필요합니다. 첫 번째로 두개는 이미 존재합니다, 그라나 마지막은 아닙니다. `nameservice` 모듈은 여러분의 상태 머신의 대부분을 정의합니다. 다음단계는 그것을 구축하는 것 입니다.

 