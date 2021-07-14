<!--
order: 6
-->

# 인코딩(Encoding)

SDK 내에서 인코딩은 주로 `go-amino` 코덱에 의해 처리되었지만, SDK는 상태 인코딩 및 클라이언트 측 인코딩 모두 `gogoprotobuf`를 사용하는 방향으로 가고 있습니다. {synopsis}

## 사전 요구 지식

- [애플리케이션의 해부](../basics/app-anatomy.md) {prereq}

## 인코딩

Cosmos SDK는 객체 인코딩 규격(specification)인 [Amino](https://github.com/tendermint/go-amino/)와 Proto3의 서브셋에 인터페이스 지원을 위한 확장을 추가한 [프로토콜 버퍼(Protocol Buffers)](https://developers.google.com/protocol-buffers)의 두 바이너리 와이어(wire) 인코딩 프로토콜을 활용합니다. Amino가 (Proto2와는 달리) 대부분 호환하는 Proto3에 대한 자세한 내용은 다음의 [Proto3 규격(spec)](https://developers.google.com/protocol-buffers/docs/proto3)을 참조하십시오.

반영 기반(reflection-based)이면서 의미있는 다국어(cross-language)/클라이언트 지원이 없어 거대한 성능적 결점이 있는 Amino를 대신하여, 프로토콜 버퍼, 특히 [gogoprotobuf](https://github.com/gogo/protobuf/)가 사용됩니다. Amino를 프로토콜 버퍼로 대체하는 작업은 아직 진행중이라는 점을 참고하십시오.

Cosmos SDK에서 타입의 바이너리 와이어 인코딩은 크게 클라이언트 인코딩과 스토어 인코딩, 두 가지로 구분할 수 있습니다. 클라이언트 인코딩은 주로 트랜잭션 처리와 서명에 주로 중점을 두고 있고, 스토어 인코딩은 상태 기계(state-machine) 전환에 사용되는 타입과 머클(Merkle) 트리에 최종적으로 저장되는 타입을 중심으로 이루어집니다.

스토어 인코딩의 경우, protobuf 정의는 모든 타입에 대해 존재할 수 있으며 일반적으로 Amino 기반의 "중간(intermediary)" 타입을 가집니다. 특히 protobuf 기반 타입 정의는 직렬화(serialization)와 지속성(persistence)을 위해 사용되는 반면, Amino 기반 타입은 자유롭게 변환될 수 있는 상태 기계 내에서 비즈니스 로직을 위해 사용됩니다. Amino 기반 타입은 향후에 점차 폐기될 수 있으므로 개발자들은 가능한 protobuf 메시지 정의를 사용하도록 주의해야 합니다.

`codec` 패키지에는 `Marshaler`와 `ProtoMarshaler`의 두 가지 핵심 인터페이스가 있습니다. `Marshaler`는 Amino 인터페이스를 캡슐화하지만, 일반화된(generic) `interface{}`가 아닌 `ProtoMarshaler`를 구현한 타입에 대해 동작하는 경우는 제외합니다.

또한 `Marshaler`의 구현은 두 가지가 존재합니다. 첫 번째는 `AminoCodec`으로 Amino를 사용해 바이너리와 JSON의 직렬화를 처리합니다. 두 번째는 `ProtoCodec`으로 Protobuf를 사용해 바이너리와 JSON의 직렬화를 처리합니다.

즉, 모듈은 Amino 또는 Protobuf 인코딩을 사용할 수 있지만 타입은 `ProtoMarshaler`를 반드시 구현해야 합니다. 만약 모듈이 타입을 위해 이 인터페이스를 구현하지 않으려는 경우 Amino 코덱을 직접 사용할 수 있습니다.

### Amino

모든 모듈은 타입과 인터페이스를 직렬화하기 위해 Amino 코덱을 사용합니다. 이 코덱은 일반적으로 해당 모듈의 도메인에서만 등록된 타입과 인터페이스(e.g. 메시지)를 갖지만, `x/gov`와 같은 예외가 있습니다. 각 모듈은 사용자가 코덱을 제공하고 모든 타입을 등록할 수 있는 `RegisterLegacyAminoCodec` 함수를 제공합니다. 애플리케이션은 필요한 각 모듈에 대해 이 함수를 호출합니다.

아래와 같이 모듈에 대한 protobuf 기반 타입 정의가 없는 경우, 미가공의(raw) 와이어 bytes을 구체적인(concrete) 타입 혹은 인터페이스로 인코딩과 디코딩하기 위해서 Amino가 사용됩니다.

```go
bz := keeper.cdc.MustMarshal(typeOrInterface)
keeper.cdc.MustUnmarshal(bz, &typeOrInterface)
```

참고로, 상기 기능에 길이 접두어(length-prefixed)를 사용한 변형이 있으며 이는 일반적으로 데이터가 스트림되거나 함께 묶이는 경우 사용됩니다(e.g. `ResponseDeliverTx.Data`).

### Gogoproto

모듈은 각각의 타입에 대해 Protobuf 인코딩을 활용하는 것이 권장됩니다. SDK에서는 Protobuf 규격(spec)의 전용 구현인 [Gogoproto](https://github.com/gogo/protobuf)를 사용하여, 공식 [Google protobuf 구현](https://github.com/protocolbuffers/protobuf)과 비교하여 속도와 DX를 향상시켰습니다.

### Protobuf 메시지 정의 가이드라인

[다음의 공식 프로토콜 버퍼 가이드라인](https://developers.google.com/protocol-buffers/docs/proto3#simple)에 더하여, 인터페이스를 다룰 때는 `.proto` 파일에 다음의 어노테이션을 사용하는 것을 권장합니다.

- 인터페이스를 수용하는 필드의 경우, `cosmos_proto.accepts_interface`를 사용하십시오
- `InterfaceRegistry.RegisterInterface`에 `protoName`과 동일한 완전 수식명(fully-qualified name)을 전달하십시오
- 인터페이스의 구현의 경우, `cosmos_proto.implements_interface`를 사용하십시오
- `InterfaceRegistry.RegisterInterface`에 `protoName`과 동일한 완전 수식명(fully-qualified name)을 전달하십시오

### 트랜잭션 인코딩

Protobuf의 또 다른 중요한 용도는 [트랜잭션](./transactions.md)의 인코딩과 디코딩입니다. 트랜잭션은 애플리케이션 또는 SDK에 의해 정의되지만, 하단의 합의 엔진에 전달되어 다른 피어에 중계됩니다. 합의 엔진은 애플리케이션에 구애받지 않기 때문에 오직 미가공의(raw) bytes 타입의 트랜잭션만 수신합니다.

- `TxEncoder` 객체는 인코딩을 수행합니다.
- `TxDecoder` 객체는 디코딩을 수행합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc4/types/tx_msg.go#L83-L87

위의 두 객체의 표준 구현은 다음의 [`auth` 모듈](../../x/auth/spec/README.md)에서 확인할 수 있습니다.:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc4/x/auth/tx/decoder.go

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc4/x/auth/tx/encoder.go

트랜잭션이 인코딩되는 과정에 대한 자세한 내용은 [ADR-020](../architecture/adr-020-protobuf-transaction-encoding.md)를 참조하십시오.

### 인터페이스의 인코딩과 `Any`의 용도

Protobuf DSL은 엄격하게 타입을 체크하기 때문에, 가변 타입의(variable-typed) 필드를 추가하는 것을 어려울 수 있습니다. 다음과 같이 [계정(Account)](../basics/accounts.md)에 대한 래퍼 역할을 수행하는 `Profile` protobuf 메시지를 생성하는 경우를 상상해보십시오:

```proto
message Profile {
  // account는 Profile에 관련된 계정입니다.
  cosmos.auth.v1beta1.BaseAccount account = 1;
  // bio는 account의 짧은 설명입니다.
  string bio = 4;
}
```

위의 `Profile` 예시에서, `account`를 `BaseAccount`로 하드코딩하였습니다. 그러나 `BaseVestingAccount` 또는 `ContinuousVestingAccount`와 같이 몇몇 다른 타입의 [권한 부여(vesting)와 관련한 사용자 계정](../../x/auth/spec/05_vesting.md)이 있습니다. 이 계정들은 서로 다르지만, 모두 `AccountI` 인터페이스를 구현합니다. `AccountI`를 사용하는 `account` 필드가 있고, 이 필드를 가진 계정들의 모든 타입을 수용하는 `Profile`을 어떻게 만들겠습니까?

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/x/auth/types/account.go#L307-L330

[ADR-019](../architecture/adr-019-protobuf-state-encoding.md)에서는 protobuf의 인터페이스를 인코딩하기 위해 [`Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto)를 사용하기로 결정했습니다. `Any`는 임의의 bytes로 직렬화된 메시지와 해당 메시지에 대하여 글로벌하게 유일한 식별자(identifier) 역할을 하는 URL을 포함합니다. 이 전략을 통해 protobuf 메시지 안에 임의의 Go 타입을 넣을(pack) 수 있습니다. 새로운 `Profile`은 다음과 같습니다:

```protobuf
message Profile {
  // account는 Profile에 관련된 계정입니다.
  google.protobuf.Any account = 1 [
    (cosmos_proto.accepts_interface) = "AccountI"; // 이 필드는 `AccountI`를 구현하는 Go 타입만을 수용한다고 주장(Assert)합니다. 현재까지는 단순 정보 제공에 불과합니다.
  ];
  // bio는 account의 짧은 설명입니다.
  string bio = 4;
}
```

profile에 계정을 추가하기 위해서, 다음과 같이 `codectypes.NewAnyWithValue`을 사용하여 `Any`에 먼저 계정을 "pack" 해야 합니다:

```go
var myAccount AccountI
myAccount = ... // BaseAccount나, a ContinuousVestingAccount, 또는 `AccountI`를 구현하는 아무 struct가 될 수 있습니다.

// Any에 account를 pack 합니다.
accAny, err := codectypes.NewAnyWithValue(myAccount)
if err != nil {
  return nil, err
}

// any와 함께 새로운 Profile을 생성합니다.
profile := Profile {
  Account: accAny,
  Bio: "some bio",
}

// 이후 평소와 같이 profile을 마셜합니다.
bz, err := cdc.Marshal(profile)
jsonBz, err := cdc.MarshalJSON(profile)
```

요약하면, 인터페이스를 인코딩하기 위해서 1/ `Any`에 인터페이스를 pack하고 2/ `Any`를 마셜(marshal)합니다. SDK는 편의상 두 과정을 합친 `MarshalInterface`를 제공합니다. [x/auth 모듈에서의 실제 예](https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/x/auth/keeper/keeper.go#L218-L221)를 참조하십시오.

반대로, `Any`의 내부로부터 구체적(concrete) Go 타입을 회수(retrieve)하는 "unpack"을 하기 위해서는 `Any`의 `GetCachedValue()`를 사용하면 됩니다.

```go
profileBz := ... // Profile의 proto로 인코딩된 (e.g. gRPC를 통해 회수된) bytes.
var myProfile Profile
// myProfile struct로 bytes의 마셜을 해제(unmarshal)한다.
err := cdc.Unmarshal(profilebz, &myProfile)

// Account 필드의 타입을 확인한다.
fmt.Printf("%T\n", myProfile.Account)                  // "Any"를 출력한다.
fmt.Printf("%T\n", myProfile.Account.GetCachedValue()) // "BaseAccount"나, "ContinuousVestingAccount", 또는 처음 Any에 pack된 무언가를 출력한다.

// account의 주소를 가져온다.
accAddr := myProfile.Account.GetCachedValue().(AccountI).GetAddress()
```

`GetCachedValue()`가 작동하려면 `Profile`(및 `Profile`을 내재한 struct)은 다음과 같이 `UnpackInterfaces` 메서드를 구현해야 하는 것을 주의하십시오:

```go
func (p *Profile) UnpackInterfaces(unpacker codectypes.AnyUnpacker) error {
  if p.Account != nil {
    var account AccountI
    return unpacker.UnpackAny(p.Account, &account)
  }

  return nil
}
```

`Any`의 `GetCachedValue()`가 올바르게 전파(populate)될 수 있도록 `UnpackInterfaces`은 이 메서드를 구현한 모든 struct로부터 재귀적으로 호출됩니다.

인터페이스 인코딩에 대한 자세한 정보(특히 `UnpackInterfaces`와 `InterfaceRegistry`를 사용하여 `Any`의 `type_url`이 확정(resolved)되는 방법)는 [ADR-019](../architecture/adr-019-protobuf-state-encoding.md)를 참조하십시오.

#### SDK의 `Any` 인코딩

위의 `Profile`의 예시는 교육적 목적으로 사용된 가상의 예시입니다. SDK에서는 다음과 같이 여러 위치에서 `Any` 인코딩을 사용합니다 (전체 목록 아님):

- `cryptotypes.PubKey` 인터페이스는 다양한 타입의 공개키들을 인코딩합니다.
- `sdk.Msg` 인터페이스는 트랜잭션 안의 다양한 `Msg`들을 인코딩합니다.
- `AccountI` 인터페이스는 (위의 예와 같이) x/auth 쿼리 응답 안의 다양한 타입의 계정들을 인코딩합니다.
- `Evidencei` 인터페이스는 x/evidence 모듈 내의 다양한 타입의 증거들을 인코딩합니다.
- `AuthorizationI` 인터페이스는 다양한 타입의 x/authz 권한(authorization)을 인코딩합니다.
- [`Validator`](https://github.com/cosmos/cosmos-sdk/blob/v0.42.5/x/staking/types/staking.pb.go#L306-L337) struct는 검증자(validator)의 정보를 포함합니다.

x/staking에서 공개키를 Validator struct 내부의 `Any` 로 인코딩하는 실제 예는 다음을 참조하십시오:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/x/staking/types/validator.go#L40-L61

## FAQ

1. protobuf 인코딩을 사용해 모듈을 어떻게 만드나요?

**모듈 타입 정의하기**

다음의 Protobuf 타입은 인코딩하도록 정의될 수 있습니다:

- `state`
- [`Msg`s](../building-modules/messages-and-queries.md#messages)
- [Query services](../building-modules/query-services.md)
- [genesis](../building-modules/genesis.md)

**명명 및 규칙(conventions)**

다음의 업계 가이드라인을 개발자에게 권장합니다: [Protocol Buffers style guide](https://developers.google.com/protocol-buffers/docs/style)와 [Buf](https://buf.build/docs/style-guide), 더욱 자세한 정보는 [ADR 023](../architecture/adr-023-protobuf-naming.md)를 참조하십시오.

2. 어떻게 모듈을 Protobuf 인코딩을 사용하도록 업데이트하나요?

모듈이 아무 인터페이스(e.g. `Account`나 `Content`)를 포함하지 않는 경우, 구체적인 Amino 코덱을 통해 인코딩되고 유지되는 기존의 타입을 Protobuf로 단순히 마이그레이션하고(추가 가이드라인은 1.을 참조), 추가 사용자 정의 없이 `ProtoCodec`을 통해 구현되는 코덱으로 `Marshaler`를 사용하면 됩니다.

만약 모듈 타입이 인터페이스를 구성하는 경우, 반드시 인터페이스를 `/types` 패키지의 `skd.Any`로 감싸야(wrap)합니다. 이 때, 각각의 메시지 타입과 인터페이스 타입을 위한 모듈 단위의 .proto 파일은 반드시 [`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto)를 사용해야 합니다.

예를 들어  모듈은 `MsgSubmitEvidence`가 사용하는 `Evidence` 인터페이스는 `x/evidence` 안에 정의됩니다. 이 때 `sdk.Any`를 사용하여 evidence 파일을 래핑하여 struct를 정의합니다. proto 파일에 다음과 같이 정의합니다:

```protobuf
// proto/cosmos/evidence/v1beta1/tx.proto

message MsgSubmitEvidence {
  string              submitter = 1;
  google.protobuf.Any evidence  = 2 [(cosmos_proto.accepts_interface) = "Evidence"];
}
```

SDK의 `codec.Codec` 인터페이스는 상태(state)를 쉽게 `Any`에 인코딩하기 위하여 `MarshalInterface`와 `UnmarshalInterface` 메서드를 지원합니다.

모듈은 인터페이스를 등록하는 메커니즘을 제공하는 다음의 `InterfaceRegistry`를 사용해야 합니다: `RegisterInterface(protoName string, iface interface{})`. 구현은 다음과 같습니다: `RegisterImplementations(iface interface{}, impls ...proto.Message)`. Amino를 사용해 타입을 등록하는 것과 유사하게 Any로 부터 안전하게 언팩할 수 있습니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc4/codec/types/interface_registry.go#L25-L66

또한 인터페이스를 언팩하기 전 `UnpackInterfaces` 단계를 미리 도입하여 직렬화를 해제(deserialization)해야합니다. 직접 또는 멤버를 통해서 protobuf `Any`를 포함하는 Protobuf 타입은 `UnpackInterfacesMessage`를 구현해야 합니다:

```go
type UnpackInterfacesMessage interface {
  UnpackInterfaces(InterfaceUnpacker) error
}
```

## 다음 {hide}

[gRPC와, REST, Tendermint 엔드포인트(endpoints)](./grpc_rest.md)에 대해 알아봅시다 {hide}
