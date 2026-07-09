# Astryx Assay — 개발 계획

> 설계는 [DESIGN.md](./DESIGN.md) 참조. 이 문서는 실행 체크리스트다.
> 기간은 **작업일(work-day)** 기준. 캘린더 환산은 §6.

## 운영 원칙

- 모든 마일스톤은 **데모 가능한 결과물**로 끝난다. 데모가 안 되면 끝난 게 아니다.
- Astryx는 베타 → `@astryxdesign/*` 버전은 고정하고, CI에 주 1회 최신 베타 스모크 테스트를 둔다.
- 막히면 우회하지 말고 M0 결정 로그(`docs/decisions.md`)에 기록하고 설계를 고친다.

---

## M0 — 스파이크: 전제 검증 (2 작업일)

목표: DESIGN.md §3의 미확정 3가지를 실물로 확정. **여기서 틀린 전제가 나오면 설계를 고치는 게 이 단계의 성공이다.**

### Day 1 — 셋업 & 해부
- [ ] `git init` + pnpm + TypeScript + vitest 뼈대
- [ ] `spike/` 폴더에 Vite + React + `@astryxdesign/core` + `theme-neutral` 설치
- [ ] Button 하나 브라우저에 렌더 (수동 확인)
- [ ] **확정 ①** `theme.css` 파싱 → 토큰 CSS 변수 네이밍/프리픽스/카테고리 구조 문서화
- [ ] **확정 ②** 라이브 테마 전환 메커니즘 (클래스? data-attr? provider?) + 다크모드 토글 방법
- [ ] **확정 ③** `astryx swizzle Button` 실행 → 출력 파일 구조 문서화
- [ ] 추가 확인: `astryx manifest --json` 실제 출력, 테마 패키지 7종 설치 시 용량/구조

### Day 2 — 렌더 파이프라인 실증
- [ ] Playwright로 Button을 **한 브라우저 세션에서** 2개 테마 × light/dark 전환하며 스크린샷 4장
- [ ] 콘솔 에러 캡처가 되는지 확인
- [ ] 결정 로그 작성 + DESIGN.md §3 미확정 항목 확정 반영

**DoD:** 스크린샷 4장 + `docs/decisions.md`에 확정 ①②③ 기록.
**Go/No-Go:** 라이브 테마 전환이 문서대로 안 되면 → "테마당 페이지 리로드" 방식으로 아키텍처 수정 후 진행 (중단 사유 아님).

---

## M1 — MVP: `check` 명령 (6 작업일)

목표: `assay check <file>` 한 방에 타입+토큰+렌더 검증. 의존성 순서대로:

### 1. 토큰 사전 추출기 (1일) — 린트의 전제
- [ ] 설치된 `@astryxdesign/theme-*` CSS 파싱 → `{ name, value, category }[]` 토큰 사전
- [ ] 색상 토큰의 OKLCH 변환(culori) — 최근접 토큰 추천의 기반
- [ ] 단위 테스트: neutral 테마 실물 CSS 스냅샷 기준

### 2. ESLint 플러그인: 룰 3개 (1.5일)
- [ ] `no-hardcoded-colors` — hex/rgb()/hsl() 리터럴 + 최근접 토큰 suggest
- [ ] `no-raw-dimensions` — spacing/radius/font-size 자리의 px 매직넘버
- [ ] `prefer-token-var` — Astryx 변수를 재정의/우회하는 커스텀 변수
- [ ] RuleTester 테스트 각 룰당 valid/invalid ≥ 5케이스, finding 코드 `ASSAY_*` 부여

### 3. typecheck 러너 (0.5일)
- [ ] tsc programmatic API, 사용자 tsconfig 상속 + 대상 파일 스코프
- [ ] 실패 시 렌더 단계 스킵 처리

### 4. 샌드박스 매니저 (1일)
- [ ] 사용자 lockfile에서 astryx 버전 감지 → `.assay/sandbox/<version>/` 캐시 생성
- [ ] 대상 컴포넌트(+상대 import 의존 파일) 복사·마운트
- [ ] fixture 규약 (CSF 호환 `*.fixture.tsx` / `*.stories.tsx` 인식, 없으면 `ASSAY_NO_FIXTURE` 경고)

### 5. 렌더 드라이버 (1일)
- [ ] Playwright: 설치된 테마 감지 → 테마×모드 매트릭스 순회 스크린샷
- [ ] 콘솔 에러 → `ASSAY_RUNTIME_ERROR`, 빈 DOM → `ASSAY_EMPTY_RENDER`

### 6. 리포트 + CLI 조립 (1일)
- [ ] 리포트 JSON 스키마(DESIGN.md §3C) + 터미널 포매터 + exit code 0/1/2
- [ ] commander로 `check` / `lint` 명령 조립

### 7. evil fixtures 코퍼스 = 통합 테스트 (동시 진행, 각 작업 완료 시 추가)
- [ ] 망가진 컴포넌트 ≥ 10종 (하드코딩 색, 다크모드 깨짐, 테마별 크래시, px 남발, 빈 렌더…)
- [ ] 각 파일 헤더에 기대 finding 코드 주석 → 통합 테스트가 검출률 검증

**DoD:** `assay check fixtures/evil/*.tsx` 실행 시 모든 기대 finding 검출(재현율 100%),
정상 컴포넌트 1종은 pass. 스크린샷 그리드 파일 생성. CI(GitHub Actions, Windows+Linux) 녹색.

---

## M2 — MCP 서버 + 공개 론칭 (5 작업일)

### 개발 (3일)
- [ ] MCP 서버(stdio): `assay_check` / `assay_lint` / `assay_render` — 스크린샷을 image content로 반환
- [ ] **Claude Code에 실제 연결해 e2e 데모**: 생성→lint→check→에이전트가 스크린샷 보고 자가 수정
- [ ] axe-core 테마 셀별 통합 (`ASSAY_AXE_*`)
- [ ] HTML 리포트 (테마×모드 스크린샷 그리드)
- [ ] pnpm workspace 전환: `eslint-plugin-astryx-assay` 패키지 분리

### 공개 (2일)
- [ ] README (영문): 데모 GIF — "에이전트가 스스로 깨진 테마를 고치는 장면"이 핵심 컷
- [ ] npm publish (`astryx-assay` — 가용 확인 완료) + GitHub public
- [ ] **론칭 액션: facebook/astryx의 "Design review automation for visual PRs" 이슈에 데모 코멘트**
- [ ] evil fixtures에서 발견한 본가 버그가 있으면 별도 이슈 제보 (기여 이력 시작)

**DoD:** 외부인이 README만 보고 5분 안에 `check` 실행 성공. 이슈 코멘트 게시.

---

## M3 — 심화 (론칭 피드백 기반, ~2주 유동)

우선순위는 론칭 반응으로 재조정. 후보:
- [ ] 타입 기반 fixture 자동 생성 (fixture 없는 컴포넌트 커버)
- [ ] visual regression: 기준선 스크린샷 대비 pixelmatch diff
- [ ] `--judge`: Claude API로 요구사항 대비 판정 (CLI 단독 사용자용; MCP 경로는 불필요)
- [ ] 이슈/피드백 대응

## M4 — swizzle 드리프트 추적 (유동)

- [ ] swizzle 시점 upstream 스냅샷 → `.assay/swizzle.lock`
- [ ] `astryx upgrade` 후 3-way diff: "내 커스텀 vs 상류 변경" 충돌 리포트

---

## 6. 캘린더 환산

총 M0+M1+M2 = **13 작업일** (론칭까지).

| 투입 | 론칭 시점 |
|---|---|
| 풀타임 | ~3주 (8월 초) |
| 사이드 프로젝트 (주 10~15h) | ~6주 (8월 중순~말) |

베타 생태계 선점이 핵심이므로 **M2 론칭까지는 속도 우선, M3부터 완성도 우선**으로 운영한다.

## 7. 지금 바로 할 다음 행동

1. `git init` + 뼈대 커밋
2. M0 Day 1 착수 (Astryx 설치 → 토큰 구조 확정)
