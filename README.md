> **📌 이 저장소는 포트폴리오용 포크입니다.**
> 원본: [team-alpha-labs/parami](https://github.com/team-alpha-labs/parami) · 팀 7명 · 2026.05
> 아래는 **임소라(flathfk)가 이 프로젝트에서 맡은 부분**이고, 원본 README는 이어서 나옵니다.

# 내가 한 일 — 결제 · 구독 · 보상조회

한경 × 토스뱅크 FullStack-LLM 부트캠프 **중간 프로젝트**
**51커밋 / 전체 271커밋 중 2위** · TypeScript · MySQL · 토스페이먼츠 · GCP Cloud Scheduler

🔗 **배포** — 데모 서버는 현재 중단된 상태입니다 (부트캠프 과정 종료로 GCP 인스턴스 정리). 코드와 아래 문서로 확인해 주세요.

**돈이 오가는 도메인**을 맡았습니다. 결제가 한 번 잘못되면 사용자가 실제로 손해를 보기 때문에, 기능을 넓히기보다 **실패 경로를 하나씩 닫는 데** 시간을 썼습니다.

## 결제 안전망 4단계

토스 결제 후 발생할 수 있는 실패 케이스를 단계별로 막았습니다.

| 단계 | 방어 대상 | 안전장치 |
| --- | --- | --- |
| 1 | 클라이언트 가격 변조 | 서버 DB 가격 재조회 후 amount 비교 |
| 2 | 토스 위변조 결제 | 토스 confirm API로 재검증 (Basic Auth) |
| 3 | DB 부분 저장 사고 | 트랜잭션 BEGIN ~ COMMIT |
| 4 | DB 저장 실패 시 사용자 돈 묶임 | 자동 환불 호출 → 환불 실패 시 운영자 알림 |

---

# 트러블슈팅

## 1. 결제 성공 직후 DB 저장 실패 → 사용자 돈이 묶임

**문제** 토스 결제는 됐는데 우리 DB INSERT가 실패하면, **사용자 돈은 빠졌고 시스템엔 기록이 없는** 최악의 상태가 됩니다.

**원인** 초기 코드는 DB 실패 시 그냥 500 응답만 줬습니다. 사용자 입장에선 "결제는 됐는데 구독은 안 된" 상태로 방치됩니다.

**해결** DB 처리를 별도 try/catch로 감싸 실패 시 토스 환불 API를 자동 호출합니다.

```ts
try {
  const result = await processConfirmedPayment(...)
  return ok(result)
} catch (dbError) {
  try {
    await cancelTossPayment(body.paymentKey, '서버 DB 처리 실패로 인한 자동 환불')
    return err('자동 환불되었습니다.', 500)
  } catch (refundError) {
    console.error('CRITICAL: 환불 자체 실패 — 운영자 수동 처리 필요', ...)
    return err('고객센터 문의 바랍니다.', 500)
  }
}
```

3단계 분기 — DB 성공 / DB 실패 후 환불 성공 / **환불도 실패**. 마지막 경우는 자동으로 못 닫으므로 운영자가 볼 수 있게 CRITICAL 로그를 남깁니다.

**교훈** 외부 결제 연동에선 자동 환불 분기가 필수. 더 정석은 환불 재시도 큐 도입(향후).

## 2. 클라이언트 가격 변조 공격

**문제** 브라우저 개발자도구로 `{ tier: 'premium', amount: 7400 }` 같은 위변조 요청이 가능합니다. 프리미엄을 베이직 가격에 사는 겁니다.

**원인** 초기 코드가 body의 tier/amount를 **그대로 토스에 전달**했습니다. 토스는 자기네 결제 정보만 검증하므로 amount만 일치하면 통과시킵니다 — **우리가 안 보면 아무도 안 봅니다.**

**해결** 서버가 DB에서 가격을 재조회해 비교하고, 적용할 티어도 **서버가 결정**합니다.

```ts
// 기존 구독자는 서버 권위로 결정 — body의 tier를 믿지 않음
const effectiveTier = currentSub
  ? (currentSub.pending_tier ?? currentSub.tier)
  : body.tier                                     // 신규 가입만 body 신뢰

const plan = await findPlanByTier(effectiveTier)
if (plan.price !== body.amount) {
  return err('결제 금액이 티어 가격과 일치하지 않습니다.', 400)
}
```

**교훈** 클라이언트 값은 절대 신뢰하지 않는다. DB가 진실의 원천.

## 3. 같은 사용자 동시 결제 → active 구독 중복 생성

**문제** 빠른 연타나 네트워크 지연으로 같은 요청 2개가 동시에 들어오면, 둘 다 "active 없음"을 보고 **둘 다 INSERT**합니다.

**원인** MySQL REPEATABLE READ 스냅샷 + **부분 UNIQUE 인덱스 미지원**으로 write skew가 발생합니다. `WHERE status='active'` 조건부 유니크를 DB가 못 걸어줍니다.

**해결** 앱 레벨 3중 안전망:

```sql
-- 1. 락 (X-lock) — 같은 user_id 요청을 직렬화
SELECT id FROM users WHERE id = ? FOR UPDATE

-- 2. 락 잡은 후 재조회 — 다른 트랜잭션이 만든 active가 이제 보임
SELECT ... FROM subscriptions
WHERE user_id = ? AND status = 'active'
ORDER BY started_at DESC, id DESC
LIMIT 1  -- 3. 안전망: 만약 2개 생겨도 1개만 반환
```

**교훈** 동시성은 단위 테스트로 못 잡습니다. 트랜잭션 + 락 + 안전망 다층 방어.

## 4. 결제 확정 중복 요청 (새로고침 / React strict mode)

**문제** 결제완료 페이지에서 새로고침하면 confirm을 또 호출합니다 → **두 번 청구** 가능. 여기에 React 19 strict mode가 `useEffect`를 두 번 실행해 같은 일이 개발 중에도 재현됐습니다.

**해결 — 두 겹**

| 겹 | 수단 |
| --- | --- |
| DB | `UNIQUE KEY uq_toss_order (toss_order_id)` → 중복 INSERT 시 `Duplicate entry` → ROLLBACK |
| 프론트 | `useRef` 가드 → strict mode 2회 실행 차단 |

```ts
const calledRef = useRef(false)
useEffect(() => {
  if (calledRef.current) return
  calledRef.current = true
  // confirm 호출 ...
}, [...])
```

**교훈** strict mode 대응은 개발용 회피가 아닙니다. **네트워크 retry, 빠른 클릭 같은 실제 운영 상황과 같은 문제**라, 대응해두면 운영에서도 그대로 효과가 있습니다. 멱등성의 최종 보루는 DB UNIQUE 제약입니다.

## 5. 티어 변경 시 차액 계산의 복잡도

**문제** 월 중간에 베이직(7,400원) → 프리미엄(20,600원) 변경 시 차액 13,200원 처리가 복잡합니다. 일할 계산 + 부분 환불은 **사용자에게 설명하기 어렵고 운영 부담도 큽니다.**

**해결 — `pending_tier` 예약 매커니즘**

차액을 계산하는 대신 **적용 시점을 미룹니다.**

```
[5/14] 베이직 결제        → tier='basic',   pending_tier=NULL
[5/20] 프리미엄 변경 요청 → tier='basic' 유지, pending_tier='premium'
[6/14] 결제 시점          → effectiveTier='premium'으로 청구
                           → tier='premium', pending_tier=NULL
```

마이그레이션 001로 컬럼 추가 + change-tier API + `processConfirmedPayment` 적용 분기 3곳에 통합.

**교훈** 정책 결정은 기술뿐 아니라 **운영 부담과 사용자 이해도까지** 고려해야 합니다. 기술적으로 가능한 것과 운영할 수 있는 것은 다릅니다.

## 6. 결제 안 한 좀비 active 구독

**문제** 단건 결제 모델이라 자동 갱신이 없는데, `next_billing_at`이 지나도 status는 계속 `active`로 남습니다 → **돈 안 낸 사람이 보상을 받는** 사고.

**원인** 초기 status가 `active` / `cancelled` 둘뿐이라 **"결제 끊김"을 표현할 상태가 없었습니다.**

**해결**

1. 마이그레이션 002에서 `status ENUM`에 `expired` 추가 — **`cancelled`(본인 의지)와 분리**
2. Cloud Scheduler가 매일 새벽 KST 03:00에 자동 만료 처리

```sql
UPDATE subscriptions SET status='expired'
WHERE status='active' AND next_billing_at < UTC_TIMESTAMP()
```

- Bearer 토큰 인증으로 외부 임의 호출 차단
- 5xx 자동 재시도

**교훈** 시간 기반 자동 처리가 좀비 데이터를 막습니다. **종료 사유를 구분해두면** 나중에 "이탈률"과 "결제 실패율"을 따로 볼 수 있습니다.

## 7. UTC와 KST 혼용으로 결제일이 어긋남

**문제** 한국 5/31 밤 11시 결제 → 서버 `NOW()`가 UTC라 6/1 02시로 기록 → 사용자는 "5월 결제"로 인식하는데 시스템은 **"6월 결제"로 처리**합니다.

**원인** MySQL `NOW()`가 서버 timezone(GCP 기본 UTC) 의존.

**해결 — 컬럼의 성격에 따라 정책을 나눴습니다**

| 컬럼 종류 | 타임존 | 이유 |
| --- | --- | --- |
| DATETIME (`started_at`, `paid_at`, `cancelled_at`) | **UTC** (`UTC_TIMESTAMP()` 명시) | 서버 timezone이 바뀌어도 값이 안 흔들림 |
| YEAR / MONTH (`billing_year`, `billing_month`) | **KST 계산** | "한국 사용자의 5월"이라는 **의미**를 담는 값 |

```ts
const kst = new Date(now.getTime() + 9 * 60 * 60 * 1000)
const billingYear = kst.getUTCFullYear()
const billingMonth = kst.getUTCMonth() + 1
```

**교훈** 글로벌 서비스가 아니어도 timezone 정책은 처음부터 명확히. **코드 헤더 주석에 명시해** 후속 컬럼도 같은 패턴을 따르도록 했습니다.

## 8. React 19 strict purity가 정상 코드를 막음

**문제** `Date.now()` / `Math.random()`을 이벤트 핸들러 안에서 호출했는데 `react-hooks/purity` lint가 **빌드를 실패**시켰습니다.

**원인** React 19 strict 룰이 컴포넌트 본체 안의 불순 함수 호출을 전부 잡습니다. 이벤트 핸들러는 렌더 중 실행되지 않지만 **정적 분석으로는 구분할 수 없습니다.**

**해결** 룰을 끄지 않고 모듈 레벨 헬퍼로 분리했습니다.

```ts
// 모듈 레벨 — lint 통과
function generateOrderId() {
  return `parami_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`
}
```

**교훈** lint false positive는 **룰을 끄지 말고 패턴을 바꿔 우회**합니다. 결과적으로 "이건 렌더와 무관한 함수"라는 의도가 코드 구조에 드러나 더 명확해졌습니다.

---

# 보안 · 설계 판단

## `toss_payment_key` 를 응답에서 제외

이 값은 **환불 권한 토큰**이라 클라이언트에 노출되면 환불 사기가 가능합니다. 실수로 흘리지 않도록 TypeScript 타입으로 강제했습니다.

```ts
export type PaymentPublic = Omit<PaymentRow, 'toss_payment_key'>
```

SELECT 쿼리에서도 해당 컬럼을 빼고 가져옵니다. **타입과 쿼리 양쪽에서** 막아, 한쪽을 잊어도 다른 쪽이 걸립니다.

## PR 리뷰에서 잡힌 것 — 코드와 안내문 불일치

| 안내문 | 실제 동작 | 조치 |
| --- | --- | --- |
| "매월 N일에 **자동 결제**됩니다" | 단건 결제라 자동 갱신 없음 | "자동으로 갱신되지 않아요. 매월 N일에 직접 결제해야 이어서 이용할 수 있어요." |
| "카드 또는 **계좌이체** 선택 가능" | `requestPayment('카드', ...)` — 카드만 | "토스페이먼츠를 통해 카드로 안전하게 결제돼요." |

**교훈** 디자인 시안을 그대로 따르기 전에 **비즈니스 정책을 먼저 확인**해야 합니다. 코드와 안내문이 어긋나면 사용자 신뢰를 잃는데, 이건 버그로 안 잡힙니다. 향후 멀티 결제수단 도입 시 `@tosspayments/payment-widget-sdk` 전환이 필요하다는 것도 코드 주석에 남겨뒀습니다.

## UX 라이팅을 근거 기반으로 통일

초기 UI 문구가 `~합니다` / `~해요` 혼재였습니다. 취향으로 정하지 않고 **Apps in Toss UX 라이팅 가이드**를 찾아 적용했습니다.

- 제품 UI 문구: `~돼요 / ~해요 / ~예요` 통일 (4개 페이지 + 토스트 전수 검토)
- 경어 의문형(`~시나요? / ~셨나요?`)은 가이드가 인정하는 예외로 유지
- 약관 본문은 제품 UI가 아니므로 격식체 유지

**교훈** 톤은 "이게 더 예쁜 것 같다"로 정하면 리뷰에서 매번 다시 논쟁합니다. **가이드 문서를 인용할 수 있으면** 한 번에 끝납니다.

---
---

# Parami

Parami는 기상 데이터가 정해진 조건을 넘으면 별도 청구 없이 보상금을 자동 지급하는 날씨 기반 구독형 보상 서비스입니다. 사용자는 플랜을 구독하고, 시스템은 정기적으로 날씨를 수집해 트리거를 판정한 뒤 자격 있는 사용자에게 포인트를 적립합니다.

## 핵심 기능

| 영역 | 내용 |
| --- | --- |
| 랜딩 | 서비스 소개, 트리거 조건, 월별 트리거 차트, 요금제 안내 |
| 인증 | 이메일 회원가입/로그인, JWT 쿠키 세션, 로그아웃, 회원 탈퇴 |
| 구독 | Basic/Standard/Premium 플랜, 티어 변경 예약, 구독 해지, 만료 처리 |
| 결제 | Toss Payments 단건 결제 승인, 금액 검증, DB 실패 시 자동 환불 시도 |
| 날씨 | 기상청/에어코리아 API 기반 현재 날씨 수집 |
| 트리거 | 강수, 폭염, 한파, 눈, 미세먼지, 맑은 날 보너스 판정 |
| 보상 | 티어별 보상금 적립, 일/월 캡, 중복 지급 방지 |
| 출금 | 포인트 출금 요청 기록 및 관리자 조회 |
| 관리자 | 유저, 결제, 트리거, 보상, 출금, 대시보드 조회 |

## 기술 스택

| 구분 | 기술 |
| --- | --- |
| Framework | Next.js 16.2.6 App Router |
| UI | React 19, Tailwind CSS v4, shadcn/ui 기반 컴포넌트 |
| State | TanStack Query |
| DB | MySQL, mysql2/promise |
| Auth | JWT, bcryptjs |
| Payment | Toss Payments SDK/API |
| Weather | KMA API, AirKorea API |
| Chart | Recharts |
| Motion/Icon | framer-motion, lucide-react |
| Deploy | Docker, GCP Cloud Run 기준 설정 |
| Scheduler | GCP Cloud Scheduler -> Next.js API Route |

> 이 프로젝트는 Next.js 16 기준입니다. 예전 Next.js 지식과 다른 부분이 있으므로 라우팅/설정 변경 전 `node_modules/next/dist/docs/` 문서를 확인하세요. 특히 기존 `middleware.ts` 대신 `proxy.ts`를 사용합니다.

## 서비스 흐름

```mermaid
flowchart LR
  User[사용자] --> Auth[회원가입/로그인]
  Auth --> Plan[요금제 선택]
  Plan --> Toss[Toss Payments 결제]
  Toss --> Sub[구독 활성화]

  Scheduler[GCP Cloud Scheduler] --> WeatherAPI[기상청/에어코리아 API]
  WeatherAPI --> WeatherLog[weather_logs 저장]
  WeatherLog --> Trigger[트리거 판정]
  Trigger --> TriggerLog[trigger_logs 저장]
  TriggerLog --> Reward[보상 지급 트랜잭션]
  Reward --> Balance[users.balance 증가]

  Admin[관리자] --> AdminAPI[관리자 API]
  AdminAPI --> DB[(MySQL)]
  Balance --> DB
  Sub --> DB
```

## 프로젝트 구조

```text
.
├─ app/
│  ├─ page.tsx                         # 랜딩 페이지
│  ├─ layout.tsx                       # 공통 레이아웃, Header/Footer/Providers
│  ├─ globals.css                      # Tailwind v4 @theme 디자인 토큰
│  ├─ (auth)/                          # 로그인, 회원가입
│  ├─ (consumer)/                      # 홈, 마이페이지, 결제, 보상, 날씨, 해지
│  ├─ admin/                           # 관리자 화면
│  └─ api/                             # 인증/결제/구독/날씨/보상/스케줄러/관리자 API
├─ components/
│  ├─ ui/                              # Button, Input, Card, Badge, Dialog 등
│  ├─ landing/                         # 랜딩 섹션 컴포넌트
│  ├─ header.tsx
│  ├─ header-weather.tsx
│  ├─ pricing-section.tsx
│  ├─ reward-calendar.tsx
│  └─ providers.tsx                    # TanStack Query, sonner Toaster
├─ lib/
│  ├─ db.ts                            # MySQL pool
│  ├─ auth.ts                          # API Route 인증 유틸
│  ├─ auth-server.ts                   # Server Component 세션 조회
│  ├─ client.ts                        # 프론트 API fetcher
│  ├─ weather.ts                       # 외부 날씨 API 호출/파싱
│  ├─ triggers.ts                      # 트리거 판정
│  ├─ rewards.ts                       # 보상 금액/KST 유틸
│  ├─ conditions.ts                    # 트리거 조건 상수
│  ├─ toss.ts                          # Toss API 헬퍼
│  └─ queries/                         # 도메인별 DB 쿼리
├─ db/
│  ├─ schema.sql                       # 기본 스키마 및 plans seed
│  ├─ schema.erd.json
│  └─ migrations/                      # 운영 반영용 증분 SQL
├─ scripts/
│  └─ check-db.mjs                     # DB 스키마/시드 sanity check
├─ types/
│  └─ db.ts
├─ proxy.ts                            # Next.js 16 라우트 가드
├─ Dockerfile
└─ next.config.ts                      # standalone output
```

## 시작하기

### 1. 의존성 설치

```bash
npm install
```

### 2. 환경변수 설정

`.env.local.example`을 복사해 `.env.local`을 만들고 값을 채웁니다.

```bash
cp .env.local.example .env.local
```

필수 환경변수:

| 이름 | 설명 |
| --- | --- |
| `DB_HOST` | MySQL 호스트 |
| `DB_PORT` | MySQL 포트, 기본값 3306 |
| `DB_USER` | MySQL 사용자 |
| `DB_PASSWORD` | MySQL 비밀번호 |
| `DB_NAME` | 사용할 DB 이름 |
| `JWT_SECRET` | JWT 서명용 secret |
| `KMA_API_KEY` | 기상청 API 키 |
| `AIRKOREA_API_KEY` | 에어코리아 API 키 |
| `NEXT_PUBLIC_TOSS_CLIENT_KEY` | 브라우저에서 사용하는 Toss client key |
| `TOSS_SECRET_KEY` | 서버 결제 승인용 Toss secret key |
| `SCHEDULER_SECRET` | 스케줄러 API 보호용 Bearer secret |

### 3. DB 준비

`db/schema.sql`을 MySQL에 적용합니다. 이 파일은 기본 테이블과 `plans` seed를 포함합니다.

```bash
mysql -h <host> -u <user> -p <db_name> < db/schema.sql
```

운영 DB나 이미 생성된 DB에는 `db/migrations/`의 SQL을 순서대로 적용합니다.

```text
001_subscriptions_add_pending_tier.sql
002_subscriptions_status_add_expired.sql
003_users_add_deleted_at.sql
004_create_withdrawal_logs.sql
005_reward_amounts_increase.sql
006_user_accounts_local_only.sql
```

DB 상태 확인:

```bash
npm run check-db
```

### 4. 개발 서버 실행

```bash
npm run dev
```

브라우저에서 `http://localhost:3000`으로 접속합니다.

## 스크립트

| 명령어 | 설명 |
| --- | --- |
| `npm run dev` | 개발 서버 실행 |
| `npm run build` | 프로덕션 빌드 및 타입 체크 |
| `npm run start` | 빌드 결과 실행 |
| `npm run lint` | ESLint 검사 |
| `npm run check-db` | DB 테이블/컬럼/seed sanity check |

## 주요 라우트

### 사용자 화면

| 경로 | 설명 |
| --- | --- |
| `/` | 랜딩 |
| `/login` | 로그인 |
| `/signup` | 회원가입 |
| `/home` | 로그인 후 홈 |
| `/weather` | 현재 날씨 및 예보 |
| `/pricing` | 요금제 선택/변경 |
| `/payment` | 결제 |
| `/payment/complete` | 결제 완료 |
| `/mypage` | 내 정보, 구독, 보상, 출금, 탈퇴 |
| `/rewards` | 보상 내역 |
| `/cancel-subscription` | 구독 해지 |

### 관리자 화면

| 경로 | 설명 |
| --- | --- |
| `/admin/dashboard` | 관리자 대시보드 |
| `/admin/users` | 유저 목록 |
| `/admin/payments` | 결제 내역 |
| `/admin/triggers` | 트리거 발동 내역 |
| `/admin/rewards` | 보상 지급 내역 |
| `/admin/withdrawals` | 출금 내역 |

## API 요약

API 응답은 기본적으로 아래 envelope를 사용합니다.

```ts
// 성공
{ success: true, data: unknown }

// 실패
{ success: false, error: string }
```

### 인증

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `POST` | `/api/auth/signup` | 회원가입 |
| `POST` | `/api/auth/login` | 로그인 |
| `POST` | `/api/auth/logout` | 로그아웃 |
| `GET` | `/api/auth/me` | 내 정보 조회 |
| `PATCH` | `/api/auth/profile` | 내 프로필 수정 |
| `DELETE` | `/api/auth/withdraw` | 회원 탈퇴 |

### 결제/구독

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `GET` | `/api/plans` | 플랜 목록 |
| `POST` | `/api/payments/confirm` | Toss 결제 승인 및 구독 반영 |
| `GET` | `/api/payments/me` | 내 결제 내역 |
| `GET` | `/api/subscriptions/me` | 내 구독 상태 |
| `PATCH` | `/api/subscriptions/change-tier` | 티어 변경 예약 |
| `PATCH` | `/api/subscriptions/cancel` | 구독 해지 |

### 날씨/보상/출금

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `GET` | `/api/weather/current` | 현재 날씨 |
| `GET` | `/api/rewards/me` | 내 보상 내역 |
| `GET` | `/api/rewards/summary` | 내 보상 요약 |
| `POST` | `/api/rewards/withdraw` | 포인트 출금 요청 |

### 스케줄러

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `POST` | `/api/scheduler/weather-check` | 날씨 수집, 트리거 판정, 보상 지급 |
| `POST` | `/api/scheduler/expire-subscriptions` | 결제 만료 구독을 `expired` 처리 |

스케줄러 호출 예시:

```bash
curl -X POST http://localhost:3000/api/scheduler/weather-check \
  -H "Authorization: Bearer $SCHEDULER_SECRET"
```

### 관리자

| Method | Endpoint | 설명 |
| --- | --- | --- |
| `GET` | `/api/admin/dashboard-stats` | 관리자 대시보드 통계 |
| `GET` | `/api/admin/users` | 유저 목록 |
| `GET` | `/api/admin/subscriptions` | 구독 목록 |
| `GET` | `/api/admin/payments` | 결제 내역 |
| `GET` | `/api/admin/triggers` | 트리거 내역 |
| `GET` | `/api/admin/rewards` | 보상 지급 내역 |
| `GET` | `/api/admin/withdrawals` | 출금 내역 |

## DB 모델

| 테이블 | 역할 |
| --- | --- |
| `users` | 회원, 권한, 잔액, soft delete |
| `user_accounts` | 로그인 계정 (MVP는 자체 이메일 로그인만) |
| `plans` | basic/standard/premium 가격표 |
| `subscriptions` | 구독 상태, 현재 티어, 다음 결제일, 변경 예약 |
| `payments` | Toss 결제 내역 |
| `weather_logs` | 외부 날씨 API 수집 이력 |
| `trigger_logs` | 트리거 발동 이력 |
| `reward_logs` | 유저별 보상 지급 이력 |
| `withdrawal_logs` | 포인트 출금 이력 |

중요 제약:

| 제약 | 목적 |
| --- | --- |
| `trigger_logs.UNIQUE(trigger_type, triggered_date)` | 같은 트리거는 하루 1회만 발동 |
| `reward_logs.UNIQUE(user_id, trigger_log_id)` | 같은 트리거에 대한 중복 보상 방지 |
| `reward_logs.idx_user_reward_month` | 월 보상 캡 조회 최적화 |
| `withdrawal_logs.idx_user_withdrawn` | 사용자별 출금 이력 조회 |

## 트리거와 보상 정책

트리거 조건은 `lib/conditions.ts`에서 관리합니다.

| 트리거 | 조건 |
| --- | --- |
| 강수 | 강수량 3mm 이상 |
| 폭염 | 기온 33도 이상 |
| 한파 | 기온 -12도 이하 |
| 눈 | 기상청 PTY 코드가 2, 3, 6, 7 중 하나 |
| 미세먼지 | PM2.5 50 이상 |
| 맑은 날 보너스 | 4,5,6,9,10,11월 중 강수 1mm 이하, PM2.5 30 이하, 풍속 5m/s 이하 |

보상 금액:

| 티어 | 1회 보상 | 월 최대 |
| --- | ---: | ---: |
| Basic | 900원 | 9,000원 |
| Standard | 1,600원 | 16,000원 |
| Premium | 2,600원 | 26,000원 |

공통 정책:

- 월 최대 보상 횟수는 10회입니다.
- KST 기준으로 일/월 캡을 계산합니다.
- active 구독자이면서 최근 결제 성공 이력이 있는 유저만 지급 대상입니다.
- 보상 지급은 `reward_logs` INSERT와 `users.balance` 증가를 한 트랜잭션으로 처리합니다.

## 결제 정책

- 클라이언트가 보낸 금액을 신뢰하지 않고, 서버가 DB의 `plans.price`와 비교합니다.
- 기존 active 구독자가 `pending_tier`를 가지고 있으면 다음 결제 승인 시 해당 티어를 적용합니다.
- Toss 결제 승인 후 DB 처리에 실패하면 `cancelTossPayment`로 자동 환불을 시도합니다.
- 결제는 월 단위 구독처럼 운영하지만, 자동 갱신 결제가 아니라 단건 결제 승인 방식입니다.

## 인증/권한

- 로그인 성공 시 JWT를 쿠키에 저장합니다.
- API Route에서는 `lib/auth.ts`의 `requireUser`, `requireAdmin`을 사용합니다.
- Server Component에서는 `lib/auth-server.ts`의 `getServerSession`을 사용합니다.
- `proxy.ts`는 라우트 접근 가드 역할을 합니다. Next.js Edge 런타임 제약 때문에 JWT role 검증은 API Route 또는 Server Component 쪽에서 처리합니다.

## 프론트엔드 개발 규칙

- 프론트 API 호출은 `lib/client.ts`의 `api.get`, `api.post`, `api.patch`, `api.delete`를 사용합니다.
- 색상/라운드/폰트 토큰은 `app/globals.css`의 `@theme` 값을 우선 사용합니다.
- UI는 `components/ui/`의 Button, Input, Card, Badge, Dialog 등을 우선 사용합니다.
- 아이콘은 `lucide-react`를 사용합니다.
- 사용자 화면은 `app/(consumer)/`, 인증 화면은 `app/(auth)/`, 관리자 화면은 `app/admin/`에 둡니다.
- Next.js 16 관련 라우팅/설정 변경 시 로컬 문서 `node_modules/next/dist/docs/01-app/`를 먼저 확인합니다.

## 배포 메모

`next.config.ts`는 Cloud Run/Docker 배포를 위해 standalone output을 사용합니다.

```ts
const nextConfig = {
  output: "standalone",
}
```

Docker 빌드:

```bash
docker build -t parami .
docker run --env-file .env.local -p 3000:3000 parami
```

운영 환경에서는 최소한 아래 작업을 확인합니다.

- `.env.local`에 해당하는 secret을 Cloud Run 환경변수로 등록
- MySQL 접근 네트워크 설정
- Cloud Scheduler에서 `/api/scheduler/weather-check`와 `/api/scheduler/expire-subscriptions` 호출
- Scheduler 요청에 `Authorization: Bearer <SCHEDULER_SECRET>` 헤더 포함

## 품질 확인 체크리스트

PR 전 최소 확인:

```bash
npm run lint
npm run build
npm run check-db
```

기능별 추가 확인:

| 변경 영역 | 확인 |
| --- | --- |
| DB 변경 | `db/schema.sql`, `db/migrations/`, `scripts/check-db.mjs` 동기화 |
| 결제 변경 | 금액 검증, Toss 실패 응답, DB 실패 후 환불 분기 |
| 스케줄러 변경 | 중복 트리거, 월 캡, 트랜잭션, 재시도 멱등성 |
| 인증 변경 | 쿠키 세션, API 권한, 관리자 접근 제한 |
| UI 변경 | 모바일/데스크탑 레이아웃, 토큰 사용, lint/build |

## 브랜치/커밋 규칙

브랜치 예시:

```text
feature/이름-기능명
fix/이름-버그명
docs/이름-문서명
style/영역-작업명
```

커밋 타입:

| 타입 | 용도 |
| --- | --- |
| `feat` | 새 기능 |
| `fix` | 버그 수정 |
| `docs` | 문서 |
| `style` | UI/스타일 |
| `refactor` | 구조 개선 |
| `chore` | 설정/패키지/기타 |

## 팀 역할

| 담당 | 역할 |
| --- | --- |
| 우석 | 스케줄러, 트리거, 보상 지급, 관리자 조회 API |
| 소라 | 결제, 구독, 보상 조회 |
| 영현 | 인증 |
| 여진 | 소비자 페이지 |
| 명진 | 관리자 페이지 |
