<!--
order: 1
-->

# High-level Overview

## SDK 가 무엇인가요?

[Cosmos SDK](https://github.com/cosmos/cosmos-sdk) 는 Cosmos Hub와 같이 다중자산(Multi-asset) 지분증명(Proof-of-Stake, PoS) 공개 
블록체인이나, 허가된 권한증명(Proof-of-Authority, PoA) 블록체인을 만들기 위한 오픈소스 프레임워크입니다.

Cosmos SDK 의 목적은 다른 블록체인과의 상호작용이 기본적으로 가능한 커스텀 블록체인을 개발자들이 쉽게 직접 만들 수 있도록 하는 것입니다. 우리는 
Cosmos SDK 가 [Tendermint](https://github.com/tendermint/tendermint) 기반에서 안정적인 블록체인 애플리케이션을 구축하기 위한 프레임워크, 
npm 같은 프레임워크가 되기를 기대합니다. Cosmos SDK 기반의 블록체인들은 컴포저블 [모듈](../building-modules/intro.md)(Composable modules)로 
만들어지며, 이들 대부분은 오픈소스이고 모든 개발자들이 쉽게 사용할 수 있습니다. 누구나 Cosmos SDK 용 모듈을 만들 수 있고, 만들어진 모듈들을 통합하는건 
당신의 블록체인 애플리케이션으로 import만 하면 됩니다. 뿐만아니라 Cosmos SDK 는 자격 기반 시스템(Capabilities-based system) 으로써 개발자들이 
모듈간 상호작용의 보안성에 대해 더 나은 판단을 할 수 있게 합니다. 자격에 대한 더 심도있는 내용을 보려면 [이 섹션](../core/ocap.md) 을 참고하세요.

## 애플리케이션별 전용 블록체인(Application-Specific Blockchains) 이란?

오늘날 블록체인 세계에서의 한 가지 개발 패러다임은 Ethereum 과 같은 가상 머신 블록체인으로써, 일반적으로는 기존 블록체인 위에 분산형 애플리케이션을 스마트 
컨트랙트로 구축하는 것이 핵심입니다. 스마트 컨트랙트는 일회용 애플리케이션(e.g. ICOs) 같은 어떤 유스케이스에는 매우 좋을 수 있지만, 복잡한 분산형 플랫폼을 
만들 때에는 부족한 경우가 많습니다. 일반적으로 스마트 컨트랙트는 유연성, 자주성과 성능 측면에서 제한적일 수 있습니다.

애플리케이션별 전용 블록체인은 가상 머신 블록체인과는 근본적으로 다른 개발 패러다임을 제공합니다. 애플리케이션별 전용 블록체인은 하나의 애플리케이션 운용을 위해 
맞춤화된 블록체인입니다: 개발자는 애플리케이션을 최적으로 실행하는데에 필요한 설계 결정을 자유롭게 내릴 수 있습니다. 또한 더 나은 자주성, 보안, 성능을 제공할 수 
있습니다.

[애플리케이션별 전용 블록체인](./why-app-specific.md) 에 대해 자세히 알아보기.

## 왜 Cosmos SDK인가?

Cosmos SDK 는 맞춤형 애플리케이션별 전용 블록체인 구축을 위한 오늘날 가장 진보한 프레임워크입니다. 다음은 왜 Cosmos SDK 로 분산형 어플리케이션을 
만들어야하는지에 대한 몇가지 이유입니다:

- SDK 에서 사용할 수 있는 기본 합의 엔진은 [Tendermint Core](https://github.com/tendermint/tendermint) 입니다. Tendermint 는 현존하는 
  BFT 합의 엔진 중 가장(그리고 유일하게) 성숙한 엔진입니다. 업계 전반에서 널리 사용되며 지분증명 시스템 구축에 최적 표준의 합의 엔진으로 여겨집니다.
- SDK 는 오픈소스이며 컴포저블 [모듈](../../x/) 로 쉽게 블록체인을 구축할 수 있도록 설계되었습니다. 오픈소스 모듈 생태계가 성장함에 따라, 이를 통해 복잡한 
  분산형 플랫폼을 구축하는것이 점점 더 쉬워질 것입니다.
- SDK 는 자격기반 보안에서 영감을 받았으며, 수년간 블록체인 상태 기계와 씨름하며 얻은 정보입니다. 따라서 Cosmos SDK 는 블록체인을 구축하기에 매우 
  안정적인 환경입니다.
- 가장 중요한 것은, Cosmos SDK 는 이미 개발 중인 많은 애플리케이션별 전용 블록체인들을 구축하는데에 사용되고 있습니다. 
  [Cosmos Hub](https://hub.cosmos.network), [IRIS Hub](https://irisnet.org), [Binance Chain](https://docs.binance.org/), 
  [Terra](https://terra.money/) or [Kava](https://www.kava.io/) 등을 예로 들 수 있습니다. 이외에도 
  [더 많은 곳](https://cosmos.network/ecosystem) 에서 Cosmos SDK 를 기반으로 구축되고 있습니다.

## Cosmos SDK 시작하기

- [SDK 애플리케이션 아키텍쳐](sdk-app-architecture.md) 자세히 알아보기
- [SDK 튜토리얼](https://cosmos.network/docs/tutorial) 을 통해 직접 애플리케이션별 전용 블록체인 구축하는 법 알아보기

## Next {hide}

[애플리케이션별 전용 블록체인](./why-app-specific.md) 알아보기 {hide}
