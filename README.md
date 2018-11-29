# SDK 어플리케이션 튜토리얼 ([원문 보러가기](https://github.com/cosmos/sdk-application-tutorial/tree/master/tutorial))

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


## 튜토리얼 파트

1. [어플리케이션 설명](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/1.%20%EC%96%B4%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98%20%EC%84%A4%EB%AA%85.md)
2. [`./app.go`에서 어플리케이션 구현 실행](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/2.app.go%EC%97%90%EC%84%9C%20%EC%96%B4%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98%20%EA%B5%AC%ED%98%84%20%EC%8B%A4%ED%96%89.md)
3. [`keeper`에서 모듈 구현](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/3.%20keeper%EC%97%90%EC%84%9C%20%EB%AA%A8%EB%93%88%EA%B5%AC%ED%98%84.md)
4. [`Msgs`와 `Handlers`를 통해 상태 트랜잭션 정의](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/4.%20Msgs%EC%99%80%20Handlers%EB%A5%BC%20%ED%86%B5%ED%95%B4%20%EC%83%81%ED%83%9C%20%ED%8A%B8%EB%9E%9C%EC%9E%AD%EC%85%98%20%EC%A0%95%EC%9D%98(SetName%2C%20BuyName).md)
   - SetName 
   - BuyName 
5. [`Queriers`에서 상태머신 조회](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/5.%20queryiers%EC%97%90%EC%84%9C%20%EC%83%81%ED%83%9C%EB%A8%B8%EC%8B%A0%20%EC%A1%B0%ED%9A%8C.md)
6. [`sdk.Codec`을 사용하여 인코딩된 타입 등](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/6.%20sdk.Codec%EC%9D%84%20%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC%20%EC%9D%B8%EC%BD%94%EB%94%A9%EB%90%9C%20%ED%83%80%EC%9E%85%20%EB%93%B1%EB%A1%9D.md)
7. [모듈과 상호작용하는 CLI 생성](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/7.%20%EB%AA%A8%EB%93%88%EA%B3%BC%20%EC%83%81%ED%98%B8%EC%9E%91%EC%9A%A9%ED%95%98%EB%8A%94%20CLI%20%EC%83%9D%EC%84%B1.md)
8. [최종적으로 만들어진 어플리케이션 import](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/8.%20%EC%B5%9C%EC%A2%85%EC%A0%81%EC%9C%BC%EB%A1%9C%20%EB%A7%8C%EB%93%A4%EC%96%B4%EC%A7%84%20%EB%AA%A8%EB%93%88%EA%B3%BC%20%EC%96%B4%ED%94%8C%EB%A6%AC%EC%BC%80%EC%9D%B4%EC%85%98%20import.md)
9. [어플리케이션을 위한 `nameserviced`와 `nameservicecli` 엔트리포인트 생성](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/9.%20%EC%97%94%ED%8A%B8%EB%A6%AC%ED%8F%AC%EC%9D%B8%ED%8A%B8%20%EC%83%9D%EC%84%B1.md)
10. [`dep`을 이용하여 의존성 관리 설정](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/10.%20dep%EC%9D%84%20%EC%9D%B4%EC%9A%A9%ED%95%9C%20%EC%9D%98%EC%A1%B4%EC%84%B1%20%EA%B4%80%EB%A6%AC%20%EC%84%A4%EC%A0%95.md)
11. [어플리케이션 빌드 그리고 실행](https://github.com/pjt3591oo/cosmos-sdk-translate/blob/master/11.%20%EB%B9%8C%EB%93%9C%2C%20%EC%8B%A4%ED%96%89.md)
