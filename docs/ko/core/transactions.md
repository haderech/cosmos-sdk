<!--
order: 2
-->

# 트랜잭션 (Transactions)

`Transactions` 는 엔드유저에 의해 생성되는 객체로써 애플리케이션에서 상태 변경을 트리거합니다. {synopsis}

## 사전 요구 학습

- [SDK Application 해부](../basics/app-anatomy.md) {prereq}

## 트랜잭션 (Transactions)

트랜잭션은 [contexts](./context.md) 의 메타데이터와 모듈의 Protobuf [`Msg` 서비스](../building-modules/msg-services.md) 를 통해
모듈 내의 상태 변경을 트리거하는 [`sdk.Msg`](../building-modules/messages-and-queries.md) 로 구성됩니다.

사용자가 애플리케이션과 상호작용하여 상태를 변경 (예: 코인 전송) 하고 싶을 때 트랜잭션을 생성합니다. 트랜잭션의 각 `sdk.Msg` 는 트랜잭션이 네트워크로 브로드캐스팅
되기 전에, 반드시 해당 계정에 연결된 개인키를 사용하여 서명되어야 합니다. 그런 다음에 트랜잭션은 합의 프로세스를 통해 네트워크가 검증, 승인하고 블록에 포함됩니다.
트랜잭션 생명주기에 대해서 더 읽어보시려면 [여기](../basics/tx-lifecycle.md) 를 클릭하세요.


## 타입 정의 (Type Definition)

트랜잭션 객체는 `Tx` 인터페이스를 구현한 SDK 타입입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/types/tx_msg.go#L49-L57

다음의 메서드들을 포함합니다:

- **GetMsgs:** 트랜잭션의 래핑을 풀고 `sdk.Msg` 목록을 반환합니다. - 하나의 트랜잭션은 하나 이상의 메시지를 포함할 수 있으며, 이는 모듈 개발자가 정합니다.
- **ValidateBasic:** 가벼운 [_stateless_](../basics/tx-lifecycle.md#types-of-checks) 검사가 포함되어 ABCI 메시지인
  [`CheckTx`](./baseapp.md#checktx) 와 [`DeliverTx`](./baseapp.md#delivertx) 에서 사용되며, 이를 통해 유효하지 않은 트랜잭션인지
  확인합니다. 예를 들어, [`auth`](https://github.com/cosmos/cosmos-sdk/tree/master/x/auth) 모듈의 `StdTx` `ValidateBasic` 함수는
  정확한 숫자의 서명자가 서명했는지 확인하고 수수료가 사용자의 최대 금액을 초과하지 않는지 확인합니다. 이 함수는 `sdk.Msg` 의 메시지에 대해 기본적인 유효성 
  검사만 수행하는 `ValidateBasic` 함수들과 구분해야한다는 점을 주의하십시오. 예를 들어,
  [`auth`](https://github.com/cosmos/cosmos-sdk/tree/master/x/auth/spec) 모듈에서 만들어진 트랜잭션을
  [`runTx`](./baseapp.md#runtx) 에서 검사할 때, 각 메시지에 대해 `ValidateBasic` 를 먼저 실행하고 트랜잭션 자체에 대해 `ValidateBasic` 을
  호출하는 `auth` 모듈 `AnteHandler` 를 실행합니다.

`Tx` 는 실제로 트랜잭션 생성에 사용되는 중간 타입이기 때문에, 개발자는 `Tx` 를 직접 조작하는 일이 거의 없습니다. 대신에 개발자는 `TxBuilder` 인터페이스를
더 선호할 것이고, [아래](#transaction-generation)에서 더 알아볼 수 있습니다.

### 트랜잭션 서명 (Signing Transactions)

트랜잭션의 모든 메시지는 해당되는 `GetSigners` 에서 지정한 주소로 서명되어야 합니다. SDK 는 현재 두 가지 방법으로 트랜잭션 서명을 허용합니다.

#### `SIGN_MODE_DIRECT` (선호됨)

가장 많이 사용되는 `Tx` 인터페이스 구현은 `SIGN_MODE_DIRECT` 에서 사용되는 Protobuf `Tx` 메시지입니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/proto/cosmos/tx/v1beta1/tx.proto#L12-L25

프로토콜 버퍼 직렬화는 결정론적이지 않기 때문에, SDK 는 추가적인 `TxRaw` 타입을 사용해서 트랜잭션이 서명되는 고정 바이트를 표시합니다. 모든 사용자는 트랜잭션에
대한 유효한 `body` 와 `auth_info` 를 생성할 수 있으며, Protobuf 를 사용해 두 메시지를 직렬화할 수 있습니다. 그런 다음 `TxRaw` 는 각각 
`body_bytes`와 `auth_info_bytes` 라고 불리는 `body` 와 `auth_info` 의 사용자의 정확한 바이너리 표현(binary representation)을 고정합니다.
`SignDoc` ([ADR-027](../architecture/adr-027-deterministic-protobuf-serialization.md) 을 사용해서 결정론적으로 직렬화된) 은
트랜잭션의 모든 서명자에 의해 서명된 문서입니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/proto/cosmos/tx/v1beta1/tx.proto#L47-L64

모든 서명자가 서명하면 `body_bytes`, `auth_info_bytes` 와 `signatures` 는 `TxRaw` 로 수집되고, 바이트 직렬화되어 네트워크로 브로드캐스트됩니다.

#### `SIGN_MODE_LEGACY_AMINO_JSON`

`x/auth` 의 `StdTx` 구조체 인 `Tx` 인터페이스의 기존 구현:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/x/auth/legacy/legacytx/stdtx.go#L120-L130

모든 서명자에 의해 서명된 문서인 `StdSignDoc`:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/x/auth/legacy/legacytx/stdsign.go#L20-L33

`StdSignDoc` 은 Amino JSON 을 사용하여 바이트로 인코드 됩니다. 모든 서명이 `StdTx` 로 수집되면, `StdTx`  는 Amino JSON 을 사용해 직렬화되고,
이 바이트들은 네트워크를 통해 브로드캐스트됩니다.

#### Other Sign Modes

기타 서명 모드로는 특히 `SIGN_MOD_TEXTUAL` 이 논의되고 있습니다. 이에 대해 더 알아보시려면
[ADR-020](../architecture/adr-020-protobuf-transaction-encoding.md) 을 참조하세요.

## 트랜잭션 프로세스

엔드유저가 트랜잭션을 보내는 프로세스는:

- 트랜잭션으로 넣을 메시지 결정
- SDK 의 `TxBuilder` 를 사용해서 트랜잭션 생성
- 가능한 인터페이스 중 하나를 사용해 트랜잭션 브로드캐스트

다음 단락에서는 이러한 구성 요소들을 다음 순서로 설명합니다.

### 메시지 (Messages)

::: tip
모듈 `sdk.Msg` 를 애플리케이션 레이어와 Tendermint 간 상호작용을 정의하는
[ABCI 메시지](https://tendermint.com/docs/spec/abci/abci.html#messages) 와 혼동하면 안됩니다.
:::

**메시지** (또는 `sdk.Msg`) 는 해당 메시지가 속한 모듈의 범위 내에서 상태 전이를 트리거 하는 모듈별 전용 객체입니다. 모듈 개발자들은
Protobuf [`Msg` 서비스](../building-modules/msg-services.md) 에 메서드를 추가해서 그들의 모듈을 위한 메시지를 정의하고, 또한 해당하는
`MsgServer` 를 구현합니다.

각 `sdk.Msg` 는 각  모듈의 `tx.proto` 파일 내에 정의된 정확한 하나의 Protobuf [`Msg` 서비스](../building-modules/msg-services.md)
RPC 와 관련되어 있습니다. SDK 앱 라우터는 해당 RPC 로 매 `sdk.Msg` 를 자동으로 매핑합니다. Protobuf 는 각 모듈 `Msg` 서비스를 위한 `MsgServer`
인터페이스를 생성하는데, 모듈 개발자는 이 인터페이스를 구현해야 합니다. 이 설계는 모듈 개발자에게 더 많은 책임을 부여하고, 애플리케이션 개발자가 상태 전이 
로직을 반복적으로 구현하지 않고 공통 기능을 재사용 하게끔 합니다.

Protobuf `Msg` 와 `MsgServer` 구현에 대해서 더 알아보시려면, [여기](../building-modules/msg-services.md) 를 클릭하세요.

상태 전이 로직에 대한 정보는 메시지에 포함되어 있고, 트랜잭션의 다른 메타 데이터 및 관련 정보는 `TxBuilder` 와 `Context` 에 저장됩니다.

### 트랜잭션 생성

`TxBuilder` 인터페이스에는 트랜잭션 생성과 밀접하게 관련된 데이터가 포함되어 있습니다. 이 데이터는 엔드유저가 원하는 트랜잭션을 생성하도록 자유롭게 설정할 수
있습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/client/tx_config.go#L32-L45

- `Msg`, 트랜잭션에 포함된 [메시지](#messages) 배열
- `GasLimit`, 지불해야 하는 가스를 계산하는 방법에 대해 사용자가 선택한 옵션
- `Memo`, 트랜잭션과 함께 전송되는 메모 또는 코멘트
- `FeeAmount`, 사용자가 수수료로 지불할 용의가 있는 최대양
- `TimeoutHeight`, 트랜잭션이 유효할 때까지의 블록 높이
- `Signatures`, 트랜잭션 모든 서명자의 서명 배열

트랜잭션 서명을 위해 현재 두 가지 서명 모드가 있으므로 `TxBuilder` 에도 두 가지 구현이 있습니다:

- `SIGN_MODE_DIRECT` 용 트랜잭션 생성을 위한 [wrapper](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/x/auth/tx/builder.go#L19-L33)
- `SIGN_MODE_LEGACY_AMINO_JSON` 용 [StdTxBuilder](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/x/auth/legacy/legacytx/stdtx_builder.go#L14-L20)

그러나 엔드유저는 `TxConfig` 인터페이스를 사용해야 하므로, `TxBuilder` 의 두 가지 구현은 엔드유저에게는 숨겨져 있습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/client/tx_config.go#L21-L30

`TxConfig` 는 트랜잭션 관리를 위한 앱 전체 (app-wide) 설정입니다. 제일 중요한건, 각 트랜잭션의 서명에 `SIGN_MODE_DIRECT` 와
`SIGN_MODE_LEGACY_AMINO_JSON` 중 어떤 것을 사용할 지에 대한 정보를 담고 있다는 것입니다. `txBuilder := txConfig.NewTxBuilder()` 를
호출하여 설정된 서명 모드로 새로운 `TxBuilder` 가 생성됩니다.

`TxBuilder` 가 위에서 말한 설정자들로 올바르게 채워지면, `TxConfig` 도 바이트를 올바르게 인코딩할 것입니다 (`SIGN_MODE_DIRECT` 또는
`SIGN_MODE_LEGACY_AMINO_JSON` 를 사용해서). 다음은 `TxEncoder()` 메서드를 사용하여 어떻게 트랜잭션을 인코드하고 생성하는지에 대한 의사 코드
스니펫입니다.

```go
txBuilder := txConfig.NewTxBuilder()
txBuilder.SetMsgs(...) // and other setters on txBuilder

bz, err := txConfig.TxEncoder()(txBuilder.GetTx())
// bz are bytes to be broadcasted over the network
```

### 트랜잭션 브로드캐스팅

트랜잭션 바이트가 생성되면, 현재 세 가지 방법으로 그것을 브로드캐스팅합니다.

#### CLI

애플리케이션 개발자는 [커맨드라인 인터페이스](../core/cli.md), [gRPC 그리고(또는) REST 인터페이스](../core/grpc_rest.md) 를 생성하여
애플리케이션으로의 진입점을 만듭니다. 이는 일반적으로 애플리케이션의 `./cmd` 폴더에서 찾을 수 있습니다.

[커맨드라인 인터페이스](../building-modules/module-interfaces.md#cli) 의 경우, 모듈 개발자는 하위 명령어들을 생성하여 애플리케이션 최상위 트랜잭션
커맨드 `TxCmd` 에 자식으로 추가합니다. CLI 커맨드는 실제로 하나의 커맨드에 트랜잭션 처리의 모든 단계가 묶여있습니다: 메시지 생성, 트랜잭션 생성 및 브로드캐스팅.
좋은 예로 [노드와 상호작용](../run-node/interact-node.md) 섹션을 보십시오. CLI 를 사용하여 만든 예시 트랜잭션:

```bash
simd tx send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake
```

#### gRPC

[gRPC](https://grpc.io) 는 SDK 의 RPC 레이어를 위한 주요 구성 요소로써 Cosmos SDK 0.40 에서 도입되었습니다. gRPC 의 주요 용도는 모듈의
[`Query` 서비스](../building-modules) 컨텍스트 입니다. 그러나 SDK 는 모든 모듈에 쓰일 수 있는 gRPC 서비스 몇 가지도 제공합니다. 그 중 하나는 `Tx`
서비스입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/proto/cosmos/tx/v1beta1/service.proto

`Tx` 서비스는 트랜잭션을 시뮬레이션하거나 트랜잭션을 쿼리하는 것과 같은 몇 가지 유틸리티 함수와 트랜잭션을 브로드캐스티하는 한가지 메서드를 제공합니다.

트랜잭션 시뮬레이션과 브로드캐스팅에 대한 예제를 [여기](../run-node/txs.md#programmatically-with-go) 에서 볼 수 있습니다.

#### REST

각 gRPC 메서드는 [gRPC-gateway](https://github.com/grpc-ecosystem/grpc-gateway) 를 사용해서 생성되는 해당 REST 엔드포인트가 있습니다.
따라서 gRPC 를 사용하는 대신에, HTTP 를 사용해서 `POST /cosmos/tx/v1beta1/txs` 엔드포인트에서 같은 트랜잭션을 브로드캐스트할 수도 있습니다.

[여기](../run-node/txs.md#using-rest) 에서 예제를 볼 수 있습니다.

#### Tendermint RPC

위에 나와있는 세 가지 메서드는 실제로 Tendermint RPC `/broadcast_tx_{async,sync,commit}` 엔드포인트 에 대한 높은 수준의 추상화 (higher
abstraction) 입니다. [여기](https://docs.tendermint.com/master/rpc/#/Tx) 문서를 참조하세요. 원한다면 Tendermint RPC 엔드포인트를
사용하여 트랜잭션을 직접 브로드캐스트할 수 있다는 뜻입니다.

## Next {hide}

[context](./context.md) 에 대해서 알아보세요 {hide}
