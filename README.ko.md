[🇬🇧 English](README.md) · [🇯🇵 日本語](README.ja.md) · [🇨🇳 中文](README.zh.md)

# AI-Native Design

**AI가 이해하고, 수정하고, 배포하기 좋은 코드 구조 설계 원칙.**

> 맥락: 이 원칙들은 [**NSKit**](https://github.com/nskit-io/nskit-io)의 뼈대입니다 — *갇혀 있어서 무한 조합*. 프로덕션 웹앱 플랫폼으로서 이 원칙을 실제로 돌아가는 서비스로 구현합니다. AI 비주얼 아티팩트 도구(Claude Design, v0, Lovable)는 다른 레이어에 있습니다 — 그들은 목업을 생성하고, NSKit은 이 원칙을 **실제로 돌아가는 코드**에 적용합니다.

> 코드 구조를 사람의 가독성이 아니라, AI가 실제로 코드를 처리하고 수정하는 방식에 맞춰 최적화하면 어떨까?

React, Vue, Angular 같은 현대 프레임워크들은 **Human Developer Experience**를 위해 훌륭하게 설계되어 있습니다 -- 모듈화, 타입 안전성, 재사용성, 관심사 분리. 사람이 주요 코드 작성자일 때 이것은 올바른 목표입니다.

하지만 AI가 주요 협업자일 때 -- 코드를 읽고, 컨텍스트를 이해하고, 변경하고, 기능을 배포할 때 -- 최적화 대상이 달라집니다. **AI-Native Design**은 "어떤 구조가 AI의 정확성과 효율성을 극대화할까?"라는 질문에서 출발한 아키텍처 원칙입니다.

이것은 프레임워크나 라이브러리가 아닙니다. 프로덕션 시스템의 실측 벤치마크에 기반한 설계 철학입니다.

---

## 목차

- [핵심 긴장 관계](#핵심-긴장-관계)
- [6가지 원칙](#6가지-원칙)
- [프론트엔드를 넘어서: AI-Native 백엔드](#프론트엔드를-넘어서-ai-native-백엔드)
- [토큰 효율 벤치마크](#토큰-효율-벤치마크)
- [실전 결과](#실전-결과)
- [언제 사용할까 / 언제 사용하지 말까](#언제-사용할까--언제-사용하지-말까)
- [FAQ](#faq)
- [이 원칙으로 만들어진 프레임워크](#이-원칙으로-만들어진-프레임워크)

---

## 핵심 긴장 관계

Human DX와 AI DX는 **서로 다른 최적화 대상**이며, **서로 다른 아키텍처 결정**으로 이어집니다.

| 관심사 | Human DX 최적화 | AI DX 최적화 |
|---|---|---|
| **파일 구조** | 명확한 import를 가진 작고 집중된 모듈 | 모든 것을 인라인한 Self-Contained 파일 |
| **의존성** | 추상화, 재사용, 컴포넌트 간 공유 | 최소화; 모든 import는 AI가 추적해야 할 컨텍스트 |
| **타입 시스템** | 안전성과 문서화를 위한 풍부한 타입 | 타입보다 컨벤션; 계약보다 패턴 |
| **파일 크기** | 최대한 작게 (SRP) | 500-800줄 스위트 스팟 (컨텍스트 윈도우 효율) |
| **빌드 프로세스** | Webpack, Vite, esbuild로 최적화 | 없음. 에러 카테고리 전체를 제거 |
| **상태 관리** | 중앙 집중 스토어 (Redux, Pinia) | 파일 스코프; 공유 가변 상태 없음 |

어느 쪽이 "더 낫다"가 아닙니다. 서로 다른 제약을 해결합니다. AI가 코드의 80% 이상을 작성할 때, 가장 중요한 제약은: **AI가 이 파일을 얼마나 정확하게 이해하고 수정할 수 있는가?**입니다.

---

## 6가지 원칙

### 1. Self-Contained 파일

**하나의 파일 = HTML + CSS + JS.** 추적할 import 없음. AI가 하나의 파일만 읽으면 전체 컨텍스트를 얻음.

```html
<!-- AI-Native: Self-Contained 컴포넌트 -->
<div class="user-card">
  <div class="avatar">...</div>
  <div class="info">...</div>

  <style>
    .user-card { border-radius: 12px; padding: 16px; }
    .user-card .avatar { width: 48px; height: 48px; }
  </style>

  <script>
  (function() {
    PART.define('user-card', {
      onInit: function(part) {
        part._data = { user: null };
      },
      setUser: function(part, user) {
        part._data.user = user;
        render(part);
      }
    });
    function render(part) { /* private 렌더링 로직 */ }
  })();
  </script>
</div>
```

모듈 기반 방식과 비교:

```javascript
// 모듈 방식: AI가 이 컴포넌트를 이해하려면 5개 이상 파일을 읽어야 함
import { Avatar } from './Avatar';
import { UserInfo } from './UserInfo';
import { useUser } from '../hooks/useUser';
import { useTheme } from '../context/ThemeContext';
import styles from './UserCard.module.css';
```

모듈 방식은 인간 팀에게 훌륭합니다 -- 명확한 분리, 테스트 가능한 단위, 공유 추상화. 하지만 AI에게는 모든 `import`가 전체 그림을 이해하기 위해 읽어야 할 파일입니다. Self-contained 파일은 AI에게 한 번의 읽기로 100% 컨텍스트를 제공합니다.

---

### 2. 500-800줄 스위트 스팟

AI-Native Design에서 가장 실행 가능한 통찰입니다. 실전 측정을 통해 파일 크기와 AI 수정 정확도 사이의 명확한 관계를 발견했습니다:

| 파일 크기 | AI 정확도 | 평가 |
|---|---|---|
| ~100줄 | ~95% | 훌륭하지만 너무 세분화 (많은 파일 = 많은 읽기) |
| ~500줄 | ~85% | **스위트 스팟 시작** |
| ~800줄 | ~77% | **스위트 스팟 끝** |
| ~1,000줄 | ~70% | 허용 가능한 최대치 |
| 2,000줄+ | ~50% | 동전 던지기 -- 피할 것 |

<img src="https://mermaid.ink/img/eHljaGFydC1iZXRhCiAgICB0aXRsZSAiQUkgQWNjdXJhY3kgdnMgRmlsZSBTaXplIgogICAgeC1heGlzICJGaWxlIFNpemUgKGxpbmVzKSIgWzEwMCwgMjAwLCAzMDAsIDQwMCwgNTAwLCA2MDAsIDcwMCwgODAwLCA5MDAsIDEwMDAsIDE1MDAsIDIwMDBdCiAgICB5LWF4aXMgIkFjY3VyYWN5ICglKSIgNDAgLS0+IDEwMAogICAgbGluZSBbOTUsIDkzLCA5MSwgODgsIDg1LCA4MywgODAsIDc3LCA3NCwgNzAsIDYwLCA1MF0=" alt="AI 정확도 vs 파일 크기" />

**NSKit 프레임워크 실측 데이터** (22개 프로덕션 JS 파일, 총 11,850줄):

| 카테고리 | 파일 | 줄 수 |
|---|---|---|
| Core | nskit-api.js | 815 |
| Core | nskit-util.js | 580 |
| Core | nskit-core.js | 406 |
| Core | nskit-lang.js | 373 |
| Core | nskit-trace.js | 207 |
| Core | nskit-auth.js | 199 |
| Core | nskit-event.js | 160 |
| Layout | nskit-dialog.js | 1,044 |
| Layout | nskit-page.js | 1,043 |
| Layout | nskit-panel.js | 937 |
| Layout | nskit-scene.js | 580 |
| Layout | nskit-part.js | 505 |
| Widget | ns-select.js | 1,040 |
| Widget | ns-table.js | 971 |
| Widget | ns-input.js | 729 |
| Widget | ns-checkbox.js | 701 |
| Widget | nskit-widget.js | 433 |
| Widget | ns-textarea.js | 331 |
| Widget | ns-button.js | 218 |
| Bootstrap | nskit-loader.js | 266 |
| Bootstrap | nskit.js | 166 |
| Bootstrap | nskit-init.js | 146 |
| **평균** | | **539줄** |

평균 파일 크기: **539줄** -- 스위트 스팟(500-800줄) 안.

---

### 3. No Build Process

Webpack 없음. Vite 없음. 트랜스파일 없음. 번들링 없음. 파일 저장, 브라우저 새로고침, 변경 확인.

옛날 방식에 대한 향수가 아닙니다. AI가 자주 발생시키는 에러 카테고리 전체를 의도적으로 제거하는 것입니다:

<img src="https://mermaid.ink/img/cGllIHRpdGxlICJCdWlsZCBFcnJvciBDYXRlZ29yaWVzIEVsaW1pbmF0ZWQgYnkgTm8tQnVpbGQiCiAgICAiRGVwZW5kZW5jeSBSZXNvbHV0aW9uIiA6IDI1CiAgICAiVHJhbnNwaWxhdGlvbiBFcnJvcnMiIDogMjAKICAgICJCdW5kbGUgQ29uZmlnIiA6IDIwCiAgICAiVHlwZSBNaXNtYXRjaCIgOiAxNQogICAgIlNvdXJjZSBNYXAgSXNzdWVzIiA6IDEwCiAgICAiSG90IFJlbG9hZCBGYWlsdXJlcyIgOiAxMA==" alt="No-Build로 제거되는 빌드 에러 카테고리" />

빌드 프로세스가 없으면 피드백 루프가 즉각적입니다: **저장, 새로고침, 확인.** "로컬에선 컴파일 되는데 CI에서 실패" 같은 문제가 완전히 사라집니다.

---

### 4. Convention over Configuration

모든 컴포넌트가 동일한 구조를 따릅니다. AI가 패턴을 한 번 배우면 모든 곳에 적용합니다.

```javascript
// Page, Scene, Dialog, Panel, Part -- 모두 동일한 구조
PART.define('component-name', {
    onInit: function(part) {
        part._data = {};           // private 상태
    },
    onShow: function(part, params) {
        // 라이프사이클: 컴포넌트가 보여질 때
    },
    setData: function(part, data) {
        // public API: 외부에서 호출
    },
    _render: function(part) {
        // private: 내부 렌더링
    },
    cleanup: function(part) {
        // 라이프사이클: 정리
    }
});
```

전체 페이지든, 다이얼로그 모달이든, 사이드바 패널이든, 재사용 파트든 -- `.define()` 패턴이 동일합니다:

- AI가 어떤 라이프사이클 메서드를 쓸지 고민할 필요 없음
- 새 개발자(와 새 AI 세션)가 하나의 패턴만 학습
- 컴포넌트 간 리팩토링이 예측 가능
- 동일 구조 = 조합 가능 = Template Mixing

---

### 5. 토큰 효율을 설계 목표로

AI가 코드를 읽을 때, 소비되는 모든 줄이 토큰 비용입니다. Self-contained 파일은 AI 작업에서 극적으로 높은 토큰 효율을 보여줍니다.

**시나리오: 카드 컴포넌트 안의 버튼 클릭 핸들러 변경.**

| | React (모듈) | AI-Native (self-contained) |
|---|---|---|
| 주요 파일 | ~150줄 | ~600줄 |
| AI가 읽어야 할 파일 수 | 8-15 (imports, hooks, context, types, styles) | 1 |
| 소비 토큰 | ~15,000 - 37,000 | ~700 - 1,200 |
| 변경 후 빌드 단계 | 필요 | 없음 |
| 사이드 이펙트 위험 | 공유 hook/context가 다른 컴포넌트에 영향 가능 | 제로 -- 파일이 self-contained |

<img src="https://mermaid.ink/img/eHljaGFydC1iZXRhCiAgICB0aXRsZSAiVG9rZW5zIFJlcXVpcmVkIGZvciBTaW5nbGUgQ29tcG9uZW50IENoYW5nZSIKICAgIHgtYXhpcyBbIkFuZ3VsYXIiLCAiUmVhY3QiLCAiVnVlIiwgIkFJLU5hdGl2ZSJdCiAgICB5LWF4aXMgIlRva2VucyIgMCAtLT4gMzUwMDAKICAgIGJhciBbMzAwMDAsIDI1MDAwLCAyMDAwMCwgODAwXQ==" alt="패러다임별 필요 토큰 수" />

**계산:**
- React 컴포넌트 변경: ~25,000 토큰 (중간값 추정)
- AI-Native 컴포넌트 변경: ~800 토큰
- **비율: ~31배 토큰 효율적**

20개 컴포넌트 수정 세션에서 500,000 토큰 vs 16,000 토큰. 복리 효과가 상당합니다 -- 특히 여러 AI 세션을 병렬로 운영할 때.

> **방법론 참고:** React/Vue/Angular 토큰 추정치는 공유 hooks, context providers, type definitions을 가진 일반적인 중간 규모 애플리케이션 구조를 기반으로 합니다. 실제 수치는 프로젝트에 따라 다릅니다. 상세 방법론은 [benchmarks/token-comparison.md](benchmarks/token-comparison.md)를 참조하세요.

---

### 6. 병렬 AI 세션

공유 가변 상태가 없는 Self-contained 파일은 강력한 기능을 열어줍니다: **여러 AI 에이전트가 동일한 코드베이스에서 머지 충돌 없이 동시 작업.**

```
전통적 (공유 상태):
  Session 1: UserCard.tsx 수정       -- useAuth hook 사용
  Session 2: ProfilePage.tsx 수정    -- useAuth hook 사용
  Session 3: AuthProvider.tsx 수정   -- Session 1, 2와 충돌!

AI-Native (self-contained):
  Session 1: part-user-card.html 수정     -- self-contained
  Session 2: scene-profile.html 수정      -- self-contained
  Session 3: part-auth-form.html 수정     -- self-contained
  결과: 3개 파일 수정, 충돌 제로, 3배 처리량
```

이론이 아닙니다. 프로덕션에서 3개 이상의 AI 세션이 서로 다른 템플릿 파일을 동시에 작업하는 것이 일상입니다.

---

## 프론트엔드를 넘어서: AI-Native 백엔드

AI-Native Design은 프론트엔드 템플릿에만 국한되지 않습니다. 동일한 원칙이 백엔드 API 아키텍처에도 적용됩니다.

### CommandHandler 패턴

전통적인 Spring REST 컨트롤러에서 AI가 하나의 엔드포인트를 이해하려면 컨트롤러, 서비스, 시큐리티 설정, DTO 등 여러 파일을 추적해야 합니다. CommandHandler 패턴은 이를 하나의 메서드로 응축합니다:

```java
@CommandHandler  // needAuth=true, allowGuest=false (기본값)
public ResponseVO getProfile(CommandContext ctx) {
    // 여기서 User는 보장됨. ctx.getUser()는 절대 null이 아님.
    // AI가 인증 미들웨어나 시큐리티 설정을 확인할 필요 없음.
    UserVO user = ctx.getUser();
    return ResponseHelper.response(200, "Success", userService.getProfile(user.getUid()));
}
```

**AI가 이 패턴을 좋아하는 이유:**

| | 전통적 REST | CommandHandler |
|---|---|---|
| **1개 API 이해** | 컨트롤러 + 서비스 + 시큐리티 설정 + DTO 읽기 | 메서드 하나만 읽기 |
| **인증 보장** | SecurityContext 수동 확인, 인터셉터 추적 | 어노테이션이 전부 — user 보장 또는 명시적으로 선택적 |
| **새 엔드포인트 추가** | URL 매핑 + 메서드 + 라우트 등록 | `@CommandHandler` 메서드 하나 추가 |
| **API 스펙 → 코드** | URL 경로 파싱, 파라미터 매핑 | `"get-profile"` → `getProfile()` — 커맨드명이 곧 메서드명 |

**인증 계약:**
- `@CommandHandler` (기본) → `ctx.getUser()`는 **절대 null이 아님**. AI가 확신을 가지고 코드 작성.
- `@CommandHandler(allowGuest = true)` → user가 **null일 수 있음**. AI가 null 체크 필요.
- `@CommandHandler(needAuth = false)` → 공개 엔드포인트. 인증 없음.

프론트엔드의 "Convention over Configuration" 원칙을 백엔드에 그대로 적용한 것입니다. AI가 패턴을 한 번 배우면 모든 모듈에 적용합니다.

전체 패턴 문서: **[command-handler](https://github.com/nskit-io/command-handler)**

---

## 토큰 효율 벤치마크

### 요약 표

| 프레임워크 | 읽어야 할 파일 | 총 줄 수 | 추정 토큰 | AI-Native 대비 |
|---|---|---|---|---|
| Angular | 11 | 1,110 | ~22,200 | 18.5x |
| React | 8 | 940 | ~18,800 | 15.7x |
| Vue | 7 | 890 | ~17,800 | 14.8x |
| **AI-Native** | **1** | **600** | **~1,200** | **1x** |

상세 방법론은 [benchmarks/token-comparison.md](benchmarks/token-comparison.md)를 참조하세요.

---

## 실전 결과

### 사례 연구: NewMyoung

[NewMyoung](https://newmyoung.com)은 AI 기반 작명 + 이름 운세 서비스로, 다수 지역에서 프로덕션 운영 중입니다:
- **한국**: newmyoung.com
- **일본**: jp.newmyoung.com
- **화교문화권** (대만, 싱가포르, 마카오, 말레이시아, 홍콩): zh.newmyoung.com

**AI-Native Design 원칙으로 구축:**

| 지표 | 값 |
|---|---|
| 핵심 기능 완성 시간 | ~2일 |
| 템플릿 파일 수 | 31 |
| 평균 파일 크기 | ~500줄 |
| 병렬 AI 세션 | 3+ |
| 빌드 프로세스 | 없음 |
| 프로덕션 안정성 | 런칭 이후 안정 운영 |

**병렬 AI 세션 운영 방식:**
- Session 1: 한국어 글자 선택 UI (part-char-select.html)
- Session 2: 운세 분석 디스플레이 (scene-analysis.html)
- Session 3: 획수 선택 (part-stroke-select.html)

세 세션이 동시 작업. 머지 충돌 없음. 공유 상태 오염 없음. 각 파일이 self-contained.

---

## 언제 사용할까 / 언제 사용하지 말까

### AI-Native Design이 잘 맞는 경우:

- AI가 주요 코드 협업자 (코드의 50% 이상 작성)
- 프로덕션으로 바로 가는 빠른 프로토타이핑
- 여러 AI 세션 병렬 운영
- 소규모 팀 (1-5명) 또는 솔로 개발자
- 빌드 설정 제로를 원할 때
- 배포 단순성 (파일 복사, 새로고침)

### AI-Native Design이 맞지 않을 수 있는 경우:

- **복잡한 공유 상태**: 실시간 협업 에디터 등 깊이 연결된 실시간 상태를 가진 애플리케이션.
- **풍부한 생태계 필요**: React/Vue/Angular 생태계 라이브러리에 크게 의존하는 경우.
- **SEO 집중 애플리케이션**: SSR/SSG가 있는 프레임워크(Next.js, Nuxt, Astro)가 더 적합.

### 대규모 프로젝트에 대한 오해

"대규모 팀에는 모듈 아키텍처가 필요하다"는 통념이 있지만, 실제 대형 프로젝트의 작업 방식을 생각해보면:

**기존 대형 프로젝트의 병목:**
```
코어팀: 프레임워크 백본 구축 → 빌드 환경 설정 → 코드 컨벤션 확정
  ↓ (이 과정이 끝나야)
모듈팀 A: 작업 시작 가능
모듈팀 B: 작업 시작 가능
모듈팀 C: 작업 시작 가능
```

코어팀이 백본을 완성하기 전까지 나머지 팀은 대기 상태입니다. 전사 빌드 규칙, 코드 컨벤션, 공유 컴포넌트가 확정되어야 모듈 단위 개발이 시작됩니다.

**AI-Native 대형 프로젝트:**
```
코어팀: NSKit 컨벤션 공유 (이미 정의됨)
  ↓ (즉시)
모듈팀 A: self-contained 파일로 독립 작업 → 완성 후 프로젝트에 통합
모듈팀 B: self-contained 파일로 독립 작업 → 완성 후 프로젝트에 통합
모듈팀 C: self-contained 파일로 독립 작업 → 완성 후 프로젝트에 통합
```

Self-contained 파일은 모듈팀 간 간섭이 없습니다. 각 팀이 NSKit 컨벤션만 지키면 백본 완성 전에도 독립적으로 작업하고, 완성된 결과물을 전체 프로젝트에 끼워넣을 수 있습니다. 오히려 자유도가 높아지고, 병목이 사라집니다.

대규모 프로젝트에 AI-Native가 맞지 않다고 단정할 수 없습니다. 오히려 **팀 간 독립성과 병렬성**에서 강점을 가집니다.

### 솔직한 트레이드오프

AI-Native Design은 **인간 중심 안전망** (타입, 린팅, 모듈 경계)을 **AI 중심 효율** (토큰 비용, 정확도, 병렬성)과 교환합니다. AI가 대부분의 작업을 할 때 이것은 좋은 거래입니다. 인간이 AI 도움 없이 자신이 작성하지 않은 코드를 유지보수해야 할 때는 나쁜 거래입니다.

---

## FAQ

### "이거 그냥 옛날 코딩 아닌가요? 전부 인라인, 모듈 없이?"

아닙니다. 옛날 인라인 코드에는 컨벤션도, 라이프사이클 관리도, 스코핑 전략도 없었습니다. AI-Native Design은 엄격한 컨벤션 (`PART.define()`, `SCENE.define()`), IIFE 기반 스코프 격리, 구조화된 라이프사이클 (`onInit`, `onShow`, `cleanup`), 의도적인 파일 크기 목표를 가지고 있습니다.

핵심 차이: 옛날 코드는 개발자가 더 나은 방법을 몰라서 인라인이었습니다. AI-Native 코드는 특정 소비자 -- AI --를 위해 *최적화*되어 self-contained입니다.

### "타입 안전은요?"

타입 안전은 가치 있습니다. AI-Native Design은 이것을 다른 종류의 안전과 교환합니다: **격리 안전**. 각 파일이 self-contained이기 때문에, 한 파일의 버그가 다른 파일로 전파될 수 없습니다. 에러의 폭발 반경이 정확히 하나의 파일입니다.

### "팀 확장은요?"

AI-Native Design은 다르게 확장합니다. 더 많은 인간 개발자를 모듈 코드베이스에 추가하는 대신, self-contained 파일에서 병렬로 작업하는 AI 세션을 추가합니다. 3개 AI 세션을 가진 한 명의 개발자가 조율 오버헤드 없이 소규모 팀의 처리량을 맞출 수 있습니다.

### "특정 AI 모델에서만 작동하나요?"

원칙은 모델에 구애받지 않습니다. 모든 LLM이 이점을 얻습니다:
- 작은 컨텍스트 윈도우 (적은 토큰 = 높은 정확도)
- Self-contained 파일 (의존성 추적 불필요)
- 일관된 패턴 (convention over configuration)

### "점진적으로 도입할 수 있나요?"

네. React 앱을 다시 작성할 필요 없습니다:

1. **새 유틸리티/스크립트**: 단일 self-contained 파일로 작성
2. **내부 도구**: 어드민 패널이나 대시보드를 AI-Native 패턴으로
3. **프로토타입**: 빠른 프로토타이핑에 AI-Native 사용, 필요시 모듈로 마이그레이션
4. **파일 크기 인식**: 모듈 코드베이스에서도 800줄 이하를 유지하면 AI 정확도가 향상

파일 크기 원칙(원칙 2)만으로도 *어떤* 코드베이스에서든 AI 협업이 개선됩니다.

---

## 이 원칙으로 만들어진 프레임워크

이 원칙들은 [NSKit](https://nskit.io)을 통해 개발되고 검증되었습니다. NSKit은 AI-인간 협업을 위해 처음부터 설계된 프레임워크입니다.

NSKit은 여러 라이브 서비스를 구동하는 프로덕션 프레임워크입니다. 이 문서의 벤치마크, 파일 크기, 사례 연구는 실제 NSKit 프로젝트에서 가져왔습니다.

---

## 기여

이것은 살아있는 문서입니다. 다음이 있다면:
- 다른 프레임워크나 AI 모델의 벤치마크 데이터
- 프로덕션에서의 AI-Native 패턴 사례 연구
- 방법론에 대한 수정 사항

이슈나 풀 리퀘스트를 열어주세요. 이 원칙들을 도전하거나 다듬는 데이터를 특히 환영합니다.

---

## 라이선스

CC BY-NC-SA 4.0 — see [LICENSE](LICENSE)

---

<p align="center">
  <i>"코드의 대부분을 작성하는 개발자에 맞춰 최적화하세요.<br/>점점 더, 그 개발자는 AI입니다."</i>
</p>
