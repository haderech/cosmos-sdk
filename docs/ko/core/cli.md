<!--
order: 8
-->

# 명령어 인터페이스 (Command-Line Interface: CLI)

본 문서는 애플리케이션에 대하여 명령어 인터페이스(CLI)가 어떻게 동작하는지 설명합니다. 명령어 인터페이스를 구현하는 방법에 대해서는 [이 문서](../building-modules/module-interfaces.md#cli)를 참조 바랍니다.

## 명령어 인터페이스(CLI)

### CLI 사용 예제

새로운 CLI를 만들기 위해 SDK 모듈은 일반적으로 [Cobra Library](https://github.com/spf13/cobra) 을 사용합니다. Cobra Library를 사용하여 CLI을 만들기 위해 commands(명령어), arguments(인자값), flags(플래그)가 정의 가능합니다.
[**Commands**](#commands)는 사용자가 실행하려는 동작에 대하여 정의합니다. 예를 들어 트랜잭션 생성 할 때는 `tx`라는 command를 사용하고, 애플리케이션에서 특정 값을 조회 할 때는 `query`라는 command를 사용하도록 정의할 수 있습니다.
또한 commands는 중첩되는 subcommands를 포함하여 추가적으로 트랜잭션 타입을 지정할 수 있습니다. 사용자는 코인을 수신하려고 하는 계정번호를 **Arguments**에 지정할 수 있고 코인 전송에 필요한 gas price나 이용하려고 하는 node 주소를 [**Flags**](#flags)에 정의하여 추가적으로 commands에 대한 다양한 요소들을 변경할 수 있습니다.

다음은 사용자가 `simd`이라고 하는 CLI를 이용하여 토큰을 전송하려고 하는 예제입니다.
```bash
simd tx bank send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake --gas auto --gas-prices <gasPrices>
```
처음 네개의 string 값들이 command에 대하여 정의하고 있습니다.
- `simd` : command 이름
- `tx` : 트랜잭션 생성
- `bank` : 사용하기 원하는 모듈 지정 ( [`x/bank`](../../x/bank/spec/README.md) 모듈)
- `send` : 트랜잭션 타입

그 다음 두개의 string 값들은 arguements 입니다.
- `from_address` : 송신자 주소
- `to_address` : 수진자 주소
- `amount` : 보내고자 하는 액수

마지막으로 나머지 strings 값들은 flags로 사용자가 얼마만큼의 사용료(사용한 gas 양과 gas price를 통해 산출)를 지불할지 정의합니다.

위 CLI는 [node](../core/node.md)를 통하여 실행되며 인터페이스는 `main.go`에 정의됩니다.


### CLI 만들기

`main.go` 파일은 root command를 생성하는 `main()` 함수를 포함해야 하며 애플리케이션에서 할 모든 명령어들은 subcommands로 추가됩니다. root command는 추가적으로 다음을 수행합니다.

- **설정 변경** : 설정파일(sdk 설정 파일)을 읽어 반영합니다.
- **flags 값 설정** : `--chain-id` 같은 새로운 flag 값을 추가합니다.
- **`codec` 실행** : 애플리케이션의 `MakeCodec()` 함수(`simapp`에서 `MakeTestEncodingConfig`)를 호출합니다. [`codec`](../core/encoding.md)은 애플리케이션에서 사용하는 데이터를 encode하고 decode 합니다. 기본적으로 Probuf를 사용하지만 개발자가 다른 serialization format을 정의하여 사용할 수 있습니다.
- **subcommand 설정** : 사용자의 편의를 위하여 [transaction commands](#transaction-commands)와 [query commands](#query-commands)를 포함한 모든 액션을 정의합니다.

`main()`함수는 root command를 [실행](https://godoc.org/github.com/spf13/cobra#Command.Execute) 할 수 있는 executor를 생성합니다. 예제로 `simapp` 애플리케이션에서 `main()` 함수를 참조하세요.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/simapp/simd/main.go#L12-L24

다음에는 각 절차마다 개발이 필요한 부분에 대하여 설명하고 `simapp` CLI 파일의 간단한 예제를 소개하겠습니다. 

## CLI에 새로운 commond를 추가하기
모든 애플리케이션에서는 먼저 root command를 만들고, `rootCmd.AddCommand()`함수를 사용하여 subcommands를 집계(subcommands이 중첩가능)하여 기능을 추가하게 됩니다. 각 애플리케이션의 고유 기능에 대해서는 `TxCmd`라는 transaction command와 `QueryCmd`라는 query command에 정의됩니다.

### Root Command
`rootCmd`라고 하는 root command는 사용하려고 하는 애플리케이션을 지정하려 사용자가 맨처음 커맨드라인에 입력하는 string 값입니다. 일반적으로 애플리케이션 이름에 `-d`를 접미사로 붙여 사용합니다. (예: `simd`, `gaiad`) root command는 다음 commands를 포함하여 애플리케이션의 기본 기능에 대하여 정의할 수 있습니다.

- **Status** : SDK rpc client tools에서 제공합니다. 연결되어 있는 [`Node`](../core/node.md)에 대한 상태정보(`NodeInfo`,`SyncInfo`, `ValidatorInfo`)를 제공합니다.
- **Keys** : SDK client tools 에서 [제공](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/client/keys) 합니다. SDK crypto tools를 사용하기 위한 subcommands가 포함되어 새로운 key를 생성하여 keyring에 저장하고, keyring에 저장되어 있는 public key를 읽어오거나 key를 삭제할 수 있습니다. 
  예를 들어 사용자가 `simd keys add <name>`를 입력하여 새로운 key를 생성하고 암호화하여 keyring에 저장하거나, `--recover`라는 flag를 사용하여 seed phrase로 부터 private key를 복구하거나, `--multisig`라는 flag를 사용하여 여러개의 key를 그룹으로 묶어 multisig key를 생성할 수 있습니다.
  `add` key command에 대한 더 자세한 정보는 이 [코드](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/client/keys/add.go) 를 참조바라고 key credential 보관을 위하여 `--keyring-backend` 사용방법에 대해서는 [keyrings docs](../run-node/keyring.md) 참조 바랍니다.
- **Server** : SDK server package에서 제공합니다. ABCI Terndermint 애플리케이션 구동에 필요한 패키지를 제공하며 [Cobra](github.com/spf13/cobra) 기반의 CLI 프레임워크도 포함하고 있습니다.
  `StartCmd`와 `ExportCmd`라고 하는 두개의 코어 함수를 포함하여 애플리케이션을 시작하고 상태를 export 합니다. 더 자세한 정보는 [여기](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/server) 를 참조 바랍니다.
- [**Transaction**](#transaction-commands)
- [**Query**](#query-commands)

다음은 `simapp` 애플리케이션의 `rootCmd` 예제 함수입니다. root command를 정의하고 [_persistent_ flag](#flags)와 `PreRun`함수를 추가하여 다른 execution전에 실행시키고 모든 필요한 subcommands를 추가합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/4eea4cafd3b8b1c2cd493886db524500c9dd745c/simapp/simd/cmd/root.go#L37-L150

`rootCmd`에는 `initAppConfig()`라는 함수가 포함되어 있는데 애플리케이션의 설정을 임의 값으로 변경할 때 사용됩니다. 애플리케이션은 기본적으로 Tendermint의 config template을 사용하는데 `initAppConfig()`을 이용하여 설정을 덮어쓸수 있습니다. 다음은 기본 설정인 `app.toml`을 덮어쓰는 예제 입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/4eea4cafd3b8b1c2cd493886db524500c9dd745c/simapp/simd/cmd/root.go#L84-L117

`initAppConfig()`는 또한 기본 SDK의 [서버설정](https://github.com/cosmos/cosmos-sdk/blob/4eea4cafd3b8b1c2cd493886db524500c9dd745c/server/config/config.go#L199) 을 덮어쓸수 있습니다.
트랜잭션을 실행할 때 validators가 요구하는 최저 gas price를 설정하는 `min-gas-prices`을 예로 들어 보겠습니다. SDK는 기본적으로 이 설정을 `""` (비어있는 string)로 설정하는데 validators는 이 경우 `app.toml`을 변경하여 비어있지 않도록 하고 그 변경이 실패할 경우 구동 후 즉시 node의 실행을 멈춥니다.
이 방식이 validators에게는 최선이 아니기 때문에 체인 개발자들은 validators들을 위하여 `initAppConfig()` 함수 안에 기본 `app.toml` 값을 설정 할 수 있습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/aa9b055ddb46aacd4737335a92d0b8a82d577341/simapp/simd/cmd/root.go#L101-L116

`status`와 `keys` subcommands는 대부분의 애플리케이션들이 지원하지만 정작 애플리케이션의 상태를 변경시키지 않습니다. 애플리케이션의 기능들 대부분은 `tx`와 `query` commands를 통하여 변경될 수 있습니다.

### Transaction Commands

[Transactions](./transactions.md)은 [`Msg`s](../building-modules/messages-and-queries.md#messages) 를 감싸는 objects로 상태를 변경 시킬수 있습니다.
일반적으로 CLI를 통하여 트랜잭션을 생성시키고자 `rootCmd` 안에 `txCmd` 함수가 포함됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/simapp/simd/cmd/root.go#L86-L92

`txCmd` 함수는 사용 가능한 모든 트랜잭션 commands을 사용자에게 제공합니다. 

- **Sign command** : [`auth`](../../x/auth/spec/README.md) 모듈에서 제공되며 메시지 서명 기능을 제공합니다. multisig를 지원하려면 `auth` 모듈의 `MultiSign` command를 추가해야 합니다. 일반적으로 트랜잭션 안에 서명이 포함되어야 그 트랜잭션이 유효하기 때문에 이 sign command는 모든 애플리케이션에 필요합니다.
- **Broadcast command** : SDK client tools에서 제공되며 트랜잭션을 broadcast 할 때 사용합니다.
- **All [module transaction commands](../building-modules/module-interfaces.md#transaction-commands)** : [기본 모듈 매니저](../building-modules/module-manager.md#basic-manager)가 `AddTxCommands()` 함수를 통하여 제공합니다.

다음은 `simapp` 애플리케이션에서 `txCmd` 함수를 사용하는 예제입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/simapp/simd/cmd/root.go#L123-L149

### Query Commands

[**Queries**](../building-modules/messages-and-queries.md#queries)는 애플리케이션의 상태를 가져올 수 있는 object 입니다. `rootCmd` 안에 `queryCmd` 함수가 포함됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/simapp/simd/cmd/root.go#L86-L92

`queryCmd` 함수는 사용 가능한 모든 트랜잭션 commands을 사용자에게 제공합니다.

- **QueryTx** : `auth` 모듈에서 제공되며 사용자는 hash, tag, block height를 사용하여 트랜잭션을 검색 할 수 있습니다. 사용자는 이 command를 사용하여 트랜잭션이 블록에 포함되어 있는지 확인해 볼 수 있습니다.
- **Account command** : `auth` 모듈에서 제공되며 계정 잔고와 같은 상태를 확인 할 수 있습니다.
- **Validator command** : SDK rpc client tools에서 제공되며 height 별로 validator 그룹을 확인 할 수 있습니다.
- **Block command** : SDK rpc client tools에서 제공되며 height 별로 블록 데이터를 확인 할 수 있습니다.
- **All [module query commands](../building-modules/module-interfaces.md#query-commands)** : [기본 모듈 매니저](../building-modules/module-manager.md#basic-manager)가 `AddQueryCommands()` 함수를 통하여 제공됩니다.

다음은 `simapp` 애플리케이션에서 `queryCmd` 함수를 사용하는 예제입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/simapp/simd/cmd/root.go#L99-L121

## Flags

Flags를 사용하여 commands를 변경할 수 있습니다. 개발자는 flags를 `flags.go`파일에 포함 할 수 있습니다. 사용자는 flags를 명확히 commands와 함께 입력할 수 있고 `app.toml`안에 사전 구성 할 수 있습니다. 일반적으로 사전 구성하는 flags는 `--node`(연결하려는 노드)와 `--chain-d`(블록체인)가 있습니다. 

Persistent flag는 지속성이 있는 flag로서 그 command로 부터 상속받는 subcommands은 그 flag 값도 상속을 받습니다. 또한 모든 flags는 기본값이 설정되어 사용자는 그 기본값을 다른 값으로 변경해야 command를 유용하게 사용할 수 있습니다. 필수 flag인 경우에도 불구하고 사용자가 입력하지 않는 경우 에러가 발생하도록 할 수 있습니다.

Flag는 command에 바로 포함시키며 (일반적으로 [모듈의 CLI 파일](../building-modules/module-interfaces.md#flags) 안에) `rootCmd` persistent flags를 제외한 다른 flag들은 애플리케이션 레벨에 포함시킬 필요가 없습니다.
일반적으로 블록체인 아이디를 지정할 수 있는 persistent flag인 `--chain-d`을 root command에 추가합니다. 블록체인 아이디는 애플리케이션이 구동 된 후 변경되어서는 안되기 때문에 이 flag는 `main()`함수 안에 정의 할 수 있는 것이 좋습니다.
다음 `simapp` 애플리케이션 예제를 참조바랍니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/simapp/simd/cmd/root.go#L118-L119

## Environment variables

flag는 그 flag의 이름으로 구성되어 있는 환경변수와 깊이 결합되어 있습니다. 환경변수는 모두 대문자로 구성되어야 하고 `basename`으로 시작해야 하고 `-` 대신 `_`을 사용합니다.
예를 들어 flag `--home`이 있고 애플리케이션의 basename이 `GAIA`라고 한다면 해당 flag는 `GAIA_HOME` 환경변수와 결합되어 있습니다. 
환경변수를 활용하면 flag를 입력할 필요가 없습니다.
예를 들면:

```sh
gaia --home=./ --node=<node address> --chain-id="testchain-1" --keyring-backend=test tx ... --from=<key name>
```

위 예제 보다 아래가 더 사용하기 편리합니다.

```sh
# 환경변수를 한번 정의하세요
GAIA_HOME=<path to home>
GAIA_NODE=<node address>
GAIA_CHAIN_ID="testchain-1"
GAIA_KEYRING_BACKEND="test"

# 나중에 사용할 때 flag를 포함시키 않아도 됩니다
gaia tx ... --from=<key name>
```

## Configurations

애플리케이션의 root command가 Cobra의 `PersistentPreRun()` command property를 사용하는 것이 중요합니다. 그래야 그 command로부터 상속받는 다른 commands이 서버와 클라이언트의 contexts에 접근이 가능합니다.
이 contexts가 그들의 기본 설정으로 세팅이 되지만 나중에 변경도 가능하고 command 별로 변경이 적용이 되는 범위도 제한할 수 있습니다.
일반적으로 `client.Context`는 모든 commands들을 위하여 미리 유용한 기본값으로 설정이 되고 필요에 따라 값이 상속되거나 다른 값으로 변경될 수 있습니다.

다음은 `simapp` 애플리케이션에서 `PersistentPreRun()`함수를 사용하는 예제입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0/simapp/simd/cmd/root.go#L54-L60

`SetCmdClientContextHandler`를 호출하면 `ReadPersistentCommandFlags`을 통하여 persistent flags를 읽고 `client.Context`를 생성하여 root command의 `Context`을 설정합니다.

`InterceptConfigsPreRunHandler`를 호출하면 `server.Context`을 생성하여 root command의 `Context`을 설정합니다.
`server.Context`는 변경이 되어 `interceptConfigs`을 호출하여 디스크에 저장이 되고 `home path`에 정의되어 있는 Terdermint 설정파일을 읽어오든지 없다면 새롭게 생성합니다. 
또한 `interceptConfigs`을 호출하여 애플리케이션 설정파일인 `app.toml`을 읽고 `server.Context`의 viper literal에 연동시킵니다. 
이는 애플리케이션이 CLI flags 뿐만 아니라 애플리케이션 설정파일에 접근하기 위해 필요한 절차입니다.

## Next {hide}

[events](./events.md) {hide}