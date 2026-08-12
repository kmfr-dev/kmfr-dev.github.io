---
title: Shipping 빌드의 Slate 기반 화면 로그
date: 2025-08-22 17:30:00 +0900
categories:
  - Project
  - The DeadLine
tags:
  - unreal-engine
  - shipping-build
  - slate
description: 최종 빌드에서만 발생하는 문제를 추적하기 위한 화면 로그를 만든 과정
image:
  path: /assets/img/the-deadline-cover.png
  alt: The DeadLine 프로젝트 대표 이미지
pin: false
---
>개발용 Development 빌드에서는 Unreal의 화면 로그를 통해 함수 호출과 상태를 바로 확인할 수 있었다. 반면 실제 배포에 사용하는 Shipping 빌드에서는 개발용 출력이 제거된다. 에디터에서는 정상인데 패키징한 게임에서만 문제가 생기면 어느 단계까지 실행됐는지 확인하기 어려웠다.

## 기존 로그 호출부는 유지하고 싶었다

프로젝트 전반의 로그 매크로를 모두 수정하면 테스트 도구를 추가하기 위해 게임 코드까지 크게 건드리게 된다. 기존 호출 방식은 유지하고 Shipping에서만 별도의 Slate 오버레이로 메시지를 보내도록 구성했다.

![Shipping 빌드의 화면 로그](/assets/img/the-deadline-shipping-log.png)
_최종 빌드에서 필요한 상태만 확인할 수 있도록 만든 화면 로그_

전역 로그 함수는 현재 게임 월드를 항상 알 수 없었기 때문에 엔진 실행 동안 유지되는 `EngineSubsystem`을 중간 통로로 사용했다. 실제 표시는 최종 화면을 관리하는 사용자 정의 `GameViewportClient`와 Slate 오버레이 `SDebugOverlay`가 담당했다.

```cpp
void ULogSubsystem::AddDebugLog(
    const FString& Message, FColor Color, float Duration)
{
#if UE_BUILD_SHIPPING
    if (!GEngine)
        return;

    if (!GEngine->GameViewport)
    {
        mLogArray.Emplace(Message, Color, Duration);
        return;
    }

    if (UProjectDViewportClient* Viewport =
        Cast<UProjectDViewportClient>(GEngine->GameViewport))
    {
        Viewport->AddDebugLog(Message, Duration, Color);
    }
#endif
}
```

## Viewport보다 로그가 먼저 들어오는 경우

초기화 도중 발생한 로그는 GameViewport가 만들어지기 전에 들어올 수 있었다. 이 메시지를 바로 출력하려 하면 로그 기능 자체가 초기화 순서에 의존하게 된다. Viewport가 준비되지 않았을 때는 메시지를 큐에 저장하고, 오버레이 생성 후 순서대로 비우도록 했다.

```cpp
const TArray<FLogData>& Logs = LogSubsystem->GetLogQueue();

for (const FLogData& Log : Logs)
    AddDebugLog(Log.LogMessage, Log._Duration, Log.LogColor);

LogSubsystem->ClearLogQueue();
```

최종 빌드에 개발 콘솔 전체를 남긴 것이 아니라 필요한 화면 로그만 제한적으로 복구했다. 실제 플레이 환경을 크게 바꾸지 않으면서도 Shipping 전용 문제를 확인할 수 있는 최소한의 통로를 만든 셈이다.
