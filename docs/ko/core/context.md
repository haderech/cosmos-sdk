<!--
order: 3
-->

# 컨텍스트(Context)

`context`는 애플리케이션의 현재 상태에 관한 정보를 함수에서 함수로 전달하는 자료 구조입니다. 이를 통해 `gasMeter`나, `block height`, `consensus parameters`등과 같은 유용한 객체와 정보들뿐만 아니라 분기된 저장소(branched storage, 전체 상태의 안전한 분기)에 접근할 수 있습니다. {synopsis}

## 사전 요구 지식

- [SDK 애플리케이션의 해부](../basics/app-anatomy.md) {prereq}
- [트랜잭션의 생명주기](../basics/tx-lifecycle.md) {prereq}

## 컨텍스트 정의

SDK `Context`는 Go의 stdlib [`context`](https://golang.org/pkg/context)을 기반으로 하고, 정의 내부에 Cosmos SDK의 전용으로 다양한 추가적인 타입을 포함하는 사용자 정의(custom) 자료 구조입니다. `Context`는 모듈이 [`multistore`](./store.md#multistore) 내부의 [store](./store.md#base-layer-kvstores)에 쉽게 접근하고 블록 헤더와 가스 미터(gas meter)와 같은 트랜잭션의 컨텍스트를 쉽게 검색할 수 있다는 점에서 트랜잭션 처리에 필수적입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/types/context.go#L16-L39

- **Context:** 기반(base) 타입은 Go [Context](https://golang.org/pkg/context)이며, 하단의 [Go Context 패키지](#Go-Context-패키지(Package))에서 자세히 설명합니다.
- **Multistore:** 모든 애플리케이션의 `BaseApp`은 `Context`가 생성될 때 제공되는 [`CommitMultiStore`](./store.md#multistore)를 포함합니다. `KVStore()`와 `TransientStore()` 메서드를 호출하여 모듈들이 고유한 `StoreKey`를 사용하여 각각의 [`KVStore`](./store.md#base-layer-kvstores)를 가져올(fetch) 수 있습니다.
- **ABCI Header:** [header](https://tendermint.com/docs/spec/abci/abci.html#header)는 ABCI 타입입니다. 블록의 높이(height)와 현재 블록의 제안자(proposer)와 같은 블록체인의 상태에 대한 중요한 정보를 담고 있습니다.
- **Chain ID:** 블록과 관련된 블록체인의 고유 식별 번호입니다.
- **Transaction Bytes:** 컨텍스트를 사용하여 처리되는 트랜잭션의 `[]byte` 표현입니다. 모든 트랜잭션은 [생명주기(lifecycle)](../basics/tx-lifecycle.md) 동안 SDK의 다양한 요소 및 합의(consensus) 엔진 (e.g. Tendermint)에 의해 처리되며, 일부는 트랜잭션의 유형에 대해 이해하지 못합니다. 따라서 트랜잭션은 [Amino](./encoding.md)같은 일종의 [인코딩 형식(encoding format)](./encoding.md)을 사용하여 일반(generic) `[]byte`타입으로 마셜링(marshal)됩니다.
- **Logger:** `logger`는 Tendermint 라이브러리에 있습니다. [여기](https://tendermint.com/docs/tendermint-core/how-to-read-logs.html#how-to-read-logs)에서 로그에 대한 자세한 내용을 참조하십시오. 모듈은 이 메서드를 호출하여 고유한 모듈별 로거를 생성합니다.
- **VoteInfo:** ABCI 타입인 [`VoteInfo`](https://tendermint.com/docs/spec/abci/abci.html#voteinfo)의 목록이며, 검증자(validator)의 이름과 그들이 블록에 사인했는지 여부를 나타내는 불리언(boolean) 값을 포함하고 있습니다.
- **Gas Meters:** 구체적으로 이 컨텍스트를 사용하여 처리되는 트랜잭션을 위한 [`gasMeter`](../basics/gas-fees.md#main-gas-meter)와 속해있는 전체 블록을 위한 [`blockGasMeter`](../basics/gas-fees.md#block-gas-meter)이 있습니다. 사용자는 트랜잭션의 실행을 위해 지불하고 싶어하는 수수료를 지정하고 가스 미터는 이 트랜잭션 또는 블록에서 현재까지 사용한 [가스(gas)](../basics/gas-fees.md)를 추적합니다. 가스 미터가 소진되면 실행은 중지(halt)됩니다.
- **CheckTx Mode:** 트랜잭션이 `CheckTx` 또는 `DeliverTx` 모드로 실행되는지를 나타내는 불리언 값입니다.
- **Min Gas Price:** 노드가 블록에 트랜잭션을 포함시키기 위해 취할 수 있는 최소 [가스(gas)](../basics/gas-fees.md) 가격입니다. 이 가격은 각 노드가 개별적으로 설정한 로컬 값이며, 따라서 **상태 전환(state-transitions)으로 이어지는 시퀀스에서 사용되는 어떠한 함수에서도 사용되어서는 안됩니다.**
- **Consensus Params:** ABCI 타입 [Consensus Parameters](https://tendermint.com/docs/spec/abci/apps.html#consensus-parameters)는 블록의 최대 가스(maximum gas for a block)와 같이 블록체인에 대한 특정 제한(limits)을 명시합니다.
- **Event Manager:** 이벤트 매니져는 `Context`에 대한 접근이 가능한 모든 호출자(caller)가 [`Events`](./events.md)를 발생시킬 수 있도록 허용합니다. 모듈은 다양한 `Types`와 `Attributes`를 정의하거나 `types/`에서 찾을 수 있는 공통 정의(common definitions)를 사용함으로써, 모듈 전용의 `Events`를 정의할 수 있습니다. 클라이언트는 이러한 `Events`를 구독(subscribe)하거나 쿼리 할 수 있습니다. 이 `Events`는 `DeliverTx`와, `BeginBlock`, `EndBlock`를 통해 수집되고(collected) 인덱싱을 위해 Tendermint에 반환됩니다. 예를 들면:

```go
ctx.EventManager().EmitEvent(sdk.NewEvent(
    sdk.EventTypeMessage,
    sdk.NewAttribute(sdk.AttributeKeyModule, types.AttributeValueCategory)),
)
```

## Go Context 패키지(Package)

기본적인 `Context`는 [Golang Context Package](https://golang.org/pkg/context)에서 정의되어 있습니다. `Context`는 API와 프로세스 전반에 걸쳐 요청 범위(request-scoped) 데이터를 전송하는 불변의(immutable) 자료 구조입니다. Contexts는 동시성(concurrency)를 허용하고 goroutine에서 사용하도록 설계되었습니다.

Context는 **불변이어야**하므로 편집되어서는 안됩니다. 대신 `With` 함수를 사용하여 부모로부터 자식 Context를 생성합니다. 예시:

```go
childCtx = parentCtx.WithBlockHeader(header)
```

[Golang Context Package](https://golang.org/pkg/context) 문서는 개발자들로 하여금 프로세스의 첫번째 매개 변수(argument)로 `ctx`라는 context를 명시적으로 넘기도록 지시합니다.

## 스토어(store) 분기(branching)

`Context`는 `CacheMultiStore`(`CacheMultiStore` 내부의 쿼리는 향후에 왕복되지(round trips) 않도록 캐시됨)를 사용하여 분기 및 개싱 기능을 허용하는 `MultiStore`를 포함하고 있습니다. 각 `KVStore`는 안전하고 구분된 일시적인(ephemeral) 저장소에 분기됩니다. 프로세스는 변경 사항을 `CacheMultiStore`에 자유롭게 쓸 수 있습니다. 상태 전환(state-transition) 시퀀스가 문제없이 수행되는 경우, 스토어 분기는 시퀀스 마지막에 내재된 스토어에 커밋(commit)될 수 있으며, 문제가 발생하는 경우 무시(disregard)할 수 있습니다. Context의 사용 패턴은 다음과 같습니다:

1. 프로세스는 수행하기 위한 정보를 제공하는 `ctx` Context를 부모 부로세스로부터 수신한다.
2. `ctx.ms`는 **분기된 스토어**(i.e. [multistore](./store.md#multistore)의 분기)이며 원본 `ctx.ms`를 변경하지 않고 실행 도중 상태를 변경할 수 있도록 만들어졌습니다. 이것은 수행 도중 특정 지점에서 변경 사항을 되돌려야 하는 경우 내재된 multistore를 보호하는 데 유용합니다.
3. 프로세스가 실행 중인 `ctx`를 읽고 쓸 수 있습니다. 이 과정은 하위 프로세스를 호출하고 필요하다면 `ctx`를 전달할 수 있습니다.
4. 하위 프로세스가 반환되면 결과가 성공인지 실패인지 여부를 확인합니다. 실패한 경우 어떠한 처리도 필요하지 않습니다. 즉, `ctx`는 단순히 폐기(discard)됩니다. 성공한 경우 `CacheMultiStore`의 변경 사항을 `Write()`를 통하여 원본 `ctx.ms`에 커밋할 수 있습니다.

예를 들어, 다음은 [`baseapp`](./baseapp.md)의 [`runTx`](./baseapp.md#runtx-and-runmsgs) 함수의 스니펫입니다:

```go
runMsgCtx, msCache := app.cacheTxContext(ctx, txBytes)
result = app.runMsgs(runMsgCtx, msgs, mode)
result.GasWanted = gasWanted

if mode != runTxModeDeliver {
  return result
}

if result.IsOK() {
  msCache.Write()
}
```

프로세스는 다음과 같습니다:

1. 트랜잭션의 메시지에 대해 `runMsgs`를 호출하기 전에 `app.cacheTxContext()`를 사용하여 context와 multistore를 분기하고 캐시합니다.
2. `runMsgCtx`는 분기된 스토어와 context이며 `runMsgs`에서 결과를 반환하기 위해 사용됩니다.
3. [`checkTxMode`](./baseapp.md#checktx)에서 프로세스가 실행 중인 경우, 변경 사항을 기록할 필요가 없으며 결과는 즉시 반환됩니다.
4. 프로세스가 [`deliverTxMode`](./baseapp.md#delivertx)에서 실행 중이고 결과가 모든 메시지에 대해 성공적으로 수행되었음을 나타낸다면, 분기된 multistore는 원본에 다시 기록됩니다.

## 다음 {hide}

[노드 클라이언트](./node.md)에 대해 알아봅시다 {hide}
