<!--
order: 4
-->

# 노드 클라이언트 (Daemon)

SDK 애플리케이션의 주요 엔드포인트는 damon 클라이언트, 아니면 풀노드 클라이언트입니다. 풀노드는 상태 기계를 수행하고, 제네시스 파일로부터 시작합니다. 이 
클라이언트는 트랜잭션을 받고, 전달하고, 블록을 제안하고 서명하기 위해 똑같은 클라이언트를 실행하는 피어들과 연결되어 있습니다. 풀노드는 Cosmos SDK 로 정의된 
애플리케이션과 ABCI 를 통해 애플리케이션에 연결된 합의 엔진으로 구성됩니다. {synopsis}

## 사전 요구 학습

- [SDK 애플리케이션 해부](../basics/app-anatomy.md) {prereq}

## `main` 함수

모든 SDK 애플리케이션의 풀노드 클라이언트는 `main` 함수 수행으로 빌드됩니다. 클라이언트는 일반적으로 애플리케이션 이름 뒤에 `-d` 를 추가하고 (예 `appd` 
의 애플리케이션 이름은 `app`), `main` 함수는 `./appd/cmd/main.go` 파일에 정의됩니다. 이 함수를 수행해 명령어 셋과 함께 제공되는 실행가능한 `appd` 가
생성됩니다. `app` 이름을 가진 앱의 풀노드를 실행하는 주 명령어는 [`appd start`](#start-command) 가 됩니다.

일반적으로, 개발자는 다음의 구조로 `main.go` 함수를 구현할 것입니다:
 
- 먼저, 애플리케이션을 위한 [`appCodec`](./encoding.md) 을 인스턴스화 합니다.
- 그리고 `config` 를 검색하고 설정 매개 변수가 설정됩니다. 이는 주로 [주소](../basics/accounts.md#addresses) 에 대한 Bech32 프리픽스 설정이 
  포함됩니다.  
  +++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/types/config.go#L13-L24

- [Cobra](https://github.com/spf13/cobra) 를 사용해 풀노드 클라이언트의 루트 명령어가 생성됩니다. 그런 다음 애플리케이션의 모든 커스텀 명령어는 
  `roomCmd` 의 `AddCommnad()` 를 사용해 추가됩니다.
- `server.AddCommnads()` 를 사용해 `roomCmd` 에 기본 서버 명령어들을 추가합니다. 이 명령어들은 표준이며 SDK 수준에서 정의되므로 위에서 추가한 
  명령어들과는 구분됩니다. 이 명령어들은 모든 SDK 기반 애플리케이션에서 공유되어야 합니다. 가장 중요한 [`start` 명령어](#start-command) 도 여기 포함되어 
  있습니다.
- `executor` 의 준비 및 실행  
  +++ https://github.com/tendermint/tendermint/blob/v0.34.0-rc6/libs/cli/setup.go#L74-L78

데모용 SDK 애플리케이션인 `simapp` 의 `main` 함수 예제를 참조하세요:  
+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/simapp/simd/main.go

## `start` 명령어

`start` 명령어는 Cosmos SDK 의 `/server` 폴더에 정의되어 있습니다.  [`main` 함수](#main-function) 에서 풀노드 클라이언트의 루트 명령어로 
추가되어있으며, 엔드유저가 자신의 노드를 시작하기 위해 호출합니다.

```bash
# "app" 이름의 앱 예시. 다음 명령어로 풀노드를 시작합니다.
appd start

# SDK 의 simapp. 다음 명령어로 simapp 노드를 시작합니다.
simd start
```

참고로 풀노드는 세 가지 개념적 레이어로 구성됩니다: 네트워킹 레이어, 합의 레이어 그리고 애플리케이션 레이어. 첫 번째와 두 번째는 합의 엔진 (기본은 
Tendermint 코어) 이라고 불리는 엔티티에 함께 묶여있습니다. 세 번째 Cosmos SDK 의 도움을 받아 정의된 상태 기계입니다. 현재 Cosmos SDK 는 
Tendermint 를 기본 합의 엔진으로 사용하는데, Tendermint 노드를 부팅하기 위한 시작 명령어가 구현되어 있습니다.

`start` 명령어의 흐름은 매우 간단합니다. 먼저 `db` (디폴트는 [`leveldb`](https://github.com/syndtr/goleveldb) 인스턴스) 를 열기 위해 
`context` 에서 `config` 를 검색합니다. 이 `db` 는 애플리케이션에 대해 알고있는 최신 상태를 포함하고 있습니다. (이 애플리케이션이 처음이라면 비어있음)  

`start` 명령어는 `db` 와 `appCreator` 함수를 사용해 애플리케이션의 새 인스턴스를 생성합니다.  
+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/server/start.go#L227-L228

`appCreator` 는 `AppCreator` 시그니처를 만족하는 함수입니다:  
+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/server/types/app.go#L48-L50

실제로, [애플리케이션의 생성자](../basics/app-anatomy.md#constructor-function) 는 `appCreator` 로 전달됩니다.  
+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/simapp/simd/cmd/root.go#L170-L215

그리고나서 `app` 의 인스턴스를 사용해 새로운 Tendermint 노드를 인스턴스화 합니다.  
+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/server/start.go#L235-L244

`app` 이 [`abci.Application` 인터페이스](https://github.com/tendermint/tendermint/blob/v0.34.0/abci/types/application.go#L7-L32) 
(`app` 이 [`baseapp`](./baseapp.md) 의 확장인 경우) 를 만족하기 때문에 Tendermint 노드는 `app` 으로 생성될 수 있습니다. `NewNode` 메소드의 
일부분으로, Tendermint 는 애플리케이션의 높이 (즉, 제네시스 블록으로부터의 블록넘버) 가 Tendermint 노드의 높이와 동일한지 확인합니다. 두 높이의 차이는 
항상 음수나 null 이 되어야 합니다. 음수라면, `NewNode` 는 애플리케이션의 높이가 Tendermint 노드의 높이에 도달할 때까지 블록을 리플레이할 것입니다. 
마지막으로, 만약 애플리케이션의 높이가 `0` 이라면 Tendermint 노드는 제네시스 파일을 사용해 상태를 초기화하기 위해 애플리케이션에서
[`InitChain`](./baseapp.md#initchain) 을 호출할 것입니다.

Tendermint 노드가 인스턴스화되고 애플리케이션과 동기화되면, 노드는 시작할 수 있습니다:  
+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/server/start.go#L250-L252

노드가 시작되면, 노드는 P2P 서버와 RPC 를 부트스트랩하고 피어에 연결을 시작합니다. 피어들과 핸드셰이크 중, 만약 다른 피어들이 앞서 있다는 것 알아채면, 노드는 
따라잡기 위해 모든 블록들을 순차적으로 쿼리할 것입니다. 그러고나서 새로운 블록 제안과 Validator 로부터 블록 처리를 위한 블록 서명을 기다립니다.

## 다른 명령어들

노드를 구체적으로 실행하고, 노드와 상호작용하는 방법은 [노드 수행, API 와 CLI](../run-node/README.md) 가이드를 참조하세요.

## Next {hide}

[store](./store.md) 에 대해서 알아보세요. {hide}