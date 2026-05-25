# Prompt Architecture Reference

## Four-Layer Composition

```
fn compose_prompt(mode: AppMode, personality: Personality) -> String {
    [
        BASE_PROMPT,          // Layer 1: Core identity + methodology
        personality.prompt(), // Layer 2: Voice/tone overlay
        mode_prompt(mode),    // Layer 3: Mode-specific permissions
        approval_prompt(),    // Layer 4: Tool approval behavior
    ].join("\n\n")
}
```

## Layer 1: Core Identity (base.md)

Source: `crates/tui/src/prompts/base.md`

Defines:
- Dual-mode classification (Chat vs Security)
- Chat: pure text assistant, no tools, no agents
- Security: coordinator role, full methodology
- "OBSERVE → HYPOTHESIZE → PREDICT → TEST → CONCLUDE" reasoning chain

## Layer 2: Personality (personalities/)

Source: `crates/tui/src/prompts/personalities/`

| File | Style |
|------|-------|
| calm.md | Cool, spatial, reserved. Engineer in a quiet room. |
| playful.md | Engaging, enthusiastic, creative. |

## Layer 3: Mode (modes/)

Source: `crates/tui/src/prompts/modes/`

| File | Permissions |
|------|------------|
| agent.md | Read-only auto; writes/patches/shell need approval |
| plan.md | Read-only; all writes blocked; investigation only |
| yolo.md | Full autonomy; all actions auto-approved |

## Layer 4: Approval (approvals/)

Source: `crates/tui/src/prompts/approvals/`

| File | Behavior |
|------|----------|
| auto.md | All tools auto-approved |
| suggest.md | Suggest approval, user confirms |
| never.md | Never approve; always ask |
