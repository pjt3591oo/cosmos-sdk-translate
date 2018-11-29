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
8. 최종적으로 만들어진 모듈과 어플리케이션 import
9. 어플리케이션을 위한 `nameserviced`와 `nameservicecli` 엔트리포인트(entry point == 연결지점?) 생성
10. `dep`을 이용하여 의존성 관리 설정
11. 어플리케이션 빌드 그리고 실행



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

 

# 3. keeper에서 모듈 구현



## Keeper 모듈

`Keeper`는 코스모스 SDK에서 메인모듈 입니다. 그것은 저장소와 함께 상요작용을 핸들링하고, 다른 모듈과의 상호작용을 위해 다른 keeper를 참조하고, 모듈의 주요 기능을 포함합니다.

`./x/nameservice/keeper.go`을 생성하여 시작합니다. 코스모스 SDK 어플리케이션에서 모듈은 `./x/` 안에 있어야 합니다.



## Keeper 구조

여러분의 SDK 모듈을 시작을 위해, `./x/nameservice/Keeper.go` 파일에 `nameservice.Keeper`를 정의합니다.

```go
package nameservice

import (
	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/cosmos/cosmos-sdk/x/bank"

	sdk "github.com/cosmos/cosmos-sdk/types"
)

// Keeper maintains the link to data storage and exposes getter/setter methods for the various parts of the state machine
type Keeper struct {
	coinKeeper bank.Keeper

	namesStoreKey  sdk.StoreKey // Unexposed key to access name store from sdk.Context
	ownersStoreKey sdk.StoreKey // Unexposed key to access owners store from sdk.Context
	pricesStoreKey sdk.StoreKey // Unexposed key to access prices store from sdk.Context

	cdc *codec.Codec // The wire codec for binary encoding/decoding.
}
```

다음은 앞의 코드에 대한 참고사항 입니다.

* 포함된 3가지의`cosmos-sdk`패키지
  * [`codec`](https://godoc.org/github.com/cosmos/cosmos-sdk/codec) - `codec`은 코드모스 인코딩 형식인  [Amino](https://github.com/tendermint/go-amino)을 사용하기 위한 도구를 제공합니다.
  * [`bank`](https://godoc.org/github.com/cosmos/cosmos-sdk/x/bank) - `bank` 모듈은 어카운트와 토큰 전송을 관리합니다.
  * [`types`](https://godoc.org/github.com/cosmos/cosmos-sdk/types) - `types`은 SDK에서 사용하는 타입을 포함합니다.
* `Keeper` 구조. keeper에는 몇 가지 핵심 요소가 있습니다.
  * [`bank.Keeper`](https://godoc.org/github.com/cosmos/cosmos-sdk/x/bank#Keeper) - `keeper`에서 `bank` 모듈로부터 참조합니다. 이것을 포함하면 모듈의 코드가 `bank` 모듈에서 함수를 호출할 수 있습니다. SDK는 [object capabilities](https://en.wikipedia.org/wiki/Object-capability_model) 방식을 사용하여 응용프로그램 상태 섹션에 접근합니다. 이는 개발자가 접근 권한이 필요하지 않은 상태의 부분에 결함이 있거나, 악의적인 모듈의 기능이 영향을 미치지 못하도록 제한함으로써 최소한의 권한 접근 방식을 사용할 수 있게 하기 위함입니다.
  * [`*codec.Codec`](https://godoc.org/github.com/cosmos/cosmos-sdk/codec#Codec) - 이것은 Amino가 이진 형태의 구조체를 인코딩하고 디코딩 할 때 사용하는 코덱에 대한 포인터입니다.
  * [`sdk.StoreKey`](https://godoc.org/github.com/cosmos/cosmos-sdk/types#StoreKey) - 이것은 상태를 유지하는 `sdk.KVStore`에 접근할 수 있습니다.
* 이 모듈에는 3가지 저장 키가 있습니다.
  * `namesStoreKey` - 주어진 이름에 대한 값을 저장하고 있는 메인 저장소 입니다 (즉. `map[name]value`).
  * `ownerStoreKey`- 이 저장소는 주어진 이름에 대한 소유자를 가져올 수 있습니다 (즉. `map[sdk_address]name`).
  * `priceStoreKey`- 이 저장소는 현재 소유자가 주어진 이름에 대한 가격이 포함되어 있습니다. 이 이름을 사는 사람은 현재 소유자보다 더 많이 지출해야 합니다(즉. `map[name]price`).



## Getter, Setter

이제 keeper를 통해 저장소와 상호 작용할 수 있는 메소드를 추가해야 합니다. 먼저, 주어진 이름에 대한 문자열로 설정하는 기능을 추가합니다.

```go
// SetName - sets the value string that a name resolves to
func (k Keeper) SetName(ctx sdk.Context, name string, value string) {
	store := ctx.KVStore(k.namesStoreKey)
	store.Set([]byte(name), []byte(value))
}
```

이 메소드는 먼저 keeper의 namesStoreKey를 사용하여 `map[name]value` 에 대한 저장소 객체를 가져옵니다.

> 참고: 이 함수는 [`sdk.Context`](https://godoc.org/github.com/cosmos/cosmos-sdk/types#Context)를 사용합니다. 이 객체는 `blockHeight` 및 `chainID`와 같은 여러 상태의 중요한 부붅에 액세스하는 함수를 가지고 있습니다.

그런 다음 `.Set([]byte, []bute) 메소드를 사용하여 <name, value> 쌍을 저장소에 추가합니다. 저장소는 []byte만 사용하므로 먼저 문자열을 []byte 형태로 변환하고 매개변수로 Set() 메소드에 사용합니다.

그런 다음 메소드를 추가하여 이름을 확인합니다(예, 이름으로 값 조회).

```go
// ResolveName - returns the string that the name resolves to
func (k Keeper) ResolveName(ctx sdk.Context, name string) string {
	store := ctx.KVStore(k.namesStoreKey)
	bz := store.Get([]byte(name))
	return string(bz)
}
```

`SetName` 메서드처럼, `StoreKey`를 이용하여 저장소에 접근합니다. 그런 다음 Store 키에 Set 메서드를 사용하는 대신 `.Get([]byte) []byte` 메서드를 사용합니다. 함수에 대한 매개 변수로 []byte로 형 변환된 이름 문자열 인 키를 전달하고 결과를 []byte 형식으로 반환합니다. 이것을 문자열로 변환하고 반환합니다.

이름 소유자를 가져오고 설정하기 위한 비슷한 기능 추가

```go
// HasOwner - returns whether or not the name already has an owner
func (k Keeper) HasOwner(ctx sdk.Context, name string) bool {
	store := ctx.KVStore(k.ownersStoreKey)
	bz := store.Get([]byte(name))
	return bz != nil
}

// GetOwner - get the current owner of a name
func (k Keeper) GetOwner(ctx sdk.Context, name string) sdk.AccAddress {
	store := ctx.KVStore(k.ownersStoreKey)
	bz := store.Get([]byte(name))
	return bz
}

// SetOwner - sets the current owner of a name
func (k Keeper) SetOwner(ctx sdk.Context, name string, owner sdk.AccAddress) {
	store := ctx.KVStore(k.ownersStoreKey)
	store.Set([]byte(name), owner)
}
```

앞의 코드에 대한 참고사항:

* `namesStoreKey` 저장소에서 데이터에 접근하는 대신 `prossessStoreKey`저장소에서 가져옵니다.
* `sdk.AccAddress`는 `[]byte`에 대한 유형병 별칭이기 때문에 기본으로 변환할 수 있습니다.
* 조건문에 사용할 수 있는 편리한 함수인 `HasOwner`를 추가합니다.

마지막으로 주어진 이름으로 가격을 가져오고 설정할 수 있는 getter와 setter를 추가합니다.

 ```go
// GetPrice - gets the current price of a name.  If price doesn't exist yet, set to 1steak.
func (k Keeper) GetPrice(ctx sdk.Context, name string) sdk.Coins {
	if !k.HasOwner(ctx, name) {
		return sdk.Coins{sdk.NewInt64Coin("mycoin", 1)}
	}
	store := ctx.KVStore(k.pricesStoreKey)
	bz := store.Get([]byte(name))
	var price sdk.Coins
	k.cdc.MustUnmarshalBinary(bz, &price)
	return price
}

// SetPrice - sets the current price of a name
func (k Keeper) SetPrice(ctx sdk.Context, name string, price sdk.Coins) {
	store := ctx.KVStore(k.pricesStoreKey)
	store.Set([]byte(name), k.cdc.MustMarshalBinary(price))
}
 ```

앞의 코드에 대한 참고사항:

* `sdk.Coins`는 바이트 형태로 인코딩하지 않습니다. 이것은  [Amino](https://github.com/tendermint/go-amino/)을 사용하여 저장소에 저장할 때 가격을 마샬링(marsalled)과 언마샬링(unmarshalled)이 필요합니다.
* 소유자가 없는(가격이 없는) 이름의 가격을 얻으려면 가격으로 `1steak`를 반환합니다.

`./x/nemeservice/keeper.go` 파일에 필요한 마지막 코드는 Keeper의 생성자 함수입니다.

```go
// NewKeeper creates new instances of the nameservice Keeper
func NewKeeper(coinKeeper bank.Keeper, namesStoreKey sdk.StoreKey, ownersStoreKey sdk.StoreKey, priceStoreKey sdk.StoreKey, cdc *codec.Codec) Keeper {
	return Keeper{
		coinKeeper:     coinKeeper,
		namesStoreKey:  namesStoreKey,
		ownersStoreKey: ownersStoreKey,
		pricesStoreKey: priceStoreKey,
		cdc:            cdc,
	}
}
```



# 4. Msgs와 Handlers 를 통해 상태 트랜잭션 정의



## Msgs와 Handlers

이제 `Keeper`가 설정습니다. Msgs와 Handler를 만들어 사용자가 실제로 이름을 구입하여 값을 설정할 수 있습니다.



## `Msgs`

`Msgs`는 상태 전환을 트리거합니다. 메시지는 클라이언트가 네트워크에 전파하는 [`Txs`](https://github.com/cosmos/cosmos-sdk/blob/develop/types/tx_msg.go#L34-L38)로 묶입니다. 코스모스 SDK는 `Txs `로 부터 `Msgs`를 래핑하거나 래핑을 풀 수 있고, 이것은 개발자가 `Msgs`를 정의하기만 하면 됩니다. `Msgs`는 다음 인터페이스를 충족해야 합니다.

```go
// Transactions messages must fulfill the Msg
type Msg interface {
	// Return the message type.
	// Must be alphanumeric or empty.
	Type() string

	// Returns a human-readable string for the message, intended for utilization
	// within tags
	Route() string

	// ValidateBasic does a simple validation check that
	// doesn't require access to any other information.
	ValidateBasic() Error

	// Get the canonical byte representation of the Msg.
	GetSignBytes() []byte

	// Signers returns the addrs of signers that must sign.
	// CONTRACT: All signatures must be present to be valid.
	// CONTRACT: Returns addrs in some deterministic order.
	GetSigners() []AccAddress
}
```

## `Handlers`

`Handlers`는 주어진 `Msgs`가 수실될 때 수행해야 하는 작업(업데이트 할 방법 및 조건을 필요로하는 저장소)을 정의합니다.

이 모듈에는 사용자가 어플리케이션 상태와 상호 작용하기 위해 보낼 수 있는 두가지 형태의 `Msgs`인 [`SetName`](https://github.com/cosmos/sdk-application-tutorial/blob/master/tutorial/set-name.md)과 [`BuyName`](https://github.com/cosmos/sdk-application-tutorial/blob/master/tutorial/buy-name.md)이 있습니다. 그들은 각각 연관된 `Handler`를 가질것 입니다.



## SetName



## `Msg`

SDK에서 `Msgs` 네이밍 규칙은 `Msg {{action}}`입니다. 구현할 첫 번째 작업은 주소 소유자가 이름을 설정하도록 허용하는 `Msgs`인 `SetName` 입니다. `./x/nameservice/msgs.go`라는 새로운 파일에서 `MsgSetName`을 정의하여 시작합니다.

```go
package nameservice

import (
	"encoding/json"

	sdk "github.com/cosmos/cosmos-sdk/types"
)

// MsgSetName defines a SetName message
type MsgSetName struct {
	NameID string
	Value  string
	Owner  sdk.AccAddress
}

// NewMsgSetName is a constructor function for MsgSetName
func NewMsgSetName(name string, value string, owner sdk.AccAddress) MsgSetName {
	return MsgSetName{
		NameID: name,
		Value:  value,
		Owner:  owner,
	}
}
```

`MsgSetName`에는 이름의 값을 설정하는 데 필요한 세 가지 속성을 가지고 있습니다.

* `name` - 이름 설정
* `value` - 이름이 결정되는 대상
* `owner` - 이름의 소유자

> 참고 :  필드 이름은 `name` 대신 `nameID`입니다. `.Route()`는 `Msg` 인터페이스 이름입니다. 이 문제는  [future update of the SDK](https://github.com/cosmos/cosmos-sdk/issues/2456)에서 풀어야합니다.

다음으로 `Msg` 인터페이스를 구현합니다.

```go
// Type should return the name of the module
func (msg MsgSetName) Route() string { return "nameservice" }

// Name should return the action
func (msg MsgSetName) Type() string { return "set_name"}
```

앞의 함수는 SDK에서 `Msgs`를 적절한 모듈로 전달하여 처리하도록 합니다. 또한 인덱싱에서 사용되는 데이터베이스 태그에 사람이 읽을 수 있는 이름을 추가합니다.

```go
// ValdateBasic Implements Msg.
func (msg MsgSetName) ValidateBasic() sdk.Error {
	if msg.Owner.Empty() {
		return sdk.ErrInvalidAddress(msg.Owner.String())
	}
	if len(msg.NameID) == 0 || len(msg.Value) == 0 {
		return sdk.ErrUnknownRequest("Name and/or Value cannot be empty")
	}
	return nil
}
```

`ValidateBasic`는 메시지의 유효성에 대한 몇가지 기본적인 **stateless** 검사를 위해 사용합니다. 첫 번째 케이스로 비어있는 속성이 없는지 확인합니다. 여기서 `sdk.Error` 타입을 사용하는것을 참고하세요. SDK는 어플리케이션 개발자가 자주접하는 에러 유형 집합을 제공합니다.

```go
// GetSignBytes Implements Msg.
func (msg MsgSetName) GetSignBytes() []byte {
	b, err := json.Marshal(msg)
	if err != nil {
		panic(err)
	}
	return sdk.MustSortJSON(b)
}
```

`GetSignBytes`는 `Msg`가 서명을 위해 인코딩되는 방법을 정의합니다. 대부분의 경우 저장된 JSON을 **마샬링(marshal)**합니다. 출력은 수정할 수 없습니다.

```go
// GetSigners Implements Msg.
func (msg MsgSetName) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{msg.Owner}
}
```

`GetSigners`는 유효한 서명을 위해 `Tx`에서 필요한 서명을 정의합니다. 이 경우 예를들어, `MsgSetName`은 이름이 가리키는 것을 재설정 하려고 할 때 소유자가 트랜잭션에 서명해야 합니다.

## **Handelr**

이제 `MsgSetName`이 지정 되었습니다. 다음 스탭은 이 메시지가 수신 될 때 취해야 할 조치를 정의하는 것입니다. 이것이 `Handler`의 역할입니다.

새로운 파일 `./x/nameservice/handler.go`에서  다음 코드로 시작하십시오.

```go
package nameservice

import (
	"fmt"

	sdk "github.com/cosmos/cosmos-sdk/types"
)

// NewHandler returns a handler for "nameservice" type messages.
func NewHandler(keeper Keeper) sdk.Handler {
	return func(ctx sdk.Context, msg sdk.Msg) sdk.Result {
		switch msg := msg.(type) {
		case MsgSetName:
			return handleMsgSetName(ctx, keeper, msg)
		default:
			errMsg := fmt.Sprintf("Unrecognized nameservice Msg type: %v", msg.Type())
			return sdk.ErrUnknownRequest(errMsg).Result()
		}
	}
}
```

`NewHandler`는 본질적으로 이 모듈로 들어오는 메시지를 적잘한 처리기로 보내는 하위 라우터입니다. 지금은 단 하나의 메시지 / `Msg/Handle`이 있습니다.

이제 `handleMsgSetName`에서 `MsgSetName` 메시지를 처리하기 위한 실제 로직을 정의해야 합니다.

> 참고: SDK에서 핸들러 네이밍 규칙은 `handlerMsg{{.Action}} 입니다.

```go
// Handle MsgSetName
func handleMsgSetName(ctx sdk.Context, keeper Keeper, msg MsgSetName) sdk.Result {
	if !msg.Owner.Equals(keeper.GetOwner(ctx, msg.NameID)) { // Checks if the the msg sender is the same as the current owner
		return sdk.ErrUnauthorized("Incorrect Owner").Result() // If not, throw an error
	}
	keeper.SetName(ctx, msg.NameID, msg.Value) // If so, set the name to the value specified in the msg.
	return sdk.Result{}                      // return
}
```

이 함수에서 `Msg`를 보낸 사람이 실제로 이름의 소유자인지 확인합니다 (`keeper.GetOwner`). 그렇다면 `Keeper`에서 함수를 호출하여 이름을 설정할 수 있습니다. 그렇지 않다면, 에러를 발생하여 사용자에게 반환합니다.



## BuyName



##  `Msg`

이제 `./nameservice/msgs.go` 파일에 이름을 구입하고 `Msg`를 정의해야 합니다. 이 코드는 `SetName`과 매우 유사합니다.

```go
// MsgBuyName defines the BuyName message
type MsgBuyName struct {
	NameID string
	Bid    sdk.Coins
	Buyer  sdk.AccAddress
}

// NewMsgBuyName is the constructor function for MsgBuyName
func NewMsgBuyName(name string, bid sdk.Coins, buyer sdk.AccAddress) MsgBuyName {
	return MsgBuyName{
		NameID: name,
		Bid:    bid,
		Buyer:  buyer,
	}
}

// Type Implements Msg.
func (msg MsgBuyName) Route() string { return "nameservice" }

// Name Implements Msg.
func (msg MsgBuyName) Type() string { return "buy_name" }

// ValidateBasic Implements Msg.
func (msg MsgBuyName) ValidateBasic() sdk.Error {
	if msg.Buyer.Empty() {
		return sdk.ErrInvalidAddress(msg.Buyer.String())
	}
	if len(msg.NameID) == 0 {
		return sdk.ErrUnknownRequest("Name cannot be empty")
	}
	if !msg.Bid.IsPositive() {
		return sdk.ErrInsufficientCoins("Bids must be positive")
	}
	return nil
}

// GetSignBytes Implements Msg.
func (msg MsgBuyName) GetSignBytes() []byte {
	b, err := json.Marshal(msg)
	if err != nil {
		panic(err)
	}
	return sdk.MustSortJSON(b)
}

// GetSigners Implements Msg.
func (msg MsgBuyName) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{msg.Buyer}
}
```

다음 으로, `./x/nameservice/handler.go` 파일에서, 모듈 라우터에 `MsgBuyName` 처리기를 추가합니다.

```go
// NewHandler returns a handler for "nameservice" type messages.
func NewHandler(keeper Keeper) sdk.Handler {
	return func(ctx sdk.Context, msg sdk.Msg) sdk.Result {
		switch msg := msg.(type) {
		case MsgSetName:
			return handleMsgSetName(ctx, keeper, msg)
		case MsgBuyName:
			return handleMsgBuyName(ctx, keeper, msg)
		default:
			errMsg := fmt.Sprintf("Unrecognized nameservice Msg type: %v", msg.Type())
			return sdk.ErrUnknownRequest(errMsg).Result()
		}
	}
```

마지막으로, 메시지에 의해 트리거 된 상태전이를 수행하는 `BuyName`의 `핸들러` 함수를 정의합니다. 이 시점에서 메시지에는 `ValidateBasic` 함수가 실행되어 일부 입력 검증합니다. 그러나 `ValidateBasic`은 어플리케이션 상태를 조회할 수 없습니다. 네트워크 상태(예, 계정 잔액)에 의존하는 유효성 검사 로직은 `handler` 기능에서 수행되어야 합니다.

```go
// Handle MsgBuyName
func handleMsgBuyName(ctx sdk.Context, keeper Keeper, msg MsgBuyName) sdk.Result {
	if keeper.GetPrice(ctx, msg.NameID).IsGTE(msg.Bid) { // Checks if the the bid price is greater than the price paid by the current owner
		return sdk.ErrInsufficientCoins("Bid not high enough").Result() // If not, throw an error
	}
	if keeper.HasOwner(ctx, msg.NameID) {
		_, err := keeper.coinKeeper.SendCoins(ctx, msg.Buyer, keeper.GetOwner(ctx, msg.NameID), msg.Bid)
		if err != nil {
			return sdk.ErrInsufficientCoins("Buyer does not have enough coins").Result()
		}
	} else {
		_, _, err := keeper.coinKeeper.SubtractCoins(ctx, msg.Buyer, msg.Bid) // If so, deduct the Bid amount from the sender
		if err != nil {
			return sdk.ErrInsufficientCoins("Buyer does not have enough coins").Result()
		}
	}
	keeper.SetOwner(ctx, msg.NameID, msg.Buyer)
	keeper.SetPrice(ctx, msg.NameID, msg.Bid)
	return sdk.Result{}
}
```

먼저, 입찰가가 현재 가격보다 높은지 먼저 확인합니다. 그런다음 이름에 이미 소유자가 있는지 확인합니다. 그럴 경우 이전 소유자는 `구매자`로부터 돈을 수령하게 됩니다.

만약 소유자가 없다면, `nameservice` 모듈은 `구매자`로 부터 소각(복구할 수 없는 주소로 보냄) 합니다.

`SubtractCoins` 또는 `SendCoins`가 nil이 아닌 오류를 반환하면, 핸들러가 오류를 발생시켜 상태 전이를 되돌립니다. 그렇지 않으면 이전에 `Keeper`에 정의된 getter 및 setter를 사용하여 핸들러가 구매자를 새 소유자로 설정하고 새 가격을 현재 입찰가로 설정합니다.

> 참고:  이 핸들러는 코인 운영을 위해 `coinKeeper`의 기능을 사용합니다. 어플리케이션에서 코인 운영 작업을 수행하는 경우  [godocs for this module](https://godoc.org/github.com/cosmos/cosmos-sdk/x/bank#BaseKeeper)를 참고하세요.



## 6. sdk.Codec을 사용하여 인코딩된 타입 등록



## Codec File

인코딩/디코딩 할 수 있도록 Amino에 유형을 등록([register your types with Amino](https://github.com/tendermint/go-amino#registering-types) )하려면 `./x/nameservice/codec.go`에 작성해야하는 약간의 코드가 있습니다. 생성 한 모든 인터페이스와 인터페이스를 구현하는 모근 구조체는 `RegisterCodec` 함수에서 선언해야 합니다. 이 모듈에서는 두 개의 `Msg` 구현(`SetName` 및 `BuyName`)을 등록해야 하지만 Whois 쿼리 반환 타입은 등록하지 않습니다.

```go
package nameservice

import (
	"github.com/cosmos/cosmos-sdk/codec"
)

// RegisterCodec registers concrete types on wire codec
func RegisterCodec(cdc *codec.Codec) {
	cdc.RegisterConcrete(MsgSetName{}, "nameservice/SetName", nil)
	cdc.RegisterConcrete(MsgBuyName{}, "nameservice/BuyName", nil)
}
```



## 7. sdk.Codec을 사용하여 인코딩된 타입 등록



## Nameservice Module CLI

코스모스 SDK는 CLI 인터페이스를 위해 [`cobra`](https://github.com/spf13/cobra) 라이브러리를 사용한다. 이 라이브러리는 각 모듈이 자신의 명령을 쉽게 노출 할 수 있게 합니다. 모듈과 사용자의 CLI 상호작용을 정의하려면 다음 파일을 작성하세요.

* `./x/nameservice/client/cli/query.go`
* `./x/nameservice/client/cli/tx.go`

`query.go`에서 시작하십시오. 여기에 각 모듈에 대한 커맨드 쿼리들(`Queries`)들을 위해 `cobra.Comman`를 정의하세요.(`resolve` 그리고 `whois`)

```go
package cli

import (
	"fmt"

	"github.com/cosmos/cosmos-sdk/client/context"
	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/spf13/cobra"
)

// GetCmdResolveName queries information about a name
func GetCmdResolveName(queryRoute string, cdc *codec.Codec) *cobra.Command {
	return &cobra.Command{
		Use:   "resolve [name]",
		Short: "resolve name",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			cliCtx := context.NewCLIContext().WithCodec(cdc)
			name := args[0]

			res, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/resolve/%s", queryRoute, name), nil)
			if err != nil {
				fmt.Printf("could not resolve name - %s \n", string(name))
				return nil
			}

			fmt.Println(string(res))

			return nil
		},
	}
}

// GetCmdWhois queries information about a domain
func GetCmdWhois(queryRoute string, cdc *codec.Codec) *cobra.Command {
	return &cobra.Command{
		Use:   "whois [name]",
		Short: "Query whois info of name",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			cliCtx := context.NewCLIContext().WithCodec(cdc)
			name := args[0]

			res, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/whois/%s", queryRoute, name), nil)
			if err != nil {
				fmt.Printf("could not resolve whois - %s \n", string(name))
				return nil
			}

			fmt.Println(string(res))

			return nil
		},
	}
}
```

앞의 코드에 대한 참고사항

* CLI가 새로운 [`CLIContext`](https://godoc.org/github.com/cosmos/cosmos-sdk/client/context#CLIContext) `컨텍스트`를 소개합니다. CLI 상호작용에 필요한 사용자 입력 및 어플리케이션 구성에 대한 데이터를 전달합니다.
* `cliCtx.QueryWithData()` 함수에 필요한 `경로`는 쿼리 라우터의 이름에 직접 매핑합니다.
  *  경로의 첫 번째 부분은 SDK 어플리케이션에서 가능한 쿼리 유형을 구분하는데 사용합니다. (의역: 사용자 정의된 쿼리를 구분하기위해 사용됩니다.)
  * 두 번째 부분(`nameservice`)은 조회를 라우팅할 모듈의 이름입니다.
  * 마지막으로 모듈에는 호출될 특정 쿼리가 있습니다.
  * 이 예에서 네 번째 부분이 쿼리입니다. 쿼리 매개변수는 간단한 문자열이기 때문에 동작합니다. 더 복잡한 쿼리입력을 사용하려면 `.QeuryWithData()` 함수의 두 번쨰 인수를 사용하여 데이터를 정달해야 합니다.

이제 쿼리 상호 작용이 정의되었으므로 `tx.go`의 트랜잭션 생성할 차례입니다.

> 참고: 여러분의 어플리케이션에서 방금 작성한 코드를 import해야 합니다. 여기에서 import 경로가 저장소로 설정됩니다. (`github.com/cosmos/sdk-application-tutorial/x/nameservice`) 자신이 설정한 경로에 따라 경로를 변경해야 합니다 (`github.com/{{ .Username }}/{{ .Project.Repo }}/x/nameservice`).

```go
package cli

import (
	"github.com/spf13/cobra"

	"github.com/cosmos/cosmos-sdk/client/context"
	"github.com/cosmos/cosmos-sdk/client/utils"
	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/cosmos/sdk-application-tutorial/x/nameservice"

	sdk "github.com/cosmos/cosmos-sdk/types"
	authcmd "github.com/cosmos/cosmos-sdk/x/auth/client/cli"
	authtxb "github.com/cosmos/cosmos-sdk/x/auth/client/txbuilder"
)

// GetCmdBuyName is the CLI command for sending a BuyName transaction
func GetCmdBuyName(cdc *codec.Codec) *cobra.Command {
	return &cobra.Command{
		Use:   "buy-name [name] [amount]",
		Short: "bid for existing name or claim new name",
		Args:  cobra.ExactArgs(2),
		RunE: func(cmd *cobra.Command, args []string) error {
			cliCtx := context.NewCLIContext().
				WithCodec(cdc).
				WithAccountDecoder(authcmd.GetAccountDecoder(cdc))

			txBldr := authtxb.NewTxBuilderFromCLI().WithCodec(cdc)

			if err := cliCtx.EnsureAccountExists(); err != nil {
				return err
			}

			coins, err := sdk.ParseCoins(args[1])
			if err != nil {
				return err
			}

			account, err := cliCtx.GetFromAddress()
			if err != nil {
				return err
			}

			msg := nameservice.NewMsgBuyName(args[0], coins, account)
			err = msg.ValidateBasic()
			if err != nil {
				return err
			}

			cliCtx.PrintResponse = true

			return utils.CompleteAndBroadcastTxCli(txBldr, cliCtx, []sdk.Msg{msg})
		},
	}
}

// GetCmdSetName is the CLI command for sending a SetName transaction
func GetCmdSetName(cdc *codec.Codec) *cobra.Command {
	return &cobra.Command{
		Use:   "set-name [name] [value]",
		Short: "set the value associated with a name that you own",
		Args:  cobra.ExactArgs(2),
		RunE: func(cmd *cobra.Command, args []string) error {
			cliCtx := context.NewCLIContext().
				WithCodec(cdc).
				WithAccountDecoder(authcmd.GetAccountDecoder(cdc))

			txBldr := authtxb.NewTxBuilderFromCLI().WithCodec(cdc)

			if err := cliCtx.EnsureAccountExists(); err != nil {
				return err
			}

			account, err := cliCtx.GetFromAddress()
			if err != nil {
				return err
			}

			msg := nameservice.NewMsgSetName(args[0], args[1], account)
			err = msg.ValidateBasic()
			if err != nil {
				return err
			}

			cliCtx.PrintResponse = true

			return utils.CompleteAndBroadcastTxCli(txBldr, cliCtx, []sdk.Msg{msg})
		},
	}
}
```

앞의 코드에 대한 참고사항

* `authcmd` 패키지는 여기에 사용됩니다. [여기로 이동하시면 더 많은 정보가 있습니다](https://godoc.org/github.com/cosmos/cosmos-sdk/x/auth/client/cli#GetAccountDecoder). CLI 제어되는 계정에 대한 액세스를 제공하고 서명을 용이하게 합니다.



## 8. 최종적으로 만들어진 모듈과 어플리케이션 import



## Import your modules and finish your application

지금 여러분의 모듈을 준비되었고, [`auth`](https://godoc.org/github.com/cosmos/cosmos-sdk/x/auth)와 [`bank`](https://godoc.org/github.com/cosmos/cosmos-sdk/x/bank) 두 모듈과 함께 `./app.go`  파일에 통합 될 수 있습니다.

> 참고: 여러분의 어플리케이션에서 방금 작성한 코드를 import해야 합니다. 여기에서 import 경로가 저장소로 설정됩니다. (`github.com/cosmos/sdk-application-tutorial/x/nameservice`) 자신이 설정한 경로에 따라 경로를 변경해야 합니다 (`github.com/{{ .Username }}/{{ .Project.Repo }}/x/nameservice`).

```go
package app

import (
	"github.com/tendermint/tendermint/libs/log"

	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/cosmos/cosmos-sdk/x/auth"
	"github.com/cosmos/cosmos-sdk/x/bank"
	"github.com/cosmos/sdk-application-tutorial/x/nameservice"

	bam "github.com/cosmos/cosmos-sdk/baseapp"
	sdk "github.com/cosmos/cosmos-sdk/types"
	abci "github.com/tendermint/tendermint/abci/types"
	cmn "github.com/tendermint/tendermint/libs/common"
	dbm "github.com/tendermint/tendermint/libs/db"
)
```

다음으로 `nameserviceApp` 구조체에 저장소의 키와 `Keepers`를 추가하고 이에 따라 생성자를 업데이트 해야합니다.

```go
const (
	appName = "nameservice"
)

type nameserviceApp struct {
	*bam.BaseApp
	cdc *codec.Codec

	keyMain          *sdk.KVStoreKey
	keyAccount       *sdk.KVStoreKey
	keyNSnames       *sdk.KVStoreKey
	keyNSowners      *sdk.KVStoreKey
	keyNSprices      *sdk.KVStoreKey
	keyFeeCollection *sdk.KVStoreKey

	accountKeeper       auth.AccountKeeper
	bankKeeper          bank.Keeper
	feeCollectionKeeper auth.FeeCollectionKeeper
	nsKeeper            nameservice.Keeper
}


func NewnameserviceApp(logger log.Logger, db dbm.DB) *nameserviceApp {

  // First define the top level codec that will be shared by the different modules
  cdc := MakeCodec()

  // BaseApp handles interactions with Tendermint through the ABCI protocol
  bApp := bam.NewBaseApp(appName, logger, db, auth.DefaultTxDecoder(cdc))

  // Here you initialize your application with the store keys it requires
	var app = &nameserviceApp{
		BaseApp: bApp,
		cdc:     cdc,

		keyMain:          sdk.NewKVStoreKey("main"),
		keyAccount:       sdk.NewKVStoreKey("acc"),
		keyNSnames:       sdk.NewKVStoreKey("ns_names"),
		keyNSowners:      sdk.NewKVStoreKey("ns_owners"),
		keyNSprices:      sdk.NewKVStoreKey("ns_prices"),
		keyFeeCollection: sdk.NewKVStoreKey("fee_collection"),
	}

  return app
}
```

이 시점에서 생성자는 여전히 중요한 논리가 부족합니다. 즉, 다음이 필요합니다.

* 각각의 모듈로부터 `Keeper`들을 요청하여 인스턴스화 합니다.
* 각각의 `Keeper`로 부터 요청된 `storeKey` 들을 생성합니다.
* 각 모듈에 `핸들러`를 등록합니다. `baseapp`의 라우터에서 `addRoute()` 메소드를 사용합니다.
* 각 모듈의 Queryiers를 등록합니다. 이 경우 `baseapp`의 queryRouter에 있는 `addRoute()` 메소드를 사용합니다.
* `baseApp` 멀티 스토어에 제공된 키에 `KVStores`를 마운트 하십시오.
* 초기 어플리케이션 상태를 정의하기 위해 `initChainer`를 설정합니다.

최종 생성자는 다음과 같습니다.

```go
// NewnameserviceApp is a constructor function for nameserviceApp
func NewnameserviceApp(logger log.Logger, db dbm.DB) *nameserviceApp {

	// First define the top level codec that will be shared by the different modules
	cdc := MakeCodec()

	// BaseApp handles interactions with Tendermint through the ABCI protocol
	bApp := bam.NewBaseApp(appName, logger, db, auth.DefaultTxDecoder(cdc))

	// Here you initialize your application with the store keys it requires
	var app = &nameserviceApp{
		BaseApp: bApp,
		cdc:     cdc,

		keyMain:          sdk.NewKVStoreKey("main"),
		keyAccount:       sdk.NewKVStoreKey("acc"),
		keyNSnames:       sdk.NewKVStoreKey("ns_names"),
		keyNSowners:      sdk.NewKVStoreKey("ns_owners"),
		keyNSprices:      sdk.NewKVStoreKey("ns_prices"),
		keyFeeCollection: sdk.NewKVStoreKey("fee_collection"),
	}

	// The AccountKeeper handles address -> account lookups
	app.accountKeeper = auth.NewAccountKeeper(
		app.cdc,
		app.keyAccount,
		auth.ProtoBaseAccount,
	)

	// The BankKeeper allows you perform sdk.Coins interactions
	app.bankKeeper = bank.NewBaseKeeper(app.accountKeeper)

	// The FeeCollectionKeeper collects transaction fees and renders them to the fee distribution module
	app.feeCollectionKeeper = auth.NewFeeCollectionKeeper(cdc, app.keyFeeCollection)

	// The NameserviceKeeper is the Keeper from the module for this tutorial
	// It handles interactions with the namestore
	app.nsKeeper = nameservice.NewKeeper(
		app.bankKeeper,
		app.keyNSnames,
		app.keyNSowners,
		app.keyNSprices,
		app.cdc,
	)

	// The AnteHandler handles signature verification and transaction pre-processing
	app.SetAnteHandler(auth.NewAnteHandler(app.accountKeeper, app.feeCollectionKeeper))

	// The app.Router is the main transaction router where each module registers it's routes
	// Register the bank and nameservice routes here
	app.Router().
		AddRoute("bank", bank.NewHandler(app.bankKeeper)).
		AddRoute("nameservice", nameservice.NewHandler(app.nsKeeper))

	// The app.QueryRouter is the main query router where each module registers it's routes
	app.QueryRouter().
		AddRoute("nameservice", nameservice.NewQuerier(app.nsKeeper))

	// The initChainer handles translating the genesis.json file into initial state for the network
	app.SetInitChainer(app.initChainer)

	app.MountStoresIAVL(
		app.keyMain,
		app.keyAccount,
		app.keyNSnames,
		app.keyNSowners,
		app.keyNSprices,
	)

	err := app.LoadLatestVersion(app.keyMain)
	if err != nil {
		cmn.Exit(err.Error())
	}

	return app
}
```

`initChainer`는 genesis.json의 계정이 초기체인 시작시 어플리케이션 상태로 매핑되는 방법을 정의합니다.

생성자는 `initChainer` 함수를 등록했지만 아직 정의되지 않았습니다. 계속해서 그것을 생성합니다.

```go
// GenesisState represents chain state at the start of the chain. Any initial state (account balances) are stored here.
type GenesisState struct {
	Accounts []auth.BaseAccount `json:"accounts"`
}

func (app *nameserviceApp) initChainer(ctx sdk.Context, req abci.RequestInitChain) abci.ResponseInitChain {
	stateJSON := req.AppStateBytes

	genesisState := new(GenesisState)
	err := app.cdc.UnmarshalJSON(stateJSON, genesisState)
	if err != nil {
		panic(err)
	}

	for _, acc := range genesisState.Accounts {
		acc.AccountNumber = app.accountKeeper.GetNextAccountNumber(ctx)
		app.accountKeeper.SetAccount(ctx, &acc)
	}

	return abci.ResponseInitChain{}
}
```

마지막으로 헬퍼함수를 추가하여 어플리케이션에서 사용하는 모든 모듈을 제대로 등록하는 amino [`*codec.Codec`](https://godoc.org/github.com/cosmos/cosmos-sdk/codec#Codec) 코덱을 생성합니다. 

```go
func MakeCodec() *codec.Codec {
	var cdc = codec.New()
	auth.RegisterCodec(cdc)
	bank.RegisterCodec(cdc)
	nameservice.RegisterCodec(cdc)
	sdk.RegisterCodec(cdc)
	codec.RegisterCrypto(cdc)
	return cdc
}
```



 ## 9. 어플리케이션을 위한 `nameserviced`와 `nameservicecli` 엔트리포인트(entry point == 연결지점?) 생성



## Entrypoints

golang에서 규칙은 프로젝트의 `./cmd` 디렉터리에 바이너리로 컴파일되는 파일을 저장하는 것 입니다. 어플리케이션을 만들려는 2가지 바이너리 파일이 있습니다

* `nameserviced`: 이 바이너리 파일은 p2p 연결을 하고 트랜잭션을 전파하고 로컬 저장소를 처리하며 네트워크와 상호 작용할 RPC 인터페이스를 제공하는 점에서 `bitcoind` 또는 다른 암호화페 데몬프로그램과 유사합니다. 이 경우 Tendermint는 네트워킹 및 거래 주문에 사용합니다.
* `Nameservicecli`: 이 바이너리 파일은 어플리케이션에서 사용자와 상호작용하는 명령어를 제공합니다.

시작하려면 바이너리를 인스턴스화 할 프로젝트를 디렉토리에 두 개의 파을을 작성합니다.

* `./cmd/nameserviced/main.go`
* `./cmd/nameservicecli/main.go`



##  `nameserviced`

`./nameserviced/main.go`에 다음 코드를 추가하여 시작하세요

> 참고: 여러분의 어플리케이션에서 방금 작성한 코드를 import해야 합니다. 여기에서 import 경로가 저장소로 설정됩니다 (`github.com/cosmos/sdk-application-tutorial`). 자신이 설정한 경로에 따라 경로를 변경해야 합니다 (`github.com/{{ .Username }}/{{ .Project.Repo }`).

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"os"

	"github.com/cosmos/cosmos-sdk/client"
	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/cosmos/cosmos-sdk/server"
	"github.com/spf13/cobra"
	"github.com/spf13/viper"
	"github.com/tendermint/tendermint/libs/cli"
	"github.com/tendermint/tendermint/libs/common"
	"github.com/tendermint/tendermint/libs/log"
	"github.com/tendermint/tendermint/p2p"

	gaiaInit "github.com/cosmos/cosmos-sdk/cmd/gaia/init"
	app "github.com/cosmos/sdk-application-tutorial"
	abci "github.com/tendermint/tendermint/abci/types"
	dbm "github.com/tendermint/tendermint/libs/db"
	tmtypes "github.com/tendermint/tendermint/types"
)

// DefaultNodeHome sets the folder where the applcation data and configuration will be stored
var DefaultNodeHome = os.ExpandEnv("$HOME/.nameserviced")

func main() {
	cobra.EnableCommandSorting = false

	cdc := app.MakeCodec()
	ctx := server.NewDefaultContext()

	appInit := server.AppInit{
		AppGenState: server.SimpleAppGenState,
	}

	rootCmd := &cobra.Command{
		Use:               "nameserviced",
		Short:             "nameservice App Daemon (server)",
		PersistentPreRunE: server.PersistentPreRunEFn(ctx),
	}

	rootCmd.AddCommand(InitCmd(ctx, cdc, appInit))

	server.AddCommands(ctx, cdc, rootCmd, appInit, newApp, exportAppStateAndTMValidators)

	// prepare and add flags
	executor := cli.PrepareBaseCmd(rootCmd, "NS", DefaultNodeHome)
	err := executor.Execute()
	if err != nil {
		// handle with #870
		panic(err)
	}
}

func newApp(logger log.Logger, db dbm.DB, traceStore io.Writer) abci.Application {
	return app.NewnameserviceApp(logger, db)
}

func exportAppStateAndTMValidators(logger log.Logger, db dbm.DB, traceStore io.Writer) (json.RawMessage, []tmtypes.GenesisValidator, error) {
	return nil, nil, nil
}

// get cmd to initialize all files for tendermint and application
// nolint: errcheck
func InitCmd(ctx *server.Context, cdc *codec.Codec, appInit server.AppInit) *cobra.Command {
	return &cobra.Command{
		Use:   "init",
		Short: "Initialize genesis config, priv-validator file, and p2p-node file",
		Args:  cobra.NoArgs,
		RunE: func(_ *cobra.Command, _ []string) error {

			config := ctx.Config
			config.SetRoot(viper.GetString(cli.HomeFlag))
			chainID := viper.GetString(client.FlagChainID)
			if chainID == "" {
				chainID = fmt.Sprintf("test-chain-%v", common.RandStr(6))
			}

			nodeKey, err := p2p.LoadOrGenNodeKey(config.NodeKeyFile())
			if err != nil {
				return err
			}
			nodeID := string(nodeKey.ID())

			pk := gaiaInit.ReadOrCreatePrivValidator(config.PrivValidatorFile())
			genTx, appMessage, validator, err := server.SimpleAppGenTx(cdc, pk)
			if err != nil {
				return err
			}

			appState, err := appInit.AppGenState(cdc, []json.RawMessage{genTx})
			if err != nil {
				return err
			}
			appStateJSON, err := cdc.MarshalJSON(appState)
			if err != nil {
				return err
			}

			toPrint := struct {
				ChainID    string          `json:"chain_id"`
				NodeID     string          `json:"node_id"`
				AppMessage json.RawMessage `json:"app_message"`
			}{
				chainID,
				nodeID,
				appMessage,
			}
			out, err := codec.MarshalJSONIndent(cdc, toPrint)
			if err != nil {
				return err
			}
			fmt.Fprintf(os.Stderr, "%s\n", string(out))
			return gaiaInit.WriteGenesisFile(config.GenesisFile(), chainID, []tmtypes.GenesisValidator{validator}, appStateJSON)
		},
	}
}

```

앞의 코드에 대한 참고사항

* 위 코드의 대부분은 다음과 같습니다.
  * Tendermint
  * Cosmos-SDK
  * 여러분의 Nameservice 모듈
* 나머지 코드는 어플리케이션 구성에서 초기 상태를 생성하는데 도와줍니다.



## `nameservicecli`

`nameservicecli` 명령을 작성하여 마무리 하세요.

> 참고: 여러분의 어플리케이션에서 방금 작성한 코드를 import해야 합니다. 여기에서 import 경로가 저장소로 설정됩니다 (`github.com/cosmos/sdk-application-tutorial`). 자신이 설정한 경로에 따라 경로를 변경해야 합니다 (`github.com/{{ .Username }}/{{ .Project.Repo }}`).

```go
package main

import (
	"os"

	"github.com/spf13/cobra"
	"github.com/tendermint/tendermint/libs/cli"
	"github.com/cosmos/cosmos-sdk/client"
	"github.com/cosmos/cosmos-sdk/client/keys"
	"github.com/cosmos/cosmos-sdk/client/rpc"
	"github.com/cosmos/cosmos-sdk/client/tx"

	authcmd "github.com/cosmos/cosmos-sdk/x/auth/client/cli"
	app "github.com/cosmos/sdk-application-tutorial"
	nameservicecmd "github.com/cosmos/sdk-application-tutorial/x/nameservice/client/cli"
)

const storeAcc = "acc"

var (
	rootCmd = &cobra.Command{
		Use:   "nameservicecli",
		Short: "nameservice Client",
	}
	defaultCLIHome = os.ExpandEnv("$HOME/.nameservicecli")
)

func main() {
	cobra.EnableCommandSorting = false
	cdc := app.MakeCodec()

	rootCmd.AddCommand(client.ConfigCmd())
	rpc.AddCommands(rootCmd)

	queryCmd := &cobra.Command{
		Use:     "query",
		Aliases: []string{"q"},
		Short:   "Querying subcommands",
	}

	queryCmd.AddCommand(
		rpc.BlockCommand(),
		rpc.ValidatorCommand(),
	)
	tx.AddCommands(queryCmd, cdc)
	queryCmd.AddCommand(client.LineBreak)
	queryCmd.AddCommand(client.GetCommands(
		authcmd.GetAccountCmd(storeAcc, cdc, authcmd.GetAccountDecoder(cdc)),
		nameservicecmd.GetCmdResolveName("nameservice", cdc),
		nameservicecmd.GetCmdWhois("nameservice", cdc),
	)...)

	txCmd := &cobra.Command{
		Use:   "tx",
		Short: "Transactions subcommands",
	}

	txCmd.AddCommand(client.PostCommands(
		nameservicecmd.GetCmdBuyName(cdc),
		nameservicecmd.GetCmdSetName(cdc),
	)...)

	rootCmd.AddCommand(
		queryCmd,
		txCmd,
		client.LineBreak,
	)

	rootCmd.AddCommand(
		keys.Commands(),
	)

	executor := cli.PrepareMainCmd(rootCmd, "NS", defaultCLIHome)
	err := executor.Execute()
	if err != nil {
		panic(err)
	}
}
```

참고:

* 위 코드의 대부분은 다음과 같습니다.
  * Tendermint
  * Cosmos-SDK
  * 여러분의 Nameservice 모듈
* [`cobra` CLI documentation](http://github.com/spf13/cobra) 문서는 앞의 코드를 이해하는데 도움을 줍니다.



## 10. dep을 이용하여 의존성 관리 설정



##  어플리케이션 빌드



## `Makefile`

일반 명령어가 포함된 Makefile을 작성하여 사용자가 응용프로그램을 빌드할 수 있습니다.

> 참고: 아래 Makefile에는 Cosmos SDK와 Tendermint Makefiles와 동일한 명령을 포함합니다.

```Makefile
DEP := $(shell command -v dep 2> /dev/null)

get_tools:
ifndef DEP
	@echo "Installing dep"
	go get -u -v github.com/golang/dep/cmd/dep
else
	@echo "Dep is already installed..."
endif

get_vendor_deps:
	@echo "--> Generating vendor directory via dep ensure"
	@rm -rf .vendor-new
	@dep ensure -v -vendor-only

update_vendor_deps:
	@echo "--> Running dep ensure"
	@rm -rf .vendor-new
	@dep ensure -v -update

install:
	go install ./cmd/nameserviced
	go install ./cmd/nameservicecli
```



## `Gopkg.toml`

고랭은 의존성을 관리하는 툴을 가지고 있습니다. 이 튜툐리얼에서는  [`dep`](https://golang.github.io/dep/)을 사용합니다. `dap` 저장소 루트에 있는 `Gopkg.toml` 파일을 사용하여 어플리케이션에 필요한 의존성을 정의합니다. 코스모드 SDK 어플리케이션은 현재 일부 라이브러리 특정 버전에 따라 다릅니다. 아래는 필요한 모든 버전이 포함되어 있습니다. 시작하려면 `./Gopkg.toml` 파일의 내용을 아래에 제약 조건 및 재정의로 바꾸세요.

```go
# Gopkg.toml example
#
# Refer to https://golang.github.io/dep/docs/Gopkg.toml.html
# for detailed Gopkg.toml documentation.
#
# required = ["github.com/user/thing/cmd/thing"]
# ignored = ["github.com/user/project/pkgX", "bitbucket.org/user/project/pkgA/pkgY"]
#
# [[constraint]]
#   name = "github.com/user/project"
#   version = "1.0.0"
#
# [[constraint]]
#   name = "github.com/user/project2"
#   branch = "dev"
#   source = "github.com/myfork/project2"
#
# [[override]]
#   name = "github.com/x/y"
#   version = "2.4.0"
#
# [prune]
#   non-go = false
#   go-tests = true
#   unused-packages = true

[[constraint]]
  name = "github.com/cosmos/cosmos-sdk"
  version = "v0.25.0"

[[override]]
  name = "github.com/golang/protobuf"
  version = "=1.1.0"

[[constraint]]
  name = "github.com/spf13/cobra"
  version = "~0.0.1"

[[constraint]]
  name = "github.com/spf13/viper"
  version = "~1.0.0"

[[override]]
  name = "github.com/tendermint/go-amino"
  version = "=v0.12.0"

[[override]]
  name = "github.com/tendermint/iavl"
  version = "=v0.11.0"

[[override]]
  name = "github.com/tendermint/tendermint"
  version = "=0.25.0"

[[override]]
  name = "golang.org/x/crypto"
  source = "https://github.com/tendermint/crypto"
  revision = "3764759f34a542a3aef74d6b02e35be7ab893bba"

[prune]
  go-tests = true
  unused-packages = true
```

> 참고: 만약 여러분의 저장소에서 시작한다면, 실행하기 전에 `dep ensure -v` 을 실행하기 전에 `github.com/cosmos/sdk-application-tutorial`을 여러분의 저장소 경로로 바꿔야 합니다. 그렇지 않으면 `dep ensure -v`를 디시 실행하기 전에 `./vendor` 디렉터리(`rm -rf ./vendor`)와 `Gopkg.lock` 파일(`rm Gopkg.lock`)을 제거해야 합니다.

이제 유지관리가 끝났으므로 의존성과 dep를 이용하여 설치를 진행합니다.

```bash
$ make get_tools
$ dep init -v
```



##  어플리케이션 빌드

```bash
# Update dependencies to match the constraints and overrides above
$ dep ensure # dep ensure -update -v 

# Install the app into your $GOBIN
$ make install

# Now you should be able to run the following commands:
$ nameserviced help
$ nameservicecli help
```



## 11. 어플리케이션 빌드 그리고 실행



##  Building And running the application



## 어플리케이션 `nameservice` 빌드

이 저장소에서 `nameservice` 어플리케이션을 빌드하여 기능을 확인하기 위해선 `dep`를 설치해야 합니다.

> 참고:  아래에는 `dep` 사이트의 쉘 스크립트를 사용하여 설치하는 명령어입니다. 만약 불편하다면 플랫폼별 [사용 지침서](https://golang.github.io/dep/docs/installation.html)를 사용하세요.

```bash
# Install dep
$ curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh

# Initialize dep and install dependencies
$ make get_tools && make get_vendor_deps

# Install the app into your $GOBIN
$ make install

# Now you should be able to run the following commands:
$ nameserviced help
$ nameservicecli help
```



## 라이브 네트워크 실행 및 명령 사용

어플리케이션에 대한 `genesis.json` 파일과 트랜잭션 계정을 초기화 하려면 다음을 실행합니다.

> 참고: init 명령어의 출력에서 chain-id를 복사하고, 두 번째 명령어의 출력에서 주소를 복사한 후 어플리케이션 명령을 조금 더 아래에서 실행할 때 사용할 수 있도록 저장합니다(즉, init을 통해 만들어진 chain-id와 keys add를 통해 만들어진 주소를 저장하여 뒤에서 사용할 수 있도록 저장해둡니다).

```bash
# Copy the chain_id and app_message.secret output here and save it for later user
$ nameserviced init

# Use app_message.secret recover jack's account. 
# Copy the `Address` output here and save it for later use
$ nameservicecli keys add jack --recover

# Create another account with random secret.
# Copy the `Address` output here and save it for later use
$ nameservicecli keys add tim
```

그런다음 생성 된 파일 `./.nameserviced/configgenesis.json` 파일을 텍스트 편집기에서 열고 `nameservicecli keys add` 명령의 주소 출력을 `"account"` 필드 아래의 `"address"` 값을 복사하세요. 이렇게 하면 로컬 네트워크를 시작할 때 코인이 있는 지갑을 제어할 수 있습니다.

여러분은 `nameserviced start` 호출을 통해 `nameserviced`를 사용할 수 있습니다. 블록이 생성되면서 발생한 로그를 실시간으로 볼 수 있습니다.

방금 생성한 네트워크에 대해 명령을 실행하려면 다른 터미널을 여세요.

> 참고:  아래 명령에서 `--chain-id` 및 `account address` 터미널 유틸니티를 사용하여 가져옵니다 (앞에서 init과 keys add를 통해 복사한 값을 넣어주면 됨). 또한 위의 네트워크 부트스트랩에서 저장된 원시 문자열을 입력 할 수도 있습니다. 다음 명령어를 사용하려면 컴퓨터에 [`jq`](https://stedolan.github.io/jq/download/)를 설치해야 합니다.

```bash
# First check the accounts to ensure they have funds
$ nameservicecli query account $(nameservicecli keys list -o json | jq -r .[0].address) \
    --indent --chain-id $(cat ~/.nameserviced/config/genesis.json | jq -r .chain_id) 
$ nameservicecli query account $(nameservicecli keys list -o json | jq -r .[1].address) \
    --indent --chain-id $(cat ~/.nameserviced/config/genesis.json | jq -r .chain_id) 

# Buy your first name using your coins from the genesis file
$ nameservicecli tx buy-name jack.id 5mycoin \
    --from     $(nameservicecli keys list -o json | jq -r .[0].address) \
    --chain-id $(cat ~/.nameserviced/config/genesis.json | jq -r .chain_id)

# Set the value for the name you just bought
$ nameservicecli tx set-name jack.id 8.8.8.8 \
    --from     $(nameservicecli keys list -o json | jq -r .[0].address) \
    --chain-id $(cat ~/.nameserviced/config/genesis.json | jq -r .chain_id)

# Try out a resolve query against the name you registered
$ nameservicecli query resolve jack.id --chain-id $(cat ~/.nameserviced/config/genesis.json | jq -r .chain_id)
# > 8.8.8.8

# Try out a whois query against the name you just registered
$ nameservicecli query whois jack.id --chain-id $(cat ~/.nameserviced/config/genesis.json | jq -r .chain_id)
# > {"value":"8.8.8.8","owner":"cosmos1l7k5tdt2qam0zecxrx78yuw447ga54dsmtpk2s","price":[{"denom":"mycoin","amount":"5"}]}

# Jack send some coin to tim, then tim can buy name from jack.  
$ nameservicecli tx send \
    --from     $(nameservicecli keys list -o json | jq -r .[0].address) \
    --to       $(nameservicecli keys list -o json | jq -r .[1].address) \
    --chain-id $(cat ~/.nameserviced/config/genesis.json | jq -r .chain_id) \
    --amount 1000mycoin

# Tim buy name from jack
$ nameservicecli tx buy-name jack.id 10mycoin \
    --from     $(nameservicecli keys list -o json | jq -r .[0].address) \
    --chain-id $(cat ~/.nameserviced/config/genesis.json | jq -r .chain_id)

```

