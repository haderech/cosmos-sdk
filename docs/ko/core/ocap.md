<!--
order: 11
-->

# 오브젝트 자격 모델 (Object-Capability Model)

## 소개

보안을 고려할 때는 구체적인 위협 모델로부터 시작하는게 좋습니다. 우리의 위협 모델은 다음과 같습니다:

> 블록체인 애플리케이션을 쉽게 구성할 수 있는 활발한 Cosmos SDK 모듈 생태계는 결함있거나 악의적인 모듈이 포함될 수 있습니다. 

Cosmos SDK 는 오브젝트 자격 모델의 기반이 되어 이런 위협에 대처하도록 설계되었습니다.

> 오브젝트 자격 시스템의 구조적 특성은 코드 설계에서는 모듈화를 선호하게 하고, 코드 구현에서는 신뢰할 수 있는 인캡슐레이션(encapsulation) 을 보장합니다.
> 
> 이러한 구조적 특성은 오브젝트 자격 프로그램 또는 시스템 운영의 일부 보안 특성을 분석하는데 도움이 됩니다. 이 중 일부 (특히, 정보 흐름 특성) 는 오브젝트 
> 동작을 결정하는 코드에 대한 지식이나 분석이 필요없이 오브젝트 참조 및 연결 수준에서 분석이 가능합니다. 
> 
> 따라서 이러한 보안 특성은 알려져있지 않고 악성일 수도 있는 코드가 포함된 새로운 오브젝트가 있는 경우에도 확립되고 유지보수할 수 있습니다.
>
> 이 구조적 특성들은 기존 오브젝트에 대한 접근을 제어하는 두 가지 규칙에서 비롯됩니다:
>
> 1. 오브젝트 A 는 B 에 대한 참조를 가지고 있을 때에만 B 로 메시지를 보낼 수 있습니다.
>
> 2. 오브젝트 A 는 C 에 대한 참조를 포함하는 메시지를 수신한 경우에만 C 에 대한 참조를 얻을 수 있습니다. 이 두 가지 규칙에 따라서, 오브젝트는 기존 참조 
     체인을 통해서만 다른 오브젝트의 참조를 얻을 수 있습니다. 간단히 말해서, "오직 연결만이 연결을 낳는다"

오브젝트 자격에 대한 소개는 [Wikipedia article](https://en.wikipedia.org/wiki/Object-capability_model) 을 참조하세요.

## 실전 Ocaps

작업에 필요한 것들만 알아봅시다.

예를 들어, 다음 코드 스니펫은 오브젝트 자격 원칙을 위반했습니다:

```go
type AppAccount struct {...}
account := &AppAccount{
    Address: pub.Address(),
    Coins: sdk.Coins{sdk.NewInt64Coin("ATM", 100)},
}
sumValue := externalModule.ComputeSumValue(account)
```

`ComputSumValue` 는 순수 함수이지만, 포인터를 받으면 그 값을 수정할 수도 있음을 의미합니다. 
이것 대신에 선호되는 메서드 시그니처 방식은 값을 복사하는 것입니다. 

```go
sumValue := externalModule.ComputeSumValue(*account)
```

Cosmos SDK 에서는 gaia 앱에서 이 원칙을 적용한 것을 볼 수 있습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.41.4/simapp/app.go#L249-L273

다음의 다이어그램은 현재 keeper 들 간의 의존관계를 보여 줍니다.

![Keeper 의존관계](../../uml/svg/keeper_dependencies.svg)

## Next {hide}

[`runTx` 미들웨어](./runtx_middleware.md) 에 대해서 알아보세요 {hide}
