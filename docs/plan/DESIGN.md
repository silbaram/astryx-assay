# Astryx Assay — 설계 문서

> **한 줄 정의:** AI 에이전트(또는 사람)가 Astryx로 만든 UI — 커스텀 컴포넌트, 신규 컴포넌트, 템플릿 — 가
> 정말 "Astryx다운지"를 자동 검증하는, 어떤 에이전트든 붙여 쓸 수 있는 CLI/MCP 기반 QA 툴킷.
>
> 슬로건: **생성은 에이전트가, 검증은 Assay가.** (assay: 귀금속의 순도를 분석·판정하는 일)

작성일: 2026-07-09 · 대상 Astryx: beta (2026-06 공개, 버전 고정 운용)

---

## 1. 설계 원칙

1. **Agent-agnostic 이중 인터페이스** — 모든 기능은 CLI 명령이면서 동시에 MCP 도구다.
   Claude Code, Cursor 등 어떤 에이전트든, 그리고 사람도 같은 표면을 쓴다. (Astryx 본가의
   "humans and AI use the same tooling" 철학을 그대로 미러링)
2. **Deterministic first** — 타입 체크·린트·렌더링 같은 결정적 검사가 기반이고, LLM 판정은
   맨 마지막 선택적 레이어다. Meta 위키의 결론("컨텍스트 주입은 선택적이라 실패한다.
   강제는 타입과 검증 도구로")을 따른다.
3. **Machine-readable 출력 + 안정적 에러 코드** — 모든 결과는 JSON이며 findings는
   `ASSAY_*` 코드로 식별한다. 에이전트는 사람용 메시지 문자열이 아니라 코드로 분기한다.
   (본가 CLI의 `ERR_UNKNOWN_COMPONENT` 방식 미러링)
4. **판정 위임(vision hand-off)** — Assay는 스크린샷을 "찍어서 돌려주는" 데까지만 책임진다.
   MCP 이미지 콘텐츠로 반환하면 호출한 에이전트가 자기 비전으로 판단한다.
   → 외부 LLM API 키 의존 없이 시각 판정이 성립하는 것이 이 설계의 핵심 차별점.
5. **샌드박스 격리** — 렌더 검증은 사용자 프로젝트 안이 아니라 Assay가 관리하는 깨끗한
   샌드박스에서 수행한다. 사용자 프로젝트의 깨진 상태와 검증 결과를 분리하기 위함.
   단, Astryx 버전은 사용자 프로젝트의 것을 감지해서 맞춘다.

---

## 2. 시스템 아키텍처

```
                        ┌─────────────────────────────────────────┐
                        │              astryx-assay               │
  사용자 / CLI ────────▶│  cli.ts (commander)                     │
                        │  mcp.ts (@modelcontextprotocol/sdk)     │──▶ 스크린샷 = MCP image content
  Claude Code 등 ──────▶│         │        │                      │    (호출 에이전트가 비전으로 판정)
   (MCP client)         │         ▼        ▼                      │
                        │  ┌──────────────────────────────┐       │
                        │  │        check pipeline        │       │
                        │  │  Stage A: 정적 검사 (빠름)    │       │
                        │  │   A1 typecheck (tsc API)     │       │
                        │  │   A2 token-lint (ESLint)     │       │
                        │  │   A3 import 위생 검사         │       │
                        │  │  Stage B: 렌더 검사 (브라우저) │       │
                        │  │   B1 샌드박스 마운트 (Vite)   │       │
                        │  │   B2 테마×모드 매트릭스 캡처   │       │
                        │  │   B3 axe-core a11y 스캔      │       │
                        │  │  Stage C: 리포트 합성         │       │
                        │  │   JSON / 터미널 / HTML       │       │
                        │  └──────────────────────────────┘       │
                        └─────────────────────────────────────────┘
                                        │
                                        ▼
                        .assay/  (버전별 샌드박스 캐시, 리포트, swizzle.lock)
```

### 디렉터리 구조 (초기 — 단일 패키지)

```
astryx-assay/
├── src/
│   ├── checkers/
│   │   ├── typecheck.ts        # tsc programmatic API
│   │   ├── lint/               # eslint-plugin-astryx-assay (룰 + 테스트)
│   │   └── render/             # 샌드박스 드라이버 + Playwright 캡처
│   ├── sandbox/                # Vite+React+astryx 템플릿 (버전별로 인스턴스화)
│   ├── report/                 # JSON 스키마, 터미널 포매터, HTML 리포트
│   ├── cli.ts
│   └── mcp.ts
├── fixtures/evil/              # 일부러 망가뜨린 컴포넌트 코퍼스 (§8 검증 전략)
├── DESIGN.md
└── package.json
```

모듈 경계를 지키면 나중에 `packages/{core,cli,mcp,eslint-plugin}` 모노레포로 쪼개는 것은
기계적인 작업이다. ESLint 플러그인은 에디터에서 단독 사용 가치가 생기는 M2 시점에 분리한다.

---

## 3. 검사 파이프라인 상세

### Stage A — 정적 검사 (브라우저 불필요, 수 초 이내)

**A1. typecheck** — `tsc --noEmit`을 programmatic API로 실행. 사용자 프로젝트의
tsconfig를 상속하되 대상 파일로 스코프를 좁힌다. 실패 시 Stage B는 건너뛴다(렌더 불가).

**A2. token-lint** — `eslint-plugin-astryx-assay`. 초기 룰 3개:

| 룰 | 잡는 것 | 예시 finding 코드 |
|---|---|---|
| `no-hardcoded-colors` | style 객체/템플릿 안의 hex·rgb()·hsl() 리터럴 | `ASSAY_HARDCODED_COLOR` |
| `no-raw-dimensions` | 토큰이 있어야 할 자리의 px 매직넘버 (spacing·radius·font-size) | `ASSAY_RAW_DIMENSION` |
| `prefer-token-var` | Astryx CSS 변수 대신 재정의한 커스텀 변수 | `ASSAY_OFF_SYSTEM_TOKEN` |

각 finding은 가능하면 **수정 제안(제일 가까운 Astryx 토큰)**을 포함한다 — 색상은 OKLCH
거리 기반 최근접 토큰 추천. 접근성 정적 검사는 룰을 새로 만들지 않고
`eslint-plugin-jsx-a11y`를 프리셋으로 묶는다.

> ⚠️ 토큰 CSS 변수의 정확한 네이밍/프리픽스는 문서에 미공개. **M0 스파이크에서
> `@astryxdesign/theme-neutral/theme.css`를 파싱해 확정**하고, 토큰 사전은 설치된
> 테마 CSS에서 빌드 타임에 추출한다(하드코딩 금지 — 베타 churn 대응).

**A3. import 위생** — deep import(`@astryxdesign/core/lib/...`), 사용자 프로젝트와
샌드박스 간 astryx 버전 불일치, 미설치 테마 참조 검출.

### Stage B — 렌더 검사

**B1. 샌드박스 마운트.** `.assay/sandbox/<astryx-version>/`에 Vite+React+astryx
템플릿을 버전별로 캐시. 사용자 프로젝트의 lockfile에서 `@astryxdesign/core` 버전을 감지해
동일 버전을 설치한다. 대상 컴포넌트 파일(+ 로컬 의존 파일)을 샌드박스로 복사해 마운트.

*props 문제:* 컴포넌트는 props 없이 렌더가 안 될 수 있다. 해결은 **fixture 규약**:

```tsx
// MyCard.fixture.tsx — Storybook CSF 호환 형태
export default { component: MyCard };
export const Default = { args: { title: "Hello" } };
export const Loading = { args: { title: "Hello", loading: true } };
```

- 이미 Storybook을 쓰는 프로젝트면 기존 `*.stories.tsx`를 그대로 인식한다.
- fixture가 없으면: M1에서는 `ASSAY_NO_FIXTURE` 경고와 함께 props 없는 렌더 시도,
  M3에서 타입 기반 최소 props 자동 생성을 시도한다.

**B2. 테마×모드 매트릭스.** 설치된 `@astryxdesign/theme-*` 패키지를 감지(기본: 공식 7종)
× light/dark = 최대 14 셀. 테마는 CSS 커스텀 프로퍼티 오버라이드로 라이브 전환 가능하므로
**페이지 리로드 없이 한 브라우저 세션에서 테마를 순회**한다(속도 핵심). 셀마다:

- Playwright(chromium) 스크린샷
- 콘솔 에러·React 에러 바운더리 캡처 → `ASSAY_RUNTIME_ERROR`
- 렌더 결과가 빈 DOM이면 → `ASSAY_EMPTY_RENDER`

**B3. a11y 스캔.** 각 테마 셀의 DOM에 `@axe-core/playwright` 실행. 특히
`color-contrast`는 테마별로 결과가 다르므로 매트릭스와 결합해야 의미가 있다
(본가에도 WCAG 대비 실패 이슈가 열려 있음 — 우리가 잡을 수 있는 실수 클래스).

### Stage C — 리포트

```jsonc
{
  "target": "src/MyCard.tsx",
  "astryxVersion": "0.4.2",
  "verdict": "fail",                      // pass | fail | error
  "checks": {
    "types":  { "status": "pass" },
    "tokens": { "status": "fail", "findings": [
      { "code": "ASSAY_HARDCODED_COLOR", "loc": "MyCard.tsx:12:8",
        "found": "#e53935", "suggest": "var(--<token-prefix>-color-danger)" } ] },
    "render": { "status": "pass",
      "matrix": [ { "theme": "neutral", "mode": "dark",
                    "screenshot": ".assay/reports/…/neutral-dark.png",
                    "consoleErrors": [] } /* … */ ] },
    "a11y":   { "status": "fail", "findings": [
      { "code": "ASSAY_AXE_COLOR_CONTRAST", "themes": ["gothic-dark"], "rule": "color-contrast" } ] }
  }
}
```

- **터미널**: 요약 테이블 + finding 목록. exit code: `0` pass / `1` findings / `2` 하네스 오류.
- **HTML 리포트**: 테마×모드 스크린샷 그리드 한 장 — 사람용 데모의 핵심 화면.
- **MCP**: 위 JSON + 스크린샷을 image content로 반환.

---

## 4. 인터페이스 설계

### CLI

```
assay check <file|dir> [--themes a,b] [--skip-render] [--report html|json]
assay lint  <file|dir>          # Stage A만
assay render <file> [--fixture Default]   # Stage B만, 스크린샷 출력
assay manifest --json           # 자기서술 계약 (본가 manifest 철학 미러링)
assay mcp                       # stdio MCP 서버 기동
```

### MCP 도구

| 도구 | 입력 | 출력 |
|---|---|---|
| `assay_check` | file, options | 리포트 JSON + 스크린샷 이미지들 |
| `assay_lint` | file | findings JSON (빠른 루프용) |
| `assay_render` | file, themes?, fixture? | 스크린샷 이미지들 |

에이전트의 기대 사용 루프: 코드 생성 → `assay_lint`(수 초)로 즉시 교정 →
`assay_check`로 최종 확인 → 반환된 스크린샷을 스스로 보고 시각 판정 → 필요 시 재수정.

---

## 5. 기술 스택과 근거

| 선택 | 대안 | 근거 |
|---|---|---|
| TypeScript + Node 20 | Python | Astryx 생태계와 동일 언어, tsc/ESLint programmatic API 직접 사용 |
| ESLint 커스텀 플러그인 | 자체 AST 스캐너 | 에디터 통합 공짜, 룰 테스트 인프라 공짜, 본가 상류 기여 가능한 형태 |
| Vite 샌드박스 | 사용자 프로젝트 내 렌더 | 재현성·격리 (원칙 5), 버전별 캐시로 속도 확보 |
| Playwright (chromium) | jsdom | CSS 커스텀 프로퍼티·실제 페인트가 검증 대상이라 실브라우저 필수 |
| @axe-core/playwright | 수동 룰 | 업계 표준, 테마 매트릭스와 결합 시 차별화 |
| @modelcontextprotocol/sdk | 자체 프로토콜 | 표준, 본가도 MCP 채택 |
| pixelmatch (M3) | — | visual regression은 후순위 |

---

## 6. 마일스톤

- **M0 — 스파이크 (1~2일):** Astryx 실제 설치. 확인할 것 ①토큰 CSS 변수 프리픽스·구조
  ②라이브 테마 전환의 정확한 메커니즘(클래스? data-attr? provider?) ③swizzle 출력 형태.
  Button 하나를 수동으로 렌더+스크린샷까지. **여기서 §3의 미확정 사항을 전부 확정한다.**
- **M1 — MVP (~1주):** `check` = typecheck + 토큰 린트 3룰 + 테마 매트릭스 스크린샷 +
  JSON/터미널 리포트. evil fixtures 코퍼스로 검출 데모.
- **M2 (~1주):** MCP 서버(이미지 반환 포함), axe-core 통합, HTML 리포트,
  ESLint 플러그인 패키지 분리. → **본가 "Design review automation" 이슈에 데모 코멘트 = 론칭.**
- **M3:** 타입 기반 fixture 자동 생성, visual regression(기준선 비교), `--judge` 옵션
  (Claude API로 요구사항 대비 판정 — MCP 경로에서는 불필요).
- **M4:** swizzle 드리프트 추적 — swizzle 시점의 upstream 스냅샷을 `.assay/swizzle.lock`에
  기록, `astryx upgrade` 후 3-way diff로 "당신의 커스텀 vs 상류 변경" 충돌 리포트.

---

## 7. 리스크와 대응

| 리스크 | 대응 |
|---|---|
| 베타 API churn | 토큰 사전을 설치된 테마 CSS에서 동적 추출(하드코딩 없음), 샌드박스 버전별 캐시, CI에서 astryx 최신 베타 대상 스모크 테스트 |
| Meta가 공식 검증 도구 출시 | ESLint 플러그인·리포트 스키마를 상류 기여 가능한 형태로 유지 — 경쟁이 아니라 합류가 출구 전략 |
| fixture 없는 컴포넌트 렌더 실패 | M1은 경고+최선 시도, M3 자동 생성. Storybook CSF 호환으로 기존 자산 재활용 |
| 14셀 렌더가 느림 | 단일 세션 라이브 테마 전환, 샌드박스 캐시, `--skip-render`/`lint` 빠른 경로 제공 |

---

## 8. 검증 전략: evil fixtures가 곧 데모다

`fixtures/evil/`에 **일부러 망가뜨린 컴포넌트 코퍼스**를 만든다: 하드코딩 색상, 다크모드에서
깨지는 배경, 특정 테마에서만 대비 실패, px 매직넘버, aria 누락, 특정 테마에서 런타임 크래시….
각 파일에 기대 finding 코드를 주석으로 명시하면 이것이 그대로 통합 테스트 스위트이자,
README의 "이만큼 잡아냅니다" 데모가 된다. 검출률(재현율)을 릴리스 기준으로 삼는다.
