# ECDC

## Project Context

@.kiro/steering/product.md
@.kiro/steering/tech.md
@.kiro/steering/structure.md

## Commands

```bash
npm run dev        # Dev server (Vite)
npm run build      # Production build
npm test           # Unit tests
npx tsc --noEmit   # Type check
```

## Language

IMPORTANT: Think in English, respond in Japanese. All project Markdown files (requirements.md, design.md, tasks.md, etc.) MUST be written in the language configured in `spec.json.language`.

## Spec-Driven Workflow

Specs: `.kiro/specs/` | Steering: `.kiro/steering/`

1. `/kiro:spec-init "description"` → `/kiro:spec-requirements {feature}` → `/kiro:spec-design {feature}` → `/kiro:spec-tasks {feature}`
2. `/kiro:spec-impl {feature} [tasks]`
3. `/kiro:spec-status {feature}` — check progress anytime

- 3-phase approval: Requirements → Design → Tasks → Implementation
- Human review required each phase; `-y` only for intentional fast-track
- Optional validation: `/kiro:validate-gap`, `/kiro:validate-design`, `/kiro:validate-impl`
- Steering management: `/kiro:steering`, `/kiro:steering-custom`

## Verification

After code changes, verify in this order:
1. `npx tsc --noEmit` — type check
2. `npm test` — tests pass
3. `npm run build` — build succeeds (for build-affecting changes)

## Context Management

When compacting, preserve: modified file list, current spec phase, and test results.
