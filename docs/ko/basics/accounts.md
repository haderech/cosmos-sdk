<!--
order: 4
-->

# 계정

이 문서는 Cosmos SDK 에 내장된 계정과 공개키 시스템에 대해서 설명합니다. {synopsis}

### 사전 요구 학습

- [SDK 애플리케이션 해부](./app-anatomy.md) {prereq}

## 계정 정의

Cosmos SDK 에서는, 계정이 _공개키_ `PubKey` 와 _개인키_ `PrivKey` 를 지정합니다. `PubKey` 를 파생해서 다양한 `Addresses` 를 생성할 수 있으며, 이를 통해 애플리케이션에서 사용자를 식별할 수 있습니다. 또한 `Addresses` 는 `message` 의 발신자를 식별하기 위한 [`message`들](../building-modules/messages-and-queries.md#messages) 과 연결됩니다. `PrivKey` 는 [디지털 서명](#signatures) 을 생성하여 `PrivKey` 와 연결된 주소가 특정 메시지를 승인했음을 증명하는데에 사용됩니다.

HD키 파생을 위해 Cosmos SDK 는 [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki) 라는 표준을 사용합니다. BIP32 는 사용자들에게 최초 비밀 시드에서 파생된 계정 세트인 HD 지갑 ([BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) 에 명시) 을 생성하게 해줍니다. 시드는 보통 12 ~ 24 단어의 니모닉으로부터 생성됩니다. 하나의 시드는 단방향 암호화 기능을 사용하여 원하는 수의 `PrivKey` 를 파생할 수 있습니다. 그런 다음 `PrivKey` 에서 `PubKey` 를 파생할 수 있습니다. 개인키는 저장된 니모닉으로부터 얼마든지 다시 생성할 수 있으므로 당연히 니모닉이 가장 민감한 정보가 됩니다.

```
        계정 0                            계정 1                              계정 2

+------------------+              +------------------+               +------------------+
|                  |              |                  |               |                  |
|   ㅤㅤ 주소 0  ㅤ   |              |       주소 1      |               |       주소 2ㅤㅤ  ㅤ|
|        ^         |              |        ^         |               |        ^         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        +         |              |        +         |               |        +         |
|ㅤㅤㅤ  공개키 0 ㅤ   |              |ㅤㅤㅤ  공개키 1ㅤㅤㅤㅤ|               |ㅤㅤㅤ  공개키 2 ㅤ   |
|        ^         |              |        ^         |               |        ^         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        +         |              |        +         |               |        +         |
| ㅤㅤㅤ 개인키 0   ㅤ |    ㅤㅤㅤㅤㅤ   | ㅤㅤㅤ 개인키 1    ㅤ|     ㅤㅤ      ㅤ|  ㅤㅤㅤ개인키 2    ㅤ|
|        ^         |              |        ^         |               |        ^         |
+------------------+              +------------------+               +------------------+
         |                                 |                                  |
         |                                 |                                  |
         |                                 |                                  |
         +--------------------------------------------------------------------+
                                           |
                                           |
                                 +---------+---------+
                                 |            ㅤㅤㅤ   |
                                 |   마스터 PrivKeyㅤㅤ |
                                 |            ㅤㅤㅤ   |
                                 +-------------------+
                                           |
                                           |
                                 +---------+---------+
                                 |            ㅤㅤㅤ   |
                                 |ㅤㅤㅤ 니모닉 (시드)ㅤㅤ |
                                 |            ㅤㅤㅤ   |
                                 +-------------------+
```

Cosmos SDK 에서는 키는 [`Keyring`](##Keyring) 이라는 객체에 의해 저장되고 관리됩니다.

## 키, 계정, 주소 그리고 서명

사용자 인증의 주요 방법은 [디지털 서명](https://en.wikipedia.org/wiki/Digital_signature) 을 사용하는 것입니다. 사용자는 그들 소유의 개인키를 사용해서 트랜잭션에 서명합니다. 서명 확인은 연결된 공개키를 통해 수행됩니다. 온체인(on-chain) 서명 확인 목적을 위해서 공개키를 `Account` 객체에 저장합니다. (올바른 트랜잭션 유효성 검사를 위해 필요한 다른 데이터와 함께)    

노드에서 모든 데이터는 프로토콜 버퍼 직렬화를 사용해서 저장됩니다.

Cosmos SDK 는 디지털 서명 생성을 위해 다음과 같은 키 체계를 지원합니다.

- `secp256k1`, [SDK 의 `crypto/keys/secp256k1` 패키지](https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/crypto/keys/secp256k1/secp256k1.go) 에 구현.
- `secp256r1`, [SDK 의 `crypto/keys/secp256r1` 패키지](https://github.com/cosmos/cosmos-sdk/blob/master/crypto/keys/secp256r1/pubkey.go) 에 구현.
- `tm-ed25519`, [SDK 의 `crypto/keys/ed25519` 패키지](https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/crypto/keys/ed25519/ed25519.go) 에 구현. 이 체계는 오직 합의만 지원합니다.

|              |   주소 길이   | 공개키 길이 | 트랜잭션 서명에 사용 | 합의에 사용 (tendermint) |
|--------------|------------|----------|-----------------|-----------------------|
| `secp256k1`  | 20         | 33       | 네               | 아니오                 |
| `secp256r1`  | 32         | 33       | 네               | 아니오                 |
| `tm-ed25519` | -- 미사용 -- | 32       | 아니오            | 네                    |

## 주소

`Addresses` 와 `PubKey` 는 둘 다 애플리케이션에서 행위자를 식별하는 공개 정보입니다. `Account` 는 인증 정보를 저장하는데에 사용됩니다. 기본 계정의 구현은 `BaseAccount` 객체를 통해 제공됩니다.

각 계정은 공개키에서 파생된 바이트 시퀀스인 `Address` 를 사용해 식별됩니다. Cosmos SDK 에서는 3가지의 주소 타입을 정의하는데, 각 주소 타입은 계정이 사용되는 컨텍스트를 지정합니다.

- `AccAddress` 는 사용자 (`message` 의 발신자) 를 식별합니다. 
- `ValAddress` 는 validator 오퍼레이터들을 식별합니다.
- `ConsAddress` 는 합의에 참여하는 validator 노드들을 식별합니다. Validator 노드들은 `ed25519` 커브를 사용하여 파생됩니다.

이 타입들은 `Address` 인터페이스를 구현합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/types/address.go#L71-L90

주소 생성 알고리즘은 [ADR-28](https://github.com/cosmos/cosmos-sdk/blob/master/docs/architecture/adr-028-public-key-addresses.md) 에 정의되어 있습니다.
다음은 `pub` 공개키로부터 계정 주소를 얻는 표준 방법입니다.

```go
sdk.AccAddress(pub.Address().Bytes())
```

참고로 `Marshal()` 및 `Bytes()` 메서드는 둘 다 동일한 로우(raw) `[]byte` 형식 주소를 반환합니다. 프로토콜 버퍼 호환성을 위해서는 `Marshal()` 을 사용해야합니다.

주소는 [Bech32](https://en.bitcoin.it/wiki/Bech32) 형식을 사용하고 `String` 메서드로 구현됩니다. Bech32 메서드는 블록체인과의 상호작용할 때 사용할 수 있는 유일한 형식입니다. 사람이 읽을 수 있는 Bech32 부분 (Bech32 prefix) 은 주소 타입을 나타내는데 사용됩니다. 예시:  

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/types/address.go#L230-L244

|                    | Address Bech32 Prefix |
| ------------------ | --------------------- |
| Accounts           | cosmos                |
| Validator Operator | cosmosvaloper         |
| Consensus Nodes    | cosmosvalcons         |

### 공개키

Cosmos SDK 의 공개키는 `cryptotypes.PubKey` 인터페이스로 정의됩니다. 저장소에 공개키가 저장되고나면, `cryptotypes.PubKey` 는 `proto.Message` 인터페이스를 확장합니다. 

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/crypto/types/types.go#L8-L17

`secp256k1` 과 `secp256r1` 직렬화에 압축 형식을 사용합니다.

- `y` 좌표가 사전식 순서로 `x` 좌표보다 크다면 첫번째 바이트는 `0x02` 가 됩니다. 
- 아니라면 첫번째 바이트는 `0x03` 이 됩니다.

이 prefix 다음에는 `x` 좌표가 붙습니다. 

공개키는 계정(또는 사용자) 참조에 사용되지 않으며 트랜잭션 메시지를 작성할 때 일반적으로 사용되지 않습니다 (몇가지 예외: `MsgCreateValidator`, `Validator` 와 `Multisig` 메시지들). `PubKey` 는 사용자 상호작용을 위해 프로토콜 버퍼 JSON ([ProtoMarshalJSON](https://github.com/cosmos/cosmos-sdk/blob/release/v0.42.x/codec/json.go#L12) 함수) 포맷으로 맞춰집니다.  
예시:  
+++ https://github.com/cosmos/cosmos-sdk/blob/7568b66/crypto/keyring/output.go#L23-L39

## Keyring

`Keyring` 은 계정을 저장하고 관리하는 객체입니다. Cosmos SDK 에서 `Keyring` 구현은 `Keyring` 인터페이스를 따릅니다. 

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/crypto/keyring/keyring.go#L51-L89

`Keyring` 의 기본 구현은 서드 파티인 [`99designs/keyring`](https://github.com/99designs/keyring) 라이브러리를 사용합니다.

`Keyring` 메서드의 몇가지 참고 사항:

- `Sign(uid string, payload []byte) ([]byte, sdkcrypto.PubKey, error)` 는 `payload` 바이트의 서명을 엄격하게 처리합니다. 트랜잭션은 반드시 표준 `[]byte` 형식으로 인코딩해야 합니다. 프로토콜 버퍼는 결정론적이지 않기 때문에, 서명할 표준 `payload` 를 [ADR-027](adr-027-deterministic-protobuf-serialization.md) 을 사용하여 결정론적으로 인코딩된 `SignDoc` 구조체로 [ADR-020](../architecture/adr-020-protobuf-transaction-encoding.md) 에서 결정하였습니다.  
  +++ https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/proto/cosmos/tx/v1beta1/tx.proto#L47-L64


- `NewAccount(uid, mnemonic, bip39Passwd, hdPath string, algo SignatureAlgo) (Info, error)` 는 [`bip44 path`](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) 기반으로 새 계정을 만들고 이를 디스크에서 유지합니다. `PrivKey` 는 **절대 암호화되지 않은 상태로 저장되지 않습니다**. 저장되기 전에 [암호 구문으로 암호화](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/crypto/armor.go) 됩니다. 이 메서드에서 키 타입과 시퀀스 번호는 니모닉에서 개인 및 공용키를 생성하는데 사용되는 BIP44 파생경로의 세그먼트(예: `0`, `1`, `2`, ...) 를 나타냅니다. 같은 니모닉과 파생경로를 사용하면, 같은 `PrivKey`, `PubKey` 그리고 `Address` 가 생성됩니다. 키링이 지원하는 키들은 다음과 같습니다.
  

- `secp256k1`

  
- `ed25519`


- `ExportPrivKeyArmor(uid, encryptPassphrase string) (armor string, err error)` 에서는 지정된 암호 구문을 사용해 ASCII-아머로 암호화된 형식의 개인키를 추출합니다. 그런 다음 `ImportPrivKey(uid, armor, passphrase string)` 함수를 사용하여 Keyring 으로 다시 개인키를 가져오거나 `UnarmorDecryptPrivKey(armorStr string, passphrase string)` 함수를 사용하여 원래의 개인키로 복호할 수 있습니다. 

## Next {hide}

[가스와 비용](gas-fees.md) 에 대해서 알아보세요.
