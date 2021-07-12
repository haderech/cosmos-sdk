<!--
order: 12
-->

# RunTx 복구(recovery) 미들웨어

`BaseApp.runTx()` 함수는 keeper가 유효하지 않은 상태에 직면하여 패닉 상태에 빠지는 등과 같은 트랜잭션 실행 중 발생할 수 있는 Golang 패닉을 처리합니다.
패닉 타입에 따라, 다른 처리기(handler)가 사용됩니다. 예를 들면 기본 처리기는 에러 로그 메시지를 출력합니다.
복구 미들웨어는 SDK 애플리케이션 개발자를 위해 사용자 정의 패닉 복구 방법을 추가하는데 사용됩니다.

자세한 내용은 다음의 [ADR-022](../architecture/adr-022-custom-panic-handling.md) 파일에서 확인하실 수 있습니다.

구현은 다음의 [recovery.go](../../baseapp/recovery.go) 파일을 참조하십시오.

## 인터페이스

```go
type RecoveryHandler func(recoveryObj interface{}) error
```

`recoveryObj`는 `buildin` Golang 패키지에 있는 `recover()` 함수의 반환 값입니다.

**컨트랙트(Contract)**

- `recoveryObj`가 처리되지 않은 경우, RecoveryHandler는 `nil`을 반환하고 다음 복구 미들웨어에게 전달해야 합니다;
- `recoveryObj`가 처리된 경우, RecoveryHandler는 `nil`이 아닌 `error`를 반환합니다.

## 사용자 정의 RecoveryHandler 등록

`BaseApp.AddRunTxRecoveryHandler(handlers ...RecoveryHandler)`

BaseApp 메서드는 기본 복구 체인에 복구 미들웨어를 추가합니다.

## 예시

특정 오류가 발생한 경우 "Consensus failure" 체인 상태를 발생시키고 싶다고 가정해봅시다.

패닉상태에 빠진 module keeper는 다음과 같습니다:

```go
func (k FooKeeper) Do(obj interface{}) {
    if obj == nil {
        // 일어나서는 안되며, 앱을 종료(crash)시켜야합니다.
        err := sdkErrors.Wrap(fooTypes.InternalError, "obj is nil")
        panic(err)
    }
}
```

기본적으로 패닉이 복구되고 에러 메시지는 로그에 출력됩니다. 이러한 동작을 재정의하려면 사용자 저으이 RecoveryHandler를 등록해야 합니다.

```go
// SDK 애플리케이션 생성자
customHandler := func(recoveryObj interface{}) error {
    err, ok := recoveryObj.(error)
    if !ok {
        return nil
    }

    if fooTypes.InternalError.Is(err) {
        panic(fmt.Errorf("FooKeeper did panic with error: %w", err))
    }

    return nil
}

baseApp := baseapp.NewBaseApp(...)
baseApp.AddRunTxRecoveryHandler(customHandler)
```

## 다음 {hide}

[IBC](./../ibc/README.md) 프로토콜에 대해 알아봅시다. {hide}
