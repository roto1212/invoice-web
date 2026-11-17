# Development Guidelines (AI Agent 전용)

## 프로젝트 개요

- 프로젝트명: **invoice-web** (Notion 견적서 뷰어 MVP, 학습용이지만 프로덕션 수준으로 구현)
- 스택: Next.js 16.4.6 (App Router, Turbopack), React 19, TypeScript 5 **strict**, TailwindCSS v4, shadcn/ui (new-york), pnpm
- 데이터 소스: Notion API (`@notionhq/client`), **서버에서만** 호출
- 기준 문서: `docs/PRD.md`(요구사항), `docs/ROADMAP.md`(Task 순서·의존성), `docs/guides/*.md`(상세 가이드)
- 코드 현황: 로그인/회원가입/Hero 등 **스타터 잔재**가 남아 있음. ROADMAP Task 001에서 정리 대상이며, 잔재를 참고해 새 기능을 만들지 않는다.

## 프로젝트 아키텍처

### 디렉토리 규칙

| 경로                      | 용도                                  | 규칙                                                     |
| ------------------------- | ------------------------------------- | -------------------------------------------------------- |
| `src/app/`                | 라우트, 레이아웃, Route Handler       | 파일명은 App Router 규약(`page.tsx`, `layout.tsx` 등)    |
| `src/components/ui/`      | shadcn/ui 컴포넌트                    | **직접 수정 최소화**, 추가는 CLI로만                     |
| `src/components/layout/`  | 레이아웃 컴포넌트                     | 비즈니스 로직 금지                                       |
| `src/components/{domain}` | 도메인 컴포넌트(견적서 등)            | 파일명 kebab-case                                        |
| `src/lib/`                | 유틸·환경변수·서버 모듈               | `env.ts`(Zod 환경변수), `utils.ts`(`cn`)                 |
| `docs/`                   | PRD, ROADMAP, 가이드                  | 구현 변경 시 관련 문서 동기화                            |
| `tasks/`                  | Task 명세 파일 (`XXX-description.md`) | Task 001에서 생성 예정, 형식은 `000-sample.md`를 따른다  |

### 계획된 라우트 (ROADMAP 기준)

| 경로                  | 파일                                           | 비고                              |
| --------------------- | ---------------------------------------------- | --------------------------------- |
| `/`                   | `src/app/page.tsx`                             | 정적 안내 페이지                  |
| `/quotes/[token]`     | `src/app/quotes/[token]/page.tsx`              | noindex, Suspense 스트리밍        |
| 접근 불가             | `src/app/quotes/[token]/not-found.tsx`         | `notFound()`로 처리               |
| `/quotes/[token]/pdf` | `src/app/quotes/[token]/pdf/route.ts`          | Node 런타임, `Cache-Control: no-store` |

## 코드 표준

### TypeScript / ESLint

- `tsconfig.json`의 `strict: true`를 유지한다. 완화·해제 **금지**.
- `any`, `@ts-ignore`, `@ts-expect-error`(사유 없는 것), `eslint-disable`로 오류를 우회하지 않는다. 타입을 고친다.
- 외부 입력(Notion 응답, 환경변수, 쿼리, 폼)은 **Zod로 파싱**한 뒤 `z.infer`로 타입을 얻는다.
- import 경로는 `@/*` 별칭(`./src/*`)을 사용한다. 깊은 상대경로(`../../..`) 금지.
- 컴포넌트·함수 이름은 PascalCase/camelCase, 파일명은 kebab-case.
- 새 코드는 Server Component가 기본. 훅·이벤트 핸들러가 필요한 최소 단위에만 `'use client'`를 붙인다.

### Prettier (`.prettierrc`)

- `semi: false`, `singleQuote: true`, `tabWidth: 2`, `trailingComma: es5`, `arrowParens: avoid`, `printWidth: 80`, `endOfLine: lf`
- 클래스 정렬은 `prettier-plugin-tailwindcss`가 처리한다. 수동 정렬 금지.
- 사용자 전역 지침의 "4 spaces"는 이 프로젝트에 적용하지 않는다. **프로젝트 `.prettierrc`가 우선**한다.

| 해야 함                                | 하지 말아야 함                       |
| -------------------------------------- | ------------------------------------ |
| `const a = 1` (세미콜론 없음, 작은따옴표 `'`) | `const a = 1;` / `"문자열"`          |
| `import { cn } from '@/lib/utils'`     | `import { cn } from '../../lib/utils'` |

### 스타일링

- TailwindCSS v4 유틸리티 클래스만 사용한다. 조건부 클래스는 `cn()`(`src/lib/utils.ts`)으로 병합한다.
- 색상은 디자인 토큰(`bg-background`, `text-muted-foreground` 등)을 사용한다. 하드코딩 hex 색상 금지.
- 다크 모드는 `next-themes`(class 방식)에 맞춘다. 상세는 `docs/guides/styling-guide.md`.
- 인쇄 레이아웃(F011)은 A4 기준으로 불필요 UI를 숨긴다.

## 기능 구현 표준

### Next.js 16 설정 의존 규칙 (`next.config.ts`)

- `typedRoutes`, `cacheComponents`, `partialPrefetching`이 **켜져 있다**. 끄지 않는다.
- 라우트를 추가·변경하면 `pnpm build`까지 실행해 Route 타입 에러를 확인한다.
- 동적 데이터(`params`, Notion 조회, 현재 시각)를 읽는 부분은 **Suspense 경계 안**에 둔다.
- Notion 조회 결과 캐싱은 `'use cache'` + `cacheTag`/`cacheLife`를 사용한다. `unstable_cache`, `fetch` 캐시 옵션은 쓰지 않는다.
- `params`는 Promise이므로 `await params`로 읽는다.
- 보안 헤더(`X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`)를 제거하지 않는다. 견적서 경로의 `X-Robots-Tag`는 Task 001에서 추가한다.

### 보안 (서버 전용)

- Notion API 키 등 비밀 값은 서버 환경변수로만 읽는다. `NEXT_PUBLIC_` 접두사 **금지**.
- 환경변수는 `src/lib/env.ts`의 Zod 스키마에 먼저 추가한 뒤 사용한다. `process.env`를 다른 파일에서 직접 읽지 않는다.
- 토큰 검증·공개 상태 판정은 서버 모듈(`access.ts`)에서만 수행한다. 클라이언트 컴포넌트에서 판정하지 않는다.
- 토큰 오류, 미공개, 만료 등 **모든 실패는 동일한 응답**(`notFound()`)으로 처리한다. 실패 사유를 응답에 노출하지 않는다.
- 토큰을 로그, 에러 메시지, 리다이렉트 쿼리에 남기지 않는다.
- 평문 비밀번호·키가 코드에서 발견되면 즉시 사용자에게 경고한다.

### PDF (F003)

- `@react-pdf/renderer`를 서버(Route Handler, Node 런타임)에서만 사용한다.
- 파일명은 `견적서_{견적번호}.pdf`. 응답에 `Cache-Control: no-store`를 지정한다.
- 한글 폰트(Noto Sans KR TTF)를 번들하고 라이선스 파일을 함께 둔다.

## 프레임워크/라이브러리 사용 표준

| 대상             | 규칙                                                                                         |
| ---------------- | -------------------------------------------------------------------------------------------- |
| 패키지 매니저    | **pnpm만 사용**. `npm`/`yarn` 명령, `package-lock.json`/`yarn.lock` 생성 금지                |
| shadcn/ui        | `pnpm dlx shadcn@latest add <name>`로만 추가. `components.json`(new-york, `@/components/ui`) 유지 |
| 아이콘           | `lucide-react`                                                                               |
| 폼               | React Hook Form + Zod + Server Actions (`docs/guides/forms-react-hook-form.md` 준수)         |
| 토스트           | `sonner` (`@/components/ui/sonner`)                                                          |
| 테마             | `next-themes` (`src/components/providers/theme-provider.tsx`)                                |
| 단위 테스트      | Vitest (`pnpm test` = `vitest run`, Task 002에서 도입)                                       |
| E2E              | Playwright MCP로 검증                                                                        |
| 의존성 추가      | 보안 > 안정성 > 최신성 순으로 선택. 추가 전 사용자에게 사유를 알린다                         |

## 워크플로우 표준

### Task 진행 순서

1. `docs/ROADMAP.md`에서 다음 Task와 선결 의존성(Task 번호, 오픈 이슈 게이트)을 확인한다.
2. 오픈 이슈 게이트가 미결정이면 **해당 Task를 시작하지 않고** 사용자에게 결정을 요청한다.
3. `tasks/XXX-description.md`를 `000-sample.md` 형식으로 만든다. API/비즈니스 로직 Task는 "## 테스트 체크리스트"를 반드시 포함한다.
4. 구현 후 각 단계마다 Task 파일의 체크박스를 갱신한다.
5. 검증을 순서대로 실행한다: `pnpm check-all` → `pnpm build`(라우트 변경 시 필수) → `pnpm test`(Task 002 이후).
6. API 연동·비즈니스 로직은 Playwright MCP로 E2E 테스트한다.
7. 단계 완료 후 **멈추고 사용자의 추가 지시를 기다린다**.
8. 완료 후 `/docs:update-roadmap`으로 ROADMAP을 갱신한다.

### 검증 명령

```bash
pnpm check-all   # typecheck + lint + format:check
pnpm build
pnpm test        # Task 002 이후
```

## 핵심 파일 연동 표준

| 변경 대상                          | 함께 수정해야 하는 파일                                                         |
| ---------------------------------- | ------------------------------------------------------------------------------- |
| 환경변수 추가                      | `src/lib/env.ts`(Zod 스키마 + `parse` 입력), 배포 환경 문서                     |
| 라우트 추가/이동                   | `docs/ROADMAP.md` 라우트 표, `docs/guides/project-structure.md`, `pnpm build` 확인 |
| 새 shadcn 컴포넌트 추가            | `src/components/ui/`(CLI 생성), `package.json`/`pnpm-lock.yaml`(자동 변경)      |
| Notion 스키마(속성) 변경           | Zod 스키마·타입 파일, fixtures, `docs/PRD.md` 데이터 모델                       |
| 완료된 Task                        | `tasks/XXX-*.md` 체크박스, `docs/ROADMAP.md` 진행 상황 라인·✅ 표시              |
| 스타터 잔재 삭제(Task 001)         | `src/app/layout.tsx` metadata(제목·설명), `docs/guides/project-structure.md`    |
| 의존성 추가/삭제                   | `package.json`, `pnpm-lock.yaml`, `CLAUDE.md` 기술 스택(해당 시)                |

## AI 의사결정 표준

- 요구사항이 모호하면 `docs/PRD.md` → `docs/ROADMAP.md` → `docs/guides/*` 순으로 근거를 찾고, 근거가 없을 때만 사용자에게 묻는다.
- 규칙 충돌 시 우선순위: **이 문서 및 프로젝트 설정 파일(`.prettierrc`, `eslint.config.mjs`, `tsconfig.json`) > 프로젝트 `CLAUDE.md` > 사용자 전역 지침**.
- 컴포넌트 배치 판단:
  - 비즈니스 로직 없는 범용 UI → `components/ui/` (shadcn 기반)
  - 페이지 틀 → `components/layout/`
  - 견적서 도메인 UI → `components/` 하위 도메인 파일
- Server/Client 판단: 상태·이벤트가 필요한가? 아니오 → Server Component. 예 → 해당 부분만 Client Component로 분리.
- 학습용 프로젝트여도 에러 처리·타입·구조는 **실무 기준**으로 구현한다.

## 금지 사항

- `strict` 완화, `any` 사용, `eslint-disable`/`@ts-ignore`로 오류 은폐
- `.env*` 파일 커밋, 비밀 값에 `NEXT_PUBLIC_` 사용, 클라이언트에서 Notion API 호출
- 접근 실패 사유를 구분해 응답하거나 토큰을 로그·URL에 노출
- `npm`/`yarn` 사용, 수동 의존성 파일 편집
- `src/components/ui/` 파일을 CLI 없이 새로 작성
- 스타터 잔재(login/signup/hero 등)를 기반으로 새 기능 확장
- `next.config.ts`의 `typedRoutes`/`cacheComponents`/`partialPrefetching` 해제
- Task 단계 완료 후 사용자 확인 없이 다음 Task로 진행
- `pnpm check-all` 또는 `pnpm build` 실패 상태로 완료 보고
