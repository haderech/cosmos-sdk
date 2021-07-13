<!--
order: 7
-->

# gRPC와, REST, Tendermint 엔드포인트(endpoints)

이 문서에서는 노드가 표시하는 다음의 모든 엔드포인트에 대한 개요를 보여줍니다: gRPC, REST, 다른 엔드포인트 {synopsis}

## 모든 엔드포인트의 개요

각 노드는 사용자가 노드와 상호작용을 할 수 있도록 다음의 엔드포인트를 제공하고, 각 엔드포인트는 다른 포트에서 제공됩니다. 각 엔드포인트를 구성하는 방법에 대한 세부 정보는 엔드포인트의 자체 섹션을 참조하십시오.

- gRPC 서버 (기본 포트: `9090`),
- REST 서버 (기본 포트: `1317`),
- Tendermint RPC 엔드포인트 (기본 포트: `26657`).

::: tip
이 노드는 Tendermint P2P 엔드포인트, 또는 Cosmos SDK와는 직접적인 관련이 없는 [Prometheus endpoint](https://docs.tendermint.com/master/nodes/metrics.html#metrics)와 같이 다른 엔드포인트 또한 제공합니다. 이러한 엔드포인트들에 대한 자세한 정보는 다음의 [Tendermint 문서](https://docs.tendermint.com/master/tendermint-core/using-tendermint.html#configuration)를 참조하십시오.
:::

## gRPC 서버

::: warning
`go-grpc v1.34.0`에 도입된 패치로 인해 gRPC가 `gogoproto` 라이브러리와 호환되지 않아 일부 [gRPC 쿼리](https://github.com/cosmos/cosmos-sdk/issues/8426)가 패닉(panic) 상태에 빠집니다. 따라서 SDK는 `go.mod`에서 `go-grpc <=v1.33.2`가 설치하는 것을 요구합니다.

gRPC가 제대로 동작하는 것을 보장하기 위해, 애플리케이션의 `go.mod`에 다음의 줄을 추가하는 것을 **강력히 권장**합니다.

```
replace google.golang.org/grpc => google.golang.org/grpc v1.33.2
```

자세한 내용은 [issue #8392](https://github.com/cosmos/cosmos-sdk/issues/8392)를 참조하십시오.
:::

Cosmos SDK v0.40에서는 Protobuf를 주요 [인코딩(encoding)](./encoding) 라이브러리로 도입하였고, 이를 통해 SDK에 연결할 수 있는 다양한 종류의 Protobuf 기반 툴이 제공됩니다. 이러한 툴 중 하나는 다양한 언어에서 클라이언트를 훌륭히 지원하는 현대적인 오픈 소스 고성능 RPC 프레임워크인 [gRPC](https://grpc.io)입니다.

각 모듈은 상태 전환과 상태 쿼리를 정의하기 위해 [`Msg`와 `Query` Protobuf 서비스](../building-modules/messages-and-queries.md)를 제공합니다. 이러한 서비스는 애플리케이션 내의 다음의 함수를 통해 gRPC에 연결됩니다.

<https://github.com/cosmos/cosmos-sdk/blob/v0.41.0/server/types/app.go#L39-L41>

`grpc.Server`는 모든 gRPC 요청을 생성 및 처리하는 구체적인(concrete) gRPC 서버입니다.

- `grpc.enable = true|false` 필드는 gRPC 서버의 활성 여부를 결정합니다. 기본값은 `true`입니다.
- `grpc.address = {string}` field defines the address (really, the port, since the host should be kept at `0.0.0.0`) the server should bind to. Defaults to `0.0.0.0:9090`.
- `grpc.address = {string}` 필드는 서버가 바인딩할 주소(호스트는 `0.0.0.0`으로 유지되어야 하므로 실제로는 포트를 의미)를 정의합니다. 기본값은 `0.0.0.0:9090`입니다..

:::tip
`~/.simapp`은 노드의 구성(configuration) 및 데이터베이스가 저장되는 디렉토리입니다. 기본값은 `~/.{app_name}`입니다.
:::

gRPC 서버가 시작되면 gRPC 클라이언트를 통해 요청을 전송할 수 있습니다. 몇가지 예가 [노드와 상호작용하기](../run-node/interact-node.md#using-grpc) 튜토리얼에 나와있습니다.

Cosmos SDK에서 제공되는 모든 사용가능한 gRPC 엔드포인트에 대한 개요는 [Protobuf 문서](./proto-docs.md)를 참조합시오.

## REST 서버

Cosmos SDK v0.40에서 노드는 REST 서버를 계속 지원합니다. 그러나 v0.39 및 이전 버젼에서 존재하던 기존 경로는 더 이상 사용되지 않음(deprecated)로 표시되며, 새로운 경로가 gRPC-gateway에 추가되었습니다.

모든 경로는 `~/.simapp/config/app.toml`에서 다음의 필드로 구성됩니다:

- `api.enable = true|false` 필드는 REST 서버의 활성화 여부를 결정합니다. 기본값은 `false`입니다.
- `api.address = {string}` 필드는 서버가 바인딩할 주소(호스트는 `0.0.0.0`으로 유지되어야 하므로 실제로는 포트를 의미)를 정의합니다. 기본값은  `tcp://0.0.0.0:1317`입니다..
- 일부 추가 API 구성 옵션은 `~/.simapp/config/app.toml`에서 정의되어 있으며, 해당 파일을 주석과 함께 참조하시기 바랍니다.

### gRPC-gateway REST 라우팅(Routes)

여러가지 이유로 인해 gRPC를 사용할 수 없는 경우(예를 들면, 작성한 웹 애플리케이션의 브라우저가 gRPC가 빌드된 HTTP2를 지원하지 않는 경우), SDK는 gRPC-gateway를 통한 REST 라우팅을 제공합니다.

[gRPC-gateway](https://grpc-ecosystem.github.io/grpc-gateway/)는 gRPC 엔드포인트를 REST 엔트포인트로 제공하는 툴입니다. SDK는 Protobuf 서비스에서 정의된 각 RPC 엔드포인트에 대한 REST의 대응(equivalent)을 제공합니다. 예를 들면 `/cosmos.bank.v1beta1.QueryAllBalances` gRPC 엔드포인트를 통해 잔고를 쿼리할 수 있는데, 그 대신으로 gRPC-gateway `"/cosmos/bank/v1beta1/balances/{address}"` REST 엔트포인트를 사용할 수 있습니다. 두 방법은 동일한 결과를 반환합니다. Protobuf 서비스에서 정의된 각 RPC 메서드에 대해 대응하는 REST 엔드포인트는 다음의 옵션으로 정의됩니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.41.0/proto/cosmos/bank/v1beta1/query.proto#L19-L22

애플리케이션의 개발자의 경우 gRPC-gateway REST 경로(routes)는 REST 서버에 연결되어야 하며, 이는 ModuleManager의 `RegisterGRPCGatewayRoutes` 함수를 호출하면 됩니다.

### 레거시 REST API 라우팅(Routes)

Cosmos SDK v0.39 및 이전 버젼에서 제공되는 REST 경로는 [HTTP deprecation header](https://tools.ietf.org/id/draft-dalal-deprecation-header-01.html)에서 사용되지 않는(deprecated) 경로로 표시됩니다. 하위 호환성을 위해 유지되지만 v0.41에서 제거될 예정입니다. 레거시 REST 경로를 새로운 gRPC-gateway REST 경로로 업데이트하려면, [마이그레이션 가이드](../migrations/rest.md)를 참조하십시오.

애플리케이션 개발자의 경우, 레거시 REST API 경로는 REST 서버와 연결되어야 하며 이는 ModuleManager의 `RegisterRESTRoutes` 함수를 호출하면 됩니다.

### Swagger

[Swagger](https://swagger.io/) (또는 OpenAPIv2) 규격(specification) 파일은 API서버의 `/swagger` 경로에서 제공됩니다. Swagger는 설명과 입력 매개 변수, 반환 타입, 각 엔드포인트의 관련 정보를 포함하여 서버가 수행하는 API 엔드포인트에 대해 설명한 개방형(open) 규격입니다.

`api.swagger`필드를 통해 `~/.simapp/config/app.toml` 내의 `/swagger` 엔드포인트를 활성화 시킬 수 있으며, 기본값은 true입니다.

애플리케이션 개발자의 경우 사용자 지정 모듈을 기반으로 자체 Swagger 정의를 생성할 수 있습니다. SDK의 [Swagger 생성 스크립트](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc4/scripts/protoc-swagger-gen.sh)를 참조하십시오.

## Tendermint RPC

Cosmos SDK와는 별도로, Tendermint 또한 RPC 서버를 제공합니다. 이 RPC 서버는 `~/.simapp/config/config.toml`의 `rpc` 테이블에서 매개 변수를 조정하는 것으로 구성할 수 있으며 기본 수신(listening) 주소는 `tcp://0.0.0.0:26657`입니다. 모든 Tendermint RPC 엔드포인트에 대한 OpenAPI 규격은 [여기](https://docs.tendermint.com/master/rpc/)에서 확인할 수 있습니다.

Some Tendermint RPC endpoints are directly related to the Cosmos SDK:

- `/abci_query`: 이 엔드포인트는 상태를 애플리케이션에 쿼리합니다. `path` 매개 변수에 다음의 문자열을 전송할 수 있습니다:
    - `/cosmos.bank.v1beta1.QueryAllBalances`와 같은 모든 Protobuf 정규화 형식의 서비스 메서드. `data` 필드는 반드시 메서드의 요청 매개변수(들)을 Protobuf를 사용하여 bytes 형식으로 인코딩해야 합니다.
    - `/app/simulate`: 트랜잭션을 시뮬레이션하고 사용된 가스와 같은 일부 정보를 반환합니다.
    - `/app/version`: 애플리케이션의 버젼을 반환합니다.
    - `/store/{path}`: 스토어를 직접 쿼리합니다.
    - `/p2p/filter/addr/{port}`: 주소 포트 별로 필터링된 노드의 P2P 피어의 목록을 반환합니다.
    - `/p2p/filter/id/{id}`: ID 별로 필터링된 노드의 P2P 피어의 목록을 반환합니다.
- `/broadcast_tx_{aync,async,commit}`: 이 3개의 엔드포인트는 다른 피어에게 트랜잭션을 브로드캐스트합니다. CLI와 gRPC, REST는 [트랜잭션을 브로드캐스트하는 방법](./transactions.md#broadcasting-the-transaction)을 제공하는데, 내부적으로는 모두 이 3개의 Tendermint RPC를 사용합니다.

## 비교표(Comparison Table)

| 이름           | 장점                                                                                                                                                            | 단점                                                                                                 |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| gRPC           | - 다양한 언어로 생성된 코드의 스텁(stub)을 사용이 가능<br>- 스트리밍 및 양방향 통신(HTTP2) 지원<br>- 작은 전송용 바이너리 크기, 빠른 전송 | - 브라우저에서 사용할 수 없는 HTTP2 기반<br>- (주로 Protobuf로 인한) 학습 곡선                      |
| REST           | - 범용적<br>- 모든 언어로 작성된 클라이언트 라이브러리, 빠른 구현<br>                                                                                        | - 단방향 요청-응답 통신 (HTTP1.1)만을 지원함<br>- 더 큰 전송 메시지 크기 (JSON) |
| Tendermint RPC | - 사용하기 편리함                                                                                                                                                         | - 더 큰 전송 메시지 크기 (JSON)                                                                   |

## 다음 {hide}

[CLI](./cli.md)에 대해 알아봅시다{hide}
