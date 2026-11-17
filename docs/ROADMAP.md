# Notion 견적서 뷰어 개발 로드맵

운영자가 Notion에 작성한 견적서를 클라이언트가 로그인 없이 토큰 URL로 확인하고 PDF로 내려받게 하는 서비스입니다.

**📅 최종 업데이트**: 2026-11-17
**📄 문서 버전**: v1.3 (근거 문서: `docs/PRD.md`, v1.2 대비 외부망 배포 전제 확정으로 보안망 관련 항목 정리)
**📊 진행 상황**: Phase 1 대기 중 (0/15 Tasks 완료)

## 개요

Notion 견적서 뷰어는 Notion으로 견적을 관리하는 1인 개발자·소규모 프리랜서(운영자)와 견적서를 받는 클라이언트를 위한 서비스입니다. 관리자 화면 없이 Notion을 CMS로 쓰며, 다음 기능을 제공합니다.

- **견적서 조회 (F001)**: Notion 견적서 DB와 견적항목 DB를 서버에서 조회해 기본정보, 항목 표, 합계, 비고를 렌더링
- **토큰 기반 접근 제어 (F002)**: URL 토큰과 공개 상태를 서버에서 검증하고, 실패 사유와 무관하게 동일한 "접근 불가" 응답
- **PDF 다운로드 (F003)**: `@react-pdf/renderer`로 서버에서 PDF를 생성해 `견적서_{견적번호}.pdf`로 다운로드 (한글 폰트 번들)
- **상태/유효기간 표시 (F004)**: 유효기간 경과 시 만료 배지 표시, 미공개 상태 차단
- **캐싱/재검증 (F010)**: Cache Components(`'use cache'` + `cacheTag`/`cacheLife`)로 Notion 조회 결과 캐싱, Suspense 스트리밍으로 로딩 표시
- **인쇄 레이아웃 (F011)**: 브라우저 인쇄 시 불필요 UI를 숨기는 A4 기준 스타일
- **서비스 안내 (F012)**: 루트 접속 시 견적서 URL로 접속하라는 정적 안내

### 기술 기준

- Next.js 16.4.6 (App Router, Turbopack) + React 19 + TypeScript 5, TailwindCSS v4 + shadcn/ui (new-york), pnpm
- `next.config.ts`에 `typedRoutes`, `cacheComponents`, `partialPrefetching`이 켜져 있음
  - 라우트를 추가·변경한 뒤에는 `pnpm build`까지 돌려야 Route 타입 에러가 드러남
  - 동적 데이터(`params`, Notion 조회, 현재 시각)를 읽는 부분은 Suspense 경계 안에 둬야 함
- 데이터 소스는 `@notionhq/client`이며 서버에서만 사용. Notion 응답은 Zod로 파싱
- 테스트: 순수 로직·서버 모듈은 Vitest 단위 테스트(`pnpm test` = `vitest run`, Task 002에서 도입), 사용자 플로우·API 연동은 Playwright MCP로 E2E 검증
- 보안 원칙: Notion API 키는 서버 환경변수로만 사용, `.env*` 커밋 금지(`.gitignore` 등록 완료), 토큰 검증 등 보안 로직은 서버에서만 처리

### 라우트 구조 (계획)

| 경로                      | 페이지/핸들러              | 기능 ID                      | 비고                                       |
| ------------------------- | -------------------------- | ---------------------------- | ------------------------------------------ |
| `/`                       | 안내 페이지                | F012                         | 정적 페이지                                |
| `/quotes/[token]`         | 견적서 보기 페이지         | F001, F002, F004, F010, F011 | noindex, Suspense 스트리밍                 |
| `/quotes/[token]` 실패 시 | 접근 불가 안내 (not-found) | F002, F004                   | `notFound()`로 처리해 URL·응답 형태를 통일 |
| `/quotes/[token]/pdf`     | PDF 다운로드 Route Handler | F003                         | Node 런타임, `Cache-Control: no-store`     |

> 접근 불가 안내를 별도 경로로 리다이렉트하지 않고 `not-found.tsx`로 처리하는 이유: 리다이렉트를 쓰면 실패 사유별 응답 차이가 생기기 쉽고, 토큰이 리다이렉트 쿼리에 남을 수 있습니다. Task 008에서 최종 확정합니다.
>
> 견적서 경로의 보안 헤더(`X-Robots-Tag`, `Referrer-Policy`)는 Task 001에서 골격을 잡고, 접근 불가 응답 헤더는 Task 008, PDF 응답 헤더는 Task 010에서 최종 확정합니다.

### Phase 구성

| Phase   | 이름           | Task    | 핵심 산출물                                                                                                 |
| ------- | -------------- | ------- | ----------------------------------------------------------------------------------------------------------- |
| Phase 1 | 프로젝트 골격  | 001     | 스타터 정리, 라우트·레이아웃 골격, 보안 헤더 골격, `/tasks` 디렉토리                                        |
| Phase 2 | 공통 모듈      | 002~006 | 타입·Zod·환경변수·회사 상수, Vitest, Notion 클라이언트, `access.ts`, `validity.ts`, 포맷·금액 유틸, 공통 UI |
| Phase 3 | 핵심 기능      | 007~011 | F001~F004 + F010: 견적서 화면, 접근 제어·실데이터 조회·캐싱, PDF, 통합 테스트                               |
| Phase 4 | 추가 기능      | 012~013 | F011 인쇄 레이아웃, F012 안내 페이지                                                                        |
| Phase 5 | 최적화 및 배포 | 014~015 | 전체 E2E·보안·성능 점검, 배포 환경, 운영자 가이드, 스모크 테스트                                            |

## 개발 워크플로우

1. **작업 계획**
   - 기존 코드베이스를 학습하고 현재 상태를 파악
   - 새로운 작업을 포함하도록 `ROADMAP.md` 업데이트
   - 우선순위 작업은 마지막 완료된 작업 다음에 삽입
   - Phase 시작 전에 아래 "오픈 이슈 결정 게이트"에서 해당 Phase의 선결 이슈가 결정되었는지 확인

2. **작업 생성**
   - 기존 코드베이스를 학습하고 현재 상태를 파악
   - `/tasks` 디렉토리에 새 작업 파일 생성 (디렉토리와 `000-sample.md`는 Task 001 시작 시 함께 생성)
   - 명명 형식: `XXX-description.md` (예: `001-setup.md`)
   - 고수준 명세서, 관련 파일, 관련 기능 ID(F001 등), 수락 기준, 구현 단계 포함
   - **API/비즈니스 로직 작업 시 "## 테스트 체크리스트" 섹션 필수 포함 (Playwright MCP 테스트 시나리오 작성)**
   - 직전에 완료된 작업 파일 2개를 예시로 참조. 완료된 파일은 체크된 박스와 변경 사항 요약을 담고 있으므로, 새 작업 파일은 `000-sample.md`처럼 빈 박스로 시작하고 변경 사항 요약은 비워 둠

3. **작업 구현**
   - 작업 파일의 명세서를 따름
   - 기능과 기능성 구현
   - **API 연동 및 비즈니스 로직 구현 시 Playwright MCP로 테스트 수행 필수**
   - 각 단계 후 작업 파일 내 단계 진행 상황 업데이트
   - 구현 완료 후 Playwright MCP를 사용한 E2E 테스트 실행
   - `pnpm check-all` 통과 후 `pnpm build` 통과 확인 (라우트 변경 시 build 필수)
   - `pnpm test`(Vitest 단위 테스트) 통과 확인 (Vitest를 도입하는 Task 002 이후 모든 작업에 적용)
   - 테스트 통과 확인 후 다음 단계로 진행
   - 각 단계 완료 후 중단하고 추가 지시를 기다림

4. **로드맵 업데이트**
   - `/docs:update-roadmap`으로 완료된 작업과 하위 항목을 ✅로 표시
   - Phase 내 모든 Task가 끝나면 Phase 제목에 ✅ 추가, 진행 상황 라인 갱신

## 오픈 이슈 결정 게이트

PRD의 오픈 이슈 5건을 언제까지 결정해야 하는지 정리했습니다. 결정 전에는 해당 Task를 시작하지 않습니다.

| #   | 오픈 이슈                                             | 결정 시점 (게이트)                                                                           | 영향 Task          | MVP 기본 제안                                                         |
| --- | ----------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------ | --------------------------------------------------------------------- |
| 1   | 부가세 처리 (10% 고정 vs Notion 속성 입력)            | **Task 003 전** (Notion 스키마·타입 확정), 늦어도 **Task 005 전**(금액 재계산 유틸)          | 002, 003, 005, 008 | Notion `세금` 수식 속성(소계 × 10%)을 읽고 서버에서 재계산해 검증     |
| 2   | 토큰 URL 영구 유효 vs 상태 변경으로만 차단            | **Task 004 전** (Phase 2 중, `access.ts` 공개 조건 확정)                                     | 004, 008           | 영구 유효, `상태`(발송·확정만 공개)로만 차단                          |
| 3   | 로고/회사 정보/계좌 정보: 고정 텍스트 vs Notion 속성  | **Task 007 전**(화면 레이아웃), 늦어도 **Task 009 전**(PDF 레이아웃). Task 002는 자리만 마련 | 002, 007, 009, 013 | 서버 상수(`src/lib/company.ts`)로 고정, Notion 속성화는 MVP 이후      |
| 4-a | 배포 환경 (보안망 여부)                               | **결정 완료**                                                                                | 015                | 학습용이라 외부망(Vercel 등)에 배포, 보안망 제약은 고려하지 않음      |
| 4-b | 한글 폰트 라이선스                                    | **Task 009 전** (Phase 3 PDF 작업 착수 전)                                                   | 009                | Noto Sans KR(OFL) TTF 번들 + 라이선스 파일 동봉                       |
| 5   | 항목 수가 많은 견적서(수십 행)의 PDF 페이지 분할 규칙 | **Task 009 전** (Phase 3 PDF 작업 착수 전)                                                   | 009                | 표 헤더 반복 + 행 단위 분할(행 중간 잘림 금지), 합계 블록은 분할 금지 |

## 의존성 개요

```
[Phase 1] Task 001 (골격)
              │
              v
[Phase 2] Task 002 (타입/Zod/환경변수/회사 상수 + Vitest 도입)
              ├─> Task 003 (Notion 연동) ─> Task 004 (access.ts 접근 제어 모듈)
              ├─> Task 005 (validity.ts · 포맷 · 금액 재계산 유틸)   (003·004와 병렬 가능)
              └─> Task 006 (공통 UI 컴포넌트 · fixtures)             (003·004와 병렬 가능)
              │
              v
[Phase 3] Task 007 (화면 UI 더미, 005·006) ─> Task 008 (접근 제어 연동·실데이터 조회·캐싱, 004) ─┐
          Task 009 (PDF 레이아웃·폰트, 005·006) ───────────────────────────────────────────────┤
                                                                                                v
                                                                 Task 010 (PDF 다운로드, 004·008·009)
                                                                                                │
                                                                                                v
                                                                 Task 011 (핵심 기능 통합 테스트)
              │
              v
[Phase 4] Task 012 (인쇄 레이아웃, 007 이후)    Task 013 (안내 페이지, 002 이후 언제든 착수 가능)
              │
              v
[Phase 5] Task 014 (전체 E2E·보안·성능 점검) ─> Task 015 (배포·운영 가이드·스모크 테스트)
```

- Phase 2에서 Notion 계열(Task 003 → 004)과 순수 로직·UI 계열(Task 005, 006)은 병렬로 진행할 수 있습니다. 모두 Task 002의 타입과 Vitest 설정에만 의존합니다.
- Phase 3의 Task 007(화면 UI)과 Task 009(PDF 레이아웃)는 Task 006의 fixtures로 먼저 만들 수 있어, Notion 연동(Task 008)과 병렬이 가능합니다. 1인 개발 기준으로는 번호 순서대로 진행하는 것을 권장합니다.
- Task 008은 접근 제어 연동과 실데이터 조회·캐싱을 한 번에 끝내므로, 정상 토큰인데 더미 데이터가 보이는 중간 상태가 생기지 않습니다.
- `access.ts`(Task 004)와 `validity.ts`·포맷 유틸(Task 005)은 화면(Task 007~008)과 PDF(Task 009~010)가 공용으로 재사용하므로, 각 기능 Task에서 같은 로직을 다시 만들지 않습니다.
- Task 013(안내 페이지)은 Notion·견적 데이터에 의존하지 않아 Task 002 이후 언제든 착수할 수 있습니다.

## 개발 단계

### Phase 1: 프로젝트 골격

**목표**: 스타터 템플릿을 견적서 뷰어 구조로 정리하고, 라우트·레이아웃·보안 헤더 골격과 작업 관리 디렉토리를 먼저 갖춥니다.
**선결 이슈**: 없음 (#4-a 배포 환경은 외부망으로 결정 완료)

- **Task 001: 스타터 정리 및 라우트 골격 구축** - 우선순위
  - 관련 기능: F001, F002, F012 (구조)
  - 스타터 예제 제거: `src/app/login`, `src/app/signup`, `login-form.tsx`, `signup-form.tsx`, `components/sections/*`, 헤더 내비게이션 메뉴(PRD: 헤더는 로고 텍스트만)
  - 빈 라우트 생성: `src/app/page.tsx`(안내), `src/app/quotes/[token]/page.tsx`, `src/app/quotes/[token]/not-found.tsx`, `src/app/quotes/[token]/loading.tsx` 또는 Suspense 경계, `src/app/quotes/[token]/pdf/route.ts`(빈 핸들러)
  - 공통 레이아웃 정리: 로고 텍스트 헤더, 최소 푸터, `print:hidden` 적용 대상 표시
  - 견적서 경로 noindex 골격: `generateMetadata`에서 `robots: { index: false, follow: false }`, `next.config.ts` headers에 `/quotes/:path*` 대상 `X-Robots-Tag: noindex, nofollow` 및 `Referrer-Policy: no-referrer` 추가(토큰이 Referer로 외부 유출되지 않도록)
  - Route 타입 확인: 동적 경로 링크/리다이렉트가 있으면 `Route` 타입으로 작성
  - `/tasks` 디렉토리와 `000-sample.md` 생성
  - 완료 기준: 세 경로가 빈 화면으로 렌더링되고 `pnpm check-all`, `pnpm build` 통과

**Phase 1 테스트 체크리스트 (Playwright MCP)**

- `/`, `/quotes/{임의 토큰}`이 빈 화면으로 렌더링되고, 삭제한 `/login`, `/signup`은 404
- `/quotes/{임의 토큰}` 응답에 `X-Robots-Tag: noindex, nofollow`, `Referrer-Policy: no-referrer` 헤더와 `robots` noindex 메타 존재
- 헤더에 로고 텍스트만 있고 내비게이션 메뉴가 없음

**Phase 1 완료 조건**

- 스타터 예제 코드가 제거되고 3개 페이지 + PDF 핸들러 경로가 존재
- 견적서 경로 noindex·`Referrer-Policy` 헤더 골격 적용
- `/tasks` 디렉토리와 `000-sample.md` 생성
- `pnpm check-all` 통과
- `pnpm build` 통과
- (Vitest는 Task 002에서 도입하므로 `pnpm test`는 Phase 2부터 완료 조건에 포함)

### Phase 2: 공통 모듈

**목표**: 화면·PDF·조회 계층이 함께 쓰는 타입, 환경변수, 단위 테스트 환경, Notion 클라이언트, 접근 제어·유효기간 판정·포맷·금액 유틸, 공통 UI를 한 번만 만들어 둡니다.
**선결 이슈**: #1 부가세 처리(Task 003 전, 늦어도 Task 005 전), #2 토큰 유효 정책(Task 004 전), #3 회사 정보 관리 방식(Task 002는 자리만 마련, 결정은 Task 007 전)

- **Task 002: 도메인 타입, 환경변수 스키마, 회사 정보 상수 정의 및 Vitest 도입**
  - 관련 기능: F001, F002, F003, F004
  - 도메인 타입: `src/lib/quotes/types.ts`에 `Quote`, `QuoteItem`, `QuoteStatus`(작성중/발송/확정/취소), `QuoteView`(화면/PDF 공용 표시 모델), `QuoteLookupResult`(`ok` | `unavailable` 판별 유니온)
  - Zod 스키마: 토큰 형식(예: `^[A-Za-z0-9_-]{32,128}$`), Notion 페이지 속성 파싱 스키마(견적서/견적항목)
  - 환경변수: `src/lib/env.ts`에 `NOTION_API_KEY`, `NOTION_QUOTES_DB_ID`(또는 data source ID), `NOTION_QUOTE_ITEMS_DB_ID`, 캐시 만료 설정값 추가. 서버 전용 모듈로 분리(`import 'server-only'`)해 클라이언트 번들 유입 차단
  - `.env.example` 작성(키 이름만, 값 비움) 및 커밋. `.env.local`은 커밋 금지 재확인
  - 회사 정보 상수 자리 마련(`src/lib/company.ts`: 상호, 연락처, 계좌 등 키 구조만 정의, 값은 오픈 이슈 #3 결과 반영)
  - Vitest 도입(이 Task에 두는 이유: 첫 단위 테스트 대상인 토큰 형식 Zod 스키마가 여기서 생기고, Phase 1에는 테스트할 로직이 없어 빈 러너만 남기 때문)
    - `pnpm add -D vitest`, `package.json`에 `"test": "vitest run"` 추가(감시 모드가 필요하면 `"test:watch": "vitest"`)
    - `vitest.config.ts`: `@/` → `./src` 경로 별칭 반영(`resolve.alias` 또는 Next 공식 예제의 `vite-tsconfig-paths` 중 이 Task에서 확정), 서버 모듈 위주이므로 `test.environment: 'node'`, 테스트 파일은 `src/**/*.test.ts`로 소스 옆에 배치
    - 서버 전용 모듈 처리: Vitest에는 Next의 `react-server` 조건이 없어 `import 'server-only'`가 있는 모듈을 그대로 import하면 에러가 남. `vitest.setup.ts`에서 `vi.mock('server-only', () => ({}))`로 무력화하고 `setupFiles`에 등록(Next 공식 `with-vitest` 예제 방식)
    - `next/cache`(`cacheTag`/`cacheLife`)를 import하는 모듈을 테스트할 때는 해당 함수를 `vi.mock`으로 대체하고, `'use cache'` 동작 자체는 단위 테스트 대상에서 제외(Playwright MCP로 검증)
    - 테스트 코드는 `vi.stubEnv`로 환경변수를 주입하고 실제 Notion 키를 쓰지 않음
    - `check-all` 스크립트에 `pnpm test`를 포함할지 이 Task에서 확정(포함하지 않으면 워크플로우대로 `pnpm test`를 따로 실행)
  - 단위 테스트: 토큰 형식 Zod 스키마(정상, 31자, 129자, 허용 외 문자, 공백·URL 인코딩 문자)
  - 완료 기준: 타입만으로 화면·PDF·조회 계층이 공유할 인터페이스가 확정되고 `pnpm check-all`, `pnpm test` 통과

- **Task 003: Notion DB 구성 및 서버 전용 클라이언트 연동**
  - 관련 기능: F001, F002, F010
  - Notion 측 준비: 견적서 DB(견적번호, 클라이언트명, 공유토큰, 상태, 발행일, 유효기간, 항목 관계, 소계/세금/합계, 비고)와 견적항목 DB(항목명, 견적서 관계, 수량, 단가, 금액) 구성, Integration 생성 후 두 DB에 연결
  - 테스트 데이터 준비: 발송(유효), 확정, 작성중, 취소, 유효기간 경과, 유효기간 당일, 항목 30행 이상, 비고 없음 케이스를 각 1건 이상
  - `pnpm add @notionhq/client`, `src/lib/notion/client.ts`(서버 전용) 작성. 사용하는 SDK 버전의 조회 API(databases/data sources)를 context7로 확인 후 적용
  - 조회 함수 골격: `queryQuoteByToken(token)`(공유토큰 일치 + 상태 필터를 Notion 쿼리 조건으로), `queryQuoteItems(quoteId)`(관계 필터, 100건 초과 시 페이지네이션)
  - 응답 매퍼: Notion 속성(제목, rich_text, select, date, number, rollup, formula)을 Zod로 파싱해 `Quote`/`QuoteItem`으로 변환. 파싱 실패는 예외로 던지고 내부 로그에만 상세 기록
  - 오류 처리: Notion 4xx/5xx·레이트 리밋(429) 구분 로깅, 재시도는 1회 이내로 제한
  - 단위 테스트: 응답 매퍼에 Notion 응답 샘플(JSON 픽스처)을 넣어 정상 변환과 파싱 실패 예외를 확인. 실제 Notion 호출은 단위 테스트에서 하지 않음
  - 완료 기준: 서버 전용 스크립트 또는 임시 서버 컴포넌트에서 테스트 데이터가 타입대로 조회되고, 클라이언트 번들에 `NOTION_API_KEY`가 없는지(`pnpm build` 후 `.next/static` 검색) 확인

- **Task 004: 토큰 검증 및 접근 제어 모듈 구현**
  - 관련 기능: F002, F004(미공개 차단)
  - `src/lib/quotes/access.ts`(서버 전용, `import 'server-only'`): 견적서 페이지(Task 008)와 PDF Route Handler(Task 010)가 공용으로 재사용하는 단일 진입 함수(예: `resolveQuoteAccess(token): Promise<QuoteLookupResult>`)
  - 토큰 형식 검증(Zod) 실패 시 Notion 호출 없이 즉시 `unavailable` 반환
  - 공개 조건: `공유토큰` 정확히 일치 + `상태 ∈ {발송, 확정}`(오픈 이슈 #2 결과 반영). 불일치/미공개/없음/Notion 조회 실패를 모두 동일한 `unavailable`로 수렴
  - 로깅: 실패 사유는 서버 로그에만 기록하고, 토큰은 앞 4자리만 남기고 마스킹. 반환값에는 사유를 담지 않음
  - 조회는 Task 003의 함수를 사용하고, Task 008에서 캐시 함수로 교체할 수 있도록 조회 의존성을 한 곳에 모음
  - 단위 테스트(Vitest, Notion 조회 함수는 `vi.mock`으로 모킹): 발송·확정 → `ok`, 작성중·취소·없음·조회 예외 → 동일한 `unavailable`, 형식 오류 토큰은 조회 함수가 호출되지 않음, 로그 출력에 토큰 원문이 없음
  - 완료 기준: 위 단위 테스트가 통과하고, Notion 테스트 데이터로도 발송·확정만 `ok`가 되며 `pnpm check-all`, `pnpm test` 통과

- **Task 005: 유효기간 판정 및 포맷·금액 재계산 유틸 구현**
  - 관련 기능: F001, F003, F004
  - `src/lib/quotes/validity.ts`: 유효기간 경과 판정 함수. 한국 시간(Asia/Seoul) 기준 날짜 비교, 유효기간 당일은 유효로 처리. 현재 시각은 인자로 주입받아 순수 함수로 유지(캐시 함수 안에서 호출하지 않도록)
  - 경계값 규칙: 유효기간 전날/당일/다음날, 유효기간 미입력 시 처리 규칙(기본 제안: 만료 배지 미표시)을 이 Task에서 확정
  - 금액·날짜 포맷 유틸(`src/lib/format.ts`): `Intl.NumberFormat('ko-KR')` 원화 표기, 날짜 `YYYY.MM.DD` 표기(Asia/Seoul 기준, 화면·PDF 공용)
  - 금액 재계산 유틸(`src/lib/quotes/amounts.ts`): 항목별 `수량 × 단가`, 소계, 세금(오픈 이슈 #1 결과), 합계를 정수 연산으로 재계산하고 Notion 값과의 불일치 목록을 반환하는 순수 함수(로그·표시 정책은 Task 008에서 적용)
  - 단위 테스트(Vitest)
    - `validity.ts`: 전날·당일·다음날·미입력 경계값, 한국 시간 자정 직전·직후(UTC 기준 날짜가 다른 시각 포함)
    - 금액 재계산: 일치, 항목 금액 불일치, 합계 불일치, 0원, 항목 0건, 큰 금액(정수 범위)
    - 포맷 유틸: 0원, 음수 처리 여부, 천 단위 구분, `YYYY.MM.DD` 변환과 시간대 경계
  - 완료 기준: 위 단위 테스트가 모두 통과하고 `pnpm check-all`, `pnpm test` 통과

- **Task 006: 공통 UI 컴포넌트 및 더미 데이터 구성**
  - 관련 기능: F001, F003, F004
  - shadcn/ui 컴포넌트 확인·추가: Table, Badge, Button, Skeleton, Alert(스타터에 없는 Table은 `pnpm dlx shadcn@latest add table`)
  - 더미 데이터: `src/lib/quotes/fixtures.ts`에 `QuoteView` 형태로 정상/만료/다항목(30행 이상) 3종 작성. 화면(Task 007)과 PDF(Task 009) 개발, 단위 테스트에서 사용하며 Task 008 이후 실제 화면에는 쓰지 않음
  - 다크 모드·인쇄(`print:`) 변형에서 추가한 컴포넌트가 깨지지 않는지 확인
  - 완료 기준: 추가한 컴포넌트와 fixtures가 타입 에러 없이 import되고 `pnpm check-all`, `pnpm test`, `pnpm build` 통과

**Phase 2 테스트 체크리스트 (Vitest 단위 테스트 + 조회 확인)**

- (Vitest) 토큰 형식 Zod 스키마가 정상 토큰만 통과시키고 길이·문자 규칙 위반은 거부
- (Vitest) Notion 응답 매퍼가 샘플 응답을 `Quote`/`QuoteItem`으로 변환하고, 형식이 다른 응답에서는 예외를 던짐
- (Vitest, Notion 모킹) `access.ts`가 발송·확정만 `ok`, 나머지(작성중/취소/없음/형식 오류/조회 예외)는 동일한 `unavailable` 반환, 형식 오류 토큰은 조회 함수 호출 없음, 로그에 토큰 원문 없음
- (Vitest) `validity.ts`가 유효기간 전날·당일은 유효, 다음날은 만료, 미입력은 확정한 규칙대로 판정 (Asia/Seoul 기준, 자정 전후 시각 포함)
- (Vitest) 금액 재계산 결과가 기대값과 일치하고, 불일치 데이터에서는 불일치 목록이 반환
- (Vitest) 포맷 유틸이 원화·날짜를 화면·PDF 공용 규칙대로 출력
- (서버 전용 스크립트) Notion 테스트 데이터가 `Quote`/`QuoteItem` 타입대로 조회되고, 30행 이상 견적서의 항목이 누락 없이 반환
- `pnpm build` 후 `.next/static`에 `NOTION_API_KEY` 값과 Notion DB ID가 없음

**Phase 2 완료 조건**

- 도메인 타입, Zod 스키마, 서버 전용 환경변수 모듈, 회사 정보 상수 자리가 확정
- Vitest 설정(`vitest.config.ts`, `server-only` 모킹, `@/` 별칭)과 `pnpm test` 스크립트 준비 완료
- Notion 테스트 데이터가 서버에서 타입대로 조회됨
- `access.ts`, `validity.ts`, 포맷·금액 재계산 유틸, 공통 UI 컴포넌트, fixtures 준비 완료
- 오픈 이슈 #1, #2 결정 반영
- `pnpm check-all` 통과
- `pnpm test` 통과
- `pnpm build` 통과

### Phase 3: 핵심 기능

**목표**: PRD 핵심 기능 F001~F004와 이를 받치는 F010을 완성합니다. 더미 데이터로 화면을 먼저 만든 뒤 접근 제어와 실제 Notion 조회·캐싱으로 한 번에 교체하고, 같은 데이터로 PDF를 생성합니다.
**선결 이슈**: #3 회사 정보 관리 방식(Task 007 전, PDF 포함 여부는 Task 009 전), #4-b 폰트 라이선스(Task 009 전), #5 PDF 페이지 분할 규칙(Task 009 전)

- **Task 007: 견적서 보기 화면 UI 구현 (더미 데이터, 상태·만료 배지 포함)**
  - 관련 기능: F001, F004(상태·만료 배지 UI), F011(인쇄 대상 구분)
  - 컴포넌트: `QuoteHeader`(견적번호/클라이언트명/발행일/유효기간/상태 배지), `QuoteStatusBadge`(발송/확정 + "만료" 배지), `QuoteItemsTable`(데스크톱 표), `QuoteItemsCards`(모바일 카드), `QuoteSummary`(소계/세금/합계), `QuoteNotes`, `QuoteActions`(PDF·인쇄 버튼 자리)
  - 표시 값은 Task 005의 포맷 유틸과 `validity.ts`를 사용하고, Task 006 fixtures(정상/만료/다항목)로 렌더링
  - 로딩 스켈레톤(`QuoteSkeleton`)과 접근 불가 안내 화면(일반화 문구 "유효하지 않거나 열람할 수 없는 견적서입니다" + 운영자 문의 안내) 마크업
  - 반응형: 모바일 카드형 / `md` 이상 표형, 다크 모드 대응. 헤더·버튼 등 인쇄 제외 대상에 `print:hidden` 표시
  - 회사 정보 표시 위치는 오픈 이슈 #3 결과(`src/lib/company.ts`) 반영
  - 완료 기준: 더미 데이터로 세 화면(정상/만료/접근 불가)이 확인되고 만료 fixture에 "만료" 배지가 보이며 `pnpm check-all`, `pnpm test`, `pnpm build` 통과

- **Task 008: 접근 제어 연동 및 견적서 실데이터 조회·캐싱 구현**
  - 관련 기능: F001, F002, F004(미공개 차단, 요청 시점 만료 판정), F010
  - 캐시 함수: `getQuoteByToken(token)`에 `'use cache'` + `cacheLife`(수 분 단위, 예: revalidate 300초) + ``cacheTag('quotes', `quote:${quoteId}`)`` 적용. 토큰은 함수 인자로 전달되어 캐시 키에 포함됨. `access.ts`의 조회 의존성을 이 캐시 함수로 교체(`access.ts` 단위 테스트는 모킹 대상만 바꿔 계속 통과해야 함)
  - 캐시 범위 주의: Notion 조회 실패(예외)는 캐시되지 않도록 throw로 처리. 형식 검증을 통과한 토큰만 캐시 함수에 진입시키고, 존재하지 않는 토큰 결과의 캐시 여부와 캐시 키 증가 위험을 점검
  - 페이지 연동: `params`를 Suspense 경계 안에서 `await`하고 Task 004의 `access.ts`를 호출, `unavailable`이면 `notFound()` 호출. `ok`이면 Notion 응답을 `QuoteView`로 변환해 바로 렌더링(캐시 함수를 먼저 만든 뒤 페이지를 연결해, 정상 토큰에 더미 데이터가 보이는 중간 상태를 두지 않음)
  - 응답 동일성 점검: 사유별로 HTTP 상태 코드, 본문, 응답 헤더가 같은지 확인. 스트리밍 도중 `notFound()`가 호출될 때의 상태 코드 동작을 확인하고, 차이가 있으면 검증을 셸 렌더링 전에 끝내는 구조로 조정
  - `not-found.tsx` 처리 방식(리다이렉트 미사용)과 접근 불가 응답 헤더 최종 확정. 접근 불가 페이지 noindex 확인(`robots` 메타 + `X-Robots-Tag` 헤더)
  - 만료 판정 위치: 현재 시각 기반 계산(`validity.ts`)은 캐시 함수 밖, 요청 시점 렌더링 영역(Suspense 내부)에서 수행해 만료 여부가 캐시에 고정되지 않도록 처리
  - 금액 검증: Task 005 재계산 유틸로 Notion 값과 비교. 불일치 시 서버 경고 로그 + 표시 정책 적용(오픈 이슈 #1 결정에 따름, 기본은 Notion 값 표시)
  - Suspense 스트리밍: 정적 셸(헤더/레이아웃)은 즉시, 견적 본문은 스켈레톤 후 스트리밍
  - 더미 fixtures는 테스트·개발 확인용으로만 남기고 페이지 코드에서 import를 제거
  - 완료 기준: 유효 URL에서 실제 Notion 값(모든 항목/합계)이 표시되고, 잘못된 토큰·미공개·없음은 응답이 구분되지 않으며, 더미 데이터는 더 이상 화면에 쓰이지 않음. 유효기간 경과 데이터에 만료 배지가 표시되고, Notion 수정 후 캐시 만료 시간 내 새 값이 반영됨

- **Task 009: PDF 문서 레이아웃 및 한글 폰트 번들 구성**
  - 관련 기능: F003, F004(PDF 만료 표시)
  - `pnpm add @react-pdf/renderer`, Next 서버 번들 호환성 확인(필요 시 `next.config.ts`의 `serverExternalPackages`에 추가)
  - 폰트: Noto Sans KR Regular/Bold TTF를 `src/assets/fonts/`에 두고 OFL 라이선스 파일 동봉. 서버에서 `Font.register`로 등록, 배포 산출물에 폰트 파일이 포함되는지(`outputFileTracingIncludes` 등) 확인. 용량이 크면 서브셋 적용 검토
  - `src/lib/pdf/quote-document.tsx`: A4 레이아웃, 기본정보/항목 표/합계/비고/회사 정보(오픈 이슈 #3 결과) 구성. 입력은 화면과 같은 `QuoteView`, 금액·날짜는 Task 005 포맷 유틸 사용
  - 상태·만료 표시: 화면과 같은 기준으로 PDF에도 "만료" 표시(만료 여부는 호출 측에서 계산해 전달)
  - 페이지 분할: 표 헤더 반복(`fixed`), 행 단위 분할(`wrap={false}`), 합계 블록 분할 금지, 페이지 번호 표시(오픈 이슈 #5 결과 반영)
  - 더미 데이터(Task 006 fixtures)로 정상/만료/다항목 PDF를 먼저 생성해 육안 검수
  - 완료 기준: 더미 데이터 PDF에서 한글이 깨지지 않고 30행 이상 견적서가 규칙대로 분할됨

- **Task 010: PDF 다운로드 Route Handler 및 버튼 연동**
  - 관련 기능: F003, F002(동일 검증 재사용), F004(만료 견적서 PDF 유지)
  - `src/app/quotes/[token]/pdf/route.ts`: Task 004의 `access.ts`를 그대로 재사용, 실패 시 페이지와 같은 일반화 응답(404). Node 런타임 유지(Edge 지정 금지)
  - 응답 헤더(최종 확정): `Content-Type: application/pdf`, `Content-Disposition: attachment; filename*=UTF-8''견적서_{견적번호}.pdf`(RFC 5987 인코딩 + ASCII 대체 파일명), `Cache-Control: private, no-store`, `X-Robots-Tag: noindex`
  - 파일명 생성 함수는 순수 함수로 분리해 Vitest로 특수문자·한글 견적번호 인코딩을 단위 테스트
  - 데이터는 Task 008의 캐시 함수를 재사용해 화면과 같은 값으로 생성. 만료 여부는 요청 시점에 `validity.ts`로 계산하고, 만료 견적서도 PDF 다운로드 유지(PRD 사용자 여정)
  - `QuoteActions` PDF 버튼(클라이언트 컴포넌트): 생성 중 로딩 표시, 실패 시 오류 메시지(Alert/토스트), 중복 클릭 방지
  - 완료 기준: 버튼 한 번으로 `견적서_{견적번호}.pdf`가 저장되고 화면 금액과 일치

- **Task 011: 핵심 기능 통합 테스트**
  - 관련 기능: F001, F002, F003, F004, F010
  - Playwright MCP로 유효/무효/미공개/존재하지 않음/형식 오류 토큰 시나리오를 화면(`/quotes/{token}`)과 PDF(`/quotes/{token}/pdf`) 양쪽에서 일괄 실행
  - 항목 30행 이상 견적서의 모바일·데스크톱 렌더링과 PDF 페이지 분할 확인
  - 유효기간 경과·당일 데이터의 만료 배지 표시 여부와 만료 견적서 PDF 다운로드 확인
  - Notion 장애 상황 재현(잘못된 API 키로 임시 실행) 시 접근 불가 화면 + 내부 정보 미노출 확인
  - 브라우저 네트워크 응답과 페이지 소스에 Notion API 키, Notion 내부 ID, 상세 오류가 없는지 확인
  - 완료 기준: 아래 테스트 체크리스트 전부 통과, `pnpm test` 전체 통과

**Phase 3 테스트 체크리스트 (Playwright MCP)**

- 유효 토큰(발송/확정) 접속 시 견적번호·클라이언트명·발행일·유효기간·항목·합계·비고가 Notion 값과 일치
- 잘못된 토큰, 작성중/취소 상태, 존재하지 않는 토큰, 형식 오류 토큰의 화면·상태 코드가 서로 동일
- 견적서·접근 불가 페이지에 `noindex` 메타와 `X-Robots-Tag` 헤더 존재
- 로딩 중 스켈레톤이 표시되고 본문이 스트리밍으로 교체됨
- 375px(모바일)에서 카드형, 1280px(데스크톱)에서 표형 표시
- Notion 수정 후 캐시 만료 시간 경과 시 새 값 반영
- 유효기간 경과 테스트 데이터 접속 시 "만료" 배지 표시, PDF 다운로드 가능
- 유효기간 당일 데이터는 만료 배지 미표시
- PDF 다운로드 버튼 클릭 시 파일명 `견적서_{견적번호}.pdf`로 다운로드 이벤트 발생
- 다운로드 PDF의 텍스트(견적번호, 항목, 합계)가 화면 값과 일치, 한글 깨짐 없음
- 무효/미공개 토큰으로 `/quotes/{token}/pdf` 직접 호출 시 페이지와 동일한 일반화 응답
- 생성 중 버튼 비활성화·로딩 표시, 서버 오류 재현 시 오류 메시지 표시
- 항목 30행 이상 견적서의 페이지 분할(헤더 반복, 행 잘림 없음) 확인

**Phase 3 완료 조건**

- F001, F002, F003, F004 인수 조건 충족 (PRD "사용자 스토리 및 인수 조건" 기준)
- F010 인수 조건 충족 (캐시 만료 시간 내 수정 반영)
- 화면 코드에서 더미 fixtures 사용 제거 (테스트·개발 확인용으로만 유지)
- 오픈 이슈 #3, #4-b, #5 결정 반영
- `pnpm check-all` 통과
- `pnpm test` 통과
- `pnpm build` 통과 (PDF 라이브러리·폰트 포함 상태로 빌드 성공)

### Phase 4: 추가 기능

**목표**: PDF가 안 되는 환경의 인쇄 대체 수단과 루트 안내 페이지를 마무리해 MVP 필수 지원 기능(F011, F012)을 갖춥니다.

- **Task 012: 인쇄 레이아웃 구현**
  - 관련 기능: F011
  - 인쇄 버튼(`window.print()`, 클라이언트 컴포넌트)을 `QuoteActions`에 연결
  - 인쇄 CSS: `print:hidden`으로 헤더·버튼·테마 토글 숨김, `@page { size: A4; margin }` 설정, 인쇄 시 데스크톱 표형 강제, 표 행 중간 페이지 분할 방지(`break-inside: avoid`), 다크 모드 해제
  - 만료 배지 등 상태 표시는 인쇄 결과에도 유지
  - 완료 기준: 인쇄 미리보기에서 버튼/헤더가 숨겨지고 A4 한 흐름으로 출력

- **Task 013: 서비스 안내 페이지 구현**
  - 관련 기능: F012
  - 안내 페이지(`/`): "견적서 URL로 접속해 주세요" 안내 + 운영자 연락처 고정 텍스트(Notion 미사용, `src/lib/company.ts` 상수 재사용 가능)
  - 정적 렌더링 유지(동적 API·Notion 호출 금지), 기본 메타데이터 정리
  - Task 002 이후 언제든 착수 가능
  - 완료 기준: 루트 접속 시 안내 문구 표시, `/`가 정적 페이지로 빌드됨

**Phase 4 테스트 체크리스트 (Playwright MCP)**

- `browser_emulate_media`로 print 미디어 에뮬레이션 시 헤더·버튼 숨김, 표 정상 표시
- 인쇄 에뮬레이션에서 375px 화면이라도 표형으로 출력, 다크 모드 해제
- 루트(`/`) 접속 시 안내 문구와 연락처 표시, 견적 데이터·Notion 호출 없음

**Phase 4 완료 조건**

- F011, F012 인수 조건 충족
- `pnpm check-all` 통과
- `pnpm test` 통과
- `pnpm build` 통과 (`/`가 정적 페이지로 빌드되는지 빌드 출력에서 확인)

### Phase 5: 최적화 및 배포

**목표**: 전체 사용자 여정, 보안 요구사항, 캐시 성능을 검증하고 배포합니다.
**선결 이슈**: 없음 (#4-a 배포 환경은 외부망으로 결정 완료)

- **Task 014: 전체 E2E 검증, 보안 및 성능 점검**
  - 관련 기능: F001~F004, F010~F012 전체
  - PRD 사용자 여정 1~4단계를 Playwright MCP로 처음부터 끝까지 실행(운영자 상태 변경 → 클라이언트 접속 → PDF → 인쇄)
  - 보안 점검: 클라이언트 번들·HTML·네트워크 응답에 API 키/Notion 내부 ID/상세 오류 미노출, `.env*` 미커밋(`git ls-files` 확인), 로그에 토큰 원문 미기록, 견적서 경로 `Referrer-Policy: no-referrer`·noindex 적용
  - 엣지 케이스: 항목 0건, 비고 없음, 금액 0원, 매우 긴 항목명, 특수문자 포함 견적번호(파일명 처리)
  - 성능 점검: 캐시 적중 시 응답 시간, Notion 레이트 리밋(평균 초당 3회) 대비 캐시 효과 확인
  - 완료 기준: 아래 전체 테스트 체크리스트 통과

- **Task 015: 배포 환경 구성 및 운영 가이드 작성**
  - 관련 기능: 전체 (운영)
  - 배포 대상(오픈 이슈 #4-a, 외부망으로 결정 완료): Vercel 등 Node 런타임을 지원하는 외부 플랫폼. PDF 생성이 Node 런타임을 쓰므로 Edge 지정 금지, 폰트 파일이 배포 산출물에 포함되는지 확인
  - 운영 환경변수 등록(`NOTION_API_KEY` 등)은 배포 플랫폼 시크릿 기능 사용, 저장소에는 `.env.example`만 유지
  - 운영자 가이드(`docs/guides/operator-guide.md`): Notion DB 속성 규칙, 공유토큰 생성 방법(32자 이상 랜덤, 예: `openssl rand -base64 32`의 URL 안전 변환), 공개 상태 규칙, 수정 반영 지연 시간 안내
  - 운영 환경 스모크 테스트: 유효/무효 토큰 접속, PDF 다운로드, 안내 페이지
  - 완료 기준: 운영 URL에서 스모크 테스트 통과, 운영자 가이드만 보고 견적서 발송까지 가능

**Phase 5 테스트 체크리스트 (Playwright MCP)**

- 사용자 여정 전체: 유효 URL 접속 → 내용 확인 → PDF 다운로드 → 인쇄 미리보기
- 상태를 "취소"로 변경 후 캐시 만료 시간 경과 시 접근 불가로 전환
- 상태를 "작성중"→"발송"으로 변경 후 캐시 만료 시간 내 공개 전환
- 모든 실패 경로(토큰 오류/미공개/없음/Notion 장애)의 응답 동일성 재확인
- 엣지 케이스(항목 0건, 비고 없음, 금액 0원, 긴 항목명, 특수문자 견적번호) 화면·PDF 정상 처리
- 운영 배포 URL에서 동일 시나리오 스모크 테스트

**Phase 5 완료 조건**

- PRD 인수 조건(F001, F002, F003, F004, F010, F011) 전부 충족
- 보안 점검 항목 전부 통과
- 오픈 이슈 5건 모두 결정 완료 및 문서 반영
- `pnpm check-all` 통과
- `pnpm test` 통과
- `pnpm build` 통과
- 운영 환경 배포 및 스모크 테스트 통과

## 기능 ID 추적표

| 기능 ID | 기능명                    | 구현 Task                         | 검증 Task          |
| ------- | ------------------------- | --------------------------------- | ------------------ |
| F001    | 견적서 조회               | 002, 003, 005, 006, 007, 008      | 011, 014           |
| F002    | 토큰 기반 접근 제어       | 001, 002, 004, 008, 010           | 004, 008, 011, 014 |
| F003    | PDF 다운로드              | 005, 009, 010                     | 010, 011, 014      |
| F004    | 견적서 상태/유효기간 표시 | 002, 004, 005, 007, 008, 009, 010 | 005, 011, 014      |
| F010    | Notion 데이터 캐싱/재검증 | 003, 008                          | 011, 014           |
| F011    | 인쇄 레이아웃             | 007, 012                          | 012, 014           |
| F012    | 서비스 안내               | 001, 013                          | 013, 014           |

> 검증 Task 중 004·005는 Vitest 단위 테스트, 나머지는 Playwright MCP 기반 E2E 검증입니다.

## MVP 범위 밖 (PRD 기준, 이번 로드맵에서 제외)

- 운영자용 관리자 화면 / 로그인 / 회원가입
- 견적서 승인·거절·서명, 클라이언트 코멘트
- 열람 이력 추적, 열람 알림, 이메일 자동 발송
- 토큰 만료·재발급 자동화, 비밀번호/PIN 보호
- 다국어, 다중 통화, 세금 계산 규칙 커스터마이징
- 회사 로고/브랜딩 테마 설정 UI
