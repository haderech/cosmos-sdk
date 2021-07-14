<!--
order: 1
-->

# SDK 애플리케이션의 해부

이 문서는 Cosmos SDK 애플리케이션의 핵심적인 부분을 설명합니다. 문서 전반에 걸쳐 `app`이라는 이름의 플레이스홀더(placeholder) 애플리케이션이 사용됩니다. {synopsis}

## 노드 클라이언트

데몬, 또는 [풀-노드(Full-Node) 클라이언트](../core/node.md)는 SDK-기반 블록체인의 핵심 절차입니다. 네트워크 참여자는 이 절차를 실행하여 상태 기계를 초기화하고, 다른 풀-노드에 연결하고, 새로운 블록이 들어왔을때 상태기계를 업데이트합니다.

```
                ^  +-------------------------------+  ^
                |  |                               |  |
                |  |       상태 기계 = 애플리케이션ㅤㅤㅤㅤ|  |
                |  |                               |  |   Cosmos SDK로 빌드
                |  |            ^      +           |  |
                |  +----------- | ABCI | ----------+  v
                |  |            +      v           |  ^
                |  |                               |  |
블록체인 노드     ㅤ|  |             ㅤ합의        ㅤㅤㅤㅤ|  |
                |  |                               |  |
                |  +-------------------------------+  |   Tendermint 코어
                |  |                               |  |
                |  |             네트워크        ㅤㅤㅤ|  |
                |  |                               |  |
                v  +-------------------------------+  v
```

블록체인 풀-노드는 자신을 바이너리로 나타내며, 일반적으로 데몬(daemon)에는 `-d`(e.g. `app`의 경우 `appd`, 또는 `gaia`의 경우 `gaiad`)의 접미사를 붙입니다. 이 바이너리는 `./cmd/appd/` 위치에 있는 간단한 [`main.go`](../core/node.md#main-함수)를 실행하여 빌드됩니다. 이 동작은 일반적으로 [Makefile](#의존성과-makefile)을 통하여 수행됩니다.

주요 바이너리가 빌드되면, 노드는 [`start` 명령어](../core/node.md#start-명령어)를 실행하는 것을 통해 시작될 수 있습니다. 이 명령어 함수는 주로 세가지 일을 실행합니다:

1. [`app.go`](#코어-애플리케이션-파일)에 정의된 상태 기계의 인스턴스를 생성합니다.
2. `~/.app/data` 폴더의 `db`에서 추출된 가장 최신의 알려진 상태로 상태 기계를 초기화합니다. 이 시점에, 상태 기계는 `appBlockHeight`의 높이(height)를 갖습니다.
3. 새로운 Tendermint 인스턴스를 생성하고 시작합니다. 무엇보다도, 노드는 피어(peer)들과 핸드셰이크(handshake)를 수행합니다. 노드는 피어들로부터 최신의 `blockHeight`를 얻을 수 있고, 이 높이가 `appBlockHeight`보다 높은 경우, 해당 높이까지 동기하기 위해 블록들을 재실행(replay) 합니다. `appBlockHeight`이 `0`인 경우 노드는 제네시스(genesis)로부터 시작하고, Tendermint는 ABCI를 통하여 `InitChain` 메시지를 `app`에 전송하면 [`InitChainer`](#initchainer)가 트리거됩니다.

## 코어 애플리케이션 파일

일반적으로 상태머신의 핵심은 `app.go`라는 파일에 정의됩니다. 이 파일은 주로 **애플리케이션의 타입 정의**와 **생성 및 초기화**를 위한 함수를 포함하고 있습니다.

### 애플리케이션의 유형 정의

`app.go`에서 가장 먼저 정의한 것은 애플리케이션의 `타입`입니다. 일반적으로 다음과 같이 구성됩니다:

- **[`baseapp`](../core/baseapp.md)에 대한 참조.** `app.go`에 정의된 사용자 지정 애플리케이션은 `baseapp`의 확장(extension)입니다. 트랜잭션이 Tendermint로부터 애플리케이션으로 중계되면, `app`은 `baseapp`의 메서드를 사용하여 적절한 모듈로 라우팅합니다. `baseapp`은 모든 [ABCI 메서드](https://tendermint.com/docs/spec/abci/abci.html#overview)와 [라우팅 로직](../core/baseapp.md#routing)을 포함한 애플리케이션을 위한 대부분의 핵심 로직을 구현하고 있습니다.
- **스토어 키(key) 목록**. 모든 상태를 포함하고 있는 [스토어(store)](../core/store.md)는 Cosmos SDK에서 [`멀티스토어(multistore)`](../core/store.md#multistore) (i.e. 스토어들의 스토어)로 구현됩니다. 각 모듈은 멀티스토어에 있는 하나 혹은 복수의 스토어를 사용하여 상태를 유지합니다. 이 스토어들은 `app` 유형으로 선언된 특정 키를 사용하여 접근할 수 있습니다. 이 키는 `keepers`와 함께 Cosmos SDK의 [오브젝트-자격](../core/ocap.md)모델의 핵심입니다.
- **모듈 `keeper` 목록.** 각 모듈은 모듈의 스토어(들)에 대한 읽기 및 쓰기를 처리하는 [`keeper`](../building-modules/keeper.md)라는 이름의 추상화를 정의합니다. 한 모듈의 `keeper`의 메서드는 (승인된 경우) 다른 모듈로부터 호출될 수 있으며, 이를 위해 애플리케이션의 유형으로 선언하고 다른 모듈이 승인된 함수에만 접근할 수 있도록 인터페이스로 추출합니다.
- **[`appCodec`](../core/encoding.md)에 대한 참조.** 애플리케이션의 `appCodec`은 자료 구조를 저장하기 위해 직렬화(serialize) 및 역직렬화(deserialize)하기 위해 사용됩니다. 스토어는 오직 `[]bytes`만 유지할 수 있기 때문입니다. 기본 코덱은 [Protocol Buffers](../core/encoding.md)입니다.
- **[`legacyAmino`](../core/encoding.md) 코덱에 대한 참조.** SDK의 일부는 상기의 `appCodec`을 사용하도록 이전(migrate)되지 않았으며 아직도 Amino를 사용하도록 하드코딩되어 있습니다. 다른 일부는 하위 호환성을 위해 명시적으로 Amino를 사용합니다. 이러한 이유로, 애플리케이션은 여전히 레거시 Amino 코덱에 대한 참조를 보유하고 있습니다. 이후의 릴리스에서는 SDK에서 Amino코덱이 제거될 것이라는 점을 참고하십시오.
- **모듈 매니져([module manager](../building-modules/module-manager.md#manager))와 기초 모듈 매니져([basic module manager](../building-modules/module-manager.md#basicmanager))에 대한 참조.** 모듈 매니저는 애플리케이션의 모듈 목록을 포함하는 객체입니다. 이는 [`Msg` 서비스](../core/baseapp.md#msg-services), [gRPC `Query` 서비스](../core/baseapp.md#grpc-query-services), 혹은 [`InitChainer`](#initchainer), [`BeginBlocker`와 `EndBlocker`](#beginblocker와-endblocker) 같은 모듈간에 실행된 다양한 함수들의 동작 순서를 지정하는 등의 작업을 용이하게 해줍니다.

데모 및 테스트 용도로 사용되는 SDK의 자체 앱인 `simapp`의 애플리케이션 유형 정의를 참조하십시오:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/simapp/app.go#L145-L187

### 생성자(Constructor) 함수

이 함수는 위의 섹션에서 기술한 유형들의 애플리케이션을 생성합니다. 애플리케이션의 데몬 명령어 중 [`start` 명령어](../core/node.md#start-명령어)를 사용하기 위하여 `AppCreator` 시그니쳐를 충족해야만 합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/server/types/app.go#L48-L50

다음은 이 함수에 의해 수행되는 주요 작업입니다:

- 새로운 [`코덱(codec)`](../core/encoding.md)을 인스턴스화하고 [basic manager](../building-modules/module-manager.md#basicmanager)를 사용하여 각 애플리케이션 모듈의 `코덱`을 초기화합니다.
- `baseapp` 인스턴스 및 코덱과 모든 적절한 스토어 키에 대한 참조를 사용하여 새로운 애플리케이션을 인스턴스화합니다.
- 애플리케이션의 `타입`에 정의된 모든 [`keeper`들](#keeper)을 각 애플리케이션 모듈의 `NewKeeper` 함수를 사용하여 초기화합니다. 한 모듈의 `NewKeeper`는 다른 모듈의 `NewKeeper`에 대한 참조를 해야할 수 있으므로, `keeper`들은 올바른 순서로 인스턴스화해야하는 점을 참고하십시오.
- 각 애플리케이션 모듈의 [`AppModule`](#애플리케이션-모듈-인터페이스) 객체를 사용하여 [모듈 매니져(`module manager`)](../building-modules/module-manager.md#manager)를 초기화합니다.
- 모듈 매니져를 사용하여 애플리케이션의 [`Msg` 서비스](../core/baseapp.md#msg-services)와, [gRPC `Query` 서비스](../core/baseapp.md#grpc-query-services), [레거시 `Msg` 라우팅](../core/baseapp.md#routing) 그리고 [레거시 쿼리(query) 라우팅](../core/baseapp.md#query-routing)을 초기화합니다. Tendermint에 의해 트랜잭션이 ABCI를 통하여 애플리케이션에 중계될 때, 트랜잭션은 이 곳에서 정의된 경로(route)를 통하여 적절한 모듈의 [`Msg` 서비스](#msg-서비스)로 라우팅됩니다. 마찬가지로, 애플리케이션에서 gRPC 쿼리 요청을 받았을 때, 이 곳에서 정의된 gRPC 경로를 통하여 적절한 모듈의 [`gRPC query service`](#grpc-쿼리(query)-서비스)로 라우팅됩니다. SDK는 레거시 `Msg` 경로를 사용하는 레거시 `Msg`와 레거시 쿼리 경로를 사용하는 레거시 Tendermint 쿼리를 아직 지원하고 있습니다.
- 모듈 매니져를 사용하여 [애플리케이션 모듈 불변자(invariants)](../building-modules/invariants.md)를 등록합니다. 불변자는 각 블록의 끝에 평가(evaluate)되는 변수(e.g. 토큰의 총 공급량)입니다. 불변자를 확인하는 과정은 [`InvariantsRegistry`](../building-modules/invariants.md#invariant-registry)라는 특수한 모듈을 통해 수행됩니다. 불변자의 값은 모듈에서 정의된 예측 값과 동일해야 합니다. 값이 예측 값과 다를 경우, `invariant registry`에서 정의된 특수한 로직(일반적으로 체인이 중지됨)이 수행됩니다. 이는 심각한 버그가 알려지지 않고 오랜 기간에 걸쳐 고치기 어려운 영향을 끼치는 문제를 방지하는 데에 유용합니다.
- 모듈 매니져를 사용하여 각 [애플리케이션 모듈](#애플리케이션-모듈-인터페이스) `InitGenesis`와, `BeginBlocker` 그리고 `EndBlocker`의 수행 순서를 설정합니다. 모든 모듈이 이러한 함수를 구현하지 않는다는 점을 참고하십시오.
- 다음의 나머지 애플리케이션 매개 변수(parameter)를 설정합니다:
    - [`InitChainer`](#initchainer): 애플리케이션이 처음 시작될때 애플리케이션을 초기화하는 데 사용됩니다.
    - [`BeginBlocker`, `EndBlocker`](#beginblocker와-endblocker): 모든 블록의 시작과 끝에 호출됩니다.
    - [`anteHandler`](../core/baseapp.md#antehandler): 수수료 및 서명 검증에 사용됩니다.
- 스토어를 마운트합니다.
- 애플리케이션을 반환합니다.

이 함수는 오로지 앱의 인스턴스를 생성할 뿐, 실제 상태(state)는 노드가 재시작하는 경우 `~/.app/data` 폴더에서 전달되고, 노드가 최초로 시작하는 경우 제네시스(genesis) 파일로 부터 생성되는 점을 참고하십시오.

다음은 `simapp`의 애플리케이션 생성자의 예시입니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/simapp/app.go#L198-L441

### InitChainer

`InitChainer` 제네시스 파일(i.e. 제네시스 계정의 토큰 잔고)로부터 애플리케이션의 상태를 초기화하는 함수입니다. 이 함수는 애플리케이션이 Tendemint 엔진으로부터 `InitChain` 메시지를 수신할 때 호출되며, 이는 노드가 `appBlockHeight == 0`에서 시작할 때(i.e. 제네시스) 발생합니다. 애플리케이션은 반드시 `InitChainer`를 [`SetInitChainer`](https://godoc.org/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetInitChainer) 메서드를 통하여 [생성자](#생성자(Constructor)-함수)에 설정해야 합니다.

일반적으로 `InitChainer`는 대부분 각 애플리케이션 모듈의 [`InitGenesis`](../building-modules/genesis.md#initgenesis) 함수로 구성됩니다. 모듈 매니져의 `InitGenesis` 함수를 호출하기만 하면, 모듈 안에 포함된 모듈들의 `InitGenesis` 함수를 호출하게 됩니다. 주의할 점은, [module manager의](../building-modules/module-manager.md) `SetOrderInitGenesis` 메서드를 통해 모듈 매니져에 각 모듈의 `InitGenesis` 함수간의 호출 순서가 설정되어야 한다는 것입니다. 이 작업은 [생성자](#생성자(Constructor)-함수)에서 수행되며 `SetOrderInitGenesis`는 `SetInitChainer` 이전에 호출되어야 합니다.

다음은 `simapp`의 `InitChainer` 예시입니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/simapp/app.go#L464-L471

### BeginBlocker와 EndBlocker

SDK는 개발자들에게 그들의 애플리케이션의 일부로 코드의 자동 실행을 구현할 수 있는 기능을 제공합니다. 이 기능은 `BeginBlocker`와 `EndBlocker`의 두 함수를 통해 구현됩니다. 이 함수들은 각 블록의 시작과 끝에서 Tendermint 엔진으로부터 각각 `BeginBlock`와 `EndBlock` 메시지를 수신하면 호출됩니다. 애플리케이션은 [`SetBeginBlocker`](https://godoc.org/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetBeginBlocker)와 [`SetEndBlocker`](https://godoc.org/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetEndBlocker) 메서드를 통해 [생성자](#생성자(constructor)-함수)에 `BeginBlocker`와 `EndBlocker`를 설정해야 합니다.

일반적으로 `BeginBlocker`와 `EndBlocker`는 대부분 각 애플리케이션 모듈의 [`BeginBlock`과 `EndBlock`](../building-modules/beginblock-endblock.md) 함수들로 구성됩니다. 모듈 매니져의 `BeginBlock`과 `EndBlock` 함수를 호출하기만 하면, 모듈 안에 포함된 모듈들의 `BeginBlock`과 `EndBlock` 함수를 호출하게 됩니다. 주의할 점은 각각 `SetOrderBeginBlock`과 `SetOrderEndBlock` 메서드를 사용하여 모듈 매니져의 `BegingBlock`과 `EndBlock`의 호출 순서를 설정해야 한다는 것입니다. 이 작업은 [생성자](#생성자(Constructor)-함수)에 있는 [모듈 매니져](../building-modules/module-manager.md)를 통해 진행되며 `SetOrderBeginBlock`과 `SetOrderEndBlock` 메서드는 `SetBeginBlocker`과 `SetEndBlocker` 함수 이전에 실행 되어야 합니다.

별개로, 어플리케이션별 전용 블록체인은 결정론적(deterministic)임을 명심해야합니다. 개발자들은 `BeginBlocker` 또는 `EndBlocker`에 비결정론(non-determinism)을 도입하지 않도록 주의해야만 하며, [가스(gas)](./gas-fees.md)가 `BeginBlocker`와 `EndBlocker`의 비용을 제한하지 않기 때문에 지나치게 연산을 필요하지 않도록 주의해야만 합니다.

다음은 `simapp`의 `BeginBlocker`와 `EndBlocker`의 예시입니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/simapp/app.go#L454-L462

### 코덱(Codec) 등록하기

`EncodingConfig` 구조체는 `app.go` 파일의 마지막으로 중요한 부분입니다. 이 구조체의 목적은 앱 전반적으로 사용하는 코덱을 정의하는 것입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/simapp/params/encoding.go#L9-L16

다음은 4개의 필드의 각 의미에 대한 설명입니다:

- `InterfaceRegistry`: `InterfaceRegistry`는 Protobuf 코덱에서 [`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto)를 사용하여 인코딩 및 디코딩된("언팩(unpack)"이라고도 합니다) 인터페이스를 다루기 위하여 사용됩니다. `Any`는 `type_url`(인터페이스를 구현하는 구체적 타입(concrete type)의 이름)과 `value`(인코딩된 바이트)가 포함된 구조체로 간주할 수 있습니다. `InterfaceRegistry`는 `Any`로부터 안전하게 언팩할 수 있는 인터페이스 및 구현을 등록하는 메커니즘을 제공합니다. 각 애플리케이션 모듈은 해당 모듈의 자체 인터페이스와 구현을 등록할 수 있는 `RegisterInterfaces` 메서드를 구현합니다.
    - `Any`의 자세한 내용은 [ADR-19](../architecture/adr-019-protobuf-state-encoding.md#usage-of-any-to-encode-interfaces)에서 확인할 수 있습니다.
    - 더욱 자세히 들여다보면, SDK는 [`gogoprotobuf`](https://github.com/gogo/protobuf)라는 Protobuf 명세(specification)를 구현하고 있습니다. 기본적으로, [`Any`를 구현한 `gogo protobuf`](https://godoc.org/github.com/gogo/protobuf/types)는 `Any`에 팩(pack)된 값을 구체적인(concrete) Go 타입으로 디코딩하기 위하여 [전역 타입 등록(global type registration)](https://github.com/gogo/protobuf/blob/master/proto/properties.go#L540)을 사용합니다. 이로 인해 종속 트리(dependency tree)에 존재하는 악의적인 모듈이 전역 protobuf 레지스트리에 타입을 등록하고 `type_url` 필드에서 해당 타입을 참조하는 트랜잭션을 통해 로드 및 마셜을 해제(unmarshal)할 수 있는 취약점이 발생합니다. 자세한 내용은 [ADR-019](../architecture/adr-019-protobuf-state-encoding.md)를 참조하십시오.
- `Marshaler`: SDK 전반적으로 사용되는 기본 코덱입니다. 상태를 인코딩 및 디코딩하기 위해 사용되는 `BinaryCodec`과 사용자에게 데이터를 출력하기 위해 사용되는 `JSONCodec`으로 구성됩니다 (예: [CLI](#cli)). 기본적으로 Progobuf를 `Marshaler`로 사용합니다.
- `TxConfig`: `TxConfig`는 클라이언트가 어플리케이션에서 정의한 구체적(concrete) 트랜잭션 타입을 생성하는 데 사용할 수 있는 인터페이스를 정의합니다. 현재 SDK는 `SIGN_MODE_DIRECT` (Protobuf 바이너리를 인코딩(over-the-wire encodeing)으로 사용) `SIGN_MODE_LEGACY_AMINO_JSON` (Amino에 종속)의 두가지 트랜잭션 타입을 다룹니다. 트랜잭션에 대한 자세한 내용은 [여기](../core/transactions.md)를 참조하십시오.
- `Amino`: SDK의 일부는 하위 호환을 위해 여전히 Amino를 사용합니다. 각 모듈은 Amino 내에 모듈의 특정 타입을 등록할 수 있는 `RegisterLegacyAmino` 메서드를 제공합니다. 이 `Amino` 코덱은 더 이상 앱 개발자가 사용해서는 안되며 향후 릴리스에서 제거할 예정입니다.

SDK는 `앱 생성자(NewApp)`를 위한 `EncodingConfig`를 생성하는데 사용하는 `MakeTestEncodingConfig` 함수를 제공합니다. 기본 `Marshaler`로 Protobuf를 사용합니다.
참고: 이 함수는 사용하지 않음(deprecated) 표시되었으며 앱을 생성하거나 테스트하는 경우에만 사용되어야 합니다. 코덱 관리 기능을 리팩토링하고 있으며 향후 Stargate 릴리스에서 준비하고 있습니다.

다음은 `simapp`의 `MakeTestEncodingConfig` 예시입니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/590358652cc1cbc13872ea1659187e073ea38e75/simapp/encoding.go#L8-L19

## 모듈(Module)

[모듈](../building-modules/intro.md)은 SDK 애플리케이션의 심장이자 영혼과도 같습니다. 모듈은 상태 기계 안의 상태 기계로 간주할 수 있습니다. ABCI를 통해 하위 Tendermint 엔진으로부터 애플리케이션으로 트랜잭션이 중계될 때, 처리를 위하여 [`baseapp`](../core/baseapp.md)에 의해 적절한 모듈로 라우팅됩니다. 이러한 패러다임은 종종 필요한 대부분의 모듈들이 이미 존재하기 때문에, 개발자들이 복잡한 상태 기계를 쉽게 빌드할 수 있도록 합니다. 개발자들의 SDK 애플리케이션 빌드에 관련된 대부분의 작업들은 아직 존재하지 않는 그들의 애플리케이션에 필요한 맞춤형 모듈을 개발하고 이미 존재하는 모듈들과 하나의 일관된 애플리케이션으로 통합하는 것입니다. 애플리케이션 디렉토리에서는 `x/` 폴더(이미 빌드된(already-built) 모듈이 포함된 SDK의 `x/` 폴더와 혼동하지 마십시오)에 모듈을 저장하는 것이 표준입니다.

### 애플리케이션 모듈 인터페이스

모듈은 Cosmos SDK에서 정의된 [인터페이스](../building-modules/module-manager.md#application-module-interfaces)와 [`AppModuleBasic`](../building-modules/module-manager.md#appmodulebasic), [`AppModule`](../building-modules/module-manager.md#appmodule)을 반드시 구현해야합니다. `AppModuleBasic`은 `codec`과 같이 기본적인 모듈의 비의존적 요소(non-dependant elements)를 구현하고, `AppModule`은 대부분의 모듈 메서드(다른 모듈의 `keeper`에 대한 참조를 필요로 하는 메서드들을 포함합니다)를 처리합니다. `AppModule`과 `AppModuleBasic` 타입 모두 `./module.go`라는 파일에 정의되어 있습니다.

`AppModule`은 모듈들을 하나의 일관성있는(coherent) 애플리케이션으로 구성하는 것을 용이하게 하는 유용한 메서드 모음을 모듈에 제공합니다. 이러한 메서드들은 애플리케이션의 모듈들의 모음을 관리하는 [`모듈 매니져`](../building-modules/module-manager.md#manager)로부터 호출됩니다.

### `Msg` 서비스

각 모듈은 메시지를 처리하는 각각 `Msg` 서비스와 쿼리를 처리하는 gRPC `Query`서비스의 2개의 [Protobuf 서비스](https://developers.google.com/protocol-buffers/docs/proto#services)를 정의합니다. 모듈을 상태 기계로 간주한다면, `Msg` 서비스는 상태 전환(state-transition) RPC 메서드의 집합으로 볼 수 있습니다.
각 Protobuf `Msg` 서비스 메서드는 Protobuf 요청 타입과 1:1로 대응되며, `sdk.Msg` 인터페이스를 정의해야 합니다.
`sdk.Msg`는 [트랜잭션](../core/transactions.md)에서 번들로 제공되며, 각 트랜잭션은 하나 이상의 메시지를 포함하고 있습니다.

트랜잭션들의 유효한 블록이 풀-노드(full-node)로부터 수신될 때, Tendermint는 [`DeliverTx`](https://tendermint.com/docs/app-dev/abci-spec.html#delivertx)를 통해 각 트랜잭션을 애플리케이션에 중계합니다. 그 후, 애플리케이션은 트랜잭션을 다음과 같이 처리합니다:

1. 트랜색션을 수신하면 애플리케이션은 먼저 `[]bytes`로부터 마셜을 해제합니다(unmarshall).
2. 다음으로, 트랜잭션에 포함된 `Msg`(들)를 추출하기 전에 트랜잭션과 관련된 [수수료 지불 및 서명](#gas-fees.md#antehandler)과 같은 몇가지 사항을 미리 검증합니다.
3. `sdk.Msg`는 Protobuf인 [`Any`s](#코덱(codec)-등록하기)를 사용하여 인코딩됩니다. `Any`의 각 `type_url`을 분석함으로써 `baseapp`의 `msgServiceRouter`는 `sdk.Msg`를 해당하는 모듈의 `Msg` 서비스에 라우팅합니다.
4. 메시지가 성공적으로 처리되면 상태가 업데이트됩니다.

자세한 내용은 트랜잭션 [생명주기](./tx-lifecycle.md)를 참조하십시오.

모듈 개발자는 자체 모듈을 빌드할때 맞춤형 `Msg` 서비스를 생성합니다. `tx.proto` 파일에 `Msg` Protobuf 서비스를 정의하는 것이 일반적입니다. 예를 들어 `x/bank` 모듈은 토큰을 전송하기 위해 2개의 메서드를 한 서비스에서 정의합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/proto/cosmos/bank/v1beta1/tx.proto#L10-L17

서비스 메서드는 모듈 상태를 업데이트 하기 위해 `keeper`를 사용합니다.

각 모듈은 또한 [`AppModule` interface](#애플리케이션-모듈-인터페이스)의 일부로 `RegisterServices`를 구현해야 합니다. 이 메서드는 생성된 Protobuf 코드가 제공하는 `RegisterMsgServer` 함수를 호출해야 합니다.

### gRPC 쿼리(`Query`) 서비스

gRPC `Query` 서비스는 v0.40 Stargate 릴리스에 도입되었습니다. 사용자는 [gRPC](https://grpc.io)를 사용하여 상태를 쿼리할 수 있습니다. 기본적으로 사용하도록 설정돼있으며, [`app.toml`](../run-node/run-node.md#configuring-the-node-using-apptoml)의 `grpc.enable`와 `grpc.address` 필드를 통해 설정할 수 있습니다.

gRPC `Query` 서비스는 모듈의 Protobuf 정의(definition) 파일, 특히 `query.proto` 안에 정의됩니다. `query.proto` 정의 파일은 단일 `Query` [Protobuf 서비스](https://developers.google.com/protocol-buffers/docs/proto#services)를 제공합니다. 각 gRPC 쿼리 엔드포인트(endpoint)는 `Query` 서비스 내부에 `rpc` 키워드로 시작하는 서비스 메서드에 해당합니다.

Protobuf는 각 모듈을 위한 모든 서비스 메서드를 포함하는 `QueryServer` 인터페이스를 생성합니다. 그러면 모듈의 [`keeper`](#keeper)는 `QueryServer` 인터페이스의 각 서비스 메서드를 구현(concrete implementation)해야 합니다. 이 구현은 대응하는 gRPC 쿼리 엔드포인트의 핸들러(handler)입니다.

마지막으로, 각 모듈은 [`AppModule` interface](#애플리케이션-모듈-인터페이스)의 일부로 `RegisterServices` 메서드 또한 구현해야 합니다. 이 메서드는 Protobuf 코드에서 제공하는 `RegisterQueryServer` 함수를 호출해야 합니다.

### Keeper

[`Keeper`](../building-modules/keeper.md)는 자신의 모듈 스토어(들)의 문지기입니다. 모듈 스토어를 읽거나 쓰려면 해당 `keeper`의 메서드를 반드시 사용하여야 합니다. 이것은 Cosmos SDK의 [오브젝트 자격(object-capabilities)](../core/ocap.md) 모델에 의해 보장이 됩니다. 오직 스토어의 키(key)를 보유한 객체만이 접근할 수 있고, 오직 모듈의 `keeper`만이 모듈 스토어(들)의 키(들)를 보유할 수 있습니다.

`Keepers`는 일반적으로 `keeper.go`라는 파일에 정의됩니다. 이 파일은 `keeper`의 타입 정의와 메서드를 포함합니다.

`Keeper`타입 정의는 일반적으로 다음의 내용으로 구성됩니다:

- 멀티스토어(multistore)에 있는 모듈 스토어(들)의 **키(들)**.
- **다른 모듈의 `keepers`에 대한 참조**. `keeper`가 다른 모듈 스토어(들)에 대한 접근하는 경우(읽기 혹은 쓰기)에만 필요합니다.
- 애플리케이션의 **코덱**에 대한 참조. `keeper`는 구조체를 저장하기 전 마셜(marshal)하거나 검색하기 전 마셜을 해제하기 위해 코덱의 참조가 필요한데, 이는 스토어가 오직 `[]bytes`만을 값으로 허용하기 때문입니다.

타입 정의와 함께, `keeper.go` 파일 중 다음으로 중요한 구성 요소는 `NewKeeper`라고 하는 `keeper`의 생성자 함수입니다. 이 함수는 새로운 `keeper`를 상기된 타입인 `코덱`과 스토어 `키`, 그리고 다른 모듈의 `keeper`에 대한 잠재적 참조들을 매개 변수로 인스턴스화 합니다. `NewKeeper` 함수는 [애플리케이션 생성자](#생성자(constructor)-함수)에서 호출됩니다. 다른 파일에서는 주로 getter와 setter와 같은 `keeper`의 메서드를 정의합니다.

### 커맨드라인(Command-Line), gRPC 서비스와 REST 인터페이스

각 모듈은 [애플리케이션 인터페이스](#애플리케이션-인터페이스)를 통해 최종 사용자에게 제공될 커맨드라인 명령어와, gRPC 서비스, 그리고 REST 경로를 정의합니다. 이를 통해 최종 사용자는 모듈에 정의된 타입의 메시지를 생성하거나 모듈에서 관리되는 상태의 부분 집합을 쿼리할 수 있습니다.

#### CLI

일반적으로 [모듈 관련 명령어](../building-modules/module-interfaces.md#cli)는 모듈의 폴더에 있는 `client/cli`라는 폴더에서 정의되어 있습니다. CLI는 `client/cli/tx.go`와 `client/cli/query.go`에서 각각 정의된 트랜잭션(Transactions)과 쿼리(Queries)의 두 범주로 나뉘어집니다. 다음의 두 범주 모두 [Cobra 라이브러리](https://github.com/spf13/cobra) 기반으로 빌드되었습니다:

- 트랜잭션 명령어를 통해 사용자가 생성한 새 트랜잭션을 블록에 포함하고 최종적으로 상태를 업데이트 할 수 있도록 합니다. 모듈에서 정의된 각 [메시지 타입](#Msg-서비스)에 맞추어 하나의 명령어가 생성되어야 합니다. 명령어는 최종 사용자가 제공한 매개 변수를 사용하여 메시지의 생성자를 호출하고, 트랜잭션에 담습니다(wrap). SDK는 서명과 더불어 다른 트랜잭션 메타데이터를 처리합니다.
- 쿼리 명령어를 통해 사용자가 모듈에서 정의된 상태의 부분 집합을 쿼리할 수 있습니다. 쿼리 명령어는 제공된 `queryRoute` 매개 변수를 통해 [질의자(querier)](#querier)에게 라우팅하는 [애플리케이션 쿼리 라우터](../core/baseapp.md#query-routing)에게 쿼리를 전달합니다.

#### gRPC

[gRPC](https://grpc.io)는 다수의 언어로 제공되는 현대적인 오픈 소스 고성능 RPC 프레임워크입니다. 외부 클라이언트(지갑이나 브라우저, 혹은 다른 백엔드 서비스)와 노드가 상호작용할때 권장하는 방법입니다.

각 모듈은 [서비스 메서드](https://grpc.io/docs/what-is-grpc/core-concepts/#service-definition)라고 불리는 [모듈의 Protobuf `query.proto` 파일](#grpc-쿼리(query)-서비스)에서 정의된 gRPC 엔드포인트를 제공합니다. 서비스 메서드는 이름(name), 입력 인자(input arguments) 및 출력 응답(output response)으로 정의됩니다. 모듈은 다음을 수행해야 합니다:

- `AppModuleBasic`에 `RegisterGRPCGatewayRoutes`를 정의하여 클라이언트 gRPC 요청(requests)을 모듈 내 올바른 처리기(handler)에 연결합니다.
- 각 서비스 메서드에 대응하는 처리기를 정의합니다. 처리기는 gRPC 요청을 수행하기 위한 핵심 로직을 구현하고 `keeper/grpc_query.go` 파일에 위치합니다.

#### gRPC-게이트웨이 REST 엔드포인트

일부 외부 클라이언트는 gRPC를 사용하지 않을 수 있습니다. 이 경우 SDK는 각 gRPC서비스를 대응하는 REST 엔드포인트로 제공하는 gRPC 게이트웨이 서비스를 지원합니다. 자세한 내용은 [grpc-게이트웨이](https://grpc-ecosystem.github.io/grpc-gateway/) 문서를 참고하십시오.

REST 엔드포인트는 Protobuf 어노테이션(annotation)을 사용하여 gRPC 서비스와 함께 Protobuf 파일에 정의되어 있습니다. REST 쿼리를 제공하기를 원하는 모듈은 `rpc` 메서드에 `google.api.http` 어노테이션을 추가해야 합니다. 기본적으로 SDK에 정의된 모든 REST 엔드포인트는 SDK `/cosmos/` 접두사로 시작하는 URL을 갖고 있습니다.

SDK는 또한 위의 REST 엔드포인트를 위한 [Swagger](https://swagger.io/) 정의 파일을 생성할 수 있는 개발 엔드포인트를 제공합니다. 이 엔드포인트는 `api.swagger` 키 아래의 [`app.toml`](../run-node/run-node.md#configuring-the-node-using-apptoml) 설정(config) 파일 내애서 활성화시킬 수 있습니다.

#### 레거시 API REST 엔드포인트

사용자는 [모듈 레거시 REST 인터페이스](../building-modules/module-interfaces.md#legacy-rest)를 사용하여 애플리케이션 레거시 API 서비스에 대한 애플리케이션 REST 호출(call)하는 방법으로 트랜잭션을 생성하고 상태를 쿼리할 수 있습니다. REST 경로는 `client/rest/rest.go`파일에 정의되며, 다음으로 구성됩니다:

- 파일에 정의된 각 경로를 등록하는 `RegisterRoutes` 함수. 이 함수는 애플리케이션 내에서 사용되는 각 모듈마다 [주요(main) 애플리케이션 인터페이스](#애플리케이션-인터페이스)에 의해 호출됩니다. SDK에서 사용된 라우터는 [Gorilla's mux](https://github.com/gorilla/mux)입니다.
- 제공할 각 쿼리 또는 트랜잭션 생성 함수를 위한 사용자 정의(custom) 타입에 대한 정의(definition). 이 사용자 정의 요청 타입은 Cosmos SDK의 다음의 `request` 타입을 기반으로 만들어집니다:
  +++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/types/rest/rest.go#L62-L76
- 지정된 모듈로 라우팅할 수 있는 각 요청에 대한 처리기(handler) 함수. 이 함수는 요청을 수행하기 위해 필요한 핵심 로직을 구현합니다.

이러한 레거시 API 엔드포인트는 하위 호환성을 위해 SDK에 존재하며 다음 릴리스에서 제거됩니다.

## 애플리케이션 인터페이스

최종 사용자는 [인터페이스](#커맨드라인(Command-Line),-gRPC-서비스와-REST-인터페이스)를 통해 풀-노드 클라이언트와 상호 작용할 수 있습니다. 이것은 풀-노드로부터 데이터를 쿼리하거나, 풀-노드로 중계되고 최종적으로 블록에 추가될 새로운 트랜잭션을 만들고 전송하는 것을 의미합니다.

주요 인터페이스는 [커맨드라인 인터페이스](../core/cli.md)입니다. SDK 애플리케이션의 CLI는 애플리케이션이 사용하는 각 모듈에서 정의된 [CLI 명령어](#cli)를 집계하여 만들어집니다. 이러한 애플리케이션의 CLI는 데몬(e.g. `appd`)과 동일하며, `appd/main.go` 파일에 정의됩니다. 파일은 다음의 내용을 포함합니다:

- **`main()` 함수.** 이 함수는 `appd` 인터페이스 클라이언트를 만들기 위해 실행됩니다. 그 전에 이 함수는 각 명령어를 준비하고 `rootCmd`에 해당 명령어를 추가합니다. `appd`의 루트(root)에서 이 함수는 `status`와 `keys`, `config` 같은 제너릭(generic) 함수와 쿼리 명령어, tx 명령어, `rest-server`를 추가합니다.
- **쿼리 명령어.** 이 함수는 `queryCmd` 함수를 실행하여 추가됩니다. 이 함수는 각 애플리케이션 모듈에 정의(`main()` 함수로부터 `sdk.ModuleClients`의 배열(array)로 전달됨)된 쿼리 명령어와, 블록 혹은 검증자(validator) 쿼리와 같은 다른 하위 수준(lower level) 쿼리 명령어를 포함하는 Cobra 명령어를 반환합니다. 쿼리 명령어는 CLI의 `appd query [query]` 명령어를 사용하여 호출할 수 있습니다.
- **트랜잭션 명령어.** 이 함수는 `txCmd` 함수를 호출하여 추가됩니다. `queryCmd`와 비슷하게, 이 함수는 각 애플리케이션 모듈에서 정의된 tx 명령어와, 트랜잭션 서명이나 브로드캐스팅(broadcasting)과 같은 하위 수준 tx 명령어를 포함하는 Cobra 명령어를 반환합니다. Tx 명령어는 CLI의 `appd tx [tx]`를 사용하여 호출할 수 있습니다.

[네임서비스(nameservice) 튜토리얼](https://github.com/cosmos/sdk-tutorials/tree/master/nameservice)에서 애플리케이션 주요 커맨드라인 파일의 예를 참조하십시오.

+++ https://github.com/cosmos/sdk-tutorials/blob/86a27321cf89cc637581762e953d0c07f8c78ece/nameservice/cmd/nscli/main.go

## 의존성과 Makefile

::: warning
`go-grpc v1.34.0`에 도입된 패치로 인해 gRPC가 `gogoproto` 라이브러리와 호환되지 않아 일부 [gRPC 쿼리](https://github.com/cosmos/cosmos-sdk/issues/8426)가 패닉(panic) 상태에 빠집니다. 따라서 SDK는 `go.mod`에서 `go-grpc <=v1.33.2`가 설치하는 것을 요구합니다.

gRPC가 제대로 동작하는 것을 보장하기 위해, 애플리케이션의 `go.mod`에 다음의 줄을 추가하는 것을 **강력히 권장**합니다.

```
replace google.golang.org/grpc => google.golang.org/grpc v1.33.2
```

자세한 내용은 [issue #8392](https://github.com/cosmos/cosmos-sdk/issues/8392)를 참조하십시오.
:::

개발자는 종속성(dependency) 관리자와 프로젝트 빌드 방법(builing method)를 자유롭게 선택할 수 있으므로, 이 섹션은 선택사항입니다. 그렇긴 하지만, 현재 대부분의 버젼 관리(control) 프레임워크는 [`go.mod`](https://github.com/golang/go/wiki/Modules)를 사용하고 있습니다. 이 것은 애플리케이션 전반에서 사용되는 각 라이브버리를 올바른 버젼으로 가져올 수 있도록 보장합니다. 다음의 [네임서비스 튜토리얼](https://github.com/cosmos/sdk-tutorials/tree/master/nameservice)의 참조하십시오:

+++ https://github.com/cosmos/sdk-tutorials/blob/c6754a1e313eb1ed973c5c91dcc606f2fd288811/go.mod#L1-L18

애플리케이션을 빌드하기 위해서 일반적으로 [Makefile](https://en.wikipedia.org/wiki/Makefile)이 사용됩니다. Makefile은 주로 [`appd`](#노드-클라이언트)와 [`appd`](#애플리케이션-인터페이스)의 두 개의 진입점(entry point)이 빌드되기 전에 `go.mod`가 실행되는 것을 보장합니다. 다음의 [네임서비스 튜토리얼](https://tutorials.cosmos.network/nameservice/tutorial/00-intro.html)에서 Makefile의 예시를 참조하십시오.

+++ https://github.com/cosmos/sdk-tutorials/blob/86a27321cf89cc637581762e953d0c07f8c78ece/nameservice/Makefile

## 다음 {hide}

[트랜잭션의 생명주기(Lifecycle)](./tx-lifecycle.md)에 대해 알아봅시다 {hide}
