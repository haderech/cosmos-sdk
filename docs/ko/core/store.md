<!--
order: 5
-->

# 스토어(Store)

스토어는 애플리케이션의 상태를 유지하는 자료 구조입니다. {synopsis}

### 사전 요구 지식

- [SDK 애플리케이션의 해부](../basics/app-anatomy.md) {prereq}

## SDK 스토어 소개

Cosmos SDK는 애플리케이션의 상태를 유지할 수 있는 스토어의 거대한 모음과 함께 제공됩니다. 기본적으로 SDK의 주요 스토어는 `multistore`(i.e. 스토어들의 스토어)입니다. 개발자는 애플리케이션의 필요에 따라 multistore에 키-값(key-value) 스토어를 원하는 만큼 추가할 수 있습니다. multistore는 Cosmos SDK의 모듈화를 지원하기 위해 존재하며, 이는 각 모듈이 자체적으로 상태의 부분 집합(subset)을 선언하고 관리하도록 만들기 때문입니다. multistore 안의 키-값 스토어는 특정한 자격인 `key`를 통해서만 액세스할 수 있으며, 이 키는 일반적으로 해당 스토어를 선언한 모듈의 [`keeper`](../building-modules/keeper.md)에서 보유하고 있습니다.

```
+-----------------------------------------------------+
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 1 -  Module 1의 keeper에 의해 관리  ㅤ|
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 2 - Module 2의 keeper에 의해 관리   ㅤ|   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 3 - Module 2의 keeper에 의해 관리   ㅤ|   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 4 - Module 3의 keeper에 의해 관리   ㅤ|   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|    +--------------------------------------------+   |
|    |                                            |   |
|    |  KVStore 5 - Module 4의 keeper에 의해 관리   ㅤ|   |
|    |                                            |   |
|    +--------------------------------------------+   |
|                                                     |
|                    Main Multistore                  |
|                                                     |
+-----------------------------------------------------+

                   애플리케이션의 상태(state)
```

### 스토어 인터페이스

내부적으로, Cosmos SDK `store`는 `CacheWrapper`를 포함하는 객체이며 `GetStoreType()` 메서드를 갖고 있습니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L15-L18

`GetStoreType`는 스토어의 타입을 반환하는 간단한 메서드이며, `CacheWrapper`는 캐시를 읽고 `Write` 메서드를 사용하여 분기를 작성하는 스토어를 구현하는 간단한 인터페이스입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L240-L264

분기(Branching) 및 캐싱은 Cosmos SDK에서 보편적으로 사용되며 모든 스토어 타입에 대하여 구현되어야 합니다. 스토리지 브랜치는 일시적이고 분리된 스토어의 분기를 생성하는데, 이는 내재된 기본 스토어에 영향을 주지 않고, 전달 및 업데이트가 가능하며, 이후 오류가 발생했을 때 되돌릴 수 있는 임시의 상태 전환을 트리거하는데 사용됩니다. 자세한 내용은 [context](./context.md#Store-branching)를 참조하십시오.

### Commit Store

커밋 스토어는 변경 사항을 내재된 트리나 db에 커밋할 수 있는 기능을 가진 스토어입니다. Cosmos SDK는 다음과 같이 `Committer`를 사용하여 기본 스토어를 확장 시키는 것을 통해 단순 스토어와 커밋 스토어를 구분합니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L29-L33

`Committer`는 디스크에 변경 사항을 유지하기 위한 메서드를 정의하는 인터페이스입니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L20-L27

`CommitID`는 상태 트리의 결정론적(deterministic) 커밋입니다. 해당 해시는 하단의 합의 엔진에 반환되고 블록 헤더에 저장됩니다. 커밋 스토어는 다양한 용도로 존재하는 데, 그 중 하나는 모든 객체가 스토어를 커밋할 수 없도록 보장하는 것입니다. Cosmos SDK의 [오브젝트 자격(object-capabilities) 모델](./ocap.md)의 일부로 오직 `baseapp`만이 스토어를 커밋할 수 있습니다. 예를 들어, 일반적으로 스토어에 접근하는 모듈의 `ctx.KVStore()` 메서드는 `CommitKVStore`가 아닌 `KVStore`를 반환합니다.

Cosmos SDK는 많은 종류의 스토어를 제공하며, 그 중 가장 많이 쓰이는 것으로는 [`CommitMultiStore`](#multistore)와, [`KVStore`](#kvstore), [`GasKv` 스토어](#gaskv-스토어)가 있습니다. [다른 종류의 스토어](#other-stores)로는 `Transient`와 `TraceKV`가 있습니다.

## Multistore

### Multistore 인터페이스

각 Cosmos SDK 애플리케이션은 상태를 유지하기 위해 multistore를 root에 보관합니다. multistore는 `Multistore` 인터페이스를 따르는 `KVStores`의 저장소입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L104-L133

추적(tracing)이 활성화된 경우, multistore의 분기는 먼저 내장된 [`TraceKv.Store`](#tracekv-스토어) 내부의 모든 `KVStore`를 래핑합니다.

### CommitMultiStore

Cosmos SDK에서 주로 사용되는 `Multistore`의 타입은 `CommitMultiStore`이며, `Multistore` 인터페이스의 확장입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L141-L184

구체적(concrete) 구현으로, [`rootMulti.Store`]는 `CommitMultiStore` 인터페이스의 go-to 구현입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/rootmulti/store.go#L43-L61

`rootMulti.Store`는 복수의 `KVStores`가 마운트 될 수 있는 `db` 주위에 구축된 베이스 레이어(base-layer) multistore이며, [`baseapp`](./baseapp.md)에서 사용하는 기본 multistore입니다.

### CacheMultiStore

`rootMulti.Store`의 분기가 필요할때마다 [`cachemulti.Store`](https://github.com/cosmos/cosmos-sdk/blob/v0.42.1/store/cachemulti/store.go)가 사용됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/cachemulti/store.go#L17-L28

`cachemulti.Store`는 자신의 생성자에서 모든 하위 스토어(substore)들을 (각 하위 스토어의 가상 스토어를 만들어서) 분기시키고 `Store.stores`에 보관하고 모든 읽기 쿼리를 캐시합니다. `Store.GetKVStore()`는 `Store.stores`로부터 스토어를 반환하고, `Store.Write()`는 모든 하위 스토어들에 대해 `CacheWrap.Write()`를 재귀적으로 호출합니다.

## 베이스 레이어 KVStores

### `KVStore`와 `CommitKVStore` 인터페이스

`KVStore`는 데이터를 저장하고 검색할 수 있는 간단한 키-값 스토어입니다. `CommitKVStore`는 `Committer`를 구현하는 `KVStore`입니다. 기본적으로 `baseapp`의 주요 `CommitMultiStore`에 마운트된 스토어는 `CommitKVStore`들입니다. `KVStore` 인터페이스는 committer에 모듈이 접근하는 것을 제한하는 데 주로 사용됩니다.

개별 `KVStore`는 모듈이 글로벌 상태의 부분집합을 관리하기 위해 사용됩니다. `KVStores`는 특정 키를 소유한 객체에 의해서만 접근될 수 있습니다. 이 `key`는 해당 스토어를 정의한 모듈의 [`keeper`](../building-modules/keeper.md)에 의해서만 제공되어야 합니다.

`CommitKVStore`들은 각각의 `key`의 프록시에 의해 선언되며 애플리케이션의 [main 애플리케이션 파일](../basics/app-anatomy.md#코어-애플리케이션-파일) 내의 [multistore](#multistore)에 마운트 됩니다. 동일한 파일 내에서, `key`는 또한 스토어를 관리하는 모듈의 `keeper`에 전달됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L189-L219

종래의 `Get`과 `Set` 메서드와 다르게, `KVStore`는 `Iterator` 객체를 반환하는 `Iterator(start, end)` 메서드를 반드시 제공해야 한다. 이 것은 일반적으로 공통의 접두사를 사용하는 키와 같이, 키들의 범위를 반복하는데 사용된다. 다음은 bank의 모듈 `keeper`에서 모든 계정의 잔고를 반복하는 예시이다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/x/bank/keeper/view.go#L115-L134

### `IAVL` 스토어

`baseapp`에서 사용하는 `KVStore`와 `CommitKVStore`의 기본 구현이 바로 `iavl.Store`입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/iavl/store.go#L37-L40

`iavl`스토어는 다음을 보장하는 자가 균형 이진 트리(self-balancing binary tree)인 [IAVL 트리](https://github.com/tendermint/iavl)에 기반합니다:

- `Get`과 `Set` 연산은 O(log n)이며 여기서 n은 트리의 요소 갯수입니다.
- 반복하면 범위 내에서 정렬된 요소가 효율적으로 반환됩니다.
- 각 트리 버전은 불변이며 (가지치기(pruning) 설정에 따라) 커밋 이후에도 탐색할 수 있습니다.

[여기](https://github.com/cosmos/iavl/blob/v0.15.0-rc5/docs/overview.md)에서 IAVL 트리의 문서를 참조하십시오.

### `DbAdapter` 스토어

`dbadapter.Store`는 `dbm.DB`의 `KVStore`인터페이스를 충족시키는 어댑터입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/dbadapter/store.go#L13-L16

`dbadapter.Store`는 `dbm.DB`를 포함하는데, 이는 `KVStore` 인터페이스 함수의 대부분이 구현됐다는 것을 의미합니다. 다른 함수들(대부분 기타)는 수동으로 구현됩니다. 이 스토어는 [Transient Stores](#transient-스토어) 내에서 주로 사용됩니다.

### `Transient` 스토어

`Transient.Store`는 블록 끝에 자동으로 폐기되는 베이스 레이어 `KVStore`입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/transient/store.go#L13-L16

`Transient.Store`는 `dbm.NewMemDB()`를 제공하는 `dbadapter.Store`입니다. 모든 `KVStore` 메서드는 재사용 됩니다. `Store.Commit()` 호출되면 새로운 `dbadapter.Store`가 할당되고 이전의 참조를 폐기하고 수거됩니다(garbage collected).

이 스토어 타입은 블록별로 관련된 정보를 유지하는데 유용합합니다. 한가지 예는 변경된 인자를 (i.e. 블록에서 인자가 변경되면 bool이 `true`로 변경됨) 저장하는 것입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/x/params/types/subspace.go#L20-L30

Transient stores는 일반적으로 [`context`](./context.md)의 `TransientStore()` 메서드를 통해 액세스합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/types/context.go#L232-L235

## KVStore 래퍼(Wrappers)

### CacheKVStore

`cachekv.Store`는 기반하는 `KVStore`에 버퍼링된 쓰기 / 캐시된 읽기의 기능을 제공하는  `KVStore`의 래퍼입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/cachekv/store.go#L27-L34

이 타입은 분리된 스토어를 생성하기 위하여 (일반적으로 나중에 되돌릴 수 있는 상태를 변경할 때) IAVL 스토어가 분기할때 사용됩니다.

#### `Get`

`Store.Get()`은 먼저 `Store.cache`가 키에 연결된 값을 갖고 있는지 확인합니다. 값이 존재한다면 함수는 그 값을 반환합니다. 존재하지 않다면 `Store.parent.Get()`를 호출하여 `Store.cache`에 결과를 캐시하고 그 값을 반환합니다.

#### `Set`

`Store.Set()`은 키-값 쌍(pair)을 `Store.cache`에 설정합니다. `cValue`는 캐시된 값이 기본 값과 다른지를 나타내는 dirty bool 필드를 갖고 있습니다. `Store.Set()`이 새로운 쌍을 캐시하면 `cValue.dirty`는 `true`로 설정되므로 `Store.Write()`이 호출되어 기본 값에 쓰여집니다.

#### `Iterator`

`Store.Iterator()`는 캐시된 항목과 원본 항복을 순회해야합니다. `Store.iterator()`에서는 각각을 위해 두개의 반복자(iterators)가 생성되고 병합됩니다. `memIterator`는 기본적으로 캐시된 항목을 위해 사용되는 `KVPairs`의 한 조각입니다. `mergeIterator`는 두개의 반복자를 합친것으로 양쪽에서 순회가 순서대로 이루어집니다.

### `GasKv` 스토어

Cosmos SDK 애플리케이션은 자원 사용을 추적하고 과 스팸을 방지하기 위해 [`가스(gas)`](../basics/gas-fees.md)를 사용합니다. [`GasKv.Store`](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/gaskv/store.go)는 스토어에 대한 읽기나 쓰기가 이루어질 때 자동으로 가스를 소비하는 `KVStore`의 래퍼이다. 이 것은 Cosmos SDK 애플리케이션에서 스토리지 사용을 추적할 수 있는 최적의 솔루션입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/gaskv/store.go#L13-L19

부모인 `KVStore`의 메서드가 호출되면, `GasKv.Store`는 자동적으로 `Store.gasConfig`에 따라 적절한 양의 가스를 소모합니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/gas.go#L153-L162

기본적으로 모든 `KVStores`는 검색되었을 때 `GasKv.Stores`에 의해 감싸집니다. 이 작업은 [`context`](./context.md)의 `KVStore()` 메서드에서 이루어집니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/types/context.go#L227-L230

이 경우 기본 가스 설정이 사용됩니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/gas.go#L164-L175

### `TraceKv` 스토어

`tracekv.Store`는 `KVStore`의 작업을 추적하는 `KVStore`의 래퍼입니다. 추적이 부모의 `MultiStore`에서 활성화된 경우, 이 작업은 자동적으로 Cosmos SDK에 의해 모든 `KVStore`에 적용됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/tracekv/store.go#L20-L43

각 `KVStore` 메서드가 호출되면, `tracekv.Store`는 자동으로 `traceOperation`을 `Store.writer`에 기록합니다. `traceOperation.Metadata`는 nil이 아닌 경우 `Store.context`로 채워집니다. `TraceContext`는 `map[string]interface{}`입니다.

### `Prefix` 스토어

`prefix.Store`는 `KVStore`에 자동으로 키에 접두사를 붙이는(key-prefixing) 래퍼입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/prefix/store.go#L15-L21

`Store.{Get, Set}()`이 호출되면 스토어는 호출을 키를 `Store.prefix` 접두사와 함께 부모로 전달합니다.

`Store.Iterator()`가 호출되면 단순히 `Store.prefix` 접두사를 붙이지 않습니다. 의도한대로 동작하지 않기 때문입니다. 이 경우 일부 요소는 접두사로 시작하지 않는 경우에도 순회합니다.

## 다음 {hide}

[인코딩(encoding)](./encoding.md)에 대해 알아봅시다 {hide}
