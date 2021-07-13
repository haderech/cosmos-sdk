<!--
order: 2
-->

# 트랜잭션 생명 주기

이 문서에서는 생성부터 커밋된 상태 변경까지 트랜잭션의 수명 주기에 대해 설명합니다. 트랜잭션 정의는 [다른 문서](../core/transactions.md)에 설명되어 있습니다. 트랜잭션을 'Tx'라고 합니다. {개요}입니다.

### 사전 읽기

- [Anatomy of an SDK Application](./app-anatomy.md) {prereq}

## 생성

### 트랜잭션 생성

주요 애플리케이션 인터페이스 중 하나는 커맨드-라인 인터페이스입니다. 트랜잭션 `Tx`는 사용자가 [command-line](../core/cli.md)의 다음 형식으로 명령을 입력하여 생성할 수 있으며, 이는 트랜잭션 타입의 `[command]`, 인수인 `[args]`, 가스 가격 같은 설정값인 `[flags]`로 구성됩니다.

```bash
[appname] tx [command] [args] [flags]
```

명령은 자동으로 트랜잭션을 **생성**하고, 계정의 개인키를 사용하여 **서명**한 뒤, 지정된 피어 노드로 **전송**합니다.

트랜잭션 생성을 위한 몇 가지 필수 플래그와 선택 플래그가 있습니다. '--from' 플래그는 트랜잭션을 발생시키는 [account](.accounts.md)를 지정합니다. 예를 들어, 코인을 전송하는 트랜잭션에서 코인은 지정된 `from` 주소에서 인출됩니다.

#### 가스와 가스비

또한 사용자들은 [flags](../core/cli.md)를 통하여 [가스비](./gas-fees.md)를 얼마나 지불할지 지정할 수 있습니다.

- `--gas`는 연산자원을 나타내는 [가스](.gas-fees.md)의 소비량을 나타냅니다. 가스는 거래에 따라 달라지며 실행 시까지 정밀하게 계산되지 않지만 `--gas` 값으로 `auto`를 입력하여 추정할 수 있습니다.
- `--gas-adjustment`(선택)은 `가스`를 적게 추정되지 않게 크기를 키울 때 사용됩니다. 예를 들어, 사용자는 예상 가스의 1.5배를 사용하도록 가스 조정을 1.5로 지정할 수 있습니다.
- `--gas-prices`는 사용자가 단위 가스당 얼마를 지불할 것인지를 지정합니다. 이 값은 한가지 혹은 여러 종류의 토큰으로 지정이 가능합니다. 예를 들어 `--gas-prices=0.025uatom, 0.025upho`는 사용자가 가스 단위당 0.025uatom 및 0.025uppo를 지불할 의사가 있음을 의미합니다.
- `--fees`는 사용자가 총 얼마의 수수료를 지불할 것인지를 지정합니다.
- `--timeout-height`는 tx가 특정 높이를 초과하여 커밋되지 않도록 블록 높이를 지정합니다.

최종 지불된 수수료의 가치는 가스에 가스 가격을 곱한 것과 같습니다. 다시 말해, `fees = ceil(gas * gasPrices)`입니다. 따라서 가스 가격에 따라 요금을 계산할 수 있고, 그 반대의 경우도 마찬가지이므로 사용자는 둘 중 하나만 지정합니다.

나중에 검증자는 주어진 또는 계산된 `가스비`를 `최소 가스비`와 비교하여 거래를 블록에 포함할지 여부를 결정합니다. `Tx`는 가스비가 높지 않으면 거부되기 때문에 더 많은 가스비를 지불하도록 유도됩니다.

#### CLI 예시

애플리케이션 `app` 사용자는 CLI에 다음 명령을 입력하여 1000uatom을 `senderAddress`에서 `recipientAddress`로 보내는 트랜잭션을 생성할 수 있습니다. 또한 가스 비용을 얼마까지 지불할 것인지를 명시하고 있습니다. 가스 가격을 단위 가스당 0.025Uatom으로 지정하였으며, 자동 견적을 1.5배까지 확대했습니다.

```bash
appd tx send <recipientAddress> 1000uatom --from <senderAddress> --gas auto --gas-adjustment 1.5 --gas-prices 0.025uatom
```

#### 다른 트랜잭션을 생성하는 메서드

커맨드라인은 애플리케이션과 상호 작용하기 쉬운 방법이지만, 애플리케이션 개발자가 정의한 [gRPC 또는 REST 인터페이스](../core/grpc_rest.md)나 애플리케이션 진입 포인트를 사용하여 `Tx`를 생성할 수도 있습니다. 사용자 관점에서는 자신이 사용하는 웹 인터페이스나 월렛를 사용하고(예: [Lunie.io](https://lunie.io/#/) 로 `Tx` 생성), Ledger Nano S로 서명합니다.

## 멤풀에 추가

`Tx`를 받은 각 풀 노드(Tendermint를 운영하는)는 `CheckTx`라는 [ABCI message](https://tendermint.com/docs/spec/abci/abci.html#messages) 를 유효성 확인을 위하여 애플리케이션 레이어로 전달한 뒤, `abci.ResponseCheckTx`를 받게 됩니다. 
만일 `Tx`가 검증된다면, 'Tx'는 노드의 고유한 트랜잭션 메모리 풀인 [**Mempool**](https://tendermint.com/docs/tendermint-core/mempool.html#mempool) 에 처리 대기중으로 저장됩니다. 정직한 노드에서는 유효하지 않은 트랜잭션은 삭제됩니다.
컨센서스가 진행되기 전까지, 노드들은 계속 전달된 트랜잭션의 유효성을 확인하고 이를 연결된 피어에 가십 네트워크를 통해 전달합니다.

### Types of Checks

풀 노드는 `CheckTx`를 진행하는 동안 상태와 관계없는 유효성 검증을 먼저 진행하고, 그 뒤에 상태와 관계있는 검증을 진행합니다. 
이를 통해 유효하지 않는 트랜잭션을 가능한 빠르게 파악하고 해당 트랜잭션을 거부하여 컴퓨팅 파워를 낭비를 방지합니다.

**_Stateless_** 검사에서는 노드가 상태에 접근할 필요가 없으며 - 라이트 클라이언트 또는 오프라인 노드에서도 검증이 가능 - 컴퓨팅 비용이 저렴합니다. 
상태와 관계없는 유효성 검증은 주소의 포함여부에 대한 확인, 음수가 되어서는 안되는 값에 대한 확인과 정의된 다른 지정 로직 등을 포함합니다.  

**_Stateful_** 검사는 커밋된 상태를 기반으로 트랜잭션 및 메시지의 유효성을 검증합니다. 
예를 들어 관련 값이 존재하는지, 그리고 주소와 거래할 수 있는지 검증합니다.
충분한 자금을 보유하고 있는지, 송신자가 거래할 수 있는 권한이 있거나 올바른 소유권을 가지고 있는지 검증합니다.
풀 노드는 항상 일반적으로 다양한 목적의 [여러 버전](../core/baseapp.md#volatile-states) 의 애플리케이션 상태가 있습니다.
예를 들어 트랜잭션의 유효성을 검증하는 과정 중에 노드는 상태를 변경하는데, 이 때 쿼리에 대해 응답하기 위해서 마지막으로 커밋된 상태의 복사본을 갖고 있게 됩니다.
커밋되지 않은 변경 사항이 있는 상태를 사용하여 쿼리에 응답해서는 안 됩니다.

`Tx`를 검증하기 위해서, 풀 노드는 상태 유관한 유효성 검증과 상태 무관한 유효성 검증을 포함한 `CheckTx`를 호출합니다. 
추가적인 유효성 검증은 이후에 [`DeliverTx`](#delivertx) 단계에서 진행됩니다. `CheckTx`는 `Tx` 디코딩을 시작으로 여러 단계를 거치게 됩니다.

In order to verify a `Tx`, full-nodes call `CheckTx`, which includes both _stateless_ and _stateful_
checks. Further validation happens later in the [`DeliverTx`](#delivertx) stage. `CheckTx` goes
through several steps, beginning with decoding `Tx`.

### 디코딩

애플리케이션이 컨센서스 엔진(예: Tendermint)으로부터 `Tx`를 수신하였을 때, [encoded](../core/encoding.md) `[]byte` 형태이기 때문에, `Tx`를 처리하기 위해서는 언마샬링 해야 합니다.
언마셜링 후 `runTxModeCheck` 모드에서 [`runTx`](../core/baseapp.md#runtx-and-runmsgs) 가 호출되고, 
Then, the [`runTx`](../core/baseapp.md#runtx-and-runmsgs) function is called to run in `runTxModeCheck` mode, meaning the function will run all checks but exit before executing messages and writing state changes.

### ValidateBasic

[`sdk.Msg`s](../core/transactions.md#messages) are extracted from `Tx`, and `ValidateBasic`, a method of the `sdk.Msg` interface implemented by the module developer, is run for each one. `ValidateBasic` should include basic **stateless** sanity checks. For example, if the message is to send coins from one address to another, `ValidateBasic` likely checks for nonempty addresses and a nonnegative coin amount, but does not require knowledge of state such as the account balance of an address.

### AnteHandler

After the ValidateBasic checks, the `AnteHandler`s are run. Technically, they are optional, but in practice, they are very often present to perform signature verification, gas calculation, fee deduction and other core operations related to blockchain transactions.

A copy of the cached context is provided to the `AnteHandler`, which performs limited checks specified for the transaction type. Using a copy allows the AnteHandler to do stateful checks for `Tx` without modifying the last committed state, and revert back to the original if the execution fails.

For example, the [`auth`](https://github.com/cosmos/cosmos-sdk/tree/master/x/auth/spec) module `AnteHandler` checks and increments sequence numbers, checks signatures and account numbers, and deducts fees from the first signer of the transaction - all state changes are made using the `checkState`.

### Gas

The [`Context`](../core/context.md), which keeps a `GasMeter` that will track how much gas has been used during the execution of `Tx`, is initialized. The user-provided amount of gas for `Tx` is known as `GasWanted`. If `GasConsumed`, the amount of gas consumed so during execution, ever exceeds `GasWanted`, the execution will stop and the changes made to the cached copy of the state won't be committed. Otherwise, `CheckTx` sets `GasUsed` equal to `GasConsumed` and returns it in the result. After calculating the gas and fee values, validator-nodes check that the user-specified `gas-prices` is less than their locally defined `min-gas-prices`.

### Discard or Addition to Mempool

If at any point during `CheckTx` the `Tx` fails, it is discarded and the transaction lifecycle ends
there. Otherwise, if it passes `CheckTx` successfully, the default protocol is to relay it to peer
nodes and add it to the Mempool so that the `Tx` becomes a candidate to be included in the next block.

The **mempool** serves the purpose of keeping track of transactions seen by all full-nodes.
Full-nodes keep a **mempool cache** of the last `mempool.cache_size` transactions they have seen, as a first line of
defense to prevent replay attacks. Ideally, `mempool.cache_size` is large enough to encompass all
of the transactions in the full mempool. If the the mempool cache is too small to keep track of all
the transactions, `CheckTx` is responsible for identifying and rejecting replayed transactions.

Currently existing preventative measures include fees and a `sequence` (nonce) counter to distinguish
replayed transactions from identical but valid ones. If an attacker tries to spam nodes with many
copies of a `Tx`, full-nodes keeping a mempool cache will reject identical copies instead of running
`CheckTx` on all of them. Even if the copies have incremented `sequence` numbers, attackers are
disincentivized by the need to pay fees.

Validator nodes keep a mempool to prevent replay attacks, just as full-nodes do, but also use it as
a pool of unconfirmed transactions in preparation of block inclusion. Note that even if a `Tx`
passes all checks at this stage, it is still possible to be found invalid later on, because
`CheckTx` does not fully validate the transaction (i.e. it does not actually execute the messages).

## Inclusion in a Block

Consensus, the process through which validator nodes come to agreement on which transactions to
accept, happens in **rounds**. Each round begins with a proposer creating a block of the most
recent transactions and ends with **validators**, special full-nodes with voting power responsible
for consensus, agreeing to accept the block or go with a `nil` block instead. Validator nodes
execute the consensus algorithm, such as [Tendermint BFT](https://tendermint.com/docs/spec/consensus/consensus.html#terms),
confirming the transactions using ABCI requests to the application, in order to come to this agreement.

The first step of consensus is the **block proposal**. One proposer amongst the validators is chosen
by the consensus algorithm to create and propose a block - in order for a `Tx` to be included, it
must be in this proposer's mempool.

## State Changes

The next step of consensus is to execute the transactions to fully validate them. All full-nodes
that receive a block proposal from the correct proposer execute the transactions by calling the ABCI functions
[`BeginBlock`](./app-anatomy.md#beginblocker-and-endblocker), `DeliverTx` for each transaction,
and [`EndBlock`](./app-anatomy.md#beginblocker-and-endblocker). While each full-node runs everything
locally, this process yields a single, unambiguous result, since the messages' state transitions are deterministic and transactions are
explicitly ordered in the block proposal.

```
		-----------------------
		|Receive Block Proposal|
		-----------------------
		          |
			  v
		-----------------------
		| BeginBlock	      |
		-----------------------
		          |
			  v
		-----------------------
		| DeliverTx(tx0)      |
		| DeliverTx(tx1)      |
		| DeliverTx(tx2)      |
		| DeliverTx(tx3)      |
		|	.	      |
		|	.	      |
		|	.	      |
		-----------------------
		          |
			  v
		-----------------------
		| EndBlock	      |
		-----------------------
		          |
			  v
		-----------------------
		| Consensus	      |
		-----------------------
		          |
			  v
		-----------------------
		| Commit	      |
		-----------------------
```

### DeliverTx

The `DeliverTx` ABCI function defined in [`BaseApp`](../core/baseapp.md) does the bulk of the
state transitions: it is run for each transaction in the block in sequential order as committed
to during consensus. Under the hood, `DeliverTx` is almost identical to `CheckTx` but calls the
[`runTx`](../core/baseapp.md#runtx) function in deliver mode instead of check mode.
Instead of using their `checkState`, full-nodes use `deliverState`:

- **Decoding:** Since `DeliverTx` is an ABCI call, `Tx` is received in the encoded `[]byte` form.
  Nodes first unmarshal the transaction, using the [`TxConfig`](./app-anatomy#register-codec) defined in the app, then call `runTx` in `runTxModeDeliver`, which is very similar to `CheckTx` but also executes and writes state changes.

- **Checks:** Full-nodes call `validateBasicMsgs` and the `AnteHandler` again. This second check
  happens because they may not have seen the same transactions during the addition to Mempool stage\
  and a malicious proposer may have included invalid ones. One difference here is that the
  `AnteHandler` will not compare `gas-prices` to the node's `min-gas-prices` since that value is local
  to each node - differing values across nodes would yield nondeterministic results.

- **`MsgServiceRouter`:** While `CheckTx` would have exited, `DeliverTx` continues to run
  [`runMsgs`](../core/baseapp.md#runtx-and-runmsgs) to fully execute each `Msg` within the transaction.
  Since the transaction may have messages from different modules, `BaseApp` needs to know which module
  to find the appropriate handler. This is achieved using `BaseApp`'s `MsgServiceRouter` so that it can be processed by the module's Protobuf [`Msg` service](../building-modules/msg-services.md).
  For `LegacyMsg` routing, the `Route` function is called via the [module manager](../building-modules/module-manager.md) to retrieve the route name and find the legacy [`Handler`](../building-modules/msg-services.md#handler-type) within the module.

- **`Msg` service:** a Protobuf `Msg` service, a step up from `AnteHandler`, is responsible for executing each
  message in the `Tx` and causes state transitions to persist in `deliverTxState`.

- **Gas:** While a `Tx` is being delivered, a `GasMeter` is used to keep track of how much
  gas is being used; if execution completes, `GasUsed` is set and returned in the
  `abci.ResponseDeliverTx`. If execution halts because `BlockGasMeter` or `GasMeter` has run out or something else goes
  wrong, a deferred function at the end appropriately errors or panics.

If there are any failed state changes resulting from a `Tx` being invalid or `GasMeter` running out,
the transaction processing terminates and any state changes are reverted. Invalid transactions in a
block proposal cause validator nodes to reject the block and vote for a `nil` block instead.

### Commit

The final step is for nodes to commit the block and state changes. Validator nodes
perform the previous step of executing state transitions in order to validate the transactions,
then sign the block to confirm it. Full nodes that are not validators do not
participate in consensus - i.e. they cannot vote - but listen for votes to understand whether or
not they should commit the state changes.

When they receive enough validator votes (2/3+ _precommits_ weighted by voting power), full nodes commit to a new block to be added to the blockchain and
finalize the state transitions in the application layer. A new state root is generated to serve as
a merkle proof for the state transitions. Applications use the [`Commit`](../core/baseapp.md#commit)
ABCI method inherited from [Baseapp](../core/baseapp.md); it syncs all the state transitions by
writing the `deliverState` into the application's internal state. As soon as the state changes are
committed, `checkState` start afresh from the most recently committed state and `deliverState`
resets to `nil` in order to be consistent and reflect the changes.

Note that not all blocks have the same number of transactions and it is possible for consensus to
result in a `nil` block or one with none at all. In a public blockchain network, it is also possible
for validators to be **byzantine**, or malicious, which may prevent a `Tx` from being committed in
the blockchain. Possible malicious behaviors include the proposer deciding to censor a `Tx` by
excluding it from the block or a validator voting against the block.

At this point, the transaction lifecycle of a `Tx` is over: nodes have verified its validity,
delivered it by executing its state changes, and committed those changes. The `Tx` itself,
in `[]byte` form, is stored in a block and appended to the blockchain.

## Next {hide}

Learn about [accounts](./accounts.md) {hide}
