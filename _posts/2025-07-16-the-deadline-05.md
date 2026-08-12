---
title: Editor Utility Widget 기반 스폰 데이터 관리
date: 2025-07-16 18:30:00 +0900
categories:
  - Project
  - The DeadLine
tags:
  - unreal-engine
  - editor-utility-widget
  - tool
description: 7월 16일 스폰 데이터 편집과 월드 미리보기를 하나의 에디터 도구로 묶은 과정
pin: false
---
>UI 구조를 정리한 뒤에는 콘텐츠를 반복해서 배치하는 과정에도 눈을 돌렸다. 런타임 코드를 고치는 문제가 아니라, 같은 확인 작업을 계속 반복해야 하는 제작 과정의 문제였다.
## 런타임 구조는 편했지만 제작 과정은 불편했다

매 판 아이템 위치가 같으면 Survivor가 주요 부품의 위치를 외울 수 있었다. 이를 막기 위해 맵마다 여러 스폰 후보 지점을 두고, 게임 시작 시 일부 지점과 아이템을 무작위로 선택했다.

스폰 위치와 해당 지점에 등장할 수 있는 아이템 목록은 행과 열 형태의 에셋인 DataTable로 관리했다. 런타임에서는 먼저 스폰 지점 순서를 섞고, 아이템별 최소 등장 수량을 보장한 뒤 남은 위치에 최대 수량과 확률 조건을 적용했다.

```cpp
TArray<FName> ShuffledSpawnIDs = MapRow->SpawnIDs;
Algo::RandomShuffle(ShuffledSpawnIDs);

for (const FName& SpawnID : ShuffledSpawnIDs)
{
    if (UsedSpawnIDs.Contains(SpawnID))
        continue;

    const FSpawnTableRow* SpawnRow =
        TableSubsystem->FindTableRow<FSpawnTableRow>(
            SpawnTableName, SpawnID);

    FName ItemTID;
    const FItemTableRow* ItemRow =
        PickItemForPoint(SpawnRow, ItemTID);

    if (ItemRow)
        SpawnItemInstance(SpawnRow, ItemRow, ItemTID);
}
```

각 스폰 지점에서 후보 아이템을 다시 섞고, 최대 생성 개수와 확률을 통과한 항목만 선택했다.

```cpp
TArray<FName> Candidates = SpawnRow->ItemTIDs;
Algo::RandomShuffle(Candidates);

for (const FName& ItemTID : Candidates)
{
    const FItemTableRow* Item = GetItemData(ItemTID);
    const int32 SpawnedCount = GetSpawnedCount(ItemTID);

    if (SpawnedCount < Item->MaxSpawnCount &&
        FMath::FRand() < Item->Probability)
    {
        return Item;
    }
}
```

이 구조는 코드에서 위치를 하드코딩하지 않고 데이터만 추가해 스폰 후보를 늘릴 수 있다는 장점이 있었다.

## DataTable만 수정하면 되지 않을까

DataTable은 스폰 위치와 회전값을 저장하는 데에는 충분했다. 그러나 `X=86580, Y=146570, Z=15380`이라는 값이 **게임 공간에서 올바른 자리인지**까지 알려주지는 못했다.

특히 아이템을 서랍이나 가구 안에 배치할 때는 좌표가 유효해도 다음과 같은 문제가 생길 수 있었다.

- 메시가 서랍 바닥을 뚫거나 공중에 떠 있음
- 회전 방향 때문에 아이템 일부가 가구 밖으로 노출됨
- 선택한 아이템의 크기와 배치 공간이 맞지 않음
- 좌표는 맞지만 다른 구조물에 가려 플레이어가 발견할 수 없음

더구나 실제 아이템은 여러 후보 중 무작위로 선택됐다. 잘못된 행이 매번 등장하는 것이 아니어서, 값을 수정한 뒤 PIE를 실행하는 방식만으로는 검증이 늦었다. DataTable 편집은 **데이터 입력**을 해결했지만 **공간 검증**은 해결하지 못한 셈이었다.

![스폰 데이터 편집 화면](/assets/img/the-deadline-spawn-editor.png)
_DataTable 행의 아이템 후보와 위치·회전값을 Details View에서 편집_

## Preview Actor로 데이터와 실제 공간을 연결

그래서 Editor Utility Widget에 Preview Actor 기능을 함께 넣었다. 프리뷰 버튼을 누르면 현재 입력한 위치와 회전값을 읽고, 활성 레벨의 에디터 월드에 확인용 액터를 생성했다. 프리뷰용 Static Mesh가 지정되어 있으면 액터에 적용하고, 뷰포트 카메라도 해당 위치로 이동시켰다.

![Preview Mesh 설정 화면](/assets/img/the-deadline-preview-mesh-setting.png)
_확인할 메시와 크기·회전값을 Preview Mesh 항목에서 지정_

이제 숫자를 고친 뒤 게임을 실행하고 무작위 스폰을 기다리는 대신, 같은 화면에서 다음 흐름으로 확인할 수 있었다.

`행 선택 → 좌표·회전·메시 수정 → Preview → 월드에서 배치 확인`

![바퀴 Preview Actor가 생성된 에디터 월드](/assets/img/the-deadline-preview-actor-wheel.png)
_지정한 바퀴 메시를 실제 좌표에 생성해 벽과 테이블 사이의 크기·높이·간섭 여부를 확인_

Preview Actor는 실제 배치 오브젝트가 아니라 DataTable 값을 검증하기 위한 임시 표현이었다. 런타임 스폰의 기준은 계속 DataTable 하나로 유지하고, 프리뷰가 맵 데이터와 섞이지 않도록 위젯을 닫을 때 생성했던 액터를 제거했다.

## 블루프린트에서 막힌 부분만 C++로 보완

행 추가와 수정은 편집기 기능으로 처리할 수 있었지만 DataTable의 행 삭제는 블루프린트에서 바로 사용할 수 없었다. 도구 전체를 C++로 다시 만들기보다 필요한 기능만 `BlueprintCallable` 함수로 노출했다.

```cpp
bool UGameUtils::RemoveTableRow(UDataTable* Table, FName RowName)
{
    if (!IsValid(Table) || !Table->GetRowMap().Contains(RowName))
        return false;

    Table->RemoveRow(RowName);
    Table->MarkPackageDirty();
    return true;
}
```

## 한계와 개선 방향

현재 도구는 Preview Actor를 직접 보고 배치 오류를 찾는 방식이라 판단 기준이 작업자에게 남아 있었다. 액터가 벽과 겹치거나 바닥에서 떠 있어도 자동으로 경고하지 않으며, 여러 스폰 지점을 한 번에 비교하기도 어려웠다. 값을 변경할 때마다 Preview 버튼을 다시 눌러야 하는 점도 반복 작업에서는 불편할 수 있었다.

다음 단계에서는 좌표나 메시가 변경되면 Preview Actor를 즉시 갱신하고, 모든 스폰 지점을 월드에 일괄 표시하는 방향으로 개선할 수 있다. 

여기에 충돌 검사를 추가해 구조물과 겹친 지점은 색상으로 구분하고, Row Name을 함께 표시하면 잘못된 데이터를 빠르게 찾을 수 있다. 최종적으로는 단순히 **보여주는 도구**에서 잘못된 배치를 먼저 알려주는 **검증 도구**로 확장하는 것이 목표다.

>이 도구의 핵심은 런타임 데이터를 새 형식으로 바꾸는 것이 아니었다. 기존 DataTable을 그대로 사용하면서 입력과 공간 확인만 하나의 작업 흐름으로 연결했다. 작업자는 게임을 매번 실행하지 않아도 됐고, 잘못된 좌표나 메시 선택도 에디터에서 바로 발견할 수 있었다.
