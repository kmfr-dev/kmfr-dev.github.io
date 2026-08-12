---
title: WidgetController 기반 UI 제어 흐름
date: 2025-07-09 18:30:00 +0900
categories:
  - Project
  - The DeadLine
tags:
  - unreal-engine
  - ui
  - refactoring
description: MVP 패턴을 바탕으로 WidgetController와 UI의 책임을 분리한 과정
image:
  path: /assets/img/the-deadline-cover.png
  alt: The DeadLine 프로젝트 대표 이미지
pin: false
---
>앞서 만든 UI 레이어와 서브시스템을 실제 화면에 적용하자 처음에는 보이지 않던 문제가 드러났다. 구조를 한 번에 다시 만들기보다 문제가 확인될 때마다 책임을 나누는 방향으로 고쳐 나갔다.

이전 글: [네트워크 로비의 Ready 상태 동기화]({% post_url 2025-07-05-the-deadline-ready-sync %})

## WidgetController와 MVP 패턴

`WidgetController`라는 이름만 보면 Unreal의 `PlayerController`와 무엇이 다른지 알기 어렵다. 이 객체는 UI를 Model, View, Presenter로 나누는 MVP 패턴에서 Presenter에 가까운 역할을 맡는다.

- **Model**: Character, GameState, Subsystem처럼 실제 상태와 규칙을 가진 게임 객체
- **View**: 값을 표시하고 입력을 받는 UMG 위젯
- **Presenter**: Model의 상태를 View에 전달하고, View에서 발생한 동작을 화면 흐름으로 연결하는 WidgetController

![The DeadLine UI에 적용한 MVP 구조](/assets/img/the-deadline-mvp-structure.svg)
_MVP를 Unreal UI 구조에 맞게 적용한 관계. WidgetController가 게임 객체와 UMG 사이를 연결한다._

정형화된 MVP를 그대로 구현한 것은 아니다. 별도의 View 인터페이스를 강제하지 않고 WidgetController가 UMG 위젯을 직접 참조한다. 대신 위젯이 `PlayerController`, 캐릭터, 서브시스템을 매번 찾아다니지 않도록 데이터 접근과 화면 제어의 접점을 한곳으로 모았다. 이 글에서 말하는 WidgetController는 이러한 Presenter 성격의 UI 중간 계층이다.

MVP 개념과 일반적인 관계는 [Model–View–Presenter 설명](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93presenter)과 [Wikimedia Commons의 MVP 도식](https://commons.wikimedia.org/wiki/File:Model_View_Presenter.svg)을 참고했다.

## 생성과 제어를 분리

UI 레이어를 만든 직후에는 `PlayerController`가 메인 위젯을 생성하고 세부 화면까지 직접 제어했다. 위젯도 필요한 값을 얻기 위해 다시 `PlayerController`나 캐릭터를 찾았다. 기능이 늘어나면 게임 객체와 UI가 서로의 내부 구조를 알아야 하는 형태가 될 수 있었다.

그래서 메인 위젯의 생성과 데이터 전달을 담당하는 `WidgetControllerBase`를 만들었다. 공통 동작은 기반 클래스에 두고 타이틀, 로비, 인게임 전용 동작은 파생 Controller로 분리했다. `PlayerController`는 자신에게 필요한 WidgetController를 소유하지만 구체적인 위젯을 하나씩 찾지는 않도록 했다.

```cpp
void AInGamePlayerController::BeginPlay()
{
    mWidgetController = NewObject<UInGameWidgetController>(this);

    if (mWidgetController->SetBoundWidget())
    {
        UUISubsystem::Get(this)
            ->SetCurrentWidgetController(mWidgetController);
        mWidgetController->DefaultSetting();
    }
}
```

`PlayerController`는 현재 화면의 Controller를 생성하고, WidgetController는 메인 위젯을 바인딩한 뒤 필요한 이벤트를 연결한다. 개별 위젯은 데이터의 출처를 직접 찾지 않아도 되고, 게임 객체도 구체적인 위젯 계층을 몰라도 된다.

## Delegate로 Model의 변경을 구독

WidgetController가 매 프레임 캐릭터의 값을 확인하면 UI와 게임 객체가 불필요하게 계속 연결된다. 대신 상태가 실제로 바뀌는 순간에만 Presenter가 통지받도록 Delegate를 사용했다.

체력과 스태미나를 관리하는 `StatusComponent`에는 변경된 비율을 전달하는 Delegate를 선언했다.

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(
    FOnHPChanged, float, Percent);

DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(
    FOnStaminaChanged, float, Percent);
```

플레이어가 생성되면 `InGameWidgetController`가 자신의 UI 갱신 함수를 각 Delegate에 바인딩한다.

```cpp
void UInGameWidgetController::BindSurvivor(APawn* Pawn)
{
    ASurvivor* Survivor = Cast<ASurvivor>(Pawn);
    if (!Survivor)
        return;

    if (UStatusComponent* Status = Survivor->GetStatus())
    {
        Status->mHPUpdateDelegate.AddDynamic(
            this, &UInGameWidgetController::UpdateHPBar);

        Status->mStaminaUpdateDelegate.AddDynamic(
            this, &UInGameWidgetController::UpdateSteminaBar);
    }
}
```

이후 `StatusComponent`는 체력이 바뀌었을 때 UI를 직접 찾지 않고 계산된 비율만 Broadcast한다.

```cpp
void UStatusComponent::RefreshHP()
{
    const FStatusData* MaxHP =
        GetStatusType(EStatusType::STATTYPE_MAXHP);

    if (!MaxHP)
        return;

    const float Percent = mHP / MaxHP->value;
    mHPUpdateDelegate.Broadcast(Percent);
}
```

호출 흐름은 다음과 같다.

`StatusComponent 상태 변경 → Broadcast(Percent) → WidgetController::UpdateHPBar → UMG 갱신`

Model은 어떤 위젯이 값을 표시하는지 모르고, View도 체력 데이터를 가진 컴포넌트를 직접 찾지 않는다. WidgetController가 둘 사이의 연결과 화면 갱신을 담당하므로 MVP에서 Presenter를 둔 목적이 실제 코드에도 드러난다.

같은 방식으로 인벤토리 변경, 손전등 배터리, 상호작용 진행도, 사망과 탈출 이벤트도 WidgetController에 바인딩했다. 기능별 Model은 자신의 상태 변화만 알리고, 어떤 UI를 보여줄지는 Presenter가 결정하도록 역할을 나눴다.
