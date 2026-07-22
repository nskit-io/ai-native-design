[🇰🇷 한국어](README.ko.md) · [🇯🇵 日本語](README.ja.md) · [🇨🇳 中文](README.zh.md)

# AI-Native Design

**Design principles for structuring code that AI understands, modifies, and ships.**

> Context: these principles power [**NSKit**](https://github.com/nskit-io/nskit-io) — *bound by structure, free to combine.* A production web-app platform that turns these principles into running services. AI visual-artifact tools (Claude Design, v0, Lovable) sit at a different layer — they generate mockups; NSKit applies AI-Native Design to the code that actually runs.

> What if we optimized code structure not for human reading habits, but for how AI actually processes and modifies code?

Modern frameworks like React, Vue, and Angular are masterfully designed for **Human Developer Experience** -- modularity, type safety, reusability, separation of concerns. These are the right goals when humans are the primary code authors.

But when AI is your primary collaborator -- reading your code, understanding context, making changes, and shipping features -- the optimization target shifts. **AI-Native Design** is a set of architectural principles that emerge when you ask: *"What structure lets AI work most accurately and efficiently?"*

This is not a framework or a library. It is a design philosophy, backed by real-world benchmarks from production systems.

---

## Table of Contents

- [The Core Tension](#the-core-tension)
- [The 6 Principles](#the-6-principles)
- [Beyond Frontend: AI-Native Backend](#beyond-frontend-ai-native-backend)
- [Token Efficiency Benchmark](#token-efficiency-benchmark)
- [Real-World Results](#real-world-results)
- [When to Use / When NOT to Use](#when-to-use--when-not-to-use)
- [FAQ](#faq)
- [Built on These Principles](#built-on-these-principles)

---

## The Core Tension

Human DX and AI DX are **different optimization targets** that lead to **different architectural decisions**.

| Concern | Human DX Optimization | AI DX Optimization |
|---|---|---|
| **File structure** | Small, focused modules with clear imports | Self-contained files with everything inline |
| **Dependencies** | Abstract, reuse, share across components | Minimize; every import is context AI must trace |
| **Type system** | Rich types for safety and documentation | Conventions over types; patterns over contracts |
| **File size** | As small as possible (SRP) | 500-800 lines sweet spot (context window efficiency) |
| **Build process** | Webpack, Vite, esbuild for optimization | None. Eliminates entire error categories |
| **State management** | Centralized stores (Redux, Pinia) | File-scoped; no shared mutable state |

Neither approach is "better." They solve for different constraints. When AI writes 80%+ of your code, the constraint that matters most is: **how accurately can AI understand and modify this file?**

---

## The 6 Principles

### 1. Self-Contained Files

**One file = HTML + CSS + JS.** No imports to trace. AI reads ONE file and gets full context.

```html
<!-- AI-Native: Self-Contained Component -->
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
    function render(part) { /* private rendering logic */ }
  })();
  </script>
</div>
```

Compare with the modular approach:

```javascript
// Modular: AI must read 5+ files to understand this component
import { Avatar } from './Avatar';
import { UserInfo } from './UserInfo';
import { useUser } from '../hooks/useUser';
import { useTheme } from '../context/ThemeContext';
import styles from './UserCard.module.css';
```

The modular approach is excellent for human teams -- clear separation, testable units, shared abstractions. But for AI, every `import` is a file it must read to understand the full picture. Self-contained files give AI 100% context in a single read.

---

### 2. The 500-800 Line Sweet Spot

This is arguably the most actionable insight in AI-Native Design. Through extensive real-world measurement, we found a clear relationship between file size and AI modification accuracy:

| File Size | AI Accuracy | Assessment |
|---|---|---|
| ~100 lines | ~95% | Excellent, but too granular (many files = many reads) |
| ~500 lines | ~85% | **Sweet spot begins** |
| ~800 lines | ~77% | **Sweet spot ends** |
| ~1,000 lines | ~70% | Acceptable maximum |
| 2,000+ lines | ~50% | Coin flip -- avoid |

<img src="https://mermaid.ink/img/eHljaGFydC1iZXRhCiAgICB0aXRsZSAiQUkgQWNjdXJhY3kgdnMgRmlsZSBTaXplIgogICAgeC1heGlzICJGaWxlIFNpemUgKGxpbmVzKSIgWzEwMCwgMjAwLCAzMDAsIDQwMCwgNTAwLCA2MDAsIDcwMCwgODAwLCA5MDAsIDEwMDAsIDE1MDAsIDIwMDBdCiAgICB5LWF4aXMgIkFjY3VyYWN5ICglKSIgNDAgLS0+IDEwMAogICAgbGluZSBbOTUsIDkzLCA5MSwgODgsIDg1LCA4MywgODAsIDc3LCA3NCwgNzAsIDYwLCA1MF0=" alt="AI Accuracy vs File Size" />

The sweet spot exists because:
- **Too small** (~100 lines): High accuracy per file, but AI must read 10+ files to understand a feature. Total token cost goes up, and cross-file reasoning introduces errors.
- **Too large** (2,000+ lines): AI loses track of earlier context. Modifications in one section inadvertently break another. Accuracy drops to a coin flip.
- **500-800 lines**: Enough context for a complete, self-contained component. Small enough for AI to hold the entire file in working memory with high fidelity.

**Real data from NSKit framework** (20 production JS files, 11,280 total lines):

| Category | File | Lines |
|---|---|---|
| Core | nskit-api.js | 866 |
| Core | nskit-util.js | 580 |
| Core | nskit-core.js | 329 |
| Core | nskit-lang.js | 373 |
| Core | nskit-auth.js | 199 |
| Core | nskit-event.js | 138 |
| Layout | nskit-page.js | 1,022 |
| Layout | nskit-dialog.js | 1,035 |
| Layout | nskit-panel.js | 903 |
| Layout | nskit-scene.js | 564 |
| Layout | nskit-part.js | 489 |
| Widget | ns-select.js | 1,040 |
| Widget | ns-table.js | 971 |
| Widget | ns-input.js | 729 |
| Widget | ns-checkbox.js | 701 |
| Widget | ns-textarea.js | 331 |
| Widget | nskit-widget.js | 433 |
| **Average** | | **564 lines** |

Average file size: **564 lines** -- right in the sweet spot.

---

### 3. No Build Process

No webpack. No Vite. No transpilation. No bundling. Save file, refresh browser, see changes.

This is not nostalgia for simpler times. It is a deliberate elimination of an entire category of errors that AI frequently introduces:

<img src="https://mermaid.ink/img/cGllIHRpdGxlICJCdWlsZCBFcnJvciBDYXRlZ29yaWVzIEVsaW1pbmF0ZWQgYnkgTm8tQnVpbGQiCiAgICAiRGVwZW5kZW5jeSBSZXNvbHV0aW9uIiA6IDI1CiAgICAiVHJhbnNwaWxhdGlvbiBFcnJvcnMiIDogMjAKICAgICJCdW5kbGUgQ29uZmlnIiA6IDIwCiAgICAiVHlwZSBNaXNtYXRjaCIgOiAxNQogICAgIlNvdXJjZSBNYXAgSXNzdWVzIiA6IDEwCiAgICAiSG90IFJlbG9hZCBGYWlsdXJlcyIgOiAxMA==" alt="Build Error Categories Eliminated" />

When AI modifies code in a build-based project, it must reason about:
- Will this import resolve correctly after bundling?
- Does the TypeScript type satisfy the compiler?
- Will tree-shaking remove this code path?
- Is the HMR boundary correct?

With no build process, the feedback loop is instant: **save, refresh, verify.** The entire class of "it compiles locally but fails in CI" disappears.

---

### 4. Convention over Configuration

Every component follows the same structure. AI learns the pattern once and applies it everywhere.

```javascript
// Page, Scene, Dialog, Panel, Part -- all identical structure
PART.define('component-name', {
    onInit: function(part) {
        part._data = {};           // private state
    },
    onShow: function(part, params) {
        // lifecycle: component becomes visible
    },
    setData: function(part, data) {
        // public API: external callers use this
    },
    _render: function(part) {
        // private: internal rendering
    },
    cleanup: function(part) {
        // lifecycle: teardown
    }
});
```

Whether it is a full page, a dialog modal, a sidebar panel, or a reusable part -- the `.define()` pattern is identical. This means:

- AI never guesses which lifecycle method to use
- New developers (and new AI sessions) learn ONE pattern
- Cross-component refactoring is predictable
- Template mixing becomes possible (same structure = composable)

---

### 5. Token Efficiency as a Design Goal

When AI reads your code, every line consumed is a token spent. Self-contained files are dramatically more token-efficient for AI operations.

**Scenario: Change a button's click handler in a card component.**

| | React (modular) | AI-Native (self-contained) |
|---|---|---|
| Primary file | ~150 lines | ~600 lines |
| Files AI must read | 8-15 (imports, hooks, context, types, styles) | 1 |
| Total tokens consumed | ~15,000 - 37,000 | ~700 - 1,200 |
| Build step after change | Yes | No |
| Side-effect risk | Shared hook/context may affect other components | Zero -- file is self-contained |

<img src="https://mermaid.ink/img/eHljaGFydC1iZXRhCiAgICB0aXRsZSAiVG9rZW5zIFJlcXVpcmVkIGZvciBTaW5nbGUgQ29tcG9uZW50IENoYW5nZSIKICAgIHgtYXhpcyBbIkFuZ3VsYXIiLCAiUmVhY3QiLCAiVnVlIiwgIkFJLU5hdGl2ZSJdCiAgICB5LWF4aXMgIlRva2VucyIgMCAtLT4gMzUwMDAKICAgIGJhciBbMzAwMDAsIDI1MDAwLCAyMDAwMCwgODAwXQ==" alt="Tokens Required by Paradigm" />

**The math:**
- React component change: ~25,000 tokens (midpoint estimate)
- AI-Native component change: ~800 tokens
- **Ratio: ~31x more token-efficient**

Over a session of 20 component modifications, that is 500,000 tokens vs 16,000 tokens. The compounding effect is significant -- especially when operating multiple AI sessions in parallel.

> **Note on methodology:** React/Vue/Angular token estimates are based on typical mid-size application structures with shared hooks, context providers, and type definitions. Actual numbers vary by project. See [benchmarks/token-comparison.md](benchmarks/token-comparison.md) for detailed methodology.

---

### 6. Parallel AI Sessions

Self-contained files with no shared mutable state unlock a powerful capability: **multiple AI agents working simultaneously on the same codebase with zero merge conflicts.**

```
Traditional (shared state):
  Session 1: editing UserCard.tsx       -- uses useAuth hook
  Session 2: editing ProfilePage.tsx    -- uses useAuth hook
  Session 3: editing AuthProvider.tsx   -- CONFLICT with Sessions 1 & 2

AI-Native (self-contained):
  Session 1: editing part-user-card.html     -- self-contained
  Session 2: editing scene-profile.html      -- self-contained
  Session 3: editing part-auth-form.html     -- self-contained
  Result: 3 files modified, zero conflicts, 3x throughput
```

This is not theoretical. In production, we routinely run 3+ AI sessions working on different template files simultaneously. Because each file is a closed universe -- its own HTML, CSS, JS, and scoped state -- there is no possibility of one session's changes breaking another session's work.

---

## Beyond Frontend: AI-Native Backend

AI-Native Design is not limited to frontend templates. The same principles apply to backend API architecture.

### The CommandHandler Pattern

Traditional Spring REST controllers require AI to trace through multiple files — controllers, services, security configs, DTOs — to understand a single endpoint. The CommandHandler pattern collapses this into a single method:

```java
@CommandHandler  // needAuth=true, allowGuest=false (default)
public ResponseVO getProfile(CommandContext ctx) {
    // User is GUARANTEED here. ctx.getUser() is never null.
    // AI doesn't need to check auth middleware or security config.
    UserVO user = ctx.getUser();
    return ResponseHelper.response(200, "Success", userService.getProfile(user.getUid()));
}
```

**Why AI loves this:**

| | Traditional REST | CommandHandler |
|---|---|---|
| **Understanding 1 API** | Read controller + service + security config + DTOs | Read ONE method |
| **Auth guarantee** | Check SecurityContext manually, trace interceptors | Annotation says it all — user is guaranteed or explicitly optional |
| **Adding new endpoint** | New URL mapping + method + route registration | Just add a method with `@CommandHandler` |
| **API spec → code** | URL path parsing, parameter mapping | `"get-profile"` → `getProfile()` — the command IS the method name |

**The Auth Contract:**
- `@CommandHandler` (default) → `ctx.getUser()` is **never null**. AI writes code with confidence.
- `@CommandHandler(allowGuest = true)` → user **may** be null. AI knows to add null checks.
- `@CommandHandler(needAuth = false)` → public endpoint. No auth at all.

This is the same "Convention over Configuration" principle from frontend applied to backend. AI learns the pattern once, applies it across every module.

See the full pattern documentation: **[command-handler](https://github.com/nskit-io/command-handler)**

---

## Token Efficiency Benchmark

### Methodology

We measured token consumption for a common operation: **modifying a button's click behavior inside a card component.**

**React project structure** (typical mid-size app):
```
src/
  components/
    Card/
      Card.tsx              ← 150 lines (primary file)
      Card.module.css       ← 80 lines
      Card.test.tsx         ← 120 lines
  hooks/
    useAuth.ts              ← 200 lines (shared hook)
    useApi.ts               ← 180 lines (shared hook)
  context/
    ThemeContext.tsx         ← 90 lines
  types/
    card.ts                 ← 60 lines
  utils/
    format.ts               ← 100 lines
  components/
    Button/
      Button.tsx            ← 80 lines (shared component)
```

To safely modify the button behavior, AI must:
1. Read `Card.tsx` to find the button (150 lines)
2. Read `Button.tsx` to understand the Button API (80 lines)
3. Read `useAuth.ts` to understand auth state (200 lines)
4. Read `useApi.ts` to understand API calls (180 lines)
5. Read `ThemeContext.tsx` for styling context (90 lines)
6. Read `card.ts` for type definitions (60 lines)
7. Read `format.ts` for utility functions (100 lines)
8. Read `Card.module.css` for style implications (80 lines)

**Total: ~940 lines across 8 files = ~18,800 tokens** (at ~20 tokens/line average)

Some projects are more modular (15+ files), some less (5-6 files). Our estimate range: **15,000 - 37,000 tokens.**

**AI-Native project structure:**
```
templates/
  parts/
    part-card.html          ← 600 lines (HTML + CSS + JS, complete)
```

AI reads one file: 600 lines = **~1,200 tokens.**

**Ratio: 15x to 31x more efficient**, depending on project modularity.

For full methodology and raw data, see [benchmarks/token-comparison.md](benchmarks/token-comparison.md).

---

## Real-World Results

### Case Study: NewMyoung

[NewMyoung](https://newmyoung.com) is an AI-powered baby naming and fortune service, live in production across multiple regions:
- **Korea**: newmyoung.com
- **Japan**: jp.newmyoung.com
- **Chinese-speaking regions** (Taiwan, Singapore, Macau, Malaysia, Hong Kong): zh.newmyoung.com

**Built using AI-Native Design principles:**

| Metric | Value |
|---|---|
| Time to core feature completion | ~2 days |
| Template files | 31 |
| Average file size | ~500 lines |
| AI sessions running in parallel | 3+ |
| Build process | None |
| Production uptime | Stable since launch |

**How parallel AI sessions worked:**
- Session 1: Korean character selection UI (part-char-select.html)
- Session 2: Fortune analysis display (scene-analysis.html)
- Session 3: Stroke count selection (part-stroke-select.html)

All three sessions worked simultaneously. No merge conflicts. No shared state contamination. Each file was self-contained.

**Multi-country expansion** was also accelerated by self-contained files. Localizing a template meant reading one file, understanding its complete behavior, and adapting text/layout -- without tracing imports or worrying about shared state in other locales.

---

## When to Use / When NOT to Use

### AI-Native Design works well when:

- AI is your primary code collaborator (writing 50%+ of code)
- You need rapid prototyping that goes straight to production
- You run multiple AI sessions in parallel
- Your team is small (1-5 people) or solo
- You want zero build configuration
- You value deployment simplicity (copy files, refresh)

### AI-Native Design may not fit when:

- **Complex shared state**: Applications with deeply interconnected real-time state (real-time collaborative editors) benefit from centralized state management.
- **Rich ecosystem needs**: If you depend heavily on React/Vue/Angular ecosystem libraries (component libraries, testing frameworks, dev tools), modular architecture is the pragmatic choice.
- **SEO-heavy applications**: Frameworks with SSR/SSG (Next.js, Nuxt, Astro) provide optimizations that AI-Native's no-build approach does not address.

### The misconception about large-scale projects

There is a common belief that "large teams need modular architecture." But consider how large projects actually work:

**Traditional large project bottleneck:**
```
Core team: Build framework backbone → Configure build environment → Finalize code conventions
  ↓ (only after this is done)
Module team A: Can begin work
Module team B: Can begin work
Module team C: Can begin work
```

Module teams are blocked until the core team finishes the backbone. Enterprise-wide build rules, code conventions, and shared components must be finalized before module-level development begins.

**AI-Native large project:**
```
Core team: Share NSKit conventions (already defined)
  ↓ (immediately)
Module team A: Works independently on self-contained files → Integrates into project
Module team B: Works independently on self-contained files → Integrates into project
Module team C: Works independently on self-contained files → Integrates into project
```

Self-contained files eliminate inter-team interference. Each team only needs to follow the convention. They can work independently before the backbone is complete and plug their finished results into the overall project. Freedom increases, bottlenecks disappear.

Large-scale projects are not inherently a bad fit for AI-Native. In fact, AI-Native has a distinct advantage in **team independence and parallelism**.

### The honest trade-off

AI-Native Design trades **human-centric safety nets** (types, linting, module boundaries) for **AI-centric efficiency** (token cost, accuracy, parallelism). This is a good trade when AI is doing most of the work. It is a bad trade when humans need to maintain code they did not write, without AI assistance.

---

## FAQ

### "Isn't this just old-school coding? Inline everything, no modules?"

No. Old-school inline code had no conventions, no lifecycle management, no scoping strategy. AI-Native Design has strict conventions (`PART.define()`, `SCENE.define()`), IIFE-based scope isolation, structured lifecycles (`onInit`, `onShow`, `cleanup`), and a deliberate file-size target. The files are self-contained, but they are not unstructured.

The key difference: old-school code was inline because developers did not know better. AI-Native code is self-contained because it is *optimized for a specific consumer* -- AI.

### "What about type safety?"

Type safety is valuable. It catches bugs at compile time that would otherwise surface at runtime. AI-Native Design trades this for a different kind of safety: **isolation safety.** Because each file is self-contained, a bug in one file cannot propagate to another. The blast radius of any error is exactly one file.

For applications where type safety is critical (finance, healthcare), use TypeScript with modular architecture. For applications where speed and AI efficiency matter more, AI-Native Design offers a different set of guarantees.

### "What about team scaling?"

AI-Native Design scales differently. Instead of scaling by adding more human developers to a modular codebase, you scale by adding more AI sessions working on self-contained files in parallel. A single developer with 3 AI sessions can match the throughput of a small team, with zero coordination overhead.

For large human teams (10+), modular architecture with clear module boundaries remains the better choice. The coordination problems that modular architecture solves (shared types, enforced interfaces, code review checklists) are real and important when many humans touch the same codebase.

### "Does this only work with Claude / specific AI models?"

The principles are model-agnostic. Any LLM benefits from:
- Smaller context windows (fewer tokens = higher accuracy)
- Self-contained files (no dependency tracing)
- Consistent patterns (convention over configuration)

The benchmarks in this document were measured using Claude, but the architectural principles apply equally to GPT-4, Gemini, or any future model. The 500-800 line sweet spot may shift as context windows grow, but the fundamental insight -- that AI accuracy degrades with file size -- is universal.

### "Can I adopt these principles incrementally?"

Yes. You do not need to rewrite your React app. Consider:

1. **New utilities/scripts**: Write them as single self-contained files
2. **Internal tools**: Build admin panels or dashboards using AI-Native patterns
3. **Prototypes**: Use AI-Native for rapid prototyping, then migrate to modular if needed
4. **File size awareness**: Even in modular codebases, keeping files under 800 lines improves AI accuracy

The file-size principle alone (Principle 2) improves AI collaboration in *any* codebase.

### "What about testing?"

Self-contained files are tested through integration/E2E testing rather than unit testing. Since each file is a complete component with its own HTML, CSS, and JS, the natural test boundary is the component as rendered in a browser, not individual functions in isolation.

This is a trade-off: you lose granular unit test coverage but gain tests that verify actual user-facing behavior. For many applications, this is a net positive.

---

## Built on These Principles

These principles were developed and validated through [NSKit](https://nskit.io), a framework designed from the ground up for AI-human collaboration.

NSKit is a production framework powering multiple live services. The benchmarks, file sizes, and case studies in this document are drawn from real NSKit projects.

---

## Contributing

This is a living document. If you have:
- Benchmark data from other frameworks or AI models
- Case studies of AI-Native patterns in production
- Corrections to our methodology

Please open an issue or pull request. We are especially interested in data that challenges or refines these principles.

---

## License

CC BY-NC-SA 4.0 — see [LICENSE](LICENSE)

---

<p align="center">
  <i>"Optimize for the developer who writes most of your code.<br/>Increasingly, that developer is AI."</i>
</p>
