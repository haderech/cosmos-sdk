<!--
order: 3
-->

# 블록체인 아키텍쳐

## 상태 기계 (State machine)

블록체인은 근본적으로 [복제된 결정론적 상태 기계(replicated deterministic state machine)](https://en.wikipedia.org/wiki/State_machine_replication) 입니다.

상태 기계는 기계가 다양한 상태를 가질 수 있으나, 한 시점에는 오직 하나의 상태만을 가져야하는 컴퓨터 공학 개념입니다. `state` 는 시스템의 현재 상태를 나타내고, 
`transactions` 는 상태 변화를 트리거합니다.

상태 S 와 트랜잭션 T 가 주어지면, 상태 기계는 새로운 상태 S' 를 반환할 것입니다.

```
+--------+                 +--------+
|        |                 |        |
|   S    +---------------->+   S'   |
|        |    apply(T)     |        |
+--------+                 +--------+
```

실제로는 보다 효율적인 프로세스를 위해서 트랜잭션들은 블록으로 묶음처리 됩니다. 상태 S와 트랜잭션 블록B 가 주어지면, 상태 기계는 새로운 상태 S' 를 반환할 것입니다.

```
+--------+                              +--------+
|        |                              |        |
|   S    +----------------------------> |   S'   |
|        |   For each T in B: apply(T)  |        |
+--------+                              +--------+
```

블록체인 개념 내에서, 상태 기계는 결정론적입니다. 이 뜻은 노드가 어떤 상태에서 트랜잭션 시퀀스를 수행하는 것을 동일하게 여러번 반복해도, 항상 같은 최종 상태에 
도달한다는 것입니다.

Cosmos SDK 는 개발자들에게 애플리케이션 상태와 트랜잭션 타입, 상태 변화 기능을 정의하는 것에 최대한의 유연성을 제공합니다. Cosmos SDK 와 상태 기계를 
구축하는 프로세스는 이어지는 섹션에서 보다 자세하게 설명하겠습니다. 우선, **Tendermint** 를 사용해서 어떻게 상태 기계를 복제하는지 알아보겠습니다.

## Tendermint

Cosmos SDK 덕분에, 개발자들은 단지 상태 기계만 정의하면 됩니다. 
[*Tendermint*](https://tendermint.com/docs/introduction/what-is-tendermint.html) 가 네트워크를 통해 그들을 복제하는 것을 처리할 것입니다.

```
                ^  +-------------------------------+  ^
                |  |                               |  |  Cosmos SDK로 빌드
                |  |       상태 기계 = 애플리케이션ㅤㅤㅤㅤ|  |
                |  |                               |  v
                |  +-------------------------------+
                |  |                               |  ^
    블록체인 노드 ㅤ|  |         합의(consensus)ㅤㅤㅤㅤㅤ |  |
                |  |                               |  |
                |  +-------------------------------+  |  Tendermint 코어
                |  |                               |  |
                |  |             네트워크        ㅤㅤㅤ|  |
                |  |                               |  |
                v  +-------------------------------+  v
```

[Tendermint](https://tendermint.com/docs/introduction/what-is-tendermint.html) 는 애플리케이션-애그노스틱(Application-agnostic) 
엔진으로써 블록체인의 *네트워크* 및 *합의* 레이어를 처리하는것을 책임집니다. 즉, Tendermint 는 트랜잭션 바이트의 전송 및 바이트 순서 정렬(ordering)을 
담당한다는 뜻입니다. Tendermint 코어는 비잔틴 장애 허용(Byzantine-Fault-Tolerant, BFT) 알고리즘을 사용하여 트랜잭션 순서 합의에 도달합니다.

Tendermint [합의 알고리즘](https://docs.tendermint.com/v0.34/introduction/what-is-tendermint.html#consensus-overview) 은 
*검증자 (Validator)* 라는 특수한 노드들의 세트와 함께 작동합니다. 검증자 는 트랜잭션 블록들을 블록체인에 추가하는 역할을 합니다. 어떤 블록이라도 검증자 
세트 V 가 있습니다. V 의 검증자 중 하나가 알고리즘에 의해 다음 제안자(proposer) 로 선택됩니다. V 의 검증자 3분의 2 이상이 
*[prevote](https://docs.tendermint.com/v0.34/spec/consensus/consensus.html#prevote-step-height-h-round-r)* 와 
*[precommit](https://docs.tendermint.com/v0.34/spec/consensus/consensus.html#precommit-step-height-h-round-r)* 에 서명하고, 
포함된 모든 트랜잭션이 유효한 경우 이 블록도 유효한 것으로 간주됩니다. 검증자 세트는 상태 기계 내에 작성된 규칙에 따라서 바뀔 수 있습니다. 

## ABCI

Tendermint 는 [ABCI](https://docs.tendermint.com/v0.34/spec/abci/) 라는 인터페이스를 통해 애플리케이션으로 트랜잭션을 전달합니다. 
이 ABCI 는 애플리케이션이 반드시 구현해야 하는 부분입니다.

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

**Tendermint 는 오직 트랜잭션 바이트들만 처리합니다**. 그 바이트들의 의미까지는 알지 못합니다. Tendermint 가 하는 일은 이 트랜잭션 바이트들을 결정론적으로 
정렬하는 것 뿐입니다. Tendermint 는 ABCI 를 통해 애플리케이션으로 바이트를 전달하고, 트랜잭션에 포함된 메시지 처리의 성공 여부를 반환된 코드가 알려주기를 
기대합니다.

아래는 ABCI 에서 가장 중요한 메시지들입니다.

- `CheckTx`: Tendermint 코어에서 트랜잭션을 수신하면, 애플리케이션으로 전달되어 몇가지 기본적인 요구사항이 충족되는지 확인합니다. `CheckTx` 는 스팸 
  트랜잭션들로부터 전체 노드들의 메모리 풀을 보호하는데 사용됩니다. `AnteHandler` 라는 특별한 핸들러는 수수료는 충분한지, 서명이 유효한지 등과 같은 일련의 
  검증 단계를 수행합니다. 모든 검사가 통과되면, 트랜잭션은 메모리 풀에 추가되고 피어 노드들에게 전달됩니다. 트랜잭션이 아직 블록에 담기지 않았기 때문에 
  `CheckTx` 로 트랜잭션이 처리된것은 아닙니다.(즉, 상태 변경이 일어난것이 아님)
- `DeliverTx`: Tendermint 코어에서 [유효한 블록](https://docs.tendermint.com/v0.34/spec/blockchain/blockchain.html#validation) 을 
  수신하면, 블록의 각 트랜잭션은 처리를 위해 `DeliverTx` 를 통해서 애플리케이션으로 전달됩니다. 이 단계에서 상태 변경이 발생합니다. `AnteHandler` 는 
  트랜잭션의 각 메시지를 위한 실제 [`Msg` service](../building-modules/msg-services.md) RPC 와 함께 다시 실행됩니다.
- `BeginBlock`/`EndBlock`: 이 메시지들은 블록내 트랜잭션 포함여부와 관계없이 각 블록의 시작과 끝에서 실행됩니다. 이는 로직의 자동실행을 트리거하기에 
  유용합니다. 하지만 컴퓨팅 비용이 비싼 루프로 인해 블록체인이 느려질 수도 있고, 무한 루프라면 심지어 멈출 수도 있으므로 주의해야 합니다. 

ABCI 메서드에 대해 더 자세한 내용은 [Tendermint docs](https://docs.tendermint.com/v0.34/spec/abci/abci.html#overview) 에서 찾을 수 
있습니다.

Tendermint 를 기반으로 하는 모든 애플리케이션은 레이어 하부의 로컬 Tendermint 엔진과 상호작용을 하기위해 ABCI 인터페이스를 구현해야 하지만, 다행히 이를 
직접하지 않아도 됩니다. Cosmos SDK 가 [baseapp](./sdk-design.md#baseapp) 형태로 보일러플레이트 구현을 제공합니다. 

## Next {hide}

[SDK 의 하이레벨 설계 원칙](./sdk-design.md) 에 대해서 읽어보세요 {hide}
