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



