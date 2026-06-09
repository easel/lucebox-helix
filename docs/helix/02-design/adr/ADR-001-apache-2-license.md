---
ddx:
  id: ADR-001
  status: accepted
  links:
    - id: prd
      rel: informs
---

# ADR-001: Apache 2.0 License

| Date | Status | Deciders | Related | Confidence |
|------|--------|----------|---------|------------|
| 2026-06-08 | Accepted | Erik LaBianca | — | High |

## Context

| Aspect | Description |
|--------|-------------|
| Problem | Lucebox-hub is public on GitHub with 2,200+ stars and 45+ contributors; a license decision is overdue and community expectations are already set |
| Current State | LICENSE file at repo root declares Apache 2.0, Copyright 2026 Lucebox |
| Requirements | Must not alienate contributors or prevent community distribution; must be compatible with the subscription model (curated model library, support tiers) as a business layer on top |
| Decision Drivers | Community trust already built on the assumption of open source; subscription differentiation comes from the model library and support SLA, not from locking the inference engine |

## Decision

We will license the lucebox-hub software under Apache 2.0.

**Key Points**: Already declared in the repo LICENSE file | Subscription model differentiates via curated model library and support, not software licensing | Apache 2.0 is compatible with all harness-adapter upstream tools (Claude, Codex, OpenCode) and the GGUF/llama.cpp ecosystem

## Alternatives

| Option | Pros | Cons | Evaluation |
|--------|------|------|------------|
| MIT | Simplest; maximum permissiveness | No patent grant; weaker than Apache for enterprise adopters | Rejected: Apache 2.0 is the right default for infrastructure with GPU vendor interactions |
| BSL (Business Source License) | Prevents commercial competitors from packaging and reselling | Community-trust hit; 2,200 stars built on open-source assumption; incompatible with contributor agreements already in flight | Rejected: the damage to community trust outweighs the competitive protection |
| Proprietary / open core | Maximum commercial protection on inference engine | Eliminates community contribution to the core; forks the 45-contributor base; conflicts with CUDA/HIP kernel dependencies on open components | Rejected: contradicts the product's open-source identity and existing GitHub presence |
| **Apache 2.0** | Patent grant included; enterprise-friendly; contributor-friendly; ecosystem-standard for inference infrastructure | Competitors can fork and redistribute; no time-delay protection | **Selected**: subscription model differentiates on model library and support, not software access |

## Consequences

| Type | Impact |
|------|--------|
| Positive | Community contributions remain open; enterprise adopters have clear patent grant; compatible with all upstream dependencies |
| Negative | Competitors can fork the inference engine without contributing back; no BSL-style delay period |
| Neutral | Subscription tier (curated models, support SLA) is a business layer and is not governed by this ADR |

## Risks

| Risk | Prob | Impact | Mitigation |
|------|------|--------|------------|
| Competitor forks and ships optimizations without contributing back | Med | Low | Apache 2.0 requires attribution; community momentum and hardware-specific tuning create durable moats regardless |

## Validation

| Success Metric | Review Trigger |
|----------------|----------------|
| Community contribution rate remains healthy | If a competitor ships a fork that materially harms the project, revisit BSL for new components only |

## Supersession

- **Supersedes**: None
- **Superseded by**: None

## Concern Impact

- **No concern impact**: License does not directly affect any active project concern.

## References

- `LICENSE` at lucebox-hub repo root (Apache 2.0, Copyright 2026 Lucebox)
- PRD §Constraints — "Bundled open-weight models must have licenses permitting commercial distribution"
