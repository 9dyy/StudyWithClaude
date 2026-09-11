# 5강 — Stack과 Queue

> **목표:** LIFO/FIFO라는 정의를 외우는 걸 넘어서, Undo 기능과 커맨드 처리 큐 같은 실제 게임 로직에 왜 이 구조가 자연스럽게 들어맞는지 설명할 수 있게 된다.
> **선수 지식:** 4강(배열 vs LinkedList).

---

## 1. Undo 기능을 배열 인덱스로 짜다가 꼬인 코드

레벨 에디터 툴에 "되돌리기(Ctrl+Z)" 기능을 넣어야 한다. 처음엔 그냥 배열에 액션들을 쌓고 인덱스로 관리하려고 했다.

```cpp
TArray<FEditAction> ActionHistory;
int32 CurrentIndex = -1;

void RecordAction(const FEditAction& Action)
{
    ActionHistory.Add(Action);
    CurrentIndex = ActionHistory.Num() - 1;
}

void Undo()
{
    if (CurrentIndex < 0) return;
    ActionHistory[CurrentIndex].Revert();
    CurrentIndex--;   // 이 인덱스 관리가 점점 꼬인다 — Redo까지 넣으면 더 복잡해짐
}
```

되돌리기 한 번은 되는데, "되돌리기 하다가 새 액션을 기록하면 그 뒤에 있던 히스토리를 어떻게 지울지", "Redo는 어떻게 구현할지"를 인덱스만으로 관리하려니 조건문이 계속 늘어난다. 사실 이 문제는 **"가장 최근에 한 일을 가장 먼저 되돌린다"** — 정확히 **LIFO(Last In, First Out)** 구조가 필요한 상황이다. 자료구조 이름을 몰라도 "마지막에 쌓은 것부터 꺼낸다"는 동작 자체가 필요했던 거고, 그게 바로 **Stack**이다.

---

## 2. Stack — LIFO, "마지막에 넣은 걸 먼저 꺼낸다"

```
Push(A) → Push(B) → Push(C)          Pop() → Pop()
┌───┐      ┌───┐      ┌───┐          ┌───┐      ┌───┐
│ A │      │ A │      │ A │          │ A │      │   │
└───┘      │ B │      │ B │          └───┘      └───┘
           └───┘      │ C │  ← 여기서 Pop → C가 먼저 나감
                       └───┘
```

Stack은 **한쪽 끝에서만 넣고(`Push`) 빼는(`Pop`)** 자료구조다. 접시를 쌓아놓고 위에서부터 하나씩 빼는 걸 상상하면 된다. 언리얼/표준 C++ 관점에서 Stack은 보통 배열(`TArray`)로 구현한다 — 4강에서 배웠듯 맨 뒤에 추가/제거는 `TArray`가 O(1)이라 캐시 지역성까지 챙긴 진짜 스택 구현체로 딱 맞는다.

```cpp
TArray<FEditAction> UndoStack;
TArray<FEditAction> RedoStack;

void RecordAction(const FEditAction& Action)
{
    UndoStack.Push(Action);
    RedoStack.Empty();   // 새 액션을 하면 Redo 히스토리는 무효화
}

void Undo()
{
    if (UndoStack.Num() == 0) return;

    FEditAction Action = UndoStack.Pop();   // 가장 최근 액션을 꺼냄 (LIFO)
    Action.Revert();
    RedoStack.Push(Action);
}

void Redo()
{
    if (RedoStack.Num() == 0) return;

    FEditAction Action = RedoStack.Pop();
    Action.Reapply();
    UndoStack.Push(Action);
}
```

인덱스로 손수 관리하던 코드가 Stack 두 개(Undo/Redo)로 정리됐다. **"가장 최근 걸 먼저 되돌린다"는 요구사항이 정확히 LIFO라는 걸 알아채는 게 핵심**이지, `Push`/`Pop`이라는 함수 이름을 아는 게 핵심이 아니다.

**함수 호출 스택도 정확히 이 구조다.** 함수 A가 B를 부르고 B가 C를 부르면, C가 끝나야 B로 돌아가고 B가 끝나야 A로 돌아간다 — 가장 나중에 호출된 게 가장 먼저 리턴되는 LIFO. 3강에서 본 "멤버 소멸이 생성의 역순으로 일어나는 것"도 같은 원리다.

---

## 3. Queue — FIFO, "먼저 넣은 걸 먼저 꺼낸다"

```
Enqueue(A) → Enqueue(B) → Enqueue(C)      Dequeue()
┌───┬───┬───┐                              ┌───┬───┐
│ A │   │   │                              │ B │ C │  ← A가 먼저 나감
├───┼───┼───┤                              └───┴───┘
│ A │ B │   │
├───┼───┼───┤
│ A │ B │ C │
└───┴───┴───┘
```

Queue는 **한쪽 끝에서 넣고(`Enqueue`) 반대쪽 끝에서 꺼내는(`Dequeue`)** 구조다. 놀이기구 줄서기 그대로 — 먼저 줄 선 사람이 먼저 탄다. 언리얼에서는 `TQueue`(스레드 안전 옵션 지원)를 쓴다.

**언제 쓰나 — BFS(너비 우선 탐색) 커맨드/이동 처리:** RTS 게임에서 유닛이 "이동 → 공격 → 방어" 커맨드를 순서대로 예약해두고 하나씩 실행하는 로직을 생각해보자. 명령을 내린 순서 그대로 실행돼야 하니 FIFO가 딱 맞는다.

```cpp
TQueue<FUnitCommand> CommandQueue;

void AUnit::IssueCommand(const FUnitCommand& Command)
{
    CommandQueue.Enqueue(Command);   // 명령을 큐 끝에 추가
}

void AUnit::Tick(float DeltaTime)
{
    if (bIsExecutingCommand) return;

    FUnitCommand NextCommand;
    if (CommandQueue.Dequeue(NextCommand))   // 가장 먼저 내려진 명령부터 실행
    {
        ExecuteCommand(NextCommand);
    }
}
```

BFS 알고리즘 자체도 Queue가 본질이다. 길찾기에서 "현재 노드에서 갈 수 있는 이웃들을 큐에 넣고, 큐에서 하나씩 꺼내 그 이웃들을 또 큐에 넣는" 방식으로 **가까운 노드부터 순서대로** 탐색한다.

```cpp
TQueue<FGridCell> Frontier;
TSet<FGridCell> Visited;

Frontier.Enqueue(StartCell);
Visited.Add(StartCell);

FGridCell Current;
while (Frontier.Dequeue(Current))
{
    if (Current == GoalCell) break;

    for (const FGridCell& Neighbor : GetNeighbors(Current))
    {
        if (!Visited.Contains(Neighbor))
        {
            Visited.Add(Neighbor);
            Frontier.Enqueue(Neighbor);   // 가까운 순서를 지키기 위해 큐 끝에 추가
        }
    }
}
```

만약 여기서 Queue 대신 Stack을 썼다면 **DFS(깊이 우선 탐색)**가 된다 — 가장 최근에 넣은 이웃을 먼저 파고들기 때문이다. **"BFS는 Queue, DFS는 Stack"이라는 대응 관계**는 자료구조 선택이 알고리즘의 성격 자체를 결정한다는 걸 보여주는 좋은 예다.

---

## 4. 내부적으로 뭘로 구현하나 — 배열이냐 LinkedList냐

Stack은 대부분 배열(`TArray`)로 구현하는 게 정석이다. `Push`/`Pop`이 전부 "끝에서" 일어나서 4강에서 본 배열의 강점(캐시 지역성, 끝에 추가/제거 O(1))을 그대로 가져간다.

Queue는 살짝 다르다. "앞에서 빼고 뒤에서 넣는" 구조라, **배열을 그대로 쓰면 앞에서 빼는 연산이 O(n)이 된다**(4강에서 본 "중간/앞 삽입·삭제는 배열이 O(n)"과 같은 문제). 그래서 실전 Queue 구현은 보통 다음 중 하나다.

```
방법 1: 원형 버퍼(Circular Buffer) — 배열이지만 앞/뒤 인덱스를 순환시켜서
        "앞에서 빼기"도 사실상 인덱스만 옮기는 거라 O(1)로 만든다.

┌───┬───┬───┬───┬───┐
│ C │ D │   │   │ A │B│   ← Head가 뒤쪽으로 순환해서 앞에서 뺀 자리를 재활용
└───┴───┴───┴───┴───┘
      ↑Tail          ↑Head (순환)
```

`TQueue`는 내부적으로 연결 리스트 기반 구조를 쓰지만, 원형 버퍼 방식도 실무에서 자주 쓰인다. **핵심은 "Queue는 앞/뒤 양쪽에서 O(1)로 넣고 뺄 수 있어야 진짜 쓸모가 있다"**는 거고, 그걸 어떤 내부 구조로 구현하느냐는 배열이냐 LinkedList냐를 4강의 트레이드오프로 다시 판단해야 하는 문제다.

---

## 5. 코드로 체감해보기 — 스킬 콤보 시스템

Stack이 실전 게임 로직에 자연스럽게 붙는 또 다른 예: 콤보 입력을 순서대로 검증하는 시스템.

```cpp
class FComboValidator
{
public:
    void PushInput(EComboInput Input)
    {
        InputStack.Push(Input);
        if (InputStack.Num() > MaxComboLength)
        {
            InputStack.RemoveAt(0);   // 오래된 입력 버림 (이 연산 자체는 O(n)이지만 길이가 짧아 무시 가능)
        }
    }

    bool CheckCombo(const TArray<EComboInput>& ComboPattern) const
    {
        if (InputStack.Num() < ComboPattern.Num()) return false;

        int32 Offset = InputStack.Num() - ComboPattern.Num();
        for (int32 i = 0; i < ComboPattern.Num(); ++i)
        {
            if (InputStack[Offset + i] != ComboPattern[i]) return false;
        }
        return true;
    }

private:
    TArray<EComboInput> InputStack;
    static constexpr int32 MaxComboLength = 5;
}
```

---

## 6. 흔한 오해

### 오해 1: "Stack과 Queue는 완전히 새로운 자료구조라 따로 구현해야 한다"

대부분 배열이나 LinkedList **위에 인터페이스만 씌운 것**이다. `TArray`에 `Push`/`Pop`만 있으면 Stack이고, 앞뒤 양쪽을 O(1)로 다룰 수 있게 감싸면 Queue다. "Push/Pop 순서 제약을 지키는 자료구조"라는 개념이지 완전히 다른 내부 구조가 아니다.

### 오해 2: "Queue를 배열로 구현해도 Stack이랑 성능이 비슷하다"

아니다. 배열을 그대로 써서 앞에서 원소를 빼면(`RemoveAt(0)`) 나머지 원소를 전부 앞으로 당겨야 해서 O(n)이다. Stack처럼 끝에서만 넣고 빼는 구조가 아니면 원형 버퍼 등 다른 내부 구현이 필요하다.

### 오해 3: "BFS와 DFS는 알고리즘 로직 자체가 다르다"

핵심 로직(이웃 탐색, 방문 체크)은 거의 동일하고, **탐색 순서를 관리하는 자료구조가 Queue냐 Stack이냐**만 다르다. 이 차이 하나로 "가까운 것부터"(BFS) vs "한쪽으로 깊게 파고들기"(DFS)라는 완전히 다른 탐색 패턴이 나온다.

### 오해 4: "함수 호출 스택은 그냥 비유일 뿐, 진짜 Stack 자료구조는 아니다"

아니다. 실제로 CPU가 함수 호출/리턴을 관리하는 데 쓰는 콜 스택은 진짜 LIFO 동작을 하는 메모리 영역이다(1강의 메모리 영역 논의에서 나온 "스택"이 바로 이거다). 지역 변수, 리턴 주소, 함수 인자가 이 위에 쌓였다 빠진다.

### 오해 5: "Undo/Redo는 Stack 하나로 충분하다"

Undo만이면 Stack 하나로 되지만, Redo까지 지원하려면 **Undo 스택과 Redo 스택 두 개**가 필요하다. Undo할 때 꺼낸 액션을 Redo 스택에 다시 쌓아두고, 새 액션을 기록하면 Redo 스택을 비우는 처리가 빠지면 Redo가 어긋난다.

---

## 7. 면접에서 나오면

**Q1. Stack과 Queue의 차이는 뭔가요?**
→ "Stack은 LIFO, 즉 가장 나중에 넣은 걸 가장 먼저 꺼내는 구조고, Queue는 FIFO, 가장 먼저 넣은 걸 가장 먼저 꺼내는 구조입니다. Stack은 Undo 기능처럼 최근 작업을 먼저 되돌려야 할 때, Queue는 커맨드 처리처럼 들어온 순서를 지켜야 할 때 씁니다."

**Q2. Undo 기능을 구현한다면 어떤 자료구조를 쓰시겠어요?**
→ "Stack을 쓰겠습니다. 가장 최근에 한 작업을 가장 먼저 되돌려야 하니 LIFO가 정확히 맞습니다. Redo까지 지원하려면 Undo 스택에서 꺼낸 작업을 Redo 스택에 쌓아두는 방식으로 스택 두 개를 같이 씁니다."

**Q3. BFS에서 왜 Queue를 쓰나요? Stack을 쓰면 어떻게 되나요?**
→ "BFS는 가까운 노드부터 순서대로 탐색해야 하는데, Queue는 먼저 넣은 이웃을 먼저 꺼내서 이 순서를 그대로 보장합니다. 만약 Stack을 쓰면 가장 최근에 넣은 이웃을 먼저 파고들게 되어서 BFS가 아니라 DFS가 됩니다."

**Q4. Stack은 보통 어떤 자료구조로 구현하나요?**
→ "배열로 구현하는 게 일반적입니다. Push와 Pop이 전부 배열의 끝에서 일어나기 때문에 O(1)이고, 배열의 캐시 지역성 이점도 그대로 가져갈 수 있습니다."

**Q5. Queue를 배열로 그대로 구현하면 어떤 문제가 생기나요?**
→ "앞에서 원소를 빼는 연산이 O(n)이 됩니다. 배열에서 맨 앞 원소를 지우면 나머지 원소를 전부 한 칸씩 당겨야 하기 때문입니다. 그래서 실전에서는 원형 버퍼나 연결 리스트 기반으로 구현해서 앞뒤 양쪽 연산을 O(1)로 만듭니다."

**Q6. 함수 호출과 Stack 자료구조가 무슨 관계가 있나요?**
→ "CPU가 함수 호출을 처리할 때 콜 스택이라는 메모리 영역에 지역 변수와 리턴 주소를 쌓습니다. 함수를 호출하면 그 정보가 스택에 쌓이고, 함수가 리턴하면 가장 최근에 쌓인 것부터 빠지는 LIFO 구조라서 Stack 자료구조와 원리가 같습니다."

**Q7. RTS 유닛의 커맨드 큐(이동, 공격, 대기를 순서대로 예약)를 구현한다면 어떤 자료구조가 맞을까요?**
→ "Queue가 맞습니다. 플레이어가 명령을 내린 순서대로 실행돼야 하기 때문에 FIFO 구조가 자연스럽습니다. 만약 Stack을 쓰면 가장 나중에 내린 명령이 먼저 실행돼버려서 플레이어가 의도한 순서와 어긋납니다."

면접관이 그 다음에 던지기 좋은 질문: *"그럼 우선순위가 있는 커맨드(급한 명령을 먼저 처리해야 하는 경우)는 어떻게 처리하나요?"* → 6강에서 다룰 트리 기반 자료구조(우선순위 큐, 힙)로 이어지는 질문이다.

---

## 8. 셀프 체크

- [ ] LIFO와 FIFO를 각각 정의하고 예시(Undo, 커맨드 큐)와 연결해서 설명할 수 있다.
- [ ] BFS는 Queue, DFS는 Stack이라는 대응 관계와 그 이유를 안다.
- [ ] Stack은 왜 대부분 배열로 구현하는 게 유리한지(끝에서만 연산, 캐시 지역성) 설명할 수 있다.
- [ ] Queue를 배열로 그대로 구현하면 왜 문제가 생기는지, 원형 버퍼가 왜 필요한지 안다.
- [ ] 함수 호출 스택이 실제 Stack 자료구조와 같은 원리로 동작한다는 걸 안다.

---

다음 강의: **6강 — 해시맵과 트리 맛보기**
"TMap은 내부적으로 어떻게 동작하길래 O(1)에 가까운 조회가 가능한지, 그리고 배열 대신 맵을 언제 써야 하는지"를 다룬다.
