# Changelog

## v1.1.0 (S-tier upgrade)

- **Added** prompt-injection rejection: task strings containing `<script>`, `javascript:`, or event-handler patterns are now rejected with `ValueError` (previously only logged, matching agent-swarm v1.7.6+ behavior)
- **Fixed** FAST tier fallback: was `anthropic/claude-3-haiku` (same as primary, useless); now `anthropic/claude-3.5-sonnet`
- **Fixed** RESEARCH tier fallback: same issue, now falls back to Sonnet
- **Removed** redundant COMPLEX tier (merged into QUALITY)
- **Expanded** `_meta.json` with full fields: license, homepage, 9 tags, commands, tiers, basedOn
- **Added** CHANGELOG.md documenting divergence from main agent-swarm skill

### Divergence from agent-swarm

This skill is based on agent-swarm but differs in:
- **Models**: Uses only Anthropic Claude models (Haiku, Sonnet 3.5, Sonnet 4, Opus) instead of OpenRouter multi-provider
- **Provider**: Requires Anthropic API key instead of OpenRouter API key
- **Cost profile**: Higher per-token costs but single-provider simplicity
- **Model IDs**: Uses `anthropic/...` prefix instead of `openrouter/...`

## v1.0.0 (Initial release)

- Forked from agent-swarm, adapted for Claude-only routing
- Claude Haiku as orchestrator/FAST tier
- Claude Sonnet for CODE/CREATIVE/REASONING
- Claude Sonnet 4 for QUALITY
- Claude Opus as final fallback for complex tasks
- Input validation and config patch safety from agent-swarm v1.7.3+
