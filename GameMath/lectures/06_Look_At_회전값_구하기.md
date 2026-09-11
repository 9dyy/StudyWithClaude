# 6강 — Look-At, 회전값 구하기

> **목표:** 두 지점 사이 방향 벡터로부터 회전값(FRotator)을 구하는 원리를 이해하고, FindLookAtRotation의 내부 동작을 설명할 수 있다.
> **선수 지식:** 1~5강 (벡터, 정규화, 내적, 외적, 좌표계).

---

## 1. 도입 — 터렛이 플레이어를 바라보게 하고 싶다

고정 포탑(터렛)이 플레이어를 추적해서 바라보게 만든다고 하자. 위치는 벡터로 쉽게 계산했다 — 방향 벡터는 `PlayerLocation - TurretLocation`. 그런데 이 방향 벡터를 그대로 터렛에 넣을 수는 없다. 터렛의 회전은 `FVector`가 아니라 `FRotator`(또는 `FQuat`)로 표현되기 때문이다.

```cpp
// 방향은 구했는데, 이걸 회전값으로 어떻게 바꾸지?
FVector Direction = (PlayerLocation - TurretLocation).GetSafeNormal();
// Turret->SetActorRotation(???);
```

다행히 언리얼은 이걸 함수 하나로 해결해준다.

```cpp
FRotator LookAtRotation = UKismetMathLibrary::FindLookAtRotation(TurretLocation, PlayerLocation);
Turret->SetActorRotation(LookAtRotation);
```

이 강의는 `FindLookAtRotation`이 내부적으로 뭘 계산하는지, 왜 방향 벡터 하나로 회전값 전체가 결정되는지를 파본다.

---

## 2. 개념 설명

### 2-1. 회전을 표현하는 방법 — Rotator vs Quaternion

언리얼은 회전을 두 가지 방식으로 다룬다.

- **FRotator:** Pitch(상하), Yaw(좌우), Roll(기울임) 세 각도로 표현. 사람이 이해하기 쉽고 블루프린트/에디터에서 자주 보임.
- **FQuat(쿼터니언):** 4개의 숫자(X,Y,Z,W)로 회전을 표현. 내부 계산(보간, 합성)에 훨씬 안정적이고 짐벌락(Gimbal Lock) 문제가 없음.

이 강의는 개념 이해를 위해 `FRotator` 기준으로 설명하되, 실무에서는 여러 회전을 곱하거나 부드럽게 보간할 때 `FQuat`을 쓰는 경우가 많다는 것만 기억해두면 된다. Look-At의 핵심 원리(방향 벡터 → 회전값)는 둘 다 동일하다.

### 2-2. 방향 벡터 하나로 회전이 어떻게 결정되는가

**핵심 아이디어: 방향 벡터를 알면, "그 방향을 바라보는 회전"은 유일하게 하나로 정해진다** (Roll을 0으로 고정한다는 전제 하에).

풀어서 설명하면:

1. 방향 벡터 `(X, Y, Z)`가 주어지면, 이 벡터가 XY 평면에 투영된 각도로부터 **Yaw**(좌우로 얼마나 돌았는지)를 구할 수 있다.
2. 방향 벡터가 수평면 기준으로 얼마나 위/아래를 향하는지로부터 **Pitch**(상하로 얼마나 기울었는지)를 구할 수 있다.
3. Roll은 "정면이 어디를 보는가"와 무관한 정보(카메라가 옆으로 얼마나 기울었는지)라서, 순수하게 방향만으로는 결정되지 않고 보통 0으로 둔다.

수식으로 쓰면 (라디안 기준, 언리얼 좌표계 근사):

```
Yaw   = atan2(Direction.Y, Direction.X)
Pitch = atan2(Direction.Z, sqrt(Direction.X² + Direction.Y²))
Roll  = 0  (보통 고정)
```

`atan2`는 일반 `atan`과 달리 두 값의 부호를 같이 받아서 **사분면 전체(-180도~180도)**를 올바르게 구분해준다는 게 핵심이다. 단순히 `Y/X`의 비율만으로는 각도가 1사분면인지 3사분면인지 구분이 안 되기 때문에, 방향을 각도로 바꿀 때는 항상 `atan2` 계열을 쓴다.

### 2-3. FindLookAtRotation이 하는 일

```cpp
FRotator FindLookAtRotation(const FVector& Start, const FVector& Target)
{
    FVector Direction = (Target - Start).GetSafeNormal();
    return Direction.Rotation();   // 내부적으로 atan2 기반 계산
}
```

실제 구현은 최적화가 더 되어있지만, 개념적으로는 딱 두 단계다.

1. `Target - Start`로 변위 벡터를 구하고 정규화해서 순수 방향만 남긴다.
2. 그 방향 벡터를 `Rotation()`(내부적으로 atan2 기반)으로 회전값으로 변환한다.

**결국 Look-At은 "방향 벡터 구하기"(1강) + "방향 벡터를 각도로 변환하기"(이 강의) 두 단계의 조합일 뿐이다.** 새로운 수학이 아니라 지금까지 배운 걸 그대로 재사용하는 것이다.

### 2-4. 내적/외적으로 회전을 구하는 방법 (Quaternion 기준)

`atan2` 대신, 3~4강에서 배운 내적/외적으로도 회전을 구할 수 있다. "현재 정면 방향에서 목표 방향으로 회전시키는 쿼터니언"을 구하는 방법이다.

```
회전각(θ) = acos(dot(CurrentForward, TargetDirection))   // 내적으로 각도
회전축(Axis) = cross(CurrentForward, TargetDirection)      // 외적으로 축
```

이 방식은 "현재 방향에서 목표 방향까지 얼마나, 어느 축을 기준으로 돌아야 하는가"를 직접 구하는 방식이라, 절대적인 Yaw/Pitch를 새로 계산하는 것보다 **부드러운 회전 보간**(현재 방향에서 서서히 목표 방향으로 돌아가는 연출)을 만들 때 자연스럽게 이어진다.

```cpp
// 현재 정면에서 목표 방향으로 부드럽게 회전 (개념 예시)
FVector CurrentForward = GetActorForwardVector();
FVector TargetDirection = (TargetLocation - GetActorLocation()).GetSafeNormal();

FRotator CurrentRotation = GetActorRotation();
FRotator DesiredRotation = TargetDirection.Rotation();

FRotator SmoothRotation = FMath::RInterpTo(CurrentRotation, DesiredRotation, DeltaTime, InterpSpeed);
SetActorRotation(SmoothRotation);
```

`RInterpTo`가 내부적으로 두 회전 사이를 보간하는 계산은 쿼터니언 기반이고, 그 밑바탕엔 지금 설명한 내적(각도)·외적(축) 개념이 깔려있다.

### 2-5. Pitch를 0으로 고정해야 하는 상황 — 지상 캐릭터

캐릭터의 시선 방향으로 몸 전체를 회전시키면, 위/아래를 볼 때마다 몸이 앞으로 숙여지거나 뒤로 젖혀지는 이상한 자세가 나온다. 그래서 걷는 캐릭터의 몸통 회전에는 보통 **Pitch를 0으로 강제**하고 Yaw만 적용한다.

```cpp
FRotator LookAtRotation = UKismetMathLibrary::FindLookAtRotation(GetActorLocation(), TargetLocation);
LookAtRotation.Pitch = 0.0f;   // 몸통은 좌우로만 회전
LookAtRotation.Roll = 0.0f;    // 옆으로 기울지 않음
SetActorRotation(LookAtRotation);
```

카메라나 머리 본(Bone)처럼 위아래를 실제로 봐야 하는 대상에는 Pitch를 그대로 두고, 몸통·이동 방향에는 Yaw만 쓰는 식으로 용도에 따라 나눠 쓴다.

---

## 3. 그림으로 보면

```
Target
  ●
   ╲
    ╲  Direction (정규화된 방향 벡터)
     ╲
      ╲   Pitch = atan2(Z, sqrt(X²+Y²))
       ╲  ↕
        ●─────────── XY 평면
      Start    ↔
             Yaw = atan2(Y, X)
```

방향 벡터 하나를 "수평 성분(X,Y)"과 "수직 성분(Z)"으로 나눠 보면, 수평 성분의 각도가 Yaw, 수직 성분과 수평 성분의 비율이 Pitch가 된다.

---

## 4. 코드로 — 실제 사례: 터렛의 시야각 제한 + Look-At

터렛이 무한히 아무 방향으로나 도는 게 아니라, 정해진 범위 안에서만 회전하도록 3강(내적)과 이 강의(Look-At)를 같이 써보자.

```cpp
void ATurret::TrackTarget(AActor* Target, float DeltaTime)
{
    if (!Target)
    {
        return;
    }

    FVector ToTarget = (Target->GetActorLocation() - GetActorLocation()).GetSafeNormal();
    FVector BaseForward = GetActorForwardVector();

    // 3강 내적으로 시야각 안에 있는지 먼저 체크
    float DotResult = FVector::DotProduct(BaseForward, ToTarget);
    const float TrackingCosLimit = FMath::Cos(FMath::DegreesToRadians(90.0f));
    if (DotResult < TrackingCosLimit)
    {
        return;   // 시야각 밖이면 추적 안 함
    }

    // 시야각 안이면 Look-At으로 목표를 향해 서서히 회전
    FRotator DesiredRotation = UKismetMathLibrary::FindLookAtRotation(GetActorLocation(), Target->GetActorLocation());
    DesiredRotation.Pitch = 0.0f;   // 터렛 베이스는 좌우로만 회전한다고 가정

    FRotator NewRotation = FMath::RInterpTo(GetActorRotation(), DesiredRotation, DeltaTime, 5.0f);
    SetActorRotation(NewRotation);
}
```

지금까지 배운 벡터(방향 계산), 내적(시야각 판정), Look-At(회전값 계산)이 한 함수 안에서 자연스럽게 합쳐지는 걸 볼 수 있다.

---

## 5. 흔한 오해

### 오해 1: "FindLookAtRotation은 뭔가 복잡한 특수 계산을 한다"

내부적으로는 방향 벡터를 구해서 정규화하고, 그걸 `atan2` 기반으로 각도로 바꾸는 두 단계일 뿐이다. 1강의 변위/정규화, 이 강의의 각도 변환을 그대로 조합한 것이다.

### 오해 2: "방향 벡터만 있으면 회전이 완전히 하나로 결정된다"

Yaw와 Pitch는 방향 벡터로 결정되지만, **Roll은 방향 벡터만으로는 정해지지 않는다.** 정면이 같은 방향이어도 옆으로 얼마나 기울었는지(Roll)는 별개 정보다. 그래서 보통 Roll은 0으로 고정하거나 별도로 계산한다.

### 오해 3: "atan(Y/X)로도 충분하다, 굳이 atan2일 필요 없다"

`atan(Y/X)`는 X가 음수일 때 각도의 사분면을 구분하지 못한다(같은 비율이라도 1사분면과 3사분면을 같은 값으로 취급). `atan2(Y, X)`는 X, Y의 부호를 각각 참고해서 올바른 사분면의 각도를 반환한다.

### 오해 4: "캐릭터 몸통에 Look-At을 그대로 적용해도 된다"

시선 방향을 몸통 회전에 그대로 적용하면 위/아래를 볼 때 몸이 이상하게 숙여지거나 젖혀진다. 지상 캐릭터의 몸통은 보통 Pitch/Roll을 0으로 고정하고 Yaw만 적용한다.

### 오해 5: "SetActorRotation을 매 프레임 직접 호출하면 회전이 자연스럽다"

목표 회전값으로 순간이동하듯 바로 스냅되기 때문에 뚝뚝 끊기는 느낌이 난다. 부드러운 회전이 필요하면 `RInterpTo`나 `FQuat::Slerp` 같은 보간 함수로 매 프레임 목표 값에 조금씩 다가가야 한다.

---

## 6. 면접에서 나오면

**Q1. 캐릭터나 카메라가 특정 지점을 바라보게 하려면 어떻게 회전값을 구하나요?**
→ "목표 지점에서 현재 위치를 빼서 방향 벡터를 구하고 정규화한 다음, 그 방향 벡터를 회전값으로 변환합니다. 언리얼에서는 FindLookAtRotation 함수가 이 과정을 대신해줍니다."

**Q2. FindLookAtRotation은 내부적으로 어떻게 동작하나요?**
→ "목표 위치에서 시작 위치를 뺀 방향 벡터를 정규화하고, 그 벡터의 X, Y, Z 성분을 atan2 기반으로 Yaw와 Pitch로 변환합니다. 방향 벡터 하나로 회전이 계산되는 원리입니다."

**Q3. 방향 벡터로 Roll까지 결정할 수 있나요?**
→ "아니요, Roll은 방향 벡터만으로는 결정되지 않습니다. 정면 방향이 같아도 옆으로 얼마나 기울었는지는 별개 정보라서, 보통 Roll은 0으로 고정하거나 다른 기준(예: 월드 Up 벡터)으로 따로 계산합니다."

**Q4. atan 대신 atan2를 쓰는 이유가 뭔가요?**
→ "atan은 Y/X 비율만 보기 때문에 부호 정보가 사라져서 사분면을 구분하지 못합니다. atan2는 X, Y를 따로 받아서 전체 360도 범위에서 올바른 사분면의 각도를 반환합니다."

**Q5. 캐릭터가 위/아래를 바라볼 때 몸통까지 같이 숙여지면 이상한데, 어떻게 처리하나요?**
→ "몸통 회전에는 Pitch를 0으로 고정해서 좌우(Yaw) 회전만 적용하고, 실제 시선 방향은 카메라나 머리 본 같은 별도 요소에 반영합니다."

**Q6. Look-At 회전을 매 프레임 바로 SetActorRotation으로 적용하는 것과, RInterpTo로 보간해서 적용하는 것 중 어떤 걸 고르겠어요? 왜죠?**
→ "대부분 RInterpTo로 보간하는 쪽을 고릅니다. 목표 회전으로 바로 스냅하면 캐릭터가 순간적으로 홱 도는 것처럼 보여서 부자연스럽습니다. 다만 조준 스코프처럼 즉각적인 반응이 필요한 경우엔 보간 없이 바로 적용하는 게 맞을 수도 있습니다."

**Q7. 내적/외적으로도 회전을 구할 수 있다고 했는데, 언제 이 방식을 쓰나요?**
→ "현재 방향에서 목표 방향으로의 회전각과 회전축이 필요할 때 씁니다. 내적으로 두 방향 사이 각도를, 외적으로 회전축을 구할 수 있는데, 절대 회전값을 새로 계산하기보다 현재 상태에서 상대적으로 얼마나 돌아야 하는지 다룰 때 자연스럽게 이어집니다."

---

## 7. 셀프 체크

- [ ] 방향 벡터로부터 Yaw, Pitch를 구하는 원리(atan2 기반)를 설명할 수 있다.
- [ ] Roll이 방향 벡터만으로 결정되지 않는 이유를 안다.
- [ ] atan과 atan2의 차이(사분면 구분)를 설명할 수 있다.
- [ ] FindLookAtRotation이 내부적으로 어떤 두 단계로 이뤄지는지 안다.
- [ ] 지상 캐릭터 몸통 회전에 Pitch를 0으로 고정하는 이유를 설명할 수 있다.
- [ ] 즉시 회전(SetActorRotation)과 보간 회전(RInterpTo)의 차이와 사용 시점을 안다.

---

다음 강의: **7강 — 충돌 감지 기초**
"총알이 벽에 맞았는지, 캐릭터끼리 부딪혔는지 매 프레임 어떻게 확인하는지, AABB와 구체 충돌부터 본다."
