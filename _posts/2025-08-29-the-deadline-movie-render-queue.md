---
title: Movie Render Queue의 Transform 충돌 해결
date: 2025-08-29 18:30:00 +0900
categories:
  - Project
  - The DeadLine
tags:
  - unreal-engine
  - sequencer
  - movie-render-queue
description: Take Recorder로 기록한 차량 시퀀스가 Movie Render Queue에서 다르게 움직인 원인
pin: false
---
>생존자가 탈출한 뒤 보여줄 차량 연출이 필요했다. 직접 차량을 운전한 움직임을 `Take Recorder`로 기록하고, 완성된 시퀀스를 `Movie Render Queue`로 출력했다.

## 플레이 움직임을 시퀀스로 기록

촬영용 차량과 카메라를 Take Recorder의 소스로 등록했다. PIE 상태에서 차량을 운전한 뒤 녹화를 종료하면 위치와 회전이 시퀀스의 Transform 키로 저장된다.

| 촬영 대상 등록 | 기록된 시퀀스 |
| --- | --- |
| ![Take Recorder에 등록한 차량과 카메라](/assets/img/the-deadline-mrq-rnd-2.png) | ![차량 Transform이 기록된 시퀀스](/assets/img/the-deadline-mrq-rnd-4.png) |

Sequencer에서 재생했을 때는 기록한 경로대로 차량이 움직였다. 그러나 같은 시퀀스를 Movie Render Queue로 출력하면 차량이 경로를 벗어났다.

## 두 시스템이 동시에 위치를 변경

문제가 된 차량은 일반 액터가 아니라 `Vehicle Movement Component`와 물리 시뮬레이션이 적용된 Pawn이었다.

![Vehicle Movement가 적용된 촬영용 차량](/assets/img/the-deadline-mrq-rnd-1.png)
_시퀀스의 Transform과 별개로 차량의 움직임을 계산할 수 있는 상태_

렌더링 중에는 다음 두 처리가 동시에 차량 위치를 변경하고 있었다.

- Sequencer가 녹화된 Transform을 적용
- Vehicle Movement와 물리가 다음 위치를 계산

따라서 문제는 출력 해상도나 프레임 설정이 아니라 **하나의 Transform을 두 시스템이 갱신한 충돌**이었다.

## 기록된 Transform만 사용

출력용 차량에서는 Movement Tick과 물리 시뮬레이션을 비활성화했다. 위치를 변경하는 주체가 Sequencer 하나로 줄자 미리보기와 최종 영상의 이동 경로가 같아졌다.

![Movie Render Queue 출력 설정](/assets/img/the-deadline-mrq-rnd-5.png)
_수정한 시퀀스를 최종 영상으로 출력한 Movie Render Queue 설정_

렌더 결과가 미리보기와 다를 때 출력 옵션만 확인해서는 원인을 찾기 어려웠다. 액터의 Transform을 매 프레임 누가 변경하는지 먼저 구분한 것이 해결의 핵심이었다.
