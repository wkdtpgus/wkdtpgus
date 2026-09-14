# Basic Memory 기여 패치: build_context 관계 타입/방향 필터

업스트림 `basicmachines-co/basic-memory` (기준 커밋 `b04d1b6`, 2026-09-13) 에 대한 패치 두 개입니다.

| 파일 | 내용 | PR 제목 |
|------|------|---------|
| `0001-docs-skills-...patch` | 스킬 문서 오타 수정. 산문 속 `[[링크]]`는 `references`가 아니라 `links_to` 관계를 만듭니다. 코드와 NOTE-FORMAT.md가 그렇게 동작합니다. | `docs(skills): inline wiki links create links_to relations, not references` |
| `0002-feat-mcp-...patch` | `build_context`에 `relation_types`, `exclude_relation_types`, `direction` 파라미터 추가. MCP 툴 → v2 API → MemoryClient → ContextService CTE까지 관통. CLI 옵션, 테스트, man 페이지, 문서, CHANGELOG 포함. | `feat(mcp): follow relations by type and direction in build_context` |

두 패치는 독립적이라 따로 PR을 내도 됩니다. 0001은 첫 PR로 내기 좋은 크기입니다.

## 적용 방법 (내 맥에서)

```bash
# 1. 포크 후 클론
git clone https://github.com/<내계정>/basic-memory.git
cd basic-memory
git remote add upstream https://github.com/basicmachines-co/basic-memory.git
git fetch upstream main

# 2. 브랜치 만들고 패치 적용 (-s 가 DCO Signed-off-by 를 내 이름으로 붙임)
git checkout -b feat/build-context-relation-filters upstream/main
git am -s /path/to/0001-*.patch /path/to/0002-*.patch

# 3. 로컬 검증 (Python 3.12+ 필요, 3.14rc 는 pydantic 이 깨짐)
just install
just check          # ruff lint + format + ty
just test-unit-sqlite

# 4. 푸시 후 PR
git push -u origin feat/build-context-relation-filters
```

기준 커밋 이후 upstream이 움직였다면 `git am` 이 충돌할 수 있습니다. 그때는 `git am --abort` 후 `git am -3 -s ...` 로 3-way 적용을 시도하세요.

## 커밋 메시지 주의

패치 안의 커밋 메시지에는 `Co-Authored-By: Claude ...` 줄이 들어 있습니다. 프로젝트가 LLM 협업을 명시적으로 권장하므로 그대로 두어도 되고, 빼고 싶으면 `git am` 후 `git commit --amend` 로 지우면 됩니다. DCO 체크는 `Signed-off-by` 줄만 봅니다.

## 검증 결과 (원격 컨테이너, Python 3.13.7, SQLite)

- 전체 단위 스위트 `tests/` 통과 (1893 passed, 2 skipped, 약 8분). `tests/mcp/test_tool_telemetry.py`는 span 속성 기대값에 새 파라미터 세 개를 추가해 맞췄습니다.
- 새로 추가한 테스트: 서비스 6개, MCP 툴 2개, API 라우터 2개, CLI 3개
- `ruff check src tests`, `ruff format`, `ty check` (변경 파일) 통과
- 새로 추가된 코드 라인은 전부 커버됨 (미커버 라인은 기존 코드)
- Postgres 경로(`_build_postgres_query`)는 같은 조인 조건에 같은 필터를 넣었지만 `# pragma: no cover` 영역이라 이 환경에서는 실행하지 못했습니다. PR 올리면 CI의 testcontainers Postgres 잡이 돌립니다.

## PR 본문 초안 (0002)

```markdown
## What

`build_context` (MCP tool, `/v2/projects/{id}/memory/...`, `bm tool build-context`)
gains three optional traversal filters:

- `relation_types` — follow only edges of these types
- `exclude_relation_types` — skip edges of these types; `["links_to"]` drops implicit prose mentions
- `direction` — `outgoing` (what the note declares), `incoming` (what points at it), `both` (default)

Filters apply at every hop of the recursive CTE, so `relation_types=["depends_on"]` with
`depth=2` returns transitive dependencies and nothing else. The primary note is never filtered.
With no filter the generated SQL is unchanged.

## Why

Notes already carry typed relations (`- caused_by [[Outage]]`) and prose wiki links are indexed
as `links_to`, but `find_related` joined every edge alike. Once `max_related` caps the result
set (cut by `ORDER BY depth, type, id`), mentions and typed relations compete for the same
budget and which ones survive is arbitrary. An agent that wants "the causes of this decision"
or "what depends on this spec" has no way to ask for it and pays for the whole neighborhood.

This complements #1516 (compact mode reduces *how much* of each result is returned) by
controlling *which* results make it into the budget. Related: #1296, #1291/#1260.

## How

- `ContextService.find_related` builds the edge filter fragments and binds type lists as
  expanding parameters; both the SQLite and Postgres CTEs take the same fragments.
- v2 memory router exposes them as repeatable query params; `MemoryClient` forwards them;
  the MCP tool accepts lists, comma strings or JSON-array strings (same as `search_notes`)
  and normalizes in the body too, since plain-function callers bypass the BeforeValidator.
- `bm tool build-context --relation-type/--exclude-relation-type/--direction`.
- build-context(3) SYNOPSIS regenerated via `just man-regen`; DESCRIPTION and GOTCHAS updated.

## Testing

- New tests for the service (include/exclude/direction/depth interaction/invalid direction),
  the MCP tool (typed vs prose edges, comma strings, incoming/outgoing), the API router
  (query params, 422 on bad direction) and the CLI options.
- `just check` and `just test-unit-sqlite` pass locally.
```

## 이슈 초안 (PR 전에 먼저 열고 싶다면)

```markdown
**Title:** build_context cannot follow relations by type or direction

Relations are typed at write time and prose links are `links_to`, but `build_context`
follows every edge and `max_related` cuts by `ORDER BY depth, type, id`. There is no way
to ask for "only `depends_on`", "everything except prose mentions", or "what points at
this note". Proposal: `relation_types`, `exclude_relation_types`, `direction` on the tool,
API and CLI, applied at every hop, default = current behaviour. Happy to open a PR.
```

## 다음 단계 아이디어 (이 PR 범위 밖)

- 관계 타입 "계열" 권장 어휘를 스킬/문서에 표로 추가: 선후(`leads_to`, `follows`, `replaces`), 인과(`caused_by`), 구조(`part_of`, `contains`), 참조(`links_to`, `relates_to`). 코드 강제가 아니라 컨벤션이라 프로젝트 철학("invent freely")과 충돌하지 않음.
- `output_format="text"` 에서 `### Relations` 를 타입별로 묶어 출력.
