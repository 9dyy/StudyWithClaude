# 8강 — Observer Pattern

> **목표:** HP 같은 상태가 바뀔 때 UI가 자동으로 갱신되는 구조를 "느슨한 결합" 개념으로 설명하고, 언리얼 Delegate/Event로 이걸 구현할 수 있게 된다.
> **선수 지식:** 7강(Object Pooling). 클래스 간 참조 개념.

---

## 1. HP가 바뀌는 곳마다 UI 갱신 코드를 심어야 했던 코드

캐릭터가 데미지를 받으면 HP바를 갱신해야 한다. 처음엔 데미지를 주는 곳마다 직접 UI를 갱신했다.

```cpp
void ACharacter::TakeDamage(float Damage)
{
    HP -= Damage;
    HPBarWidget->UpdateHP(HP);          // 여기서도
}

void ACharacter::ApplyPoisonTick(float Damage)
{
    HP -= Damage;
    HPBarWidget->UpdateHP(HP);          // 여기서도
}

void ACharacter::Heal(float Amount)
{
    HP += Amount;
    HPBarWidget->UpdateHP(HP);          // 여기서도... 반복
}
```

HP를 바꾸는 곳이 늘어날 때마다 `HPBarWidget->UpdateHP(HP)`를 매번 손으로 추가해야 한다. 하나라도 빠뜨리면 UI가 안 갱신되는 버그가 생긴다. 게다가 나중에 "HP가 바뀌면 미니맵의 체력 아이콘도 바꿔야 한다"는 요구사항이 오면, **모든 데미지/힐 함수를 다시 찾아다니며 코드를 추가**해야 한다. `ACharacter`가 `HPBarWidget`, `MinimapIcon` 등 자기와 상관없어야 할 UI 클래스들을 전부 알아야 하는 것도 이상하다 — 캐릭터 로직과 UI 로직이 **꽉 묶여(강한 결합)** 있는 거다.

---

## 2. Observer Pattern — "바뀌었다고 알리기만 하면, 관심 있는 쪽이 알아서 반응한다"

발상을 뒤집는다. `ACharacter`는 "HP가 바뀌었다"는 사실만 **알리고(Notify)**, HP바든 미니맵이든 **관심 있는 쪽(Observer)이 알아서 그 알림을 구독(Subscribe)해서 반응**하게 만든다.

```
[일렬로 직접 호출하던 방식]                [Observer Pattern]

Character                                  Character
   │  UpdateHP()                              │  "HP 바뀜!" 이벤트 발행(Broadcast)
   ├──→ HPBarWidget                           │
   ├──→ MinimapIcon                           ↓  (누가 듣고 있는지 Character는 모른다)
   └──→ (새 UI 생길 때마다 여기 추가)      ┌───┴───┬────────┐
                                          HPBarWidget MinimapIcon  (새 Observer)
                                          (구독)      (구독)        (그냥 구독만 추가)
```

**핵심은 방향이 뒤집힌다는 거다.** 기존 방식은 `Character`가 `HPBarWidget`을 직접 알고 호출했다(강한 결합). Observer 방식은 `Character`는 "이벤트가 발생했다"고만 외치고, 누가 듣고 있는지, 몇 명이 듣고 있는지 **전혀 몰라도 된다.** 이게 **느슨한 결합(Loose Coupling)** 이다 — 두 클래스가 서로의 존재를 몰라도 협력할 수 있는 구조.

---

## 3. 언리얼 Delegate로 구현하기

언리얼은 Observer Pattern을 언어 차원에서 지원하는 **Delegate/Event** 시스템을 제공한다.

```cpp
// Character.h
DECLARE_MULTICAST_DELEGATE_OneParam(FOnHPChanged, float /*NewHP*/);

class ACharacter : public AActor
{
public:
    FOnHPChanged OnHPChanged;   // "HP가 바뀌면 알려줄게" 라는 이벤트 창구

    void TakeDamage(float Damage)
    {
        HP -= Damage;
        OnHPChanged.Broadcast(HP);   // 알림만 발행 — 누가 듣는지 전혀 모른다
    }

    void Heal(float Amount)
    {
        HP += Amount;
        OnHPChanged.Broadcast(HP);   // 데미지든 힐이든 "HP가 바뀌었다"는 알림 하나로 통일
    }

private:
    float HP = 100.0f;
};
```

```cpp
// HPBarWidget.h — 구독하는 쪽
class UHPBarWidget : public UUserWidget
{
public:
    void BindToCharacter(ACharacter* Character)
    {
        Character->OnHPChanged.AddUObject(this, &UHPBarWidget::HandleHPChanged);   // 구독
    }

private:
    void HandleHPChanged(float NewHP)
    {
        ProgressBar->SetPercent(NewHP / MaxHP);
    }
};
```

```cpp
// MinimapIcon.h — 나중에 추가된 새 Observer. Character 쪽 코드는 한 줄도 안 바뀐다.
class UMinimapIcon : public UUserWidget
{
public:
    void BindToCharacter(ACharacter* Character)
    {
        Character->OnHPChanged.AddUObject(this, &UMinimapIcon::HandleHPChanged);   // 그냥 구독만 추가
    }

private:
    void HandleHPChanged(float NewHP)
    {
        UpdateHealthIconColor(NewHP);
    }
};
```

`TakeDamage`, `Heal`, `ApplyPoisonTick` 어디에도 `HPBarWidget`이나 `MinimapIcon`이라는 이름이 등장하지 않는다. **새 UI가 추가돼도 `ACharacter` 코드는 한 줄도 안 바뀐다** — 그저 새 클래스가 `OnHPChanged`를 구독하기만 하면 된다. 도입 문제의 "빠뜨리면 버그", "새 요구사항마다 캐릭터 코드 수정"이 둘 다 사라진다.

---

## 4. Multicast Delegate vs Single Delegate — 몇 명이 들을 수 있나

언리얼 Delegate는 크게 두 종류다.

| 종류 | 구독자 수 | 쓰는 곳 |
|------|------|------|
| `DECLARE_DELEGATE_...` (싱글) | 딱 1명만 바인딩 가능 | "이 요청의 응답은 딱 한 곳에서만 처리한다"가 명확할 때 |
| `DECLARE_MULTICAST_DELEGATE_...` | 여러 명이 동시에 구독 가능 | Observer Pattern처럼 "몇 명이 들을지 모르는" 상황 |
| `DECLARE_DYNAMIC_MULTICAST_DELEGATE_...` | 여러 명 + 블루프린트에서도 바인딩 가능 | 디자이너가 블루프린트에서 이벤트에 반응하게 하고 싶을 때 |

Observer Pattern을 구현할 땐 거의 항상 **Multicast** 계열을 쓴다. 구독자가 0명이든 10명이든 `Broadcast` 쪽 코드는 똑같기 때문에, "몇 명이 듣고 있는지 모르는" Observer Pattern의 전제와 정확히 맞아떨어진다.

---

## 5. 구독 해제를 깜빡하면 생기는 문제

Observer Pattern의 흔한 함정은 **구독은 하는데 해제를 안 하는 것**이다.

```cpp
void UHPBarWidget::NativeDestruct()
{
    if (BoundCharacter)
    {
        BoundCharacter->OnHPChanged.RemoveAll(this);   // 위젯이 사라지기 전에 구독 해제
    }
    Super::NativeDestruct();
}
```

`HPBarWidget`이 파괴됐는데 구독 해제를 안 하면, `Character`가 `Broadcast`를 호출할 때 **이미 없어진 객체의 함수를 부르려는 시도**가 일어나서 크래시로 이어질 수 있다. 3강에서 본 RAII를 여기 적용하면, 위젯의 소멸자(혹은 `NativeDestruct`처럼 소멸 시점에 자동 호출되는 함수)에서 구독 해제를 자동으로 처리해서 "구독 해제를 깜빡하는" 실수 자체를 막을 수 있다.

---

## 6. UObject가 아닌 Observer라면 — 댕글링 포인터 위험

`AddUObject`로 바인딩하면 언리얼이 대상 `UObject`의 생존 여부를 어느 정도 추적해줘서, 파괴된 객체로의 `Broadcast`를 대부분 안전하게 막아준다. 하지만 **`UObject`가 아닌 순수 C++ 클래스(`FSoundManager`처럼 static/스택에 사는 객체 등)를 콜백 대상으로 묶으면 이 안전망이 없다.**

```cpp
class FDamageLogger   // UObject가 아닌 평범한 C++ 클래스
{
public:
    void SubscribeTo(ACharacter* Character)
    {
        Character->OnHPChanged.AddRaw(this, &FDamageLogger::LogDamage);   // 원시 포인터로 바인딩
    }

private:
    void LogDamage(float NewHP) { /* ... */ }
};

void SomeFunction()
{
    FDamageLogger Logger;              // 지역 변수(스택 객체) — 3강에서 배운 스코프 종료 시 소멸
    Logger.SubscribeTo(PlayerCharacter);
}   // 여기서 Logger는 소멸되지만, 구독 해제를 안 했다면 Character는 여전히 죽은 주소를 들고 있다!

// 나중에 PlayerCharacter->TakeDamage(...)가 Broadcast를 호출하면
// 이미 사라진 Logger의 메모리를 가리키는 함수 호출 시도 → 미정의 동작(크래시 가능성 높음)
```

`AddRaw`는 이름 그대로 **원시 포인터를 그대로 저장**하기 때문에, 그 객체가 스코프를 벗어나 소멸돼도(3강의 RAII가 정확히 이 시점을 보장한다) `Character`는 이를 전혀 모른다. **RAII가 자동으로 자원을 정리해준다는 바로 그 성질이, 구독 해제를 안 챙기면 오히려 댕글링 콜백이라는 위험으로 돌아오는 셈이다.** 이런 경우 소멸자에서 반드시 `RemoveAll(this)`를 호출하도록 짜야 한다.

```cpp
class FDamageLogger
{
public:
    ~FDamageLogger()
    {
        if (SubscribedCharacter)
        {
            SubscribedCharacter->OnHPChanged.RemoveAll(this);   // RAII로 구독 해제 자동화
        }
    }
    // ...
};
```

---

## 7. 코드로 체감해보기 — 이벤트 하나로 여러 시스템을 동시에 반응시키기

```cpp
// 퀘스트 시스템도 같은 이벤트를 구독하면, Character/UI 코드를 전혀 안 건드리고
// "HP가 30% 이하로 떨어지면 위기 퀘스트 발동" 같은 로직을 추가할 수 있다.
class UQuestManager : public UObject
{
public:
    void BindToCharacter(ACharacter* Character)
    {
        Character->OnHPChanged.AddUObject(this, &UQuestManager::HandleHPChanged);
    }

private:
    void HandleHPChanged(float NewHP)
    {
        if (NewHP / MaxHP < 0.3f)
        {
            TriggerCrisisQuest();
        }
    }
};
```

같은 `OnHPChanged` 이벤트 하나에 `HPBarWidget`, `MinimapIcon`, `UQuestManager`가 **서로의 존재를 전혀 모른 채** 각자 반응한다. 이게 느슨한 결합이 실전에서 주는 이득이다 — 시스템을 추가/제거할 때 다른 시스템 코드를 건드릴 필요가 없다.

---

## 8. 흔한 오해

### 오해 1: "Observer Pattern은 언리얼 Delegate가 있어야만 쓸 수 있다"

아니다. Observer Pattern은 **개념**이고, Delegate는 언리얼이 제공하는 구현 도구 중 하나일 뿐이다. 표준 C++에서는 함수 포인터, `std::function` 목록, 콜백 인터페이스(가상 함수)로도 같은 패턴을 구현할 수 있다.

### 오해 2: "Broadcast를 부르면 구독자가 실행되는 순서가 항상 보장된다"

일반적으로 구독한 순서대로 호출되긴 하지만, 이 순서에 로직을 의존하면 안 된다. Observer들은 서로 독립적이어야 한다는 게 이 패턴의 전제라, "A가 먼저 실행되고 나서 B가 실행돼야 한다"는 요구사항이 있다면 애초에 Observer Pattern이 안 맞는 상황이다.

### 오해 3: "구독자가 많아지면 성능에 신경 쓸 필요 없다"

`Broadcast`는 구독자 수만큼 함수 호출을 순회한다. 구독자가 수백 개 단위로 매 프레임 반복 호출되는 이벤트라면 이 비용도 무시 못한다. 대부분의 게임 이벤트(HP 변화, 아이템 획득)는 빈도가 낮아서 문제없지만, 매 프레임 도는 이벤트에 무분별하게 붙이면 성능 이슈가 될 수 있다.

### 오해 4: "구독 해제는 신경 안 써도 언리얼이 알아서 처리해준다"

`UObject` 기반 Delegate는 대상 객체가 사라지면 어느 정도 안전 장치가 있지만, 완전히 믿고 구독 해제를 생략하면 안 된다. 명시적으로 `RemoveAll`/`Remove`를 호출하는 습관이 안전하다 — 특히 `UObject`가 아닌 일반 객체를 바인딩할 땐 이 안전장치 자체가 없다.

### 오해 5: "Observer Pattern을 쓰면 결합도가 아예 0이 된다"

완전히 0은 아니다. 구독하는 쪽은 여전히 "어떤 이벤트를 구독해야 하는지"는 알아야 한다(`OnHPChanged`라는 이름과 시그니처는 알아야 함). 느슨한 결합이란 **"발행자가 구독자를 모른다"**는 뜻이지, 결합이 아예 사라진다는 뜻이 아니다.

### 오해 6: "AddRaw로 바인딩해도 AddUObject처럼 안전하게 처리된다"

아니다. `AddRaw`는 원시 포인터를 그대로 저장할 뿐이라, 그 객체가 소멸돼도 발행자는 전혀 알지 못한다. `UObject`가 아닌 클래스를 구독자로 쓸 땐 소멸자에서 직접 `RemoveAll(this)`를 호출하는 걸 잊으면 댕글링 콜백으로 이어진다.

---

## 9. 면접에서 나오면

**Q1. Observer Pattern에 대해 설명해주세요.**
→ "어떤 객체(Subject)의 상태가 바뀌었을 때, 그 변화에 관심 있는 여러 객체(Observer)들에게 알림을 보내서 각자 알아서 반응하게 하는 패턴입니다. Subject는 Observer가 누구인지, 몇 명인지 몰라도 되기 때문에 두 쪽이 느슨하게 결합됩니다."

**Q2. HP가 바뀔 때 UI가 자동으로 갱신되는 구조를 어떻게 만드시겠어요?**
→ "캐릭터에 HP 변화 이벤트(멀티캐스트 델리게이트)를 두고, 데미지나 힐 로직에서는 이 이벤트를 Broadcast만 합니다. HP바 위젯이나 미니맵 아이콘 같은 UI는 이 이벤트를 구독해서 각자 필요한 방식으로 갱신합니다. 이러면 캐릭터 코드가 특정 UI 클래스를 몰라도 됩니다."

**Q3. 느슨한 결합이 왜 중요한가요?**
→ "새로운 기능(UI, 시스템)을 추가할 때 기존 코드를 수정하지 않아도 되기 때문입니다. 강하게 결합되어 있으면 요구사항이 하나 늘 때마다 원본 클래스를 계속 찾아가 코드를 추가해야 하지만, 느슨하게 결합되어 있으면 새 Observer를 구독만 추가하면 됩니다."

**Q4. 언리얼에서 Observer Pattern을 구현할 때 어떤 Delegate를 쓰나요?**
→ "구독자가 여러 명일 수 있으니 Multicast Delegate를 씁니다. 블루프린트에서도 이벤트를 받아야 한다면 Dynamic Multicast Delegate를 쓰고, 응답이 딱 한 곳에서만 처리되는 게 명확하면 싱글 Delegate로 충분합니다."

**Q5. 구독 해제를 깜빡하면 어떤 문제가 생기나요?**
→ "구독한 객체가 이미 파괴됐는데 발행자가 여전히 그 객체를 향해 Broadcast를 시도하면 크래시로 이어질 수 있습니다. 위젯이나 액터가 소멸되는 시점에 명시적으로 구독을 해제하는 코드를 반드시 챙겨야 합니다."

**Q6. Observer가 수백 개씩 붙어서 매 프레임 Broadcast가 일어난다면 성능에 문제가 없을까요?**
→ "문제가 될 수 있습니다. Broadcast는 구독자 수만큼 함수 호출을 순회하기 때문에, 매 프레임 도는 이벤트에 구독자가 많이 붙으면 그 자체가 비용이 됩니다. 이런 경우엔 이벤트 빈도를 줄이거나, 정말 매 프레임 반응이 필요한 대상만 별도로 관리하는 걸 고려해야 합니다."

**Q7. Observer Pattern 대신 직접 함수를 호출하는 기존 방식을 계속 써도 되는 경우는 언제인가요?**
→ "관심을 가질 대상이 명확히 하나뿐이고 앞으로도 늘어날 가능성이 거의 없다면 직접 호출이 더 단순하고 디버깅하기도 쉽습니다. Observer Pattern은 구독자가 여러 개이거나 자주 늘어나는 상황에서 이득이 커지는 패턴이라, 무조건 쓰는 게 능사는 아닙니다."

**Q8. UObject가 아닌 클래스를 Observer로 쓸 때 특히 조심해야 할 게 뭔가요?**
→ "AddRaw로 바인딩하면 원시 포인터를 그대로 저장하기 때문에, UObject처럼 생존 여부를 추적해주는 안전장치가 없습니다. 그 객체가 소멸될 때 소멸자에서 직접 RemoveAll을 호출해서 구독을 해제하지 않으면, 발행자가 이미 사라진 객체를 향해 Broadcast를 시도하는 댕글링 콜백 문제가 생길 수 있습니다."

면접관이 그 다음에 던지기 좋은 질문: *"이런 이벤트 시스템을 게임 전역에서 하나만 두고 관리하고 싶다면 어떻게 설계하겠어요?"* → 9강(Singleton과 Factory)에서 다룰 GameInstance/Subsystem 패턴으로 이어지는 질문이다.

---

## 10. 셀프 체크

- [ ] 기존 직접 호출 방식과 Observer Pattern의 차이를 결합도 관점에서 설명할 수 있다.
- [ ] 언리얼 Multicast Delegate로 Observer Pattern을 구현하는 코드를 짤 수 있다.
- [ ] Broadcast 하는 쪽이 구독자를 몰라도 되는 게 왜 유리한지 새 기능 추가 시나리오로 설명할 수 있다.
- [ ] 구독 해제를 깜빡하면 왜 위험한지, 언제 해제해야 하는지 안다.
- [ ] AddRaw로 바인딩한 non-UObject 구독자가 왜 댕글링 콜백 위험에 더 취약한지 설명할 수 있다.
- [ ] Observer Pattern이 항상 정답이 아닌 경우(구독자가 명확히 하나뿐)를 판단할 수 있다.

---

다음 강의: **9강 — Singleton과 Factory**
"언리얼의 GameInstance/Subsystem이 왜 사실상 Singleton 역할을 하는지, 팩토리로 액터를 생성하면 뭐가 좋아지는지"를 다룬다.
