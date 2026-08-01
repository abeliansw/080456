# Test Coverage Analysis & Improvement Proposal

Analysis date: 2026-08-01
Scope: entire repository (`abeliansw/080456`)

---

## 1. Current state: coverage is 0%

There is no test code of any kind in this repository:

| Signal | Result |
| --- | --- |
| Test files (`test_*.py`, `*_test.py`, `tests/`) | **0** |
| Test framework config (`pytest.ini`, `pyproject.toml`, `tox.ini`, `conftest.py`) | **0** |
| CI configuration (`.github/workflows/`, or any other) | **0** |
| Dependency manifest (`requirements.txt`, `pyproject.toml`, lockfile) | **0** |
| Assertions anywhere in the codebase | **0** |
| `.py` source files | **0** |

The codebase is **17 Jupyter notebooks / 400 cells / 239 code cells**, plus a README
and a PDF. All executable logic lives inside notebook cells; nothing is importable,
so nothing is testable in its present form.

The closest thing to a test today is `LangSmith/성능_평가하기.ipynb`, which builds a
3-example LangSmith dataset of question/answer pairs. It uploads the dataset but
never runs an evaluator against it and never asserts anything — it is a manual demo,
not a regression check.

### Why this matters for this repository specifically

This is book source code. The reader experience *is* the product: a reader opens a
notebook in Colab, runs the cells, and expects them to work. A broken cell is a
book erratum. Right now there is no mechanism — automated or otherwise — that
would tell the authors a notebook has stopped working.

And notebooks have in fact stopped working. Measurements below.

---

## 2. Defects already present that a minimal test suite would have caught

These are not hypotheticals. Each was found by a check that takes seconds to
automate, and each is currently shipped to readers on `main`.

### 2.1 One notebook is not a valid notebook — `testCRM20251111.ipynb`

The file contains a single newline byte (1 byte total). It is not valid JSON and
will not open in Jupyter, Colab, or nbformat:

```
$ python -c "import json; json.load(open('testCRM20251111.ipynb'))"
json.decoder.JSONDecodeError: Expecting value: line 2 column 1 (char 1)
```

A one-line `nbformat.read()` check over every `*.ipynb` catches this.

### 2.2 80 of 239 code cells (33%) are not valid Python

`testCRMSimple.ipynb` has markdown headings pasted into **code** cells — 80 cells
containing text like:

```python
## 분석 결과 및 시사점

### Subtask:
학습된 선형 회귀 모델의 성능 평가 결과와 ...
```

Every one of these raises `SyntaxError` on execution. An `ast.parse()` pass over
each code cell (with `!`/`%` lines stripped) catches all 80.

### 2.3 85 error tracebacks are committed into notebook outputs

The stored outputs prove the notebooks were shipped in a failing state:

| Error | Cells | Notebook |
| --- | --- | --- |
| `SyntaxError: invalid syntax` | 80 | `testCRMSimple.ipynb` |
| `AttributeError: module 'matplotlib.font_manager' has no attribute '_rebuild'` | 2 (cells 19, 23) | `testCRMSimple.ipynb` |
| `AssertionError: The shape of the shap_values matrix does not match ...` | 2 (cells 245, 247) | `testCRMSimple.ipynb` |
| `UFuncTypeError: Cannot cast ufunc 'add' output from float64 to int64` | 1 (cell 228) | `testCRMSimple.ipynb` |

`fm._rebuild()` was removed in matplotlib 3.6. The notebook even documents the
failure in a later cell (`# fm._rebuild() # Removed as it's causing an AttributeError`)
— but the broken cells were never deleted, so a reader hits the error twice before
reaching the working version.

A check for `output_type == "error"` in committed outputs is trivial and would have
blocked all four of these.

### 2.4 A silent bug: `tool=` instead of `tools=` in CrewAI

`CrewAI/특정_유튜브_채널에서_데이터_검색하기.ipynb` cell 3 constructs both agents with
the wrong keyword:

```python
searching = Agent(
    role='YouTube 비디오에서 전문가 수준의 콘텐츠를 연구',
    ...
    tool = [yt_tool],      # <- should be tools=[yt_tool]
    allow_delegation=True
)
```

CrewAI's `Agent` parameter is `tools`. The sibling notebook
`CrewAI/데이터_검색_및_내용_작성하기.ipynb` gets it right (`tools=[search_tool]`) in the
same position, which confirms this is a typo, not a deliberate API difference. The
consequence is quiet: the agent is constructed without its YouTube search tool, so
the notebook appears to run while the tool it is meant to demonstrate is never used.
This is exactly the class of bug that no amount of "the cell didn't crash" checking
finds — it needs an assertion on the constructed object.

### 2.5 12 uses of deprecated or already-removed library APIs

Nothing in the repository pins a version except `httpx==0.27.2`. Every `!pip install`
resolves to whatever is latest on the day the reader runs it:

```
!pip install langgraph langchain langchain_openai tavily-python langchain_community "httpx==0.27.2"
```

Against current library versions, the following are deprecated or gone:

| Notebook | API | Status |
| --- | --- | --- |
| `LangGraph/Tavily를_이용한_정보검색하기` | `langgraph.prebuilt.ToolInvocation`, `ToolExecutor` | **removed** in langgraph ≥ 0.2 |
| `LangChain/랭체인에서_에이전트_사용하기` | `from langchain.chat_models import ChatOpenAI` | moved to `langchain_openai` |
| `LangChain/랭체인에서_에이전트_사용하기` | `initialize_agent`, `ConversationBufferMemory`, `DocstoreExplorer`, `TavilyAnswer` | deprecated |
| `LangGraph/RAG_&_검색_에이전트_생성하기` | `langchain.embeddings.openai`, `langchain_community.vectorstores.Chroma` | deprecated |
| `LangGraph/멀티_에이전트_생성하기` | `from langchain.llms import OpenAI` | moved |
| `testCRMSimple` | `matplotlib.font_manager._rebuild` | **removed** in matplotlib ≥ 3.6 |

`ToolInvocation`/`ToolExecutor` are the serious one: that notebook cannot run at all
on a current langgraph install.

### 2.6 23 hardcoded API-key assignment sites

Every notebook that needs credentials sets them as string literals in a code cell:

```python
os.environ["OPENAI_API_KEY"] = "sk"      # 12 notebooks
os.environ["TAVILY_API_KEY"] = "tvly"    # 5 notebooks
os.environ["LANGCHAIN_API_KEY"] = "lsv2" # 3 notebooks
os.environ["SERPER_API_KEY"] = ""        # 1 notebook
```

The placeholders are currently harmless. The risk is structural: the repository's
documented workflow instructs the reader (and the author) to type a real key into a
cell that is tracked by git. One `git commit` after a working session leaks a live
credential. Notebook outputs are also committed (see below), which is a second leak
path — LangSmith project names are already visible in `LANGCHAIN_PROJECT`.

### 2.7 Committed outputs make review impossible

Outputs account for **71–109%** of stored file size (base64 PNGs inflate the ratio
past 100% before JSON escaping):

| Notebook | Size | Outputs |
| --- | --- | --- |
| `testCRMSimple.ipynb` | 1131 KB | 831 KB (73%) |
| `CrewAI/데이터_검색_및_내용_작성하기.ipynb` | 114 KB | 113 KB (99%) |
| `CrewAI/특정_유튜브_채널에서_데이터_검색하기.ipynb` | 106 KB | 107 KB (101%) |
| `AutoGPT/AutoGPT.ipynb` | 89 KB | 85 KB (96%) |

A one-line source change produces a multi-hundred-KB diff. This is why the 85 error
tracebacks went unnoticed: nobody can read these diffs.

---

## 3. Proposed test improvements, in priority order

The recommendation is deliberately staged. Tiers 1 and 2 need no API keys, no
network, and no money — they would have caught every defect in Section 2 except 2.4.
Tier 3 costs real API calls and should run on a schedule, not per-commit.

### Tier 1 — Static notebook validation (highest value, lowest cost)

Add `tests/test_notebooks_static.py` running under pytest, parametrized over every
`*.ipynb`. No dependencies beyond `pytest` and `nbformat`. Target runtime: < 5 s.

1. **Notebook parses** — `nbformat.read()` succeeds and validates.
   *Catches 2.1. Currently fails on `testCRM20251111.ipynb`.*
2. **Every code cell is valid Python** — strip `!`/`%` lines, then `ast.parse()`.
   *Catches 2.2. Currently fails on 80 cells.*
3. **No committed error outputs** — assert no cell has `output_type == "error"`.
   *Catches 2.3. Currently fails on 85 cells.*
4. **No credential-shaped literals** — reject any string matching
   `sk-[A-Za-z0-9]{20,}`, `tvly-…`, `lsv2_…`, in both sources **and** outputs.
   *Guards 2.6.*
5. **No banned/removed APIs** — a small deny-list (`ToolInvocation`, `ToolExecutor`,
   `fm._rebuild`, `langchain.chat_models`, …) asserted absent, so a known-dead API
   cannot be reintroduced. *Catches 2.5 as a regression guard.*
6. **Outputs are stripped** — assert `outputs == []` and `execution_count is None`
   for all cells, enforced by `nbstripout` as a pre-commit hook. *Fixes 2.7 and makes
   every later diff reviewable.*

Note that checks 1–3 and 6 currently **fail**. That is the point: adopt them as a
red build, fix the notebooks, then keep them green.

### Tier 2 — Extract agent logic and unit-test it with fake LLMs

The genuinely interesting logic in this repository is the **graph control flow**, and
it is all pure or near-pure — it takes a state dict and returns a routing decision.
It needs no LLM to test. Today it is untestable only because it is trapped in cells.

Proposal: extract these into a small importable package (e.g. `src/agents/`), have
the notebooks `import` from it, and unit-test the functions directly using
`langchain_core.language_models.fake.FakeListLLM` / `FakeMessagesListChatModel`.

Highest-value targets:

| Function | Notebook | What to assert |
| --- | --- | --- |
| `should_continue` | `LangSmith/LangGraph와_디버깅_연동하기`, `LangGraph/Tavily…` | returns `"tools"` when the last message has tool calls, `"__end__"` when it does not; **and does not crash on an empty message list** |
| `should_end` | `LangGraph/LangGraph에서_ReAct…` | returns `END` when `response` is set, `"agent"` when it is absent *or empty* |
| `analyze_question` routing | `LangGraph/멀티_에이전트_생성하기` | see robustness note below |
| `relevant_documents` | `LangGraph/RAG_&_검색…` | keeps docs graded `yes`, drops `no`, sets `web_search` correctly; **behavior when the grader returns malformed JSON** |
| `state_transition` | `Autogen/두_개_이상의_AssistantAgent…` | full speaker chain proxy → design → marketing → pm → `None` |
| `execute_step` | `LangGraph/LangGraph에서_ReAct…` | task string formatting; `past_steps` accumulates rather than overwrites |

Two robustness gaps these tests would expose immediately:

- **`멀티_에이전트_생성하기` routing is fragile.** `analyze_question` returns
  `response.content.strip().lower()` and the conditional edge map has exactly two
  keys, `"code"` and `"general"`. If the model answers `"code."` or `"general question"`
  — both plausible — the graph raises instead of routing. A fake-LLM test
  parametrized over realistic model replies pins this down; the fix is a
  `startswith` match plus a default branch.
- **`relevant_documents` trusts the grader blindly.** It does
  `score = retrieval_grader.invoke(...)` then `grade = score['score']`, where the
  chain ends in `JsonOutputParser()`. Any non-conforming model output is a `KeyError`
  or `TypeError` that aborts the whole graph. A test with a fake LLM returning
  malformed JSON documents the required fallback.

Also worth noting: the same `should_continue` logic appears in two notebooks with a
subtle difference — one checks `"function_call" not in last_message.additional_kwargs`
(the legacy OpenAI functions API), the other `"tool_calls" not in ...` (the current
tool-calling API). Extracting to one tested implementation removes the inconsistency.

### Tier 3 — Environment and integration checks (scheduled, not per-commit)

7. **Dependency manifest + import smoke test.** Add a pinned `requirements.txt`
   (ideally per framework directory) capturing versions the notebooks are known to
   work against, and a test that imports every symbol the notebooks import. This is
   the check that catches 2.5 *before a reader does*. Run it weekly against pinned
   versions (must pass) and against latest (allowed to fail, but files an issue) —
   that unpinned-canary job is what gives advance warning of the next
   `ToolInvocation`-style removal.
8. **Constructor/wiring assertions.** Build each `Agent`/`Crew`/graph and assert its
   shape without invoking an LLM: `assert searching.tools == [yt_tool]` catches 2.4;
   `assert set(graph.get_graph().nodes) == {...}` catches an accidentally dropped node.
   Cheap, no network.
9. **End-to-end notebook execution**, via `nbmake` (`pytest --nbmake`) or `papermill`,
   with secrets from CI and `vcrpy`/cassettes to replay recorded HTTP where possible.
   Gate on a marker so it never blocks a docs-only commit. This is the only tier that
   spends money, and it should be nightly at most.
10. **Turn `LangSmith/성능_평가하기` into a real evaluation.** It already has a
    3-example dataset; add `evaluate()` with a correctness evaluator and a score
    threshold. That converts the repository's one existing "evaluation" notebook from
    a demo into an actual quality gate — and it is a better book chapter, since it
    shows readers the full evaluate-and-assert loop rather than dataset upload alone.

### Tier 0 — Prerequisite: CI

None of the above runs today because there is no CI. A single GitHub Actions workflow
on push/PR running Tiers 1–2 (seconds, no secrets, no cost) is the enabling step, plus
a `.pre-commit-config.yaml` with `nbstripout` so item 6 is enforced at commit time.

---

## 4. Suggested sequencing

| Step | Work | Effort | Catches |
| --- | --- | --- | --- |
| 1 | CI workflow + `nbstripout` pre-commit | ~1 h | makes diffs reviewable |
| 2 | Tier 1 static checks | ~2–3 h | 2.1, 2.2, 2.3, 2.6, 2.7 |
| 3 | Fix the 3 currently-red checks (delete the 80 markdown-in-code cells, remove the dead `_rebuild` cells, restore or delete `testCRM20251111.ipynb`) | ~2 h | the actual erratum |
| 4 | Pinned `requirements.txt` + import smoke test | ~2 h | 2.5 |
| 5 | Extract graph logic to `src/`, unit-test with fake LLMs | ~1–2 d | routing bugs, 2.4 |
| 6 | Nightly `nbmake` run + LangSmith eval threshold | ~1 d | live-API drift |

Steps 1–3 are where nearly all the value is, and they cost under a day. Step 3 is the
one that fixes what readers are hitting right now.

## 5. What is out of scope for testing

`AutoGPT/AutoGPT.ipynb` is five shell commands that clone an archived upstream repo
at tag `v0.4.7` and run `./run.sh`. There is no logic to unit-test. The only useful
check is that the upstream tag still resolves — and given the project is archived, a
note in the notebook stating the pinned tag and its archived status would serve
readers better than a test.
