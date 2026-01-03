# CLAUDE.md

Guidance for Claude Code. Keep this file concise—details belong in agent files.

## Git Workflow
- Default branch: `main`
- One commit = one logical change, short message
- No Co-Authored-By
- **Use separate commands** instead of chaining (e.g., run `git add`, then `git commit`, then `git push` separately) - pre-approved commands only work when run individually
- **Meaningful branch names**: Use `<type>/<description>` format (e.g., `feature/add-copy-trading`, `fix/sync-cleanup`, `refactor/repository-pattern`)

### Stacked PRs
When branching from an unmerged PR:
- Add `> **Depends on:** #XX` at the top of PR description
- Mark as **Draft** until parent PR is merged
- Keep stack depth ≤ 4 PRs
- Avoid overlapping file changes between stacked PRs
- After parent merges: rebase onto main, update PR base branch

## Parallel Development (Multiple Claude Instances)

Always work in a separate directories as if you're fully independent.

### Picking Tasks from GitHub Issues

1. **Pull latest main**: `git pull origin main` - Always sync before starting new work
2. **List open issues**: `gh issue list --label ready --state open`
3. **Check assignee**: If issue has an assignee, someone is already working on it
4. **Assign yourself**: `gh issue edit <number> --add-assignee @me`
5. **When done**: PR will auto-close issue via "Closes #XX" in description

## Philosophy

**KISS** - Simple solutions over complex ones
**YAGNI** - Build only what's needed now
**DRY** - Single source of truth
**Small Steps** - Minimal changes, commit often

## Testing

- **Prefer property-based tests** over example-based tests
- Test behavior, not implementation
- Cover happy paths, edge cases, isolation
- Each test verifies one behavior
- **Always use `@test-writer` agent** to write tests (ensures property tests are used)

**Python**: Use `hypothesis` for property tests (`@given`, strategies)
**Solidity**: Use fuzz testing (`testFuzz_`), invariants, `makeAddr()`
**Frontend**: Use `getByRole` > `getByLabelText` > `getByText` > `getByTestId`

## Design Principles

- Single Responsibility, Open/Closed, Dependency Inversion
- Pure functions, minimal side effects, early returns
- Fail fast, explicit inputs/outputs
- **No hardcoded values** - Extract all constants to config files

## Python OOP+FP Hybrid (Scala-style)

Full design: `docs/python-oop-fp-hybrid.md`

### When to Use Classes vs Functions

| Use Classes For | Use Pure Functions For |
|-----------------|------------------------|
| DB/API boundaries (Repository, Client) | Data transformations |
| Resource management (connections, sessions) | Business logic, validation |
| Dependency injection containers | Calculations, filtering |
| Stateful components (trackers, caches) | Mapping, composing |

### Core Patterns

**Repository Pattern**: Classes encapsulate session, expose domain operations
```python
class TradeRepository(SessionRepository[Trade, int]):
    async def save(self, trade: Trade) -> Trade: ...
    async def find_by_trader(self, trader: str) -> Sequence[Trade]: ...
```

**Unit of Work**: Transaction boundaries, lazy repository access
```python
async with UnitOfWork(session_factory) as uow:
    await uow.trades.save(trade)
    await uow.leaders.increment_trade_count(...)
    await uow.commit()
```

**Pure Functions in domain/**: Stateless, testable logic
```python
def classify_transfer(event: TransferEvent) -> Literal["BUY", "SELL"] | None
def normalize_amount(wei_value: int, decimals: int = 18) -> Decimal
```

**Immutable Value Objects**: `@dataclass(frozen=True)` for data transfer
```python
@dataclass(frozen=True)
class TradeData:
    tx_hash: str
    trader: str
    side: Literal["BUY", "SELL"]
```

### File Structure
```
backend/src/polycopy/
    domain/           # Pure functions (trade_logic.py, value_objects.py)
    repositories/     # SessionRepository classes + UnitOfWork
    services/         # Orchestration (IndexerService, CopyTradingService)
    clients/          # External APIs (Web3Client, PolymarketClient)
```

## Agents

**TDD Cycle:**
- `@test-writer` - RED: write failing test
- `@implementer` - GREEN: minimal code to pass
- `@refactorer` - REFACTOR: improve structure

**Quality Gates:**
- `@code-reviewer` - quality, maintainability
- `@security-auditor` - vulnerabilities, OWASP
- `@architect-reviewer` - architecture, SOLID

**Orchestration:**
- `@orchestrator` - coordinate parallel workers

## TDD Workflow (MANDATORY)

**ALWAYS follow TDD for any new feature or bugfix. No exceptions.**

```
1. RED:      @test-writer "<requirement>" → failing test
2. GREEN:   @implementer → minimal passing code
3. REFACTOR: @refactorer → improve (tests stay green)
4. REPEAT
```

**Rules:**
- **NEVER write implementation before a failing test exists**
- Never skip RED phase
- Never modify tests to make them pass
- Minimal implementation only
- If you catch yourself writing code first, STOP and write the test

## Commands

| Command | Purpose |
|---------|---------|
| `/tdd <feature>` | Full TDD cycle |
| `/red <req>` | Write failing test |
| `/green` | Implement to pass |
| `/refactor` | Improve code |
| `/swarm <task>` | Parallel workers |
| `/review <scope>` | Quality gate reviews |

## Swarm Mode

`/swarm <task>` - Orchestrator breaks work into parallel subtasks with non-overlapping file scopes.

**Parallel**: Independent modules, features, directories
**Sequential**: TDD phases, dependent migrations, API→implementation

## Quality Gates

| Stage | Check | Required |
|-------|-------|----------|
| 1. Tests | `npm test` / `forge test` | Always |
| 2. Static | Lint, typecheck, compiler | Always |
| 3. Code Review | `@code-reviewer` | PR |
| 4. Security | `@security-auditor` | Deploy |
| 5. Architecture | `@architect-reviewer` | Major changes |

Verdicts: ✅ Approved | ⚠️ Needs changes | ❌ Rejected

## Project Overview

Project goal is to create a simple and useful copy-trading for Polymarket.
Design is available ad docs/design.md
Useful info about competitors might be found in docs/knowledge