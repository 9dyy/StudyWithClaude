# 3강 — 내적 (Dot Product)

> **목표:** 내적이 뭘 계산하는 건지, 왜 각도·투영과 연결되는지 이해하고 시야각(FOV) 판정을 직접 짤 수 있다.
> **선수 지식:** 1강(벡터), 2강(정규화).

---

## 1. 도입 — "적이 내 시야 안에 있는지" 어떻게 판단하나

몬스터 AI를 짜다 보면 이런 요구사항이 자주 나온다: "플레이어가 몬스터의 정면 90도 시야각 안에 들어오면 몬스터가 발견한다." 거리는 `FVector::Dist`로 구할 수 있는데, **각도**는 어떻게 구할까?

각도를 구하려면 `atan2`나 삼각함수를 떠올리기 쉬운데, 실무에서는 훨씬 싸고 간단한 방법을 쓴다 — **내적(Dot Product)**이다. 벡터 두 개를 내적하면 그 사이의 각도와 직결된 숫자 하나가 바로 나온다.

```cpp
// 몬스터가 플레이어를 "보고 있는지" 판정
FVector ToPlayer = (PlayerLocation - MonsterLocation).GetSafeNormal();
FVector MonsterForward = Monster->GetActorForwardVector();

float DotResult = FVector::DotProduct(MonsterForward, ToPlayer);
// DotResult가 클수록(1에 가까울수록) 정면에 가깝다
```

이 강의는 저 `DotProduct`가 정확히 뭘 계산하는지, 왜 그렇게 각도와 연결되는지를 파본다.

---

## 2. 개념 설명

### 2-1. 내적의 정의 — 계산은 아주 단순하다

내적은 두 벡터의 같은 축 성분끼리 곱해서 다 더하는 것뿐이다.

```
A · B = A.X * B.X + A.Y * B.Y + A.Z * B.Z
```

```cpp
FVector A = FVector(1, 0, 0);
FVector B = FVector(0.7071f, 0.7071f, 0);   // 45도 방향 단위 벡터

float Dot = FVector::DotProduct(A, B);   // 1*0.7071 + 0*0.7071 + 0*0 = 0.7071
```

결과는 **스칼라 하나**다. 벡터끼리 계산했는데 벡터가 안 나오고 숫자 하나가 나온다는 게 내적의 특징이다.

### 2-2. 내적과 각도의 관계 — 코사인 그 자체

내적에는 이런 성질이 있다.

```
A · B = |A| * |B| * cos(θ)     (θ = A와 B 사이의 각도)
```

만약 **A와 B를 둘 다 정규화**해서 크기를 1로 만들면(`|A| = |B| = 1`), 이 식은 이렇게 단순해진다.

```
A · B = cos(θ)      (A, B가 단위 벡터일 때)
```

**정규화된 두 벡터를 내적하면, 그 결과값이 곧 두 벡터 사이 각도의 코사인 값이다.** 이게 내적이 각도 판정에 쓰이는 이유의 전부다. `acos()`를 써서 실제 각도(도 단위)로 변환할 수도 있지만, 대부분의 판정은 각도로 안 바꾸고 **코사인 값 자체를 그대로 비교**해서 끝낸다 — `acos()`도 비싼 연산이라 굳이 쓸 필요가 없을 때가 많다.

| 내적 결과 | 각도(대략) | 의미 |
|-----------|-----------|------|
| 1.0 | 0° | 완전히 같은 방향 (정면) |
| 0.7 | 45° | 대각선 정도 |
| 0.0 | 90° | 완전히 수직 |
| -0.7 | 135° | 거의 반대 방향 |
| -1.0 | 180° | 정반대 방향 |

### 2-3. 시야각(FOV) 판정 — 내적의 대표 활용

"정면 기준 좌우 60도(전체 120도) 안에 들어오면 발견"이라는 요구사항을 내적으로 짜보자.

```cpp
bool AMonster::CanSeeTarget(AActor* Target, float HalfFOVDegrees) const
{
    FVector ToTarget = (Target->GetActorLocation() - GetActorLocation()).GetSafeNormal();
    FVector Forward = GetActorForwardVector();   // 이미 단위 벡터

    float DotResult = FVector::DotProduct(Forward, ToTarget);

    // 코사인은 각도가 커질수록 작아지므로, "HalfFOV보다 안쪽"은 "cos(HalfFOV)보다 큰 값"과 같다
    float CosHalfFOV = FMath::Cos(FMath::DegreesToRadians(HalfFOVDegrees));

    return DotResult >= CosHalfFOV;
}
```

여기서 헷갈리기 쉬운 지점: **각도가 커질수록 코사인 값은 작아진다.** 그래서 "60도 이내"를 판정하려면 `DotResult >= cos(60도)`처럼 부등호 방향이 직관과 반대로 보일 수 있다. 이 관계표를 몸에 익혀두면 실수를 줄일 수 있다.

### 2-4. 내적의 또 다른 얼굴 — 투영(Projection)

내적은 각도 판정 말고 "한 벡터가 다른 벡터 방향으로 얼마나 나아갔는가"를 구하는 데도 쓰인다. 이걸 **투영(Projection)**이라 부른다.

캐릭터가 경사면을 오를 때, 이동 입력 벡터를 경사면 방향으로 투영해서 "실제로 경사면을 따라 얼마나 이동해야 하는가"를 구하는 게 대표적인 예다.

```
투영된 길이 = A · B_hat     (B_hat은 B의 정규화된 단위 벡터)
```

```cpp
// 캐릭터의 이동 입력을 경사면 방향으로 투영
FVector InputDir = FVector(1, 0, 0);          // 정면 이동 입력
FVector SlopeDir = SlopeNormal.GetSafeNormal(); // 경사면을 따라가는 방향

float ProjectedLength = FVector::DotProduct(InputDir, SlopeDir);
FVector ProjectedMovement = SlopeDir * ProjectedLength;
```

각도 판정과 투영은 같은 공식(`A · B_hat`)에서 나온 두 가지 다른 해석일 뿐이다 — "코사인 값으로 볼 것인가", "그림자 길이로 볼 것인가"의 차이다.

### 2-5. 내적으로 락온(Lock-On) 우선순위 정하기

액션 게임의 타겟팅 시스템은 "화면 안에 여러 적이 있을 때, 정면에 가장 가까운 적을 우선 락온"하는 로직을 자주 쓴다. 이것도 내적 하나로 처리된다 — 후보들 중 **내적 값이 가장 큰(정면에 가장 가까운) 대상**을 고르면 된다.

```cpp
AActor* AMyCharacter::FindBestLockOnTarget(const TArray<AActor*>& Candidates) const
{
    FVector Forward = GetActorForwardVector();
    AActor* BestTarget = nullptr;
    float BestDot = -1.0f;   // 내적의 최솟값(정반대)보다 낮은 값으로 초기화

    for (AActor* Candidate : Candidates)
    {
        FVector ToCandidate = (Candidate->GetActorLocation() - GetActorLocation()).GetSafeNormal();
        float Dot = FVector::DotProduct(Forward, ToCandidate);

        if (Dot > BestDot)
        {
            BestDot = Dot;
            BestTarget = Candidate;
        }
    }

    return BestTarget;
}
```

거리(`Size`)와 각도(내적)를 같이 가중치로 섞어서 "가깝고 정면에 있는 적"을 우선하는 식으로 확장하는 것도 흔한 패턴이다 — 실제 상용 게임의 락온 시스템 대부분이 이런 가중치 조합을 쓴다.

---

## 3. 그림으로 보면

```
      Forward (몬스터 정면)
        ↑
        │       ● Target
        │      ╱
        │     ╱ ToTarget
        │    ╱
        │   ╱  θ (각도)
        │  ╱ ⌒
    ────●────────────
      Monster

Dot(Forward, ToTarget) = cos(θ)
θ가 작을수록(정면에 가까울수록) Dot 값은 1에 가까움
```

투영은 이렇게 생각하면 된다: 해가 정확히 위에서 비칠 때, 벡터 A가 벡터 B 위에 드리우는 **그림자의 길이**가 곧 `A · B_hat`이다.

---

## 4. 코드로 — 실제 사례: 백어택(Backstab) 판정

RPG에서 흔한 "적의 뒤에서 공격하면 크리티컬" 판정도 내적 하나로 끝난다.

```cpp
bool AMyCharacter::IsBackstabAttack(AActor* Enemy) const
{
    FVector EnemyForward = Enemy->GetActorForwardVector();
    FVector EnemyToAttacker = (GetActorLocation() - Enemy->GetActorLocation()).GetSafeNormal();

    // 적의 정면 방향과 "적→공격자" 방향의 내적
    // 값이 음수(반대 방향에 가까움)면 공격자가 적의 등 뒤에 있다는 뜻
    float DotResult = FVector::DotProduct(EnemyForward, EnemyToAttacker);

    const float BackstabThreshold = -0.5f;   // 대략 후방 120도 범위
    return DotResult <= BackstabThreshold;
}
```

`EnemyForward`와 `EnemyToAttacker`가 거의 반대 방향(내적이 -1에 가까움)이라는 건, 공격자가 적이 등지고 있는 방향, 즉 적의 뒤쪽에 있다는 뜻이다.

---

## 5. 흔한 오해

### 오해 1: "내적을 하면 항상 각도가 바로 나온다"

내적 결과는 **코사인 값**이지 각도(도) 자체가 아니다. 실제 각도(예: 45도)가 필요하면 `acos()`를 한 번 더 거쳐야 한다. 대부분의 판정 로직은 그럴 필요 없이 코사인 값끼리 비교하는 것으로 충분하다.

### 오해 2: "정규화 안 한 벡터로 내적해도 각도 판정에 쓸 수 있다"

정규화 안 된 벡터로 내적하면 `A · B = |A||B|cos(θ)`라서, 결과에 **두 벡터의 크기까지 섞여버린다.** 순수하게 각도만 비교하고 싶으면 반드시 두 벡터 모두 정규화한 뒤 내적해야 한다.

### 오해 3: "내적 결과가 크면 두 벡터가 비슷한 방향이다 — 항상 그렇다"

정규화된 벡터끼리는 맞는 말이지만, 정규화 안 된 벡터라면 **한쪽 벡터가 그냥 크기가 커서** 내적 값이 커질 수도 있다. "값이 크다"와 "방향이 비슷하다"를 동일시하려면 항상 정규화가 전제되어야 한다.

### 오해 4: "내적이 0이면 두 벡터 중 하나가 0벡터라는 뜻이다"

내적이 0이 되는 훨씬 흔한 경우는 **두 벡터가 서로 수직(90도)**일 때다. 0벡터가 아니어도 방향이 직각이면 내적은 0이 된다.

### 오해 5: "시야각 판정에서 각도가 커지면 내적 조건도 완화된다(부등호 방향이 직관적이다)"

코사인은 각도가 커질수록 **작아지는** 함수라서, "더 넓은 각도까지 허용"하려면 오히려 **더 작은 내적 값까지** 허용해야 한다(`Dot >= cos(FOV)`에서 FOV가 커질수록 `cos(FOV)`는 작아짐). 부등호 방향을 반대로 짜는 실수가 은근히 잦다.

---

## 6. 면접에서 나오면

**Q1. 내적이 뭔가요?**
→ "두 벡터의 같은 축 성분끼리 곱해서 더한 값입니다. 결과는 벡터가 아니라 스칼라 하나이고, 두 벡터를 정규화한 상태에서 내적하면 그 값이 곧 둘 사이 각도의 코사인 값이 됩니다."

**Q2. 내적으로 시야각(FOV) 판정을 어떻게 구현하나요?**
→ "몬스터의 정면 벡터와 타겟 방향 벡터를 정규화해서 내적을 구하고, 그 값을 원하는 시야각의 코사인 값과 비교합니다. 내적이 그 값보다 크면 시야각 안에 있다고 판정합니다."

**Q3. 두 벡터가 정확히 반대 방향이면 내적 값은 어떻게 되나요?**
→ "-1이 나옵니다. cos(180도)가 -1이기 때문입니다. 같은 방향이면 1, 수직이면 0, 반대 방향이면 -1이라는 걸 기준으로 삼으면 판정 로직 짜기 편합니다."

**Q4. 내적을 각도 판정이 아니라 다른 용도로 쓴 적 있나요? 혹은 다른 용도를 아나요?**
→ "투영(Projection)에 씁니다. 예를 들어 캐릭터의 이동 입력을 경사면 방향으로 투영해서, 실제로 경사면을 따라 얼마나 이동해야 하는지 구할 때 내적을 씁니다."

**Q5. 백어택 판정처럼 '적의 등 뒤에 있는지'는 내적으로 어떻게 판정하나요?**
→ "적의 정면 벡터와, 적에서 공격자 쪽을 향하는 벡터를 내적합니다. 이 값이 충분히 음수라면 두 벡터가 반대 방향에 가깝다는 뜻이니까, 공격자가 적이 등지고 있는 방향, 즉 뒤쪽에 있다고 판정할 수 있습니다."

**Q6. 내적 대신 acos로 실제 각도를 구해서 비교하는 것과, 내적 값을 코사인 값 그대로 비교하는 것 중 뭘 고르겠어요?**
→ "코사인 값 그대로 비교하는 쪽을 고릅니다. acos는 삼각함수라 상대적으로 연산 비용이 크고, 어차피 판정 목적이면 각도로 바꾸지 않고 코사인 값끼리 비교해도 결과가 동일합니다. 실제 각도 숫자를 UI에 보여줘야 하는 경우가 아니면 acos를 쓸 이유가 없습니다."

**Q7. 정규화 안 된 벡터로 내적을 하면 어떤 문제가 생기나요?**
→ "내적 값에 두 벡터의 크기까지 섞여 들어가서, 순수하게 각도만 비교하려는 목적이면 결과가 왜곡됩니다. 각도 판정에 쓸 거면 두 벡터를 먼저 정규화해야 합니다."

---

## 7. 셀프 체크

- [ ] 내적의 계산 공식(성분곱의 합)을 안다.
- [ ] 정규화된 벡터의 내적이 코사인 값과 같다는 걸 설명할 수 있다.
- [ ] 시야각(FOV) 판정을 내적으로 구현할 수 있다.
- [ ] 각도가 커질수록 코사인 값이 작아진다는 관계를 알고, 부등호 방향을 헷갈리지 않는다.
- [ ] 투영이 내적의 또 다른 활용이라는 걸 설명할 수 있다.
- [ ] 내적 값이 1, 0, -1일 때 각각 어떤 방향 관계인지 안다.

---

다음 강의: **4강 — 외적(Cross Product)**
"왼쪽/오른쪽, 앞/뒤 같은 상대적 위치를 어떻게 판정하는지, 그리고 회전축을 어떻게 구하는지 본다."
