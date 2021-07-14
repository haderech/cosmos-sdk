<!--
order: 15
-->

# 인-플레이스(In-Place) 스토어(Store) 마이그레이션

::: warning
동작 중인(live) 체인에서 마이그레이션을 하는 경우 인-플레이스 스토어 마이그레이션 문서를 전부 읽고 이해한 후 수행하십시오.
:::

사용자 지정 인-플레이스 마이그레이션 로직을 사용하여 앱 모듈을 원활하게 업그레이드 하십시오. {synopsis}

Cosmos SDK는 다음 두 방법을 사용하여 업그레이드를 수행합니다.

- 애플리케이션의 전체 상태를 `export` CLI 명령어를 사용하여 JSON 파일로 내보내서 변경한 후, genesis 파일로 변경된 JSON 파일을 사용하여 새 바이너리를 시작하십시오. [체인 업그레이드 가이드](../migrations/chain-upgrade-guide-040.md#upgrade-procedure)를 참조하십시오.
- 버젼 v0.43 이상에서는 더 거대한 상태를 가진 체인을 현격히 빠르게 인-플레이스 업그레이드를 수행할 수 있습니다. [마이그레이션 업그레이드 가이드](../building-modules/upgrade.md)를 참조하여 인-플레이스 업그레이드를 활용하여 애플리케이션 모듈을 설정하십시오.

위 문서에서는 인-플레이스 스토어 마이그레이션 업그레이드 메서드를 사용하는 단계를 다룹니다.

## 모듈 버젼 추적하기

모듈 개발자는 각 모듈의 합의 버젼을 지정합니다. 합의 버젼은 모듈의 호환성을 깨는(breaking change) 버젼을 의미합니다. SDK는 x/upgrade `VersionMap` 스토어에서 모든 모듈의 합의 버젼을 추적합니다. 업그레이드 과정에서 Cosmos SDK에 의해 기존의 `VersionMap`과 새로운 `VersionMap` 간의 변경이 계산됩니다. 식별된 각 변경 마다 모듈 전용의 마이그레이션이 실행되며 각각의 업그레이드된 모듈의 합의 버젼은 증가합니다.

## Genesis 상태

새로운 체인을 시작할 때 애플리케이션의 genesis 도중에 각 모듈의 합의 버젼은 반드시 상태로 저장되어야 합니다. 합의 버젼을 저장하기 위해서 `app.go`의 `InitChainer` 메서드에 다음 줄을 추가하십시오:

```diff
func (app *MyApp) InitChainer(ctx sdk.Context, req abci.RequestInitChain) abci.ResponseInitChain {
  ...
+ app.UpgradeKeeper.SetModuleVersionMap(ctx, app.mm.GetVersionMap())
  ...
}
```

이 정보는 Cosmos SDK가 모듈의 새로운 버젼이 앱에 도입되는 것을 감지하기 위해 사용합니다.

### 합의 버젼

모듈 개발자는 각 모듈의 합의 버젼을 지정합니다. 합의 버젼은 모듈의 호환성을 깨는(breaking change) 버젼을 의미합니다. 합의 버젼은 어느 모듈이 업그레이드가 필요한지를 SDK에 알려줍니다. 예를 들어 bank 모듈이 버젼 2를 사용하고 업그레이드에서 bank 모듈 3이 도입되는 경우, SDK는 bank 모듈을 업그레이드 하고 "version 2 to 3" 마이그레이션 스크립트를 실행합니다.

### 버젼 맵(Map)

버젼 맵은 합의 버젼을 모듈명에 매핑한 것입니다. 버젼 맵은 인-플레이스 마이그레이션 도중 사용하기 위하여 x/upgrade의 상태로 보관됩니다. 마이그레이션이 완료되면 업데이트된 버젼 맵이 상태로 유지됩니다.

## 처리기(handler) 업그레이드

마이그레이션을 용이하기 위해 업그레이드는 `UpgradeHandler`를 사용합니다. 앱 개발자가 구현하는 `UpgradeHandler` 함수는 다음의 함수 시그니쳐와 일치해야합니다. 이 함수는 x/upgrade의 상태로부터 `VersionMap`을 검색하고 새로운 `VersionMap`을 반환하여 업그레이드 이후 x/upgrade에 저장하도록 합니다. 두 `VersionMap`간 차이에 의해 어느 모듈이 업그레이드가 필요한지 결정됩니다.

```golang
type UpgradeHandler func(ctx sdk.Context, plan Plan, fromVM VersionMap) (VersionMap, error)
```

이러한 함수 내부에서, 제공된 `plan`에 수행할 모든 업그레이드 로직을 포함시켜야합니다. 모든 업그레이드 처리기의 함수는 다음 코드 줄과 함께 끝나야합니다.

```golang
  return app.mm.RunMigrations(ctx, cfg, fromVM)
```

## 마이그레이션 실행하기

마이그레이션은 `app.mm.RunMigrations(ctx, cfg, vm)`를 사용하여 `UpgradeHandler` 내부에서 실행됩니다. `UpgradeHandler` 함수는 업그레이드 도중 발생하는 기능을 나타냅니다. `RunMigration` 함수는 `VersionMap` 매개 변수를 통해 반복적으로 호출되고 새로운 바이너리 앱 모듈보다 작은 모든 버전에 대해 마이그레이션 스크립트를 실행합니다. 마이그레이션이 완료되면 새로운 `VersionMap`이 반환되어 업그레이드된 모듈 버젼의 상태로 유지됩니다.

```golang
cfg := module.NewConfigurator(...)
app.UpgradeKeeper.SetUpgradeHandler("my-plan", func(ctx sdk.Context, plan upgradetypes.Plan, vm module.VersionMap) (module.VersionMap, error) {

    // ...
    // 업그레이드 로직 수행
    // ...

    // RunMigrations는 업데이트된 모듈 합의 버젼의
    // VersionMap을 반환합니다.
    return app.mm.RunMigrations(ctx, vm)
})
```

모듈의 마이그레이션 스크립트를 구성하는 방법에 대한 자세한 내용은 [마이그레이션 업그레이드 가이드](../building-modules/upgrade.md)를 참조하십시오.

## 업그레이드 도중 새로운 모듈 추가하기

업그레이드 도중 애플리케이션에 완전히 새로운 모듈을 도입할 수 있습니다. 아직 모듈이 `x/upgrade`의 `VersionMap` 스토어에 등록되지 않았기 때문에 새로운 모듈로 인식됩니다. 이 경우, `RunMigrations`는 해당하는 모듈의 `InitGenesis`를 호출하여 초기 상태를 설정합니다.

### 새로운 모듈을 위한 StoreUpgrades 추가하기

인-플레이스 스토어 마이그레이션을 실행하기를 원하는 모든 체인들은 수동으로 새 모듈의 스토어 업그레이드를 자동으로 추가하고, 업그레이드를 적용하기 위한 스토어 로더를 구성해야 합니다. 이 작업은 마이그레이션이 시작하기 전에 multistore에 새로운 모듈의 스토어가 추가되었음을 보장해줍니다.

```golang
upgradeInfo, err := app.UpgradeKeeper.ReadUpgradeInfoFromDisk()
if err != nil {
	panic(err)
}

if upgradeInfo.Name == "my-plan" && !app.UpgradeKeeper.IsSkipHeight(upgradeInfo.Height) {
	storeUpgrades := storetypes.StoreUpgrades{
		// 새로운 모듈을 위한 스토어 업그레이드 추가한다.
		// 예:
		//    Added: []string{"foo", "bar"},
		// ...
	}

	// version == upgradeHeight를 확인하는 스토어 로더를 구성하고 스토어 업그레이드를 적용한다.
	app.SetStoreLoader(upgradetypes.UpgradeStoreLoader(upgradeInfo.Height, &storeUpgrades))
}
```

## Genesis 함수 덮어쓰기(overwriting)

Cosmos SDK는 애플리케이션 개발자가 앱으로 가져올(Import) 수 있는 모듈을 제공합니다. 이 중 많은 모듈들이 기정의된 `InitGenesis` 함수를 갖고 있습니다.

가져온 모듈에 대하여 자체적인 `InitGenesis` 함수를 작성할 수 있습니다. 이를 위하여 업그레이드 처리기에서 사용자 정의 genesis 함수를 수동으로 트리거합니다.

::: warning
`UpgradeHandler` 함수에 전달되는 버젼 맵 내부의 합의 버젼을 반드시 설정하십시오. 그러지 않은 경우, `UpgradeHandler`의 사용자 정의 함수를 트리거한 경우에도 SDK는 모듈의 기존 `InitGenesis`를 실행합니다.
:::

```go
import foo "github.com/my/module/foo"

app.UpgradeKeeper.SetUpgradeHandler("my-plan", func(ctx sdk.Context, plan upgradetypes.Plan, vm module.VersionMap)  (module.VersionMap, error) {
    // SDK가 기본 InitGenesis 함수를 실행하는 것을
    // 피하기 위해 버젼 맵에 합의 버젼을 등록한다.
    vm["foo"] = foo.AppModule{}.ConsensusVersion()

    // foo의 사용자 정의 InitGenesis를 실행한다.
    app.mm["foo"].InitGenesis(ctx, app.appCodec, myCustomGenesisState)

    return app.mm.RunMigrations(ctx, cfg, vm)
})
```

사용자 지정 genesis 함수가 존재하지 않고 모듈의 기본 genesis 함수를 건너뛰려는 경우, 다음의 예와 같이 `UpgradeHandler`에 있는 버젼 맵에 모듈을 간단히 등록하면 됩니다:

```go
import foo "github.com/my/module/foo"

app.UpgradeKeeper.SetUpgradeHandler("my-plan", func(ctx sdk.Context, plan upgradetypes.Plan, vm module.VersionMap)  (module.VersionMap, error) {
    // food의 버젼을 VersionMap의 최신의 ConsensusVersion으로 설정합니다.
    // 이 경우 Foo의 InitGenesis를 건너뜁니다.
    vm["foo"] = foo.AppModule{}.ConsensusVersion()

    return app.mm.RunMigrations(ctx, cfg, vm)
})
```

## 풀-노드를 업그레이드된 블록체인과 동기화 하기

Cosmovisor를 사용하여 업그레이드된 기존의 블록체인과 풀-노드를 동기화 할 수 있습니다.

성공적으로 동기화 하기 위해서는 반드시 블록체인이 시작한 genesis 위치의 초기 바이너리를 사용하여 시작해야합니다. Cosmovisor는 각각의 순차적 업그레이드와 관련된 바이너리에 대한 다운로드 및 전환을 처리합니다.

Cosmovisor에 대한 자세한 내용은 [Cosmovisor 퀵 스타트](../run-node/cosmovisor.md)를 참조하십시오.
