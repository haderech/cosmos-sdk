<!--
order: 4
-->

# 노드 클라이언트 (데몬)

SDK 애플리케이션의 주요 엔드포인트는 풀-노드 클라이언트라고도 하는 데몬 클라이언트입니다. 풀-노드는 genesis 파일로부터 상태 기계를 동작시킵니다. 또한 트랜잭션을 수신하고 중계하기 위하여 동일한 클라이언트를 실행하는 피어에 접속합니다. 풀-노드는 Cosmos SDK의 정의에 의해 애플리케이션과 해당 애플리케이션에 ABCI를 통하여 연결된 합의 엔진으로 구성됩니다. {synopsis}

## 사전 요구 지식

- [SDK 애플리케이션의 해부](../basics/app-anatomy.md) {prereq}

## `main` 함수

모든 SDK 애플리케이션의 풀-노드 클라이언트는 `main` 함수를 실행하여 구축됩니다. 클라이언트는 일반적으로 애플리케이션의 이름에 `-d` 접미사를 추가하여 명명되고(e.g. `app` 애플리케이션을 위한 `appd`, `main` 함수는 `./appd/cmd/main.go` 파일에서 정의됩니다. 이 함수를 실행하면 명령어의 집합과 함께 제공되는 `appd` 실행 파일이 생성됩니다. `app`이라는 앱의 경우 주요 명령어는 [`appd start`](#start-명령어)이며 풀-노드를 실행합니다.

일반적으로 개발자는 다음의 구조를 사용하여 `main.go` 함수를 구현합니다:

- 먼저 애플리케이션을 위하여 [`appCodec`](./encoding.md)을 인스턴스화합니다.
- 그 후 `config`를 불러오고 config 매개 변수가 설정됩니다. 여기에는 [주소(addresses)](../basics/accounts.md#addresses)에 대한 Bech32 접두사 설정이 주로 포함되어 있습니다.
  +++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/types/config.go#L13-L24
- [cobra](https://github.com/spf13/cobra)를 사용하여 풀-노드의 루트 명령어를 생성합니다. 그 후, `rootCmd`의 `AddCommand()` 메서드를 사용하여 애플리케이션의 모든 사용자 지정 명령어를 추가합니다.
- `server.AddCommands()` 메서드를 사용하여 기본 서버 명령어를 `rootCmd`에 추가합니다. 이 명령어들은 위 항목의 명령어들과는 별개로 구분되는데 그 이유는 표준이면서 SDK 수준에서 정의되었고 모든 SDK 기반의 애플리케이션과 공유되어야하기 때문입니다. 이 명령어들은 가장 중요한 명령어인 [`start` command](#start-명령어)를 포함합니다.
- `executor`를 준비하고 실행합니다.
   +++ https://github.com/tendermint/tendermint/blob/v0.34.0-rc6/libs/cli/setup.go#L74-L78

SDK의 데모용 애플리케이션인 `simapp` 애플리케이션에서 다음의 `main` 함수의 예를 참조하십시오:
+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/simapp/simd/main.go

## `start` 명령어

`start` 명령어는 Cosmos SDK의 `/server` 폴더에 정의되어 있습니다. 이 명령어는 [`main` 함수](#main-함수) 안에서 풀-노드의 루트 명령어에 추가되고 최종 사용자가 노드를 시작하기 위해 다음과 같이 호출됩니다:

```bash
# "app"이라는 이름의 예제 앱의 경우, 다음의 명령어를 사용하여 풀-노드를 시작하십시오.
appd start

# SDK의 자체 simapp을 사용하는 경우, 다음의 명령어를 사용하여 simapp 노드를 시작하십시오.
simd start
```

풀-노드는 네트워크 레이어와, 합의 레이어, 애플리케이션 레이어, 이 세 가지 개념적(conceptual) 레이어들로 구성되어 있다는 것을 상기하십시오. 처음 두 레이어는 일반적으로 합의 엔진(기본값은 Tendermint Core)이라는 엔티티로 함께 묶이며, 세번째 레이어는 Cosmos SDK의 도움을 받아 상태 기계로 정의됩니다. 현재 Cosmos SDK는 Tendermint를 기본 합의 엔진으로 사용하는데, 이는 start 명령어가 Tendermint 노드를 부팅하도록 구현되었음을 의미합니다.

`start` 명령어의 흐름은 꽤 간단명료합니다. 우선 `db`(기본값은 [`leveldb`](https://github.com/syndtr/goleveldb) 인스턴스) `context`로부터 `config`를 불러옵니다. 이 `db`는 애플리케이션에서의 알려진 가장 최신의 상태를 포함하고 있습니다(애플리케이션이 최초로 생성한경우 비어 있음).

`start` 명령어는 `appCreator` 함수를 사용하여 애플리케이션의 새로운 인스턴스를 `db`와 함께 생성합니다:
+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/server/start.go#L227-L228

`appCreator`는 `AppCreator` 시그니쳐를 충족하는 함수라는 점을 참고하십시오:
+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/server/types/app.go#L48-L50

실제로 [애플리케이션의 생성자](../basics/app-anatomy.md#생성자(constructor)-함수)는 `appCreator`로서 전달됩니다.
+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/simapp/simd/cmd/root.go#L170-L215

그 다음에, `app`의 인스턴스를 사용하여 새로운 Tendermint 노드를 인스턴스화합니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/server/start.go#L235-L244

Tendermint 노드는 `app`을 사용하여 생성할 수 있는데, 그 이유는 `app`이 [`abci.Application` 인터페이스](https://github.com/tendermint/tendermint/blob/v0.34.0/abci/types/application.go#L7-L32) (`app`이 [`baseapp`](./baseapp.md)을 확장하므로)를 충족하기 때문입니다. `NewNode` 메서드의 도중, Tendermint는 애플리케이션의 높이(height, i.e. genesis로부터의 블록 갯수)가 Tendermint 노드의 높이와 동일한지를 확인합니다. 두 높이간의 차이는 언제나 음수이거나 null이어야 합니다. 엄밀하게는 음수의 경우, `NewNode`는 애플리케이션의 높이가 Tendermint 노드의 높이와 동일해질 때까지 블록을 재실행(replay)합니다. 마지막으로 애플리케이션의 높이가 `0`인 경우, Tendermint 노드는 애플리케이션에 [`InitChain`](./baseapp.md#initchain)을 호출하여 genesis 파일로부터 상태를 초기화합니다.

Tendermint 노드가 인스턴스화되고 애플리케이션과 동기화되면 노드는 시작할 수 있습니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/server/start.go#L250-L252

노드가 시작되면, 노드는 RPC와 P2P서버를 부트스트랩하고 피어에 연결하기 시작합니다. 피어와 핸드셰이크 도중에 자신이 피어보다 뒤쳐졌음을 인지하면, 노드는 모든 블록을 순차적으로 쿼리하여 따라잡습니다. 그 후 작업을 위해 새로운 블록 제안과 검증자(validator)로부터의 블록 서명을 기다립니다.

## 다른 명령어

노드를 구체적으로 실행 및 상호작용하는 방법에 대한 자세한 내용은 [노드와, API, CLI 실행하기](../run-node/README.md)가이드를 참조하십시오.

## 다음 {hide}

[스토어(store)](./store.md)에 대하여 알아봅시다 {hide}
