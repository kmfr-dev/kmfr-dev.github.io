---
title: 네트워크 로비의 Ready 상태 동기화
date: 2025-07-05 18:30:00 +0900
categories:
  - Project
  - The DeadLine
tags:
  - unreal-engine
  - multiplayer
  - replication
description: 로비 Ready 상태를 동기화하며 RPC 소유권과 RepNotify의 차이를 이해한 과정
image:
  path: /assets/img/the-deadline-cover.png
  alt: The DeadLine 프로젝트 대표 이미지
pin: false
---
>멀티플레이 로비에서 Ready 버튼을 누르면 모든 플레이어의 화면이 같은 상태를 보여야 했다. 기능 자체는 단순해 보였지만, 처음 만든 구조는 어떤 객체가 클라이언트의 요청을 서버에 전달할 수 있는지 잘못 이해한 상태에서 출발했다.

이전 글: [UI 레이어 기반 화면 관리]({% post_url 2025-06-10-the-deadline-02 %})

![플레이어별 Ready 상태가 표시되는 로비](/assets/img/the-deadline-ready-before.png)
_각 플레이어의 준비 상태를 모두 같은 화면으로 유지해야 했다._

## Server RPC는 호스트의 컨트롤러에서 실행될까

처음에는 클라이언트가 Server RPC를 호출하면 호스트가 사용하는 PlayerController에서 함수가 실행된다고 생각했다. 실제 로그를 비교해 보니 서버에는 접속한 플레이어마다 대응하는 Controller가 존재했고, 요청은 호출한 클라이언트에 대응하는 서버 측 인스턴스에서 처리됐다.

클라이언트는 Ready 변경을 요청하고, 서버는 해당 플레이어의 PlayerState를 찾아 값을 바꾸는 구조로 다시 정리했다. 서버가 전체 Ready 상태를 검사해야 하므로 개인 상태를 Pawn이 아닌 PlayerState에 두는 편이 자연스러웠다.

## 이벤트를 뿌리는 것보다 상태를 복제

초기 R&D에서는 서버가 모든 클라이언트에서 함수를 실행하는 Multicast RPC로 Ready 표시를 전달했다. 이미 접속한 사람의 화면은 맞출 수 있었지만 Ready는 한 번 발생하고 끝나는 이벤트가 아니었다. 늦게 접속한 플레이어도 현재 상태를 알아야 했다.

최종 구현에서는 값이 복제될 때 지정된 함수를 호출하는 RepNotify를 사용했다.

```cpp
UPROPERTY(ReplicatedUsing = OnRep_Ready)
bool bReady = false;

void ALobbyPlayerState::OnRep_Ready()
{
    if (ALobbyCharacter* Character = GetPawn<ALobbyCharacter>())
        Character->SetReadyUI(bReady);
}
```

Ready의 원본은 서버의 PlayerState가 소유하고, 각 클라이언트의 `OnRep_Ready`는 복제된 값으로 캐릭터 위의 UI를 갱신한다. 따라서 새로 접속한 클라이언트도 현재 상태를 복원할 수 있었다. 호스트가 서버와 플레이어를 겸하는 Listen Server에서는 값이 이미 로컬에 있어 OnRep가 호출되지 않는 경우가 있으므로, 서버가 값을 변경한 직후 호스트 화면은 별도로 갱신했다.

![모든 플레이어가 준비됐을 때 활성화된 시작 버튼](/assets/img/the-deadline-ready-complete.png)
_서버가 전체 Ready 상태를 확인한 뒤 호스트의 시작 버튼을 활성화했다._

>Ready 버튼을 구현하며 중요한 것은 RPC 호출 자체가 아니었다. 누가 값을 소유하고, 그 값이 이벤트인지 유지되어야 하는 상태인지 먼저 구분해야 했다.
