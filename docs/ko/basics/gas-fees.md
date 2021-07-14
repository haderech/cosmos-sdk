<!--
order: 5
-->

# 가스와 비용

이 문서는 Cosmos SDK 애플리케이션 내의 가스와 비용을 다루는 기본 전략에 대해서 설명합니다. {synopsis}

### 사전 요구 학습

- [SDK 애플리케이션 해부](./app-anatomy.d) {prereq}

## `Gas` 와 `Fees` 소개

Cosmos SDK 에서 `gas` 는 실행에 사용되는 자원 소모량을 추적하는데 사용되는 특별한 단위입니다. `gas` 는 일반적으로 저장소에서 읽고 쓸때마다 소비되고, 비싼 계산을 해야할 때에도 소비됩니다. 가스는 크게 두가지 목적이 있습니다.

- 블록들이 너무 많은 자원을 소비하지 않고 종료되도록 합니다. SDK 에서 [block gas meter](#block-gas-meter) 를 통해 기본형이 구현되어 있습니다.
- 엔드유저의 무분별한 사용이나 스팸을 방지합니다. 이를 위해, `gas` 는  [`message`](../building-modules/messages-and-queries.md#messages) 실행에는 일반적인 가격이 책정되어 있고, `fee` (`fees = gas * gas-prices`)가 됩니다. `fees` 는 일반적으로 `message` 의 발신자에게 부과됩니다. SDK 는 기본적으로 `gas` 가격을 강제하지 않고, 다른 방법으로 스팸을 막습니다(예: 대역폭 체계). 그러나 대부분의 애플리케이션이 스팸을 방지하기 위해 `fee` 메커니즘을 구현합니다. 이는  [`AnteHandler`](#antehandler) 를 통해 수행됩니다. 

## 가스 미터

Cosmos SDK 에서, `gas` 는 `uint64` 이며, _gas meter_ 라고 불리는 객체에 의해 관리됩니다. _gas meter_ 는 `Gasmeter` 인터페이스를 구현합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/store/types/gas.go#L34-L43

where:

- `GasConsumed()` 는 가스 미터 인스턴스가 소비된 가스의 양을 반환해 줍니다.
- `GasConsumedToLimit()` 는 가스 미터 인스턴스가 소비된 가스의 양을 반환해주거나 제한에 다다랐을 때 그 양을 반환합니다.
- `Limit()` 가스 미터 인스턴스의 제한값을 반환합니다. 가스 미터가 무한이라면 `0` 을 반환합니다.
- `ConsumeGas(amount Gas, descriptor string)` 는 제공된 `gas` 양을 소비합니다. `gas` 가 넘치면, `descriptor` 메시지와 함께 패닉상태가 됩니다.
- `IsPastLimit()` 는 가스 미터 인스턴스가 소비하는 가스의 양이 제한을 초과하면 `true` 를 반환하고, 그렇지 않다면 `false` 를 반환합니다.
- `IsOutOfGas()` 는 가스 미터 인스턴스가 소비하는 가스의 양이 제한과 같거나 초과하면 `true` 를 반환하고, 그렇지 않다면 `false` 를 반환합니다.

가스 미터는 일반적으로 [`ctx`](../core/context.md) 에 있습니다. 그리고 가스 소비는 다음의 패턴으로 수행됩니다.

```go
ctx.GasMeter().ConsumeGas(amount, "description")
```

기본적으로, Cosmos SDK 는 두가지의 가스미터를 사용하는데, [메인 가스 미터](#메인-가스-미터) 와 [블록 가스 미터](#블록-가스-미터) 입니다.


### 메인 가스 미터

`ctx.GasMeter()`  는 애플리케이션의 메인 가스 미터입니다. 메인 가스 미터는 `setDeliverState` 를 통해 `BeginBlock` 에서 초기화되며, 그러고 나서 상태 변경으로 이어지는 실행 시퀀스 동안의 가스 소비량을 추적합니다. 즉, 원래  [`BeginBlock`](../core/baseapp.md#beginblock), [`DeliverTx`](../core/baseapp.md#delivertx) and [`EndBlock`](../core/baseapp.md#endblock) 에 의해 트리거됩니다. 각 `DeliverTx` 의 시작 시 메인 가스 미터가 `AnteHandler` 에서 __반드시 0으로 설정되어야__ 매 트랜잭션의 가스 소비량을 추적할 수 있습니다.      

가스 소비량은 일반적으로 [`BeginBlocker`, `EndBlocker`](../building-modules/beginblock-endblock.md) 또는 [`Msg` 서비스](../building-modules/msg-services.md) 에서 모듈 개발자가 수동으로 설정할 수도 있지만, 대부분 저장소에서 읽고 쓸 때마다 자동으로 처리됩니다. 이 자동 가스 소비 로직은 [`GasKv`](../core/store.md#gaskv-store) 라고 불리는 특별한 저장소에 구현되어 있습니다.

### 블록 가스 미터

`ctx.BlockGasMeter()` 는 블록당 가스 소비량을 추적하고, 일정 한계를 초과하지 않도록 하기위해 사용되는 가스 미터입니다. `BlockGasMeter` 는 [`BeginBlock`](../core/baseapp.md#beginblock) 이 호출될 때 마다 새 인스턴스가 만들어집니다. `BlockGasMeter` 는 유한하며, 각 블럭의 가스 한도는 애플리케이션의 합의 매개 변수에서 정의됩니다. 기본적으로 Cosmos SDK 애플리케이션이 사용하는 기본 합의 매개 변수는 Tendermint 가 제공합니다. 

+++ https://github.com/tendermint/tendermint/blob/v0.34.0-rc6/types/params.go#L34-L41

새 [트랜잭션](../core/transactions.md) 이 `DeliverTx` 를 통해 처리될 때, `BlockGasMeter` 의 현재 값이 한계인지 체크됩니다. 만약 한계 값이라면 `DeliverTx` 는 즉시 반환됩니다. `BeginBlock` 자체가 가스를 소비할 수도 있기 때문에, 이는 블록의 첫 번째 트랜잭션에서도 발생할 수 있는 일입니다. 만약 한계 값이 아니라면 트랜잭션은 정상적으로 처리됩니다. `DeliverTx` 의 마지막에, `ctx.BlockGasMeter()` 로 추적되는 가스는 트랜잭션 처리에 소비된 양만큼 증가됩니다:  

```go
ctx.BlockGasMeter().ConsumeGas(
	ctx.GasMeter().GasConsumedToLimit(),
	"block gas meter",
)
```

## AnteHandler

The `AnteHandler` is run for every transaction during `CheckTx` and `DeliverTx`, before a Protobuf `Msg` service method for each `sdk.Msg` in the transaction. `AnteHandler`s have the following signature:

`AnteHandler` 는 `CheckTx` 와 `DeliverTx` 동안 모든 트랜잭션에서 수행되는데, 트랜잭션의 각 `sdk.Msg` 에 대한 프로토콜 버퍼 `Msg` 서비스 메서드 이전에 수행됩니다. `AnterHandler` 는 다음과 같은 형태를 지닙니다.  

```go
// AnteHandler authenticates transactions, before their internal messages are handled.
// If newCtx.IsZero(), ctx is used instead.
type AnteHandler func(ctx Context, tx Tx, simulate bool) (newCtx Context, result Result, abort bool)
```

`AnteHandler` 는 SDK 코어가 아니라 모듈에 구현됩니다. 이를 통해 개발자들이 그들의 애플리케이션 필요에 맞는 `AnteHandler` 버전을 선택할 수 있습니다. 그렇긴해도 오늘날 대부분의 애플리케이션들이 [`auth` 모듈](https://github.com/cosmos/cosmos-sdk/tree/master/x/auth) 에 정의된 기본 구현을 사용합니다. 보통의 Cosmos SDK 애플리케이션에서의 `AnteHandler` 가 수행하는 일들은 다음과 같습니다. 

- 트랜잭션이 올바른 타입인지 검사합니다. 트랜잭션 타입은 `AnteHandler` 를 구현하는 모듈에 정의되어 있으며, 트랜잭션 인터페이스를 따릅니다: +++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/types/tx_msg.go#L49-L57
 이를 통해 개발자들는 그들의 애플리케이션 트랜잭션을 위해 다양한 타입을 사용할 수 있습니다. 기본 `auth` 모듈에서 기본 트랜잭션 타입은 `Tx` 입니다: +++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/proto/cosmos/tx/v1beta1/tx.proto#L12-L25
  
- 트랜잭션에 포함된 각 [`message`](../building-modules/messages-and-queries.md#messages) 의 서명을 검사합니다. 각 `message` 는 한명 혹은 여러 발신자에 의해 서명되어야 하며, 이 서명들은 반드시 `AnteHandler` 에 의해 검증되어야 합니다.

- `CheckTx` 동안, 거래와 함께 제공되는 가스 가격이 로컬 `min-gas-prices` 보다 큰지 검사합니다.(리마인드하자면, 가스 가격은 다음의 등식으로 도출됩니다: `fees = gas * gas-prices`). `min-gas-prices` 는 각 풀노드의 로컬 매개 변수이며, `CheckTx` 에서 최소 비용을 제공하지 않은 트랜잭션을 버릴 때 사용됩니다.

- 트랜잭션의 발신자가 비용을 감당할 수 있는 충분한 자금을 보유하고 있는지 검사합니다. 엔드유저가 트랜잭션을 생성하면, 그들은 반드시 다음 3개의 매개 변수 중 2개를 알려줘야 합니다(세 번째는 암시적): `fees`, `gas` 그리고 `gas-prices`. 이는 발신자들이 그들의 트랜잭션 실행을 실행하는 노드들을 위해 얼마만큼을 지불할 용의가 있는지 알려줍니다. 제공된 `gas` 값은 `GasWanted` 라는 매개 변수로 저장되고 나중에 사용됩니다.

- `newCtx.gasMeter` 를 0으로 설정하고 ,`GasWanted` 를 한도로 둡니다. __이 단계는 매우 중요한데,__ 트랜잭션이 가스를 무한으로 소비하지 못하도록 할 뿐만 아니라, `ctx.GasMeter` 가 각 `DeliverTx` 사이에서 재설정 되도록 합니다.(`ctx` 는 `Antehandler` 가 실행된 후 `newCtx` 로 설정되며, `AnteHandler` 는 `DeliverTx` 가 호출될 때 마다 수행됩니다.)

위에서 설명한 바와 같이, `AnteHandler` 는 트랜잭션 실행에 소비할 수 있는 가스의 최대 한도인 `GasWanted` 를 반환합니다. 최종적으로 실제 소비량은 `GasUsed` 로 표시되고, 반드시 `GasUsed =< GasWanted` 여야 합니다. `GasWanted` 와 `GasUsed` 모두 [`DeliverTx`](../core/baseapp.md#delivertx) 가 리턴될 때 합의 엔진으로 전달됩니다.  

## Next {hide}

[baseapp](../core/baseapp.md) 에 대해서 알아보세요 {hide}
