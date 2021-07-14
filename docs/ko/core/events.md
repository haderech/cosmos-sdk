<!--
order: 9
-->

# 이벤트(Events)

`Event`는 애플리케이션을 실행하기 위한 관련 정보를 포함한 객체입니다. 블록 탐색기(explorer)나 지갑같은 서비스 제공자(provider)들이 다양한 메시지와 트랜잭션을 인덱싱하기 위해 주로 사용합니다. {synopsis}

## 사전 요구 지식

- [SDK 애플리케이션의 해부](../basics/app-anatomy.md) {prereq}
- [이벤트 관련 Tendermint 문서](https://docs.tendermint.com/master/spec/abci/abci.html#events) {prereq}

## 이벤트

이벤트는 ABCI `Event` 타입의 별칭으로 Cosmos SDK에 구현되며 형식은 다음과 같습니다: `{eventType}.{attributeKey}={attributeValue}`.

+++ https://github.com/tendermint/tendermint/blob/v0.34.8/proto/tendermint/abci/types.proto#L304-L313

이벤트는 다음을 포함합니다:

- 상위 레벨(high-level)에서 이벤트를 분류하기 위한 `type`. 예를 들면, SDK는 이벤트를 `"message"`타입을 사용하여 `Msg`로 필터링합니다.
- `attributes` 목록은 분류된 이벤트에 대한 자세한 정보를 제공하는 키-값의 쌍(key-value pairs)입니다. 예를 들어 `"message"`유형의 경우, `message.action={some_action}`나, `message.module={some_module}`, 혹은 `message.sender={some_sender}`를 사용하여 키-값 쌍으로 이벤트를 필터링할 수 있습니다.

::: tip
속성(attrivute) 값을 string으로 파싱하려면, 각 속성 값 양 옆으로 `'`(작은 따옴표)를 추가해야합니다.
:::

이벤트와 `type`, `attributes`는 모듈의 `/types/events.go`파일 내의 **모듈 별 기준**에 정의되며, 모듈의 Protobuf [`Msg` 서비스](../building-modules/msg-services.md)에 의해 [`EventManager`](#eventmanager)를 사용하여 트리거됩니다.
또한 각 모듈은 `spec/xx_events.md`에 이벤트를 문서화(documents)합니다.

이벤트는 하단의(underlying) 합의 엔진으로 다음의 ABCI 메시지에 대한 응답으로서 반환됩니다:

- [`BeginBlock`](./baseapp.md#beginblock)
- [`EndBlock`](./baseapp.md#endblock)
- [`CheckTx`](./baseapp.md#checktx)
- [`DeliverTx`](./baseapp.md#delivertx)

### 예시

다음 예시는 SDK를 사용하여 이벤트를 쿼리하는 방법을 보여줍니다.

| 이벤트                                            | 설명                                                                                                                                              |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tx.height=23`                                   | 높이 23의 모든 트랜잭션을 쿼리                                                                                                                      |
| `message.action='/cosmos.bank.v1beta1.Msg/Send'` | x/bank `Send` [서비스 `Msg`](../building-modules/msg-services.md)를 포함한 모든 트랜잭션을 쿼리. 값을 둘러싼 `'`를 주의하십시오.                  |
| `message.action='send'`                          | x/bank `Send` [레거시 `Msg`](../building-modules/msg-services.md#legacy-amino-msgs)를 포함한 모든 트랜잭션을 쿼리. 값을 둘러싼 `'`를 주의하십시오. |
| `message.module='bank'`                          | x/bank module로부터의 메시지를 포함한 모든 트랜잭션을 쿼리. . 값을 둘러싼 `'`를 주의하십시오.                                                       |
| `create_validator.validator='cosmosval1...'`     | x/staking 특정 이벤트, [x/staking SPEC](../../../cosmos-sdk/x/staking/spec/07_events.md)를 참조하십시오.                                                         |

## EventManager

Cosmos SDK 애플리케이션에서 이벤트는 `EventManager`라는 추상화에 의해 관리됩니다.
내부적으로 이 `EventManager`는 트랜잭션이나 `BeginBlock`/`EndBlock`의 전체 동작 흐름에 대한 이벤트 목록을 추적합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/types/events.go#L17-L25

`EventManager`는 이벤트를 관리하기 위한 유용한 메서드 모음과 함께 제공됩니다. 모듈과 애플리케이션이 가장 많이 사용하는 메서드는 `EmitEvent`로, `EventManager` 내에서 이벤트를 추적합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/types/events.go#L33-L37

모듈 개발자는 각 메시지 처리기(`Handler`)와, `BeginBlock`/`EndBlock` 처리기에서 `EventManager#EmitEvent`를 통해 발생하는 이벤트를 처리해야합니다. `EventManager`는 [`Context`](./context.md)를 통해 접근할 수 있으며, 이벤트 발생은 일반적으로 다음의 형식과 같습니다:

```go
ctx.EventManager().EmitEvent(
    sdk.NewEvent(eventType, sdk.NewAttribute(attributeKey, attributeValue)),
)
```

모듈의 `handler` 함수는 `context`에 새로운 `EventManager`를 다음과 같이 설정하여 발생한 이벤트들을 `message` 별로 구분할 수 있습니다:

```go
func NewHandler(keeper Keeper) sdk.Handler {
    return func(ctx sdk.Context, msg sdk.Msg) (*sdk.Result, error) {
        ctx = ctx.WithEventManager(sdk.NewEventManager())
        switch msg := msg.(type) {
```

일반적으로 이벤트를 구현하고 모듈의 `EventManager`를 사용하는 자세한 내용은 [`Msg` services](../building-modules/msg-services.md) 컨셉(concept)문서를 참조하십시오.

## 이벤트 구독(subscribe)

Tendermint의 [Websocket](https://docs.tendermint.com/master/tendermint-core/subscription.html#subscribing-to-events-via-websocket)를 사용하면 다음과 같이 `subscribe` RPC 메서드를 호출하여 이벤트를 구독할 수 있습니다:

```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "id": "0",
  "params": {
    "query": "tm.event='eventCategory' AND eventType.eventAttribute='attributeValue'"
  }
}
```

구독할 수있는 주요 `eventCategory`는 다음과 같습니다:

- `NewBlock`: `BeginBlock`과 `EndBlock` 도중 트리거된 이벤트.
- `Tx`: `DeliverTx` (i.e. 트랜잭션 처리) 도중 트리거된 이벤트.
- `ValidatorSetUpdates`: 블록을 위한 검증자 모음(set) 업데이트.

이러한 이벤트는 블록이 커밋(commit)된 이후 `state` 패키지에 의해 트리거됩니다. [Tendermint의 Godoc 페이지](https://godoc.org/github.com/tendermint/tendermint/types#pkg-constants)에서 이벤트 카테고리의 전체 리스트를 확인할 수 있습니다..

`query`의 `type`과 `attribute` 값을 통해 원하는 특정 이벤트를 필터링 할 수 있습니다. 예를 들면 `transfer` 트랜잭션은 `Transfer` 타입의 이벤트를 트리거하고 `attributes`인 `Recipient`와 `Sender`  (`bank` 모듈의 [`events.go`](https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/x/bank/types/events.go) 파일에 정의됨)를 갖고 있습니다. 이 이벤트에 구독하는 방법은 다음과 같습니다:

```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "id": "0",
  "params": {
    "query": "tm.event='Tx' AND transfer.sender='senderAddress'"
  }
}
```

`senderAddress`는 [`AccAddress`](../basics/accounts.md#addresses)의 형식을 따르는 주소입니다.

## 유형화된(typed) 이벤트 (공개 임박)

앞서 설명한 것처럼, 이벤트는 모듈 단위로 정의됩니다. 이벤트 타입과 이벤트 속성을 정의하는 것은 모듈 개발자의 몫입니다. 불행히도 `spec/XX_events.md`을 파일을 제외하고는 이러한 이벤트 타입과 속성은 쉽게 검색할 수 없으므로, SDK는 이벤트를 발생시키고 쿼리를 하기 위해 Protobuf에서 정의한 [Typed Events](../architecture/adr-032-typed-events.md)를 제안합니다.

유형화된(Typed) 이벤트 제안은 아직 완전히 구현되지 않았습니다. 문서는 아직 제공되지 않습니다.

## 다음 {hide}

SDK [텔레메트리(telemetry)](./telemetry.md)에 대해 알아봅시다 {hide}
