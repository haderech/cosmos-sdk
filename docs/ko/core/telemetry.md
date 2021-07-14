<!--
order: 10
-->

# 원격 측정

사용자가 원하는 다양한 데이터를 분석하여 애플리케이션과 모듈에 대한 상태를 원격으로 모니터 합니다. {synopsis}

운용자와 개발자는 Cosmos SDK가 제공하는 `telemetry` 패키지를 사용하여 애플리케이션의 성능과 상태를 모니터 할 수 있습니다.
한개의 API endpoint를 `/metrics?format={text|prometheus}`에 설정하여 외부에 있는 Prometheus 서버가 데이터를 `text` 형태로 읽어 갈 수 있습니다.

원격 측정이 활성화가 되면 [go-metrics](https://github.com/armon/go-metrics) 라이브러리를 통하여 측정항목들이 수집되어 API 호출을 통하여 외부에서 읽어갈 수 있습니다.

예제:
```go
func EndBlocker(ctx sdk.Context, k keeper.Keeper) {
  defer telemetry.ModuleMeasureSince(types.ModuleName, time.Now(), telemetry.MetricKeyEndBlocker)

  // ...
}
```

개발자들은 직접 `telemetry` 패키지를 사용하여 측정항목들을 감싸 알아보기 쉽게 라벨을 붙일 수 있고 아니면 직접 `go-metrics` 라이브러리를 사용할 수 있습니다.
하지만 측정항목들에 대하여 알맞은 맥락과 적절한 값들을 표기할 수 있는 `telemetry` 패키지 사용이 일반적으로 더 선호되는 방식입니다.

Cosmos SDK는 측정항목들에 대하여 다음과 같은 타입을 지원합니다:
* 게이지 (gauges)
* 요약 (summaries)
* 카운터 (counters)

## 라벨 (Labels)

모듈의 몇가지 항목들은 그 이름이 자동으로 라벨로 붙게 됩니다 (예:`BeginBlock`).
또한 오퍼레이터는 `telemetry` 패키지를 사용하여 애플리케이션에 제공하는 측정항목들 대하여 글로벌 라벨을 지정할 수도 있습니다 (예: chain-id).
글로벌 라벨은 [name, value] 튜플로 리스트를 만들어 설정 가능합니다.

예제:

```toml
global-labels = [
  ["chain_id", "chain-OfXo4V"],
]
```

## 카디널리티 (Cardinality)

카디널리티는 하나의 세트나 그룹안에 몇개의 유니크한 값들이 있는지를 나타내는 숫자인데 측정항목들을 얼마나 자세히 세분화하여 나타낼지 잘 고려가 되어야 보다 효율적으로 원격으로 값들을 읽어가거나 인텍싱 할 수 있습니다.

개발자들은 특히 측정항목들에 대하여 세분화 할 때 유용한 값들을 나타낼 수 있도록 충분한 카디널리티를 사용해야 하지만 씽크(sink)의 리밋을 초과하지 않도록 조심해야 합니다. 일반적으로 카디널리티는 10을 넘지않도록 합니다.

다음 측정항목들에 대해서는 충분한 세부사항과 알맞은 카디널리티를 가지고 있습니다:

* 블록의 시작과 종료 시간
* 사용한 트랜잭션 개스
* 사용한 블록 개스
* 새로 발급한 토큰 갯수
* 새로 발급한 계정 갯수

하지만 다음 측정항목들은 너무 많은 카디널리티를 가지고 있어 크게 쓸모가 없어 보입니다:

* 계좌 송금 액수
* 특정 주소로부터 투표나 입금액수

## 지원되는 측정항목들

| 측정항목                          | 설명                                                                                       | 유니트            | 타입    |
|:--------------------------------|:------------------------------------------------------------------------------------------|:----------------|:--------|
| `tx_count`                      | `DeliverTx`를 통해 수행한 전체 트랜잭션 수                                                       | tx              | 카운터 |
| `tx_successful`                 | `DeliverTx`를 통해 성공적으로 수행한 트랜잭션 수                                                   | tx              | 카운터 |
| `tx_failed`                     | `DeliverTx`를 통해 실패한 트랜잭션 수                                                           | tx              | 카운터 |
| `tx_gas_used`                   | 트랜잭션이 사용한 개스 양                                                                       | gas             | 게이지 |
| `tx_gas_wanted`                 | 트랜잭션이 요청한 개스 양                                                                       | gas             | 게이지 |
| `tx_msg_send`                   | `MsgSend`를 통해 보낸 토큰수량 (per denom)                                                     | token           | 게이지 |
| `tx_msg_withdraw_reward`        | `MsgWithdrawDelegatorReward`를 통해 출금한 토큰수량 (per denom)                                 | token           | 게이지 |
| `tx_msg_withdraw_commission`    | `MsgWithdrawValidatorCommission`를 통해 출금한 토큰수량 (per denom)                             | token           | 게이지 |
| `tx_msg_delegate`               | `MsgDelegate`를 통해 일임한 토큰수량                                                            | token           | 게이지 |
| `tx_msg_begin_unbonding`        | `MsgUndelegate`를 통해 일임을 해제한 토큰수량                                                    | token           | 게이지 |
| `tx_msg_begin_begin_redelegate` | `MsgBeginRedelegate`를 통해 다시 일임한 토큰수량                                                 | token           | 게이지 |
| `tx_msg_ibc_transfer`           | `MsgTransfer`를 통해 이체한 토큰수량 (source 혹은 sink chain)                                    | token           | 게이지 |
| `ibc_transfer_packet_receive`   |`FungibleTokenPacketData`를 통해 수신한 토큰수량 (source 혹은 sink chain)                         | token           | 게이지 |
| `new_account`                   | 신규 발행한 계좌 수                                                                            | account         | 카운터 |
| `gov_proposal`                  | 거버넌스 제안 수                                                                              | proposal        | 카운터 |
| `gov_vote`                      | 거버넌스 투표 수                                                                              | vote            | 카운터 |
| `gov_deposit`                   | 거버넌스 입금 건수                                                                             | deposit         | 카운터 |
| `staking_delegate`              | 일임을 수행한 수                                                                              | delegation      | 카운터 |
| `staking_undelegate`            | 일임을 취소한 수                                                                              | undelegation    | 카운터 |
| `staking_redelegate`            | 다시 일임한 수                                                                               | redelegation    | 카운터 |
| `ibc_transfer_send`             | 체인(source 혹은 sink)에서 보낸 IBC 이체 수                                                     | transfer        | 카운터 |
| `ibc_transfer_receive`          | 체인(source 혹은 sink)에서 수신한 IBC 이체 수                                                    | transfer        | 카운터 |
| `ibc_client_create`             | 생성한 클라이언트 수                                                                           | create          | 카운터 |
| `ibc_client_update`             | 갱신한 클라이언트 수                                                                           | update          | 카운터 |
| `ibc_client_upgrade`            | 업그레이드 한 클라이언트 수                                                                     | upgrade         | 카운터 |
| `ibc_client_misbehaviour`       | 잘못된 행동을 한 클라이언트 수                                                                   | misbehaviour    | 카운터 |
| `ibc_connection_open-init`      | `OpenInit` 커넥션 수                                                                        | handshake       | 카운터 |
| `ibc_connection_open-try`       | `OpenTry` 커넥션 수                                                                         | handshake       | 카운터 |
| `ibc_connection_open-ack`       | `OpenAck` 커넥션 수                                                                         | handshake       | 카운터 |
| `ibc_connection_open-confirm`   | `OpenConfirm` 커넥션 수                                                                     | handshake       | 카운터 |
| `ibc_channel_open-init`         | `OpenInit` 채널 수                                                                          | handshake       | 카운터 |
| `ibc_channel_open-try`          | `OpenTry` 채널 수                                                                           | handshake       | 카운터 |
| `ibc_channel_open-ack`          | `OpenAck` 채널 수                                                                           | handshake       | 카운터 |
| `ibc_channel_open-confirm`      | `OpenConfirm` 채널 수                                                                       | handshake       | 카운터 |
| `ibc_channel_close-init`        | `CloseInit` 채널 수                                                                         | handshake       | 카운터 |
| `ibc_channel_close-confirm`     | `CloseConfirm` 채널 수                                                                      | handshake       | 카운터 |
| `tx_msg_ibc_recv_packet`        | IBC 패킷 수신 수                                                                             | packet          | 카운터 |
| `tx_msg_ibc_acknowledge_packet` | 인정한 IBC 패킷 수                                                                            | acknowledgement | 카운터 |
| `ibc_timeout_packet`            | 시간 만료된 IBC 패킷 수                                                                        | timeout         | 카운터 |
| `abci_check_tx`                 | ABCI `CheckTx` 기간                                                                         | ms              | 요약 |
| `abci_deliver_tx`               | ABCI `DeliverTx` 기간                                                                       | ms              | 요약 |
| `abci_commit`                   | ABCI `Commit` 기간                                                                          | ms              | 요약 |
| `abci_query`                    | ABCI `Query` 기간                                                                           | ms              | 요약 |
| `abci_begin_block`              | ABCI `BeginBlock` 기간                                                                      | ms              | 요약 |
| `abci_end_block`                | ABCI `EndBlock` 기간                                                                        | ms              | 요약 |
| `begin_blocker`                 | 모듈에 대한 `BeginBlock` 기간                                                                  | ms              | 요약 |
| `end_blocker`                   | 모듈에 대한 `EndBlock` 기간                                                                    | ms              | 요약 |
| `store_iavl_get`                | IAVL `Store#Get` 통화시간                                                                    | ms              | 요약 |
| `store_iavl_set`                | IAVL `Store#Set` 통화시간                                                                    | ms              | 요약 |
| `store_iavl_has`                | IAVL `Store#Has` 통화시간                                                                    | ms              | 요약 |
| `store_iavl_delete`             | IAVL `Store#Delete` 통화시간                                                                 | ms              | 요약 |
| `store_iavl_commit`             | IAVL `Store#Commit` 통화시간                                                                 | ms              | 요약 |
| `store_iavl_query`              | IAVL `Store#Query` 통화시간                                                                  | ms              | 요약 |
| `store_gaskv_get`               | GasKV `Store#Get` 통화시간                                                                   | ms              | 요약 |
| `store_gaskv_set`               | GasKV `Store#Set` 통화시간                                                                   | ms              | 요약 |
| `store_gaskv_has`               | GasKV `Store#Has` 통화시간                                                                   | ms              | 요약 |
| `store_gaskv_delete`            | GasKV `Store#Delete` 통화시간                                                                | ms              | 요약 |
| `store_cachekv_get`             | CacheKV `Store#Get` 통화시간                                                                 | ms              | 요약 |
| `store_cachekv_set`             | CacheKV `Store#Set` 통화시간                                                                 | ms              | 요약 |
| `store_cachekv_write`           | CacheKV `Store#Write` 통화시간                                                               | ms              | 요약 |
| `store_cachekv_delete`          | CacheKV `Store#Delete` 통화시간                                                              | ms              | 요약 |

## 다음 {hide}

[object-capability](./ocap.md)  {hide}