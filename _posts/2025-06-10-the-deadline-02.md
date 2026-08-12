---
title: UI 레이어 기반 화면 관리
date: 2025-06-10 18:30:00 +0900
categories:
  - Project
  - The DeadLine
tags:
  - unreal-engine
  - ui
  - architecture
description: 6월 9일부터 10일까지 UISubsystem과 UI 레이어의 첫 구조를 설계한 과정
pin: false
---
프로젝트의 방향과 협업 방식을 정한 뒤, 본격적인 개발에 들어가기 전에 UI 공통 구조부터 만들기로 했다.

이전 글: [팀 협업 프로세스와 SVN 형상관리]({% post_url 2025-06-07-the-deadline-03 %})

## UI를 계속 직접 생성해도 될까

프로젝트 초반에는 타이틀, 로비, 인게임에서 필요한 위젯을 각각 생성해도 큰 문제가 없어 보였다. 하지만 기획된 UI 목록을 정리해 보니 메인 화면뿐 아니라 설정창, 확인창, 인게임 알림처럼 성격이 다른 화면이 계속 추가될 예정이었다. 

이 상태로 기능부터 만들면 각 코드가 위젯을 직접 생성하고 `ZOrder`, 즉 화면이 겹칠 때 앞뒤에 표시되는 순서까지 따로 결정하게 될 가능성이 높았다.

따라서 6월 9일부터 이틀 동안 개별 UI 제작보다 공통 구조를 먼저 잡았다. 

>목표는 세 가지였다. 레벨마다 하나의 메인 위젯만 생성하고, 그 아래에서 UI의 앞뒤 순서를 통제하며, 생성 경로를 한곳에서 관리하는 것이었다.

## 역할이 다른 UI를 레이어로 분리

메인 위젯 안을 `Main`, `Modal`, `Popup`, `Top` 패널로 나눴다. 기본 HUD는 Main, 뒤쪽 입력을 막아야 하는 확인창은 Modal, 잠시 나타나는 알림은 Popup, 항상 가장 위에 보여야 하는 요소는 Top에 배치했다. 이후 새 화면이 추가되어도 임의의 `ZOrder` 숫자를 정하지 않고 같은 규칙을 따를 수 있었다.

![MainWidget의 패널 구성 코드](/assets/img/the-deadline-ui-layers.png)
_화면의 앞뒤 순서를 숫자가 아닌 패널의 역할로 구분_

```cpp
UPROPERTY(meta = (BindWidget))
TObjectPtr<UCanvasPanel> MainPannel;

UPROPERTY(meta = (BindWidget))
TObjectPtr<UCanvasPanel> ModalPannel;

UPROPERTY(meta = (BindWidget))
TObjectPtr<UCanvasPanel> PopupPannel;
```

레벨별 메인 위젯 클래스는 `GameInstanceSubsystem` 기반의 `UISubsystem`에서 관리했다. `GameInstanceSubsystem`을 선택한 이유는 타이틀에서 로비, 인게임으로 레벨이 바뀌어도 동일한 접근 지점을 사용할 수 있기 때문이었다. `PlayerController`는 현재 레벨에 맞는 메인 위젯을 요청하고, 메인 위젯은 정해진 패널 아래에서 자식 UI를 관리하도록 책임을 나눴다.

```cpp
UUISubsystem* UUISubsystem::Get(const UObject* WorldContext)
{
    UGameInstance* GameInstance =
        UGameplayStatics::GetGameInstance(WorldContext);

    return GameInstance ? GameInstance->GetSubsystem<UUISubsystem>() : nullptr;
}
```

## 빠르게 검증한 만큼 한계도 명확했다

초기 구현에서는 위젯 클래스 경로를 `StaticLoadClass`로 불러와 맵에 등록했다. 이틀 안에 구조를 확인하기에는 가장 단순했지만, 에셋 경로가 바뀔 때 C++ 코드도 수정해야 했다. 또한 `UISubsystem`이 클래스 탐색과 생성 정보까지 모두 가지면서 역할이 커지는 문제가 남았다.

그래도 이 단계에서 UI가 놓일 자리와 기본 생성 규칙을 먼저 정해 둔 덕분에 이후 화면을 같은 기준으로 추가할 수 있었다.
