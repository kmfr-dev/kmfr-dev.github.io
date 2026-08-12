---
title: Multicast RPC 기반 수리 진행도 송수신
date: 2025-09-05 17:30:00 +0900
categories:
  - Project
  - The DeadLine
tags:
  - unreal-engine
  - multiplayer
  - rpc
description: 탈출 수단의 부품 수리 UI를 이벤트 단위로 동기화한 이유
image:
  path: /assets/img/the-deadline-cover.png
  alt: The DeadLine 프로젝트 대표 이미지
pin: false
---
>Survivor는 맵에서 부품을 찾아 차량이나 보트 같은 탈출 수단을 수리한다. 탈출 수단마다 필요한 부품이 다르며, 수리가 진행될 때 오브젝트 위에 표시되는 월드 UI의 현재 개수와 완료 상태를 모든 플레이어에게 동일하게 보여줘야 했다.

![탈출 수단에 표시되는 부품별 수리 현황](/assets/img/the-deadline-repair-overview.png)
_탈출 수단마다 필요한 부품 종류와 현재 개수를 월드 UI로 표시했다._

## 배열 전체를 복제할 필요가 있을까

RepNotify로 부품 배열을 계속 복제하면 늦게 들어온 클라이언트의 상태 복원에는 유리하지만, 각 클라이언트가 배열을 다시 순회해 어떤 부품이 변했는지 찾아야 했다. 배열에서 바뀐 항목만 효율적으로 복제하는 FastArray도 선택지였지만 탈출 수단 하나에 필요한 부품은 최대 세 종류였고 수리 이벤트도 경기 중 자주 발생하지 않았다.

게임 규칙의 판정은 서버가 담당했다. 서버는 수리가 성공한 순간 이미 변경된 부품 종류와 현재 개수, 완료 여부를 알고 있으므로, 이 세 값만 Multicast RPC 인자로 전달해 접속 중인 모든 클라이언트의 월드 UI를 갱신했다.

![수리 진행도 Multicast 구현](/assets/img/the-deadline-repair-multicast.png)
_서버가 확정한 부품 종류와 개수를 이벤트 단위로 전달_

```cpp
UFUNCTION(NetMulticast, Reliable)
void Multicast_SetTextWidgetComp(
    EItemPartType Type, int32 Count, bool IsRepaired);
```

UI가 보이지 않는 클라이언트에도 RPC가 호출되는 비용은 있었다. 다만 최대 인원과 호출 횟수가 작았고, 복제 상태를 하나 더 관리하지 않아도 된다는 장점이 더 크다고 판단했다. 

| 수리 전 | 수리 완료 |
| --- | --- |
| ![수리 전 부품 UI](/assets/img/the-deadline-repair-before.png) | ![수리 완료 부품 UI](/assets/img/the-deadline-repair-after.png) |

이 방식은 현재 접속 중인 플레이어에게 드문 UI 변경 이벤트를 전달한다는 조건에 맞춘 선택이었다. 도중 참가나 재접속을 지원한다면 Multicast만으로 과거 상태를 복구할 수 없으므로, 그때는 복제 상태를 함께 두어야 한다.
