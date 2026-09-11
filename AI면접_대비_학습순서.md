# AI 면접(코딩테스트 대체) 대비 — 과목 통합 학습 순서

> 코딩 테스트를 대체하는 'AI 면접'에서 게임 프로그래머의 기본기·공통역량을 평가한다는 전제로, 실제 면접 질문 15개(면접질문모음.md)와 CS/게임 공통역량 출제 빈도를 종합해 우선순위를 매긴 것.
> DB는 제외. 각 시리즈의 보너스(BEST20) 파일은 마무리 복습용이라 순위에서 제외.
> 작성 시점: 2026-09-11. 학습이 더 진행되면 갱신 필요.

---

## S — 실제로 질문받았거나 거의 확실히 나오는 것

| 순위 | 강의 | 근거 |
|---|---|---|
| 1 | [Network 06 TCP_vs_UDP_게임은_뭘쓰나](Network/lectures/06_TCP_vs_UDP_게임은_뭘쓰나.md) | 실제 질문(TCP/UDP 차이, 게임 예시) — 게임 네트워크의 대표 단골 |
| 2 | [OS 10 Deadlock](OS/lectures_md/10_Deadlock.md) | 실제 질문(starvation 해결법) 포함 |
| 3 | [OS 07 Race_Condition_임계영역](OS/lectures_md/07_Race_Condition_임계영역.md) | 동기화 파트의 핵심, 거의 항상 나옴 |
| 4 | [OS 08 Mutex_Lock_CriticalSection](OS/lectures_md/08_Mutex_Lock_CriticalSection.md) | 07강과 세트로 항상 같이 물어봄 |
| 5 | [Cpp 01 static의_여러_얼굴](Cpp/lectures/01_static의_여러_얼굴.md) | 실제 질문 |
| 6 | [Cpp 02 캐스팅_4형제](Cpp/lectures/02_C++_캐스팅_4형제.md) | 실제 질문 |
| 7 | [Cpp 04 배열_vs_LinkedList](Cpp/lectures/04_배열_vs_LinkedList.md) | 실제 질문 |
| 8 | [Cpp 11 스마트_포인터](Cpp/lectures/11_스마트_포인터.md) | 실제 질문(멀티스레드 shared_ptr) |
| 9 | [GameMath 03 내적_Dot_Product](GameMath/lectures/03_내적_Dot_Product.md) | 실제 질문(내적/외적) |
| 10 | [GameMath 04 외적_Cross_Product](GameMath/lectures/04_외적_Cross_Product.md) | 실제 질문(내적/외적) |
| 11 | [GameMath 07 충돌_감지_기초](GameMath/lectures/07_충돌_감지_기초.md) | 실제 질문(충돌 감지 방법) |
| 12 | [GameMath 06 Look_At_회전값_구하기](GameMath/lectures/06_Look_At_회전값_구하기.md) | 실제 질문(Look-at 원리) |
| 13 | [GameMath 08 점과_다각형](GameMath/lectures/08_점과_다각형.md) | 실제 질문(점-다각형 판정) |
| 14 | [Network 14 언리얼_네트워크_모델](Network/lectures/14_언리얼_네트워크_모델.md) | 실제 질문(reliable/unreliable 예시) |
| 15 | [OS 09 Atomic_LockFree](OS/lectures_md/09_Atomic_LockFree.md) | 실제 질문(shared_ptr 스레드안전)의 배경지식 — Cpp 11강과 항상 세트로 꼬리질문 옴 |

## A — 일반 CS/게임 공통역량 면접에서 매우 흔한 것

직접 질문 목록엔 없지만 출제 확률이 높은 항목.

| 순위 | 강의 | 근거 |
|---|---|---|
| 16 | [OS 04 프로세스_vs_스레드](OS/lectures_md/04_프로세스_vs_스레드.md) | CS 기본 중의 기본, 거의 항상 나옴 |
| 17 | [OS 03 프로세스_메모리구조](OS/lectures_md/03_프로세스_메모리구조.md) | 스택/힙 구분 — 신입 필수 |
| 18 | [Cpp 07 Object_Pooling](Cpp/lectures/07_Object_Pooling.md) | 게임 프로그래머 특화 단골(실제 질문에도 있었음, S로 올려도 될 수준) |
| 19 | [Cpp 08 Observer_Pattern](Cpp/lectures/08_Observer_Pattern.md) | 게임 특화 단골(실제 질문에도 있었음) |
| 20 | [OS 06 CPU_스케줄링_컨텍스트_스위칭](OS/lectures_md/06_CPU_스케줄링_컨텍스트_스위칭.md) | CS 기본 |
| 21 | [OS 11 가상메모리와_페이징](OS/lectures_md/11_가상메모리와_페이징.md) | CS 기본, 자주 나옴 |
| 22 | [Network 04 TCP_신뢰성의_대가](Network/lectures/04_TCP_신뢰성의_대가.md) | 06강 전제지식, TCP 세부 질문 대비 |
| 23 | [Network 05 UDP_빠름의_대가](Network/lectures/05_UDP_빠름의_대가.md) | 06강 전제지식 |
| 24 | [GameMath 01 벡터란_무엇인가](GameMath/lectures/01_벡터란_무엇인가.md) | 내적/외적의 전제지식, 사실상 필수 |
| 25 | [GameMath 05 좌표계와_변환](GameMath/lectures/05_좌표계와_변환.md) | 월드/로컬, 트랜스폼 — 자주 나옴 |
| 26 | [Cpp 12 정렬_알고리즘과_시간복잡도](Cpp/lectures/12_정렬_알고리즘과_시간복잡도.md) | Big-O는 코딩테스트 대체 면접의 단골 설명형 질문 |
| 27 | [OS 13 캐시와_메모리계층](OS/lectures_md/13_캐시와_메모리계층.md) | 04강(배열vs링크드리스트) 캐시지역성과 직결 |
| 28 | [Cpp 03 생성자_소멸자와_RAII](Cpp/lectures/03_생성자_소멸자와_RAII.md) | 스마트포인터 전제지식 |
| 29 | [GameMath 02 벡터의_크기와_정규화](GameMath/lectures/02_벡터의_크기와_정규화.md) | 벡터 기초, 내적 전제지식 |
| 30 | [Network 09 블로킹_논블로킹_IO멀티플렉싱](Network/lectures/09_블로킹_논블로킹_IO멀티플렉싱.md) | 비동기 처리 개념, 자주 나옴 |
| 31 | [OS 15 블로킹_논블로킹_비동기IO](OS/lectures_md/15_블로킹_논블로킹_비동기IO.md) | 위와 세트 |
| 32 | [Cpp 05 Stack과_Queue](Cpp/lectures/05_Stack과_Queue.md) | 자료구조 기본기 |

## B — 가끔 나옴 (여유 있으면 준비)

- [Network 03 포트와_소켓](Network/lectures/03_포트와_소켓.md)
- [Network 08 소켓_프로그래밍_기초](Network/lectures/08_소켓_프로그래밍_기초.md)
- [Network 07 흐름제어_혼잡제어_Nagle](Network/lectures/07_흐름제어_혼잡제어_Nagle.md)
- [Network 10 패킷_설계와_직렬화](Network/lectures/10_패킷_설계와_직렬화.md)
- [Network 15 레이턴시와_지연보상](Network/lectures/15_레이턴시와_지연보상.md)
- [Cpp 09 Singleton과_Factory](Cpp/lectures/09_Singleton과_Factory.md)
- [Cpp 10 State_Pattern](Cpp/lectures/10_State_Pattern.md)
- [Cpp 06 해시맵과_트리_맛보기](Cpp/lectures/06_해시맵과_트리_맛보기.md)
- [OS 05 언리얼의_스레드_구조](OS/lectures_md/05_언리얼의_스레드_구조.md)
- [OS 02 커널모드_유저모드_시스템콜](OS/lectures_md/02_커널모드_유저모드_시스템콜.md)
- [OS 12 페이지폴트와_스왑](OS/lectures_md/12_페이지폴트와_스왑.md)
- [OS 14 파일시스템_기본](OS/lectures_md/14_파일시스템_기본.md)

## C — 드묾 (후순위, 배경지식 정도로 훑기)

- [OS 01 운영체제란](OS/lectures_md/01_운영체제란.md)
- [Network 01 네트워크란](Network/lectures/01_네트워크란.md)
- [Network 02 IP주소와_라우팅](Network/lectures/02_IP주소와_라우팅.md)
- [Network 11 HTTP_HTTPS](Network/lectures/11_HTTP_HTTPS.md)
- [Network 12 DNS](Network/lectures/12_DNS.md)
- [Network 13 TLS와_게임보안](Network/lectures/13_TLS와_게임보안.md)

---

## 요약 기준

S/A는 15개 실제 질문에 직접 매핑되거나(S), 그 질문의 필수 전제지식이거나 거의 항상 짝으로 따라오는 CS 기본기(A)다. B/C는 게임 클라 신입 "공통역량" 범위에서 상대적으로 덜 물어보는(주로 서버/인프라 성격이 강한) 영역이라 후순위로 뺐다.

시간이 부족하면 **S 15개만 확실히 하고 A로 넘어가는 순서**를 추천한다.
