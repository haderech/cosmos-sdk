<!--
order: 3
-->

# 쿼리(Query) 생명주기(Lifecycle)

이 문서에서는 SDK 애플리케이션에서 유저 인터페이스부터 애플리케이션 스토어 및 뒷단(back)까지 쿼리의 생명주기에 대해 설명합니다. {synopsis}

## 사전 요구 지식

- [트랜잭션(transaction) 생명주기](./tx-lifecycle.md) {prereq}

## 쿼리 생성

[**쿼리**](../building-modules/messages-and-queries.md#queries)는 인터페이스를 통해 애플리케이션의 최종 사용자가 만들고 풀-노드(full-node)에서 처리되는 정보(information)에 대한 요청입니다. 사용자는 애플리케이션 스토어(stores)나 모듈에서 직접 네트워크와 애플리케이션 자체, 애플리케이션의 상태에 대한 정보는 쿼리(query)할 수 있습니다. 쿼리는 [트랜잭션](../core/transactions.md) ([여기](./tx-lifecycle.md)에서 생명주기를 볼 수 있습니다)과 다르다는 것을 참고하십시오. 특히 쿼리는 상태 전환(state-transition)을 트리거하지 않으므로 처리를 위해 합의(consensus)를 요구하지 않고 따라서 하나의 풀-노드만으로 완전히 처리 될 수 있습니다.

쿼리 생명주기를 설명하기 위한 일환으로, `MyQuery`가 `simapp`이라는 애플리케이션에서 특정 위임자(delegator) 주소에서 만들어진 위임의 목록을 요청한다고 가정해 보겠습니다. 예상한대로 [`staking`](../../x/staking/spec/README.md) 모듈이 이 쿼리를 처리할 것입니다. 그 전에 먼저, 사용자가 `MyQuery`를 생성할 수 있는 몇가지 방법을 알아봅시다.

### CLI

어플리케이션의 주요 인터페이스는 커맨드라인 인터페이스입니다. 사용자는 풀-노드에 접속하여 자신의 머신에서 CLI를 직접 실행합니다. 즉, CLI는 풀-노드와 직접 상호 작용합니다. `MyQuery`를 터미널에서 생성하려면 사용자는 다음의 명령어(command)를 입력합니다:

```bash
simd query staking delegations <delegatorAddress>
```

이 쿼리 명령어는 [`staking`](../../x/staking/spec/README.md) 모듈 개발자가 정의했으며 애플리케이션 개발자가 CLI를 생성할 때 하위 명령어(subcommand) 목록으로 추가되었습니다.

일반적인 형식은 다음과 같습니다:

```bash
simd query [moduleName] [command] <arguments> --flag <flagArg>
```

`--node` (CLI가 연결하는 풀-노드)와 같은 값을 제공하기 위하여, 사용자는 [`app.toml`](../run-node/run-node.md#configuring-the-node-using-apptoml) 설정(config) 파일을 사용하여 해당 값을 설정하거나 플래그로 제공할 수 있습니다.

CLI는 애플리케이션 개발자가 정의한 계층(hierarchical) 구조의 다음과 같은 특정 명령어 집합을 이해합니다: [루트(root) 명령어](../core/cli.md#root-command) (`simd`), 명령어의 타입 (`Myquery`), 해당 명령어가 포함된 모듈 (`staking`), 명령어 (`delegations`). 따라서 CLI는 정확히 어느 모듈이 이 명령어를 처리하는지 알고, 그 모듈에 호출을 직접적으로(directly) 전달합니다.

### gRPC

::: warning
`go-grpc v1.34.0`에 도입된 패치로 인해 gRPC가 `gogoproto` 라이브러리와 호환되지 않아 일부 [gRPC 쿼리](https://github.com/cosmos/cosmos-sdk/issues/8426)가 패닉(panic) 상태에 빠집니다. 따라서 SDK는 `go.mod`에서 `go-grpc <=v1.33.2`가 설치하는 것을 요구합니다.

gRPC가 제대로 동작하는 것을 보장하기 위해, 애플리케이션의 `go.mod`에 다음의 줄을 추가하는 것을 **강력히 권장**합니다.

```
replace google.golang.org/grpc => google.golang.org/grpc v1.33.2
```

자세한 내용은 [issue #8392](https://github.com/cosmos/cosmos-sdk/issues/8392)를 참조하십시오.
:::

사용자가 쿼리를 만들 수 있는 Cosmos SDK v0.40에 도입된 또다른 인터페이스는 [gRPC server](../core/grpc_rest.md#grpc-server)에 대한 [gRPC](https://grpc.io) 요청입니다. 엔드포인트(endpoint)는 `.proto` files 파일에 정의된 [Protocol Buffers](https://developers.google.com/protocol-buffers) 서비스 메서드이며, 이는 Protobuf의 자체 언어에 구애받지 않는(language-agnostic) 인터페이스 정의 언어(IDL, Interface Definition Language)로 쓰여졌습니다. Protobuf 생태계(ecosystem)는 `*.proto`파일로부터 다양한 언어로 코드를 생성하는 툴을 개발하였습니다. 이러한 툴을 통해 gRPC 클라이언트를 쉽게 빌드할 수 있습니다.

위 툴 중 하나는 [grpcurl](https://github.com/fullstorydev/grpcurl 이며, 이 클라이언트를 사용하여 `MyQuery`에 대한 gRPC 요청은 다음과 같습니다:

```bash
grpcurl \
    -plaintext                                           # 결과를 일반 텍스트 형식으로 원합니다.
    -import-path ./proto \                               # 해당 .proto 파일을 가져옵니다.
    -proto ./proto/cosmos/staking/v1beta1/query.proto \  # 쿼리 protobuf 서비스를 위해 .proto file를 들여다 봅니다.
    -d '{"address":"$MY_DELEGATOR"}' \                   # 쿼리 매개 변수
    localhost:9090 \                                     # gRPC 서버 엔드포인트
    cosmos.staking.v1beta1.Query/Delegations             # 서비스 메서드의 완전 수식명(fully-qualified name)
```

### REST

[REST server](../core/grpc_rest.md#rest-server)는 사용자가 HTTP 요청을 통해 쿼리를 요청을 할 수 있는 또 다른 인터페이스입니다. REST 서버는 [gRPC-gateway](https://github.com/grpc-ecosystem/grpc-gateway)를 사용하여 Protobuf 서비스로부터 완전히 자동적으로 생성됩니다.

`MyQuery`를 위한 HTTP 요청에 대한 예는 다음과 같습니다:

```bash
GET http://localhost:1317/cosmos/staking/v1beta1/delegators/{delegatorAddr}/delegations
```

## CLI에서 쿼리를 처리하는 방법

위의 예시들은 외부 사용자가 노드 상태를 쿼리하여 노드와 상호작용하는 방법을 보여줍니다. 이 쿼리의 정확한 생명주기를 더 자세히 이해하기 위해, CLI에서 쿼리를 준비하는 방법과 노드가 쿼리를 처리하는 방법을 들여다보겠습니다. 사용자 관점에서의 상호 작용은 조금 다르지만, 내재적인(underlying) 함수들은 모듈 개발자에 의해 동일한 명령을 구현한 것이므로 대부분 동일합니다. 이 처리 단계는 CLI또는 gRPC, 혹은 REST 서버에서 이루어지며 `client.Context`와 밀접하게 연관됩니다.

### Context

`client.Context`는 CLI 명령을 수행할때 가장 먼저 생성됩니다. `client.Context`는 요청을 처리하기 위한 모든 데이터를 사용자 측에서 저장하는 객체입니다. 특히, `client.Context`는 다음을 저장합니다:

- **Codec**: 애플리케이션에서 Tendermint RPC 요청을 만들기 전에 매개 변수(parameter)와 쿼리를 마셜(marshal)하고 반환된 응답(response)를 JSON 객체로 마셜을 해제(unmarshal)해제하기 위한 [인코더/디코더](../core/encoding.md)입니다. CLI에서 사용하는 기본 코덱은 Protobuf입니다.
- **Account decoder**: `[]byte`를 계정으로 변환하는 [`auth`](../..//x/auth/spec/README.md) 모듈의 계정 디코더입니다.
- **RPC Client**: 요청이 중계(relay)될 Tendermint RPC 클라이언트 혹은 노드입니다.
- **Keyring**: 트랜잭션을 서명하고 키(key)와 관련된 다른 동작을 처리하는 [Key Manager](../basics/accounts.md#keyring)입니다.
- **Output Writer**: 응답을 출력하기 위한 [Writer](https://golang.org/pkg/io/#Writer)입니다.
- **Configurations**: 사용자가 해당 명령에 대해 설정한 플래그로, 쿼리할 블록체인의 높이(height)를 특정하는 `--height`와 JSON 응답의 들여쓰기를 추가하는 `--indent`를 포함합니다.

`client.Context`는 또한 RPC 클라이언트를 검색하고 풀-노드에 쿼리를 중계할 ABCI 호출을 수행하는 `Query()`와 같은 다양한 함수를 포함하고 있습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/client/context.go#L20-L50

`client.Context`의 주요 역할은 최종 사용자와의 상호 작용중에 사용되는 데이터를 저장하고 이 데이터와의 상호 작용하는 메서드를 제공하는 것입니다. 이는 풀-노드에서 쿼리가 처리되기 전과 후에 사용됩니다. 특히 `MyQuery`를 처리함에 있어서, 쿼리 매개변수를 인코딩하고, 풀-노드를 검색하고, 결과를 출력할때 `client.Context`가 활용됩니다. 풀-노드로 중계하기 전에 쿼리는 `[]byte` 형태로 인코딩 되어야하는데, 이는 풀-노드는 애플리케이션에 구애받지 않기 때문에(application-agnostic) 특정 타입을 이해할 수 없기 때문입니다. 풀-노드(RPC 클라이언트) 자체는 사용자 CLI가 연결된 노드가 무엇인지 알고 있는 `client.Context`를 사용하여 검색됩니다. 이후 쿼리는 이를 처리할 풀-노드로 중계됩니다. 마지막으로, `client.Context`는 응답이 반환되었을때 결과를 출력할 `Writer`를 포함합니다. 이 과정은 이후 섹션에서 자세히 설명할 예정입니다.

### 인자(Arguments) 및 경로(Route) 생성

생명주기 중 이 시점에서, 사용자는 쿼리에 포함할 모든 데이터와 함께 CLI 명령어를 생성하였습니다. `client.Context`는 `MyQuery`의 나머지 단계를 돕기 위해 존재합니다. 이제 다음 단계에서는 명령어나 요청을 파싱(parse)하고 인자를 추출한 후, 모두 인코딩합니다. 이 단계는 사용자가 상호 작용 중인 인터페이스 내의 사용자 측에서 이루어집니다.

#### 인코딩(Encoding)

현재의 (계정의 위임을 쿼리하는) 경우, `MyQuery`는 [주소(address)](./accounts.md#addresses)인 `delegatorAddress`를 유일한 인자로 포함합니다. 그러나, 요청은 애플리케이션 타입에 대한 내재된 지식이 없는 풀-노드의 합의 엔진(e.g. Tendermint Core)에 중계되기 때문에 `[]byte`만을 포함할 수 있습니다. 따라서 `client.Context`의 `codec`이 주소를 마셜하는데 사용됩니다.

CLI 명령의 코드는 다음과 같습니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/x/staking/client/cli/query.go#L324-L327

#### gRPC 쿼리 클라이언트 생성

SDK는 Protobuf 서비스에서 생성된 코드를 활용하여 쿼리를 수행합니다. `staking` 모듈의 `MyQuery` 서비스는 CLI에서 쿼리를 만드는데 사용할 `queryClient`를 생성합니다. 관련 코드는 다음과 같습니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/x/staking/client/cli/query.go#L318-L342

내부적으로, `client.Context`는 사전에 설정된 노드를 검색하고 쿼리를 그 곳에 중계하는 `Query()`라는 함수를 갖고 있습니다. 이 함수는 쿼리의 서비스 메서드 완전 수식명(우리의 경우: `/cosmos.staking.v1beta1.Query/Delegations`)을 경로로 사용하고, 인자를 매개 변수로 갖습니다. `client.Context`는 먼저 사용자가 이 쿼리를 중계하도록 설정한 ([**node**](../core/node.md)라는 이름의) RPC 클라이언트를 검색하고, (ABCI 호출을 위한 양식으로 변경된 매개 변수) `ABCIQueryOptions`를 생성합니다. 그 다음, 노드를 사용하여 `ABCIQueryWithOptions()`라는 ABCI 호출을 생성합니다.

코드는 다음과 같습니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/client/query.go#L65-L91

## RPC

`ABCIQueryWithOptions()` 호출을 통해, `MyQuery`가 [풀-노드](../core/encoding.md)로 부터 수신하면 요청이 처리됩니다. 참고로 RPC는 풀-노드의 합의 엔진(e.g. Tendermint Core)으로 만들어지지만, 쿼리는 합의의 일부가 아니며 네트워크의 나머지 부분으로 브로드캐스트되지 않습니다. 이는 네트워크가 동의해야 할 어떤 것도 필요하지 않기 때문입니다.

ABCI 클라이언트 및 Tendermint RPC에 대한 자세한 내용은 [여기](https://tendermint.com/rpc)의 Tendermint 문서를 확인하십시오.

## 애플리케이션 쿼리 처리

쿼리가 기반이 된 합의 엔진으로부터 중계된 후 풀-노드에서 수신되면, 애플리케이션 전용의 타입을 이해하고  및 상태(state)의 복사본을 가진 환경에서 처리됩니다. [`baseapp`](../core/baseapp.md)은 ABCI [`Query()`](../core/baseapp.md#query) 함수를 구현하고 gRPC 쿼리를 처리합니다. 쿼리 경로는 파싱되어 (일반적으로 모듈 중 한곳에서) 존재하는 서비스 메서드의 완전 수식명과 일치하면, `baseapp`은 요청을 관련 모듈로 중계합니다.

gRPC 경로 외에도, `baseapp`은 `app`과, `store`, `p2p`, `custom`의 4가지 다른 타입의 쿼리 또한 처리합니다. 첫 3개의 타입(`app`, `store`, `p2p`)은 순수하게 애플리케이션 수준이기 때문에 `baseapp` 또는 스토어에서 처리되지만, `custom` 쿼리 타입은 쿼리를 `baseapp`이 모듈의 [legacy queriers](../building-modules/query-services.md#legacy-queriers)에 라우팅하도록 요구합니다. 이러한 쿼리들에 대한 자세한 내용은 [이 가이드](../core/grpc_rest.md#tendermint-rpc)를 참조하십시오.

`MyQuery`는 `staking` 모듈의 Protobuf 서비스 메서드의 완전 수식명(`/cosmos.staking.v1beta1.Query/Delegations`를 상기하십시오)을 갖고 있기 때문에 `baseapp`은 먼저 경로를 파싱한다음, 내부의 `GRPCQueryRouter`를 사용하여 대응하는 gRPC 처리기(handler)를 검색하고, 쿼리를 모듈에 라우팅합니다. gRPC 처리기는 이 쿼리를 인식하고 애플리케이션의 스토어로부터 적절한 값을 검색하여 응답을 반환합니다. 쿼리 서비스에 대한 자세한 내용은 [여기](../building-modules/query-services.md)를 참조하십시오.

일단 질의자 `querier`로부터 결과가 수신되면, `baseapp`은 응답을 사용자에게 반환하는 절차를 시작합니다.

## 응답(Response)

`Query()`는 ABCI 함수이므로 `baseapp`은 응답을 [`abci.ResponseQuery`](https://tendermint.com/docs/spec/abci/abci.html#messages)타입으로 반환합니다. `client.Context`의 `Query()` 루틴은 응답을 수신합니다.

### CLI 응답(Response)

애플리케이션 [`코덱`](../core/encoding.md)을 사용하여 응답을 JSON으로 마샬을 해제하고, `client.Context`는 결과를 출력 타입(text나 JSON, YAML)과 같은 모든 설정(configuration)을 적용하여 커맨드라인에 출력합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/client/context.go#L248-L283

마지막으로 쿼리의 결과는 CLI에 의해 콘솔에 출력됩니다.

## 다음 {hide}

[계정](./accounts.md)에 대하여 알아봅시다. {hide}
