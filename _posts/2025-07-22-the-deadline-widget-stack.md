---
title: 위젯 스택 기반 퀵메뉴 제어
date: 2025-07-22 18:30:00 +0900
categories:
  - Project
  - The DeadLine
tags:
  - unreal-engine
  - ui
  - stack
description: 여러 모달 화면에서 ESC 입력의 닫기 순서를 관리한 과정
pin: false
---
>인게임에서 ESC를 누르면 설정과 종료 버튼이 있는 퀵메뉴가 열린다. 이 위에서 도움말이나 종료 확인창을 다시 열 수 있게 되면서 ESC 입력의 의미가 모호해졌다. 무
>
>조건 퀵메뉴를 닫으면 그 위의 창이 남고, 화면마다 조건문을 추가하면 새 모달이 생길 때마다 입력 코드도 수정해야 했다.

## 마지막에 연 화면부터 닫기

표시된 모달을 스택에 넣고 ESC 입력 시 마지막 요소를 닫았다. 스택은 마지막에 넣은 값을 먼저 꺼내는 자료구조이므로, 화면이 열린 순서의 반대로 닫는 동작과 맞았다. 열린 모달이 없을 때만 퀵메뉴를 표시해 같은 입력이 현재 UI 상태에 맞게 동작하도록 했다.

![퀵메뉴 위젯 구성](/assets/img/the-deadline-stack-rnd-1.png)
_설정과 종료처럼 다른 모달로 이어지는 퀵메뉴_

```cpp
void UWidgetControllerBase::ToggleESC()
{
    if (mWidgetStack.IsEmpty())
    {
        ShowModalWidget(UUIDefine::SubWidgetName::QUICKMENU);
        return;
    }

    UModalWidgetBase* LastWidget = mWidgetStack.Last();
    const FName* WidgetName = mWidgetMap.FindKey(LastWidget);

    if (WidgetName)
        HideModalWidget(*WidgetName);
}
```

개별 위젯이 서로의 존재를 확인하지 않아도 열림 순서 자체가 닫기 순서가 됐다. 이후 모달이 추가돼도 ESC 처리 코드는 바뀌지 않았다.

![UI 입력 액션과 매핑 컨텍스트](/assets/img/the-deadline-stack-rnd-3.png)
_게임 조작과 분리해 관리한 UI 전용 입력 매핑_
