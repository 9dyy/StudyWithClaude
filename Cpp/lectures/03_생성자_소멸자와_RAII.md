# 3강 — 생성자·소멸자와 RAII

> **목표:** 생성자가 호출되는 순서(멤버 초기화 리스트 vs 본문), 소멸자가 호출되는 정확한 시점, 그리고 이 둘을 이용한 RAII 패턴이 왜 스마트 포인터로 이어지는지 설명할 수 있게 된다.
> **선수 지식:** 2강(캐스팅). 클래스/포인터 기본 문법.

---

## 1. 파일을 열어놓고 리턴해버린 함수

레벨 로딩 중에 설정 파일을 읽는 함수를 짰다.

```cpp
void LoadLevelConfig(const FString& Path)
{
    FILE* File = fopen(TCHAR_TO_ANSI(*Path), "r");

    if (File == nullptr)
    {
        return;   // 파일 못 열면 그냥 리턴
    }

    if (!ParseConfig(File))
    {
        return;   // 파싱 실패해도 그냥 리턴 — fclose(File) 까먹음!
    }

    fclose(File);
}
```

파싱이 실패하는 경로로 빠지면 `fclose(File)`을 아예 안 부르고 함수가 끝난다. 파일 핸들이 새는(leak) 거다. 함수에 `return`이 세 군데면 세 군데 다 `fclose`를 챙겨야 하는데, 사람이 손으로 관리하면 언젠가 하나는 까먹는다. `new`로 잡은 메모리, 열어놓은 소켓, 잠근 뮤텍스도 똑같은 문제를 겪는다 — "정리(cleanup)를 사람이 매번 챙겨야 한다"는 게 근본 원인이다.

이 문제를 C++이 언어 차원에서 푸는 방법이 있다. 바로 **생성자/소멸자를 이용해 "정리를 객체의 죽음에 자동으로 묶어버리는"** RAII 패턴이다.

---

## 2. 생성자 — 객체가 태어날 때 자동으로 실행되는 코드

생성자는 객체가 만들어질 때 딱 한 번, 자동으로 호출된다. 여기서 자주 헷갈리는 게 **멤버 초기화 리스트와 생성자 본문의 차이**다.

```cpp
class AWeapon : public AActor
{
public:
    // 멤버 초기화 리스트로 초기화
    AWeapon()
        : Damage(10.0f)
        , AmmoCount(30)
        , OwnerName(TEXT("Unknown"))
    {
        // 본문은 텅 비어있어도 됨
    }

private:
    float Damage;
    int32 AmmoCount;
    FString OwnerName;
};
```

```cpp
// 본문에서 대입하는 방식 — 겉보기엔 비슷해 보이지만 다르다
AWeapon()
{
    Damage = 10.0f;       // 이미 "기본 생성"된 Damage에 값을 대입
    AmmoCount = 30;
    OwnerName = TEXT("Unknown");
}
```

**차이는 "생성이냐 대입이냐"다.** 멤버 초기화 리스트는 멤버를 **그 값으로 바로 생성**한다. 본문에서 대입하는 방식은 멤버가 **먼저 기본값으로 생성된 다음, 대입 연산으로 값을 덮어쓴다** — 즉 한 단계를 더 거친다. `int32`, `float` 같은 기본 타입은 차이가 미미하지만, `FString`이나 다른 클래스 멤버라면 "기본 생성자 호출 + 대입 연산자 호출" 두 번 일하는 셈이라 비효율적이다. 게다가 **`const` 멤버나 참조(`&`) 멤버는 대입이 아예 불가능**해서 반드시 초기화 리스트를 써야 한다.

```cpp
class AWeapon : public AActor
{
public:
    AWeapon(const AActor* InOwner)
        : Owner(InOwner)   // const 멤버는 초기화 리스트로만 값을 줄 수 있음
    {
    }

private:
    const AActor* const Owner;   // 본문에서 대입 시도하면 컴파일 에러
};
```

**초기화 순서는 초기화 리스트에 적은 순서가 아니라, 클래스에 멤버를 선언한 순서로 결정된다.** 초기화 리스트에 `AmmoCount, Damage` 순으로 적어도 클래스 선언이 `Damage, AmmoCount` 순이면 `Damage`가 먼저 초기화된다. 순서를 헷갈려서 초기화 안 된 멤버를 참조하는 실수를 막으려면 **선언 순서 = 초기화 리스트 순서**로 맞추는 습관이 안전하다.

---

## 3. 암시적 생성자 — 아무것도 안 써도 컴파일러가 만들어주는 것들

클래스에 생성자를 하나도 안 쓰면 컴파일러가 **기본 생성자, 복사 생성자, 소멸자, 대입 연산자**를 알아서 만들어준다.

```cpp
class FInventorySlot
{
public:
    int32 ItemID;
    int32 Count;
    // 생성자를 하나도 안 썼다
};

FInventorySlot Slot;          // 암시적 기본 생성자 호출 (멤버는 초기화 안 될 수 있음!)
FInventorySlot Copy = Slot;   // 암시적 복사 생성자 — 멤버를 하나하나 그대로 복사
```

**함정은 암시적 기본 생성자가 기본 타입(`int32`, `float`, 포인터)까지 0으로 초기화해준다는 보장이 없다는 거다.** 클래스 멤버(예: `FString`)는 자기 기본 생성자가 있어서 확실히 초기화되지만, `int32 ItemID`는 쓰레기 값을 가진 채로 남을 수 있다. `FInventorySlot Slot;` 만들고 바로 `ItemID`를 읽으면 미정의 값이 나올 수 있다는 뜻이다. 그래서 멤버 변수는 클래스 안에서 `int32 ItemID = 0;`처럼 **기본값을 직접 지정**해두는 습관이 안전하다.

한 가지 더: 클래스에 **포인터 멤버**가 있고 그 포인터가 가리키는 자원을 소유하고 있다면, 암시적 복사 생성자는 **포인터 값만 그대로 복사**한다(얕은 복사). 두 객체가 같은 메모리를 가리키게 되고, 하나가 소멸자에서 그 메모리를 해제하면 다른 하나는 이미 해제된 메모리를 가리키는 댕글링 포인터가 된다.

---

## 4. 소멸자 — 객체가 죽을 때 자동으로 실행되는 코드

소멸자는 객체의 생명이 끝나는 시점에 자동으로, 반드시 호출된다. **정확히 언제 끝나는지**가 핵심이다.

| 객체가 사는 곳 | 소멸자 호출 시점 |
|------|------|
| 지역 변수(스택) | 그 변수가 선언된 스코프(`{ }`)를 벗어날 때 |
| `new`로 만든 힙 객체 | `delete`를 명시적으로 호출할 때 |
| 스마트 포인터가 감싼 힙 객체 | 마지막 소유자가 사라질 때 (자동) |
| 전역/static 객체 | 프로그램 종료 시 |

```cpp
class FScopedLog
{
public:
    FScopedLog(const FString& InName) : Name(InName)
    {
        UE_LOG(LogTemp, Log, TEXT("[%s] 시작"), *Name);
    }

    ~FScopedLog()
    {
        UE_LOG(LogTemp, Log, TEXT("[%s] 종료"), *Name);   // 스코프 벗어나면 자동 호출
    }

private:
    FString Name;
};

void LoadLevel()
{
    FScopedLog Log(TEXT("LoadLevel"));   // 여기서 생성자 → "시작" 로그

    if (!CheckPrerequisites())
    {
        return;   // 여기서 스코프를 벗어나므로 Log의 소멸자가 자동 호출됨 → "종료" 로그
    }

    DoActualLoading();
}   // 정상 경로도 여기서 소멸자 호출 → "종료" 로그
```

`return`이 몇 군데 있든, 예외가 던져지든 상관없이 **스코프를 벗어나는 모든 경로에서 소멸자는 반드시 호출된다.** 이게 도입 문제의 해결 실마리다 — "정리 코드를 매 `return`마다 손으로 챙기지 말고, 스코프를 벗어날 때 자동으로 실행되는 소멸자에 맡기자."

---

## 5. RAII — 자원 해제를 소멸자에 묶어버리기

**RAII(Resource Acquisition Is Initialization)** 는 이름 그대로 "자원 획득은 초기화다" — **생성자에서 자원을 획득하고, 소멸자에서 반드시 해제**하도록 클래스로 감싸는 패턴이다. 도입의 파일 핸들 문제를 RAII로 다시 짜보자.

```cpp
class FScopedFile
{
public:
    FScopedFile(const FString& Path)
    {
        File = fopen(TCHAR_TO_ANSI(*Path), "r");   // 생성자에서 자원 획득
    }

    ~FScopedFile()
    {
        if (File)
        {
            fclose(File);   // 소멸자에서 반드시 해제 — 손으로 안 챙겨도 됨
        }
    }

    FILE* Get() const { return File; }

private:
    FILE* File = nullptr;
};

void LoadLevelConfig(const FString& Path)
{
    FScopedFile ScopedFile(Path);

    if (ScopedFile.Get() == nullptr)
    {
        return;   // 여기서 리턴해도 ScopedFile 소멸자가 fclose를 대신 해줌
    }

    if (!ParseConfig(ScopedFile.Get()))
    {
        return;   // 여기서도 마찬가지
    }
}   // 정상 종료 경로도 마찬가지
```

`fclose`를 어디에도 직접 안 썼는데, 함수가 어떤 경로로 빠져나가든 `FScopedFile`의 소멸자가 자동으로 실행되면서 파일을 닫는다. **자원 해제를 "사람이 기억해야 할 일"에서 "언어가 보장해주는 일"로 옮긴 것**이 RAII의 핵심이다. 언리얼의 `FScopeLock`이 정확히 이 패턴이다 — 생성자에서 락을 잡고, 소멸자에서 락을 푼다(9강에서 다시 볼 락 계열과 연결된다).

```cpp
void UpdateSharedData()
{
    FScopeLock Lock(&CriticalSection);   // 생성자에서 Lock() 호출
    SharedData++;
}   // 함수 끝나면 소멸자에서 Unlock() 자동 호출 — Unlock 깜빡할 일이 없음
```

---

## 6. 스마트 포인터로 이어지는 이유

RAII의 자연스러운 다음 단계가 **스마트 포인터**다. `new`로 잡은 메모리도 결국 "획득한 자원"이니, 똑같이 소멸자에 `delete`를 묶어버리면 된다.

```cpp
class FSimpleOwningPtr
{
public:
    FSimpleOwningPtr(AActor* InPtr) : Ptr(InPtr) {}
    ~FSimpleOwningPtr() { delete Ptr; }   // 소멸자에서 자동 delete

    AActor* Get() const { return Ptr; }

private:
    AActor* Ptr;
};
```

이게 사실상 `TUniquePtr`/`std::unique_ptr`의 원리 그대로다. `TSharedPtr`는 여기에 "몇 명이 같이 소유하고 있는지" 세는 참조 카운트를 얹어서, **마지막 소유자의 소멸자가 실행될 때만** `delete`하도록 확장한 것뿐이다. `new`/`delete`를 손으로 직접 짝 맞추던 걸 RAII로 자동화하면, "delete 깜빡함", "이미 delete한 걸 또 delete함" 같은 실수가 원천적으로 줄어든다.

```cpp
void SpawnAndUseWeapon()
{
    TUniquePtr<FWeaponData> WeaponData = MakeUnique<FWeaponData>();
    WeaponData->Damage = 50.0f;

    if (!IsValidLoadout())
    {
        return;   // WeaponData의 소멸자가 자동으로 메모리 해제 — delete 안 써도 됨
    }

    ApplyWeaponData(WeaponData.Get());
}   // 여기서도 자동 해제
```

---

## 7. 코드로 체감해보기 — 생성/소멸 순서 확인

멤버 초기화 순서와 소멸 순서가 실제로 어떻게 도는지 확인해보자.

```cpp
class FInner
{
public:
    FInner(const FString& InName) : Name(InName)
    {
        UE_LOG(LogTemp, Log, TEXT("Inner(%s) 생성"), *Name);
    }
    ~FInner()
    {
        UE_LOG(LogTemp, Log, TEXT("Inner(%s) 소멸"), *Name);
    }
private:
    FString Name;
};

class FOuter
{
public:
    FOuter()
        : A(TEXT("A"))   // 선언 순서가 A, B라서 A가 먼저 생성됨
        , B(TEXT("B"))
    {
        UE_LOG(LogTemp, Log, TEXT("Outer 생성"));
    }
    ~FOuter()
    {
        UE_LOG(LogTemp, Log, TEXT("Outer 소멸"));
    }
private:
    FInner A;
    FInner B;
};

// 출력 순서:
// Inner(A) 생성
// Inner(B) 생성
// Outer 생성
// Outer 소멸
// Inner(B) 소멸   ← 생성의 역순으로 소멸!
// Inner(A) 소멸
```

**멤버는 선언 순서대로 생성되고, 소멸은 그 정반대 순서로 일어난다.** 나중에 생긴 게 먼저 죽는 거다(스택처럼 LIFO — 이 구조는 5강에서 다시 나온다). 이 순서를 알아야 "A가 B에 의존하는데 B가 먼저 소멸돼버려서 문제가 생기는" 종류의 버그를 예상하고 피할 수 있다.

---

## 8. 흔한 오해

### 오해 1: "멤버 초기화 리스트나 본문에서 대입하나 결국 똑같다"

기본 타입은 거의 차이 없지만, 클래스 멤버는 "기본 생성 + 대입" 두 단계를 거치는 본문 대입이 더 비효율적이다. 게다가 `const`/참조 멤버는 본문 대입 자체가 불가능해서 초기화 리스트가 필수다.

### 오해 2: "생성자를 안 쓰면 멤버는 자동으로 0으로 초기화된다"

아니다. 클래스 타입 멤버는 자기 기본 생성자로 초기화되지만, `int32`/`float`/포인터 같은 기본 타입은 암시적 기본 생성자가 값을 보장해주지 않는다. 클래스 안에서 `int32 Count = 0;`처럼 직접 기본값을 줘야 안전하다.

### 오해 3: "포인터 멤버가 있는 클래스는 복사해도 알아서 잘 복사된다"

아니다. 컴파일러가 만들어주는 암시적 복사 생성자는 포인터 **값만** 그대로 복사한다(얕은 복사). 그 포인터가 자원을 소유하고 있다면 두 객체가 같은 자원을 가리키게 되고, 이중 해제(double free)나 댕글링 포인터로 이어진다.

### 오해 4: "소멸자는 delete를 명시적으로 불러야만 실행된다"

지역 변수(스택 객체)는 스코프를 벗어나는 순간 자동으로 소멸자가 실행된다. `delete`가 필요한 건 `new`로 만든 힙 객체뿐이다. RAII가 성립하는 이유가 바로 이 "스코프 종료 = 자동 소멸자 호출" 보장 때문이다.

### 오해 5: "예외가 발생하면 소멸자가 안 불릴 수도 있다"

아니다. 예외로 함수를 빠져나가는 경우(스택 언와인딩)에도, 그 스코프 안에 있던 지역 객체들의 소멸자는 정상적으로 호출된다. RAII가 예외 안전성 확보에도 쓰이는 이유다.

---

## 9. 면접에서 나오면

**Q1. 멤버 초기화 리스트와 생성자 본문에서 대입하는 것의 차이는 뭔가요?**
→ "초기화 리스트는 멤버를 해당 값으로 바로 생성하고, 본문 대입은 먼저 기본 생성한 뒤 대입 연산자로 덮어씁니다. 클래스 타입 멤버는 그만큼 한 단계를 더 거쳐 비효율적이고, const나 참조 멤버는 대입 자체가 불가능해서 반드시 초기화 리스트를 써야 합니다."

**Q2. 소멸자는 정확히 언제 호출되나요?**
→ "지역 변수는 선언된 스코프를 벗어날 때, 힙에 new로 만든 객체는 delete를 호출할 때, 스마트 포인터로 감싼 객체는 마지막 소유자가 사라질 때 자동으로 호출됩니다. return이나 예외로 스코프를 빠져나가는 경우에도 그 스코프의 지역 객체 소멸자는 반드시 실행됩니다."

**Q3. RAII가 뭔가요?**
→ "자원 획득을 객체의 생성자에, 자원 해제를 소멸자에 묶어서 관리하는 패턴입니다. 객체가 스코프를 벗어나면 소멸자가 자동으로 호출되는 걸 이용해서, 파일 핸들이나 락, 메모리 같은 자원을 사람이 매 return마다 직접 해제할 필요 없이 언어 차원에서 보장받습니다."

**Q4. 클래스에 포인터 멤버가 있을 때 기본으로 생성되는 복사 생성자를 그대로 써도 되나요?**
→ "그 포인터가 자원을 소유하고 있다면 안 됩니다. 암시적 복사 생성자는 포인터 값만 그대로 복사하는 얕은 복사라서, 두 객체가 같은 메모리를 가리키게 되고 한쪽이 소멸자에서 해제하면 다른 쪽은 댕글링 포인터가 됩니다. 이런 경우엔 복사 생성자를 직접 정의하거나 스마트 포인터로 소유권을 명확히 해야 합니다."

**Q5. RAII가 스마트 포인터로 어떻게 이어지나요?**
→ "new로 획득한 메모리도 결국 관리해야 할 자원이라, 그 해제를 소멸자에 묶으면 스마트 포인터가 됩니다. unique_ptr은 소유자가 하나뿐이라 소멸자에서 바로 delete하고, shared_ptr은 참조 카운트를 둬서 마지막 소유자의 소멸자가 실행될 때만 delete하도록 확장한 겁니다."

**Q6. 멤버 초기화 순서를 초기화 리스트에 적은 순서와 다르게 쓰면 어떻게 되나요?**
→ "실제 초기화 순서는 초기화 리스트에 적은 순서가 아니라 클래스에 멤버를 선언한 순서를 따릅니다. 리스트 순서와 선언 순서가 다르면 헷갈릴 뿐 아니라, 한 멤버가 다른 멤버를 참조해서 초기화하는 경우 아직 초기화 안 된 멤버를 참조하는 버그로 이어질 수 있어서 선언 순서와 리스트 순서를 맞춰 쓰는 게 안전합니다."

**Q7. 함수 안에서 예외가 발생해도 RAII로 관리되는 자원이 제대로 해제되나요?**
→ "네, 됩니다. 예외로 스택 언와인딩이 일어나도 그 스코프에 있던 지역 객체들의 소멸자는 정상적으로 호출됩니다. 그래서 락이나 파일 핸들을 RAII 클래스로 감싸두면 예외 상황에서도 자원이 새지 않는다는 걸 보장할 수 있습니다."

면접관이 그 다음에 던지기 좋은 질문: *"그럼 shared_ptr의 참조 카운트 증가/감소는 스레드 안전한가요?"* → OS 09강(Atomic과 Lock-Free)에서 이미 다룬 내용과 연결된다. 참조 카운트 자체는 atomic으로 안전하지만, 가리키는 객체의 동시 접근은 별개 문제라는 점을 짚어주면 된다.

---

## 10. 셀프 체크

- [ ] 멤버 초기화 리스트와 본문 대입의 차이, 그리고 const/참조 멤버가 초기화 리스트를 강제하는 이유를 안다.
- [ ] 암시적 기본 생성자가 기본 타입 멤버를 0으로 보장해주지 않는다는 걸 안다.
- [ ] 포인터 멤버를 가진 클래스에서 암시적 복사 생성자가 왜 위험할 수 있는지(얕은 복사) 설명할 수 있다.
- [ ] 소멸자가 호출되는 정확한 시점(스코프 종료, delete, 스마트 포인터 소유자 소멸)을 구분한다.
- [ ] RAII가 뭔지, 그리고 그게 왜 스마트 포인터의 원리로 이어지는지 설명할 수 있다.

---

다음 강의: **4강 — 배열 vs LinkedList**
"삽입/삭제/탐색 시간복잡도만 외우지 말고, 캐시 지역성 때문에 TArray가 TLinkedList보다 실제로 왜 더 빠른 경우가 많은지"를 다룬다.
