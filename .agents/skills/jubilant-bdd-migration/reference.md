# Reference

Deep rules and configuration details for the `jubilant-bdd-migration`
skill. See `SKILL.md` for the compact workflow and `examples.md` for
concrete before/after code.

## Gherkinator YAML schema (source of truth)

`test-plan.yaml` is a multi-document YAML file. Each document is one
`TestPlan`. The schema is enforced by `gherkinator validate`.

```yaml
# ── Required fields ────────────────────────────────────────────────────────

feature: "Slurmctld deployment"   # Human-readable feature name.
                                  # Becomes: Feature: Slurmctld deployment.

type: functional                  # One of: functional | solution | performance
                                  #         reliability | security.
                                  # Becomes tag: @functional.

status: planned                   # One of: planned | implemented | deprecated.
                                  # Becomes tag: @planned.
                                  # Use `planned` for newly migrated tests;
                                  # flip to `implemented` once the BDD suite
                                  # passes for this plan.

risk: stable                      # One of: edge | beta | candidate | stable.
                                  # Becomes tag: @stable.
                                  # Use `stable` for core charm features,
                                  # `candidate` for secondary features,
                                  # `beta`/`edge` for less critical paths.

scenarios:                        # List of multi-line strings. Each entry:
  - |-                            #   - First line = scenario title
    Deploy slurmctld              #   - Subsequent lines = Gherkin steps
    Given I add model 'test'
    Given I deploy 'slurmctld' from channel 'latest/edge'
    Then the workload status for app 'slurmctld' is 'active'

# ── Optional fields ────────────────────────────────────────────────────────

description: "End-to-end deployment check for slurmctld."

issues: "https://github.com/canonical/slurm-charms/issues/123"
docs: "https://docs.canonical.com/charmed-hpc"

background: |-                    # Steps that run before every scenario.
  Given I add model 'test'        # Common setup shared by all scenarios.

examples:                         # Parametrised data rows. Use <param> tokens
  - - alice                       # in scenario steps; the first row supplies
    - admin                       # column headers derived from those tokens.
  - - bob                         # Triggers Scenario Outline mode.
    - viewer
```

### Output filename

The `feature` field is lowercased and underscored to derive the output
filename:

- `feature: "GPU job submission"` → `gpu_job_submission.feature` / `.md`
- `feature: "Slurmctld deployment"` → `slurmctld_deployment.feature` / `.md`

### Generated Gherkin shape

For each plan document, `gherkinator generate --format gh` produces:

```gherkin
@functional @planned @stable
Feature: Slurmctld deployment

  End-to-end deployment check for slurmctld.

  Background:
    Given I add model 'test'

  Scenario: Deploy slurmctld
    Given I deploy 'slurmctld' from channel 'latest/edge'
    Then the workload status for app 'slurmctld' is 'active'
```

When `examples` is present, the corresponding scenario becomes:

```gherkin
  Scenario Outline: <title>
    Given ... with username '<username>' and role '<role>'

    Examples:
      | username | role |
      | alice | admin |
      | bob | viewer |
```

The generated Gherkin is validated against the official Cucumber Gherkin
parser (`github.com/cucumber/gherkin/go/v28`); malformed output is
rejected at generation time.

### Risk/status filtering for incremental migration

| Filter | Flag | Behaviour |
|---|---|---|
| Risk (cumulative) | `--risk edge\|beta\|candidate\|stable` | `edge` → only `edge`; `beta` → `edge` + `beta`; `candidate` → `edge` + `beta` + `candidate`; `stable` → all. |
| Status (exact) | `--status planned\|implemented\|deprecated` | Only plans matching the exact value. |

Filters intersect. To migrate only the highest-criticality work first:

```bash
# Generate .feature files for newly migrated (status=planned) edge-risk
# plans only.
gherkinator generate --format gh tests/integration/features/test-plan.yaml \
  --output-dir tests/integration/features \
  --status planned --risk edge
```

## Gherkinator command reference (migration-relevant subset)

| Command | Purpose | Key flags |
|---|---|---|
| `gherkinator init <dir> --name <file>` | Create a YAML test plan directory with a skeleton file. | `--name` / `-n` (default `test-plan.yaml`; `.yaml` appended if missing). |
| `gherkinator edit <file>` | Open the YAML in `$VISUAL`/`$EDITOR` (falls back to `editor`, `vi`, `emacs`, `nano`) with a leading comment that documents the schema. Validates on save. | — |
| `gherkinator validate <file>` | Check the YAML against the schema (per document). Reports each invalid document with file path, document index, and reason. Exits non-zero on failure. | — |
| `gherkinator generate <file>` | Transpile YAML to `.feature` files. | `--format gh\|md` (default `gh`); `--output-dir` / `-o` (default `.`); `--risk` (cumulative); `--status` (exact). |
| `gherkinator delete -i <file> "<feature>"` | Remove a plan by feature name (case-insensitive). | `--input` / `-i` (required); `--yes` / `-y` (skip confirmation). |
| `gherkinator clean -d <dir>` | Remove generated `.feature`, `.md`, and `.gherkindocs/`. | `--dir` / `-d` (default `.`). |

Positional arguments to `init`, `validate`, and `generate` accept any mix
of YAML files (`.yaml`/`.yml`) and directories; directories are scanned
non-recursively. With no arguments, the current working directory is
scanned.

## Target directory layout

After migration, the charm repo's test tree looks like:

```
tests/integration/
├── features/                              # gherkinator-controlled
│   ├── test-plan.yaml                     # SOURCE OF TRUTH (YAML TestPlan docs)
│   ├── slurmctld_deployment.feature       # GENERATED: gherkinator generate
│   ├── slurmctld_integration.feature      # GENERATED
│   └── slurmctld_operations.feature       # GENERATED
├── conftest.py                            # UPDATED: charm-specific fixtures only
├── test_deployment.py                     # NEW: scenarios() + custom steps
├── test_integration.py
├── test_operations.py
└── constants.py                           # PRESERVED: charm test constants
```

- **`test-plan.yaml`** is the only file the agent/author hand-edits. It
  is the single source of truth for every migration scenario in this repo.
- **`.feature` files are generated**. Do not hand-edit them; they will be
  overwritten on the next `gherkinator generate`. To change a scenario,
  edit the YAML and re-run `gherkinator generate`.
- Each `test_*.py` module loads its scenarios with
  `scenarios("features/<name>.feature")` and defines any custom steps the
  framework doesn't already provide.

## Pattern mapping: existing test → gherkinator YAML → BDD

| Existing jubilant pattern | YAML TestPlan field | BDD equivalent |
|---|---|---|
| `juju.add_model(...)` / `jubilant.temp_model()` | `background` (or inline in `scenarios`) | `Given I add model 'name'` (framework suffixes the real model name) |
| `juju.deploy("charm", "app")` (Charmhub) | `scenarios[i]` (steps) | `Given I deploy 'app' [from channel '...'] [on base '...'] [with 'N' units]` |
| `juju.deploy(local_charm, "app")` | `scenarios[i]` (steps) | `Given I deploy 'app' from a local charm [located at '/path/to.charm']` (or set `<APP>_CHARM_PATH`) |
| `juju.integrate("a", "b")` | `scenarios[i]` (steps) | `Given I integrate 'a' with 'b'` |
| `juju.config("app", values={...})` | `scenarios[i]` (steps) | `Given I set 'key' for app 'app' to 'value'` |
| `juju.config("app", reset=["key"])` | `scenarios[i]` (steps) | `Given I reset 'key' for app 'app'` |
| `juju.add_unit("app", num_units=N)` | `scenarios[i]` (steps) | `Given I add 'N' units to app 'app'` |
| `juju.run("app/0", "action", params=...)` | `scenarios[i]` (steps) | `When I run action 'action' on unit 'app/0' [with parameters 'k=v']` |
| `juju.exec("cmd", unit="app/0")` | `scenarios[i]` (steps) | `When I execute 'cmd' on unit 'app/0'` |
| `juju.wait(lambda s: s.apps[...].status == "active")` | `scenarios[i]` (steps) | `Then the workload status for app 'app' is 'active'` (framework polls via `context.wait()`). Do NOT put `juju.wait()` inside a custom deploy step if integrations happen in later steps — charms can't reach active without integrations. |
| Custom status checker `def _ready(s): ...` passed to `juju.wait` | custom step | Keep as a helper, or wrap in a custom Then step that calls `context.wait(ready=...)` |
| `tenacity.Retrying(...)` around an assertion | `scenarios[i]` (steps) | Drop tenacity; use `context.wait()` (or a built-in Then step, which already polls). The `ready` function must catch `Exception` (not just `AssertionError`) because `juju.exec()` raises `jubilant.TaskError` on non-zero exit codes. |
| `sleep(N)` after `juju.wait()` | custom step with `context.wait()` | Charm `active` status doesn't guarantee installed binaries are on `PATH` yet. Poll for the binary (e.g., `juju.exec("apptainer --version")`) instead of a fixed sleep. |
| `@pytest.mark.order(N)` | (not migrated to YAML) | Preserve with the marker on the scenario-loading module; ordering is a charm-repo concern, not framework-level |
| N/A | `feature` | `Feature: <feature>` line in generated `.feature`. |
| N/A | `type` | First tag (`@functional`, `@reliability`, ...). |
| N/A | `status` | Second tag (`@planned`, `@implemented`, `@deprecated`). |
| N/A | `risk` | Third tag (`@edge`, `@beta`, `@candidate`, `@stable`). |
| N/A | `description` | Free-form text rendered under `Feature:`. |
| N/A | `background` | Shared `Background:` block. |
| N/A | `examples` | `Scenario Outline:` + `Examples:` table. |

## Configuration snippets

### `pyproject.toml`

Add `pytest-jubilant-bdd` to the integration test extras. It pulls in
`jubilant` and `pytest-bdd`.

```toml
[project.optional-dependencies]
integration = [
    "pytest-jubilant-bdd ~= 0.2",
    # jubilant and pytest-bdd are pulled in transitively.
    # pytest-order is optional — add it only if you rely on cross-file ordering.
    # "pytest-order ~= 1.0",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
markers = [
    "integration: Integration tests requiring a Juju controller",
]
```

### `conftest.py`

Do **not** define a `juju` fixture. The plugin provides `context`. Keep
only charm-specific fixtures (charm path, base, env vars), shared custom
steps used by multiple feature files, and pytest hooks
(`pytest_addoption`, `pytest_configure`, etc.).

```python
"""Integration test fixtures and shared step definitions."""
import os
from pathlib import Path

import pytest
from pytest_bdd import parsers, when

from pytest_jubilant_bdd import Context


@pytest.fixture
def scenario_state() -> dict:
    """Per-scenario mutable state shared between Given/When/Then steps.

    Must live in conftest.py — pytest does not discover fixtures from
    helper modules like bdd_utils.py.
    """
    return {}


@pytest.fixture(scope="session")
def charm_base(request: pytest.FixtureRequest) -> str:
    """Charm deployment base."""
    return request.config.getoption("--charm-base", default="ubuntu@24.04")


# Custom step shared across multiple .feature files — must be in
# conftest.py because pytest-bdd resolves steps per-module.
@when(parsers.parse("I reset the node configuration on unit '{unit}'"))
def reset_node_config(context: Context, unit: str) -> None:
    """Custom step available to ALL test_*_bdd.py modules."""
    juju = context.get_juju()
    juju.run(unit, "set-node-config", params={"reset": True})


def pytest_addoption(parser: pytest.PytestParser) -> None:
    """Add charm-specific CLI options."""
    parser.addoption("--charm-base", default="ubuntu@24.04")
    parser.addoption("--keep-models", action="store_true")
```

Fixtures that need lifecycle management (e.g., starting/stopping an SMTP
server) also belong here — plain step functions have no teardown hook.

## CLI options provided by the plugin

- `--juju-bdd-no-teardown` — skip destroying models at the end of the
  session.
- `--juju-bdd-wait-timeout <sec>` — default wait timeout (default 180s)
  used by `context.wait()` and the built-in Then steps.

## Preserving test ordering

The framework does not prescribe ordering. If the original tests used
`@pytest.mark.order(N)`, apply the marker to the scenario-loading module:

```python
import pytest
from pytest_bdd import scenarios

@pytest.mark.order(2)
scenarios("features/integration.feature")
```

Only do this if cross-file ordering genuinely matters; intra-feature
scenario order is already top-to-bottom in the `.feature` file.

## Verification checklist

After migration, confirm:

- [ ] `gherkinator validate tests/integration/features/test-plan.yaml`
      passes.
- [ ] `gherkinator generate --format gh
      tests/integration/features/test-plan.yaml --output-dir
      tests/integration/features` produces one `.feature` file per
      `TestPlan` document.
- [ ] Generated `.feature` files match the YAML source (no hand-edits).
- [ ] Every original test scenario has a YAML `TestPlan` equivalent in
      `test-plan.yaml`.
- [ ] Each YAML plan uses `status: planned` initially; flip to
      `implemented` only after the BDD suite is green for that plan.
- [ ] Gherkin steps use the exact phrasing from the step reference in
      `SKILL.md`.
- [ ] No step handlers are imported from `pytest_jubilant_bdd._main`.
- [ ] No custom `juju` fixture is defined; the plugin's `context` fixture
      is used.
- [ ] Custom steps use `context.wait()` for polling, not `tenacity`.
- [ ] All `@pytest.fixture` definitions are in `conftest.py`, not in
      helper modules like `bdd_utils.py`.
- [ ] All `context.wait(ready=fn)` functions catch `Exception`, not just
      `AssertionError`.
- [ ] Custom steps shared across multiple `.feature` files are defined
      in `conftest.py`.
- [ ] `pytest-jubilant-bdd` is in the integration extras.
- [ ] `pytest tests/integration/ -v` passes.
- [ ] Original non-BDD tests are retained until the BDD suite is green.
- [ ] `LOCAL_*` / charm-path environment variables still work.

## Troubleshooting

**`gherkinator validate` reports an invalid field**
- Check the message for the field name and the allowed enum values. Fix
  the YAML; do not hand-edit the generated `.feature` file.

**"Step not found" / "Ambiguous step"**
- Re-check the exact phrasing against the reference table in `SKILL.md`
  (single quotes, `I add model`, `I deploy ... from a local charm`, etc.).
- Ensure a custom step's `parsers.parse` pattern doesn't collide with a
  built-in step. Prefer the built-in.

**Eventually-consistent assertion fails intermittently**
- Use a built-in Then step (it already polls via `context.wait()`).
- For custom Then steps, call `context.wait(ready=...)` instead of
  `@retry`.

**`deploy_local` fails with `CharmNotFoundError`**
- Provide `located at '/path/to.charm'` in the step, or set the
  `<APP>_CHARM_PATH` environment variable (e.g. `SLURMCTLD_CHARM_PATH`).

**`TooManyDeployedAppsError`**
- More than one model has an app with the same name. Add `in model
  'name'` to the relevant step.

**`fixture 'X' not found`**
- The fixture is defined in a helper module (e.g., `bdd_utils.py`)
  instead of `conftest.py`. Move `@pytest.fixture` definitions to
  `conftest.py` — pytest does not discover fixtures from non-conftest,
  non-`test_*` modules.

**`StepDefinitionNotFoundError` for a custom step used in multiple
features**
- The step is defined in one `test_*_bdd.py` module but used by a
  `.feature` file loaded by a different module. pytest-bdd resolves
  steps per-module — move the step to `conftest.py`.

**`ValueError: option names {'--X'} already added`**
- pytest found two `conftest.py` files defining the same
  `pytest_addoption`. Check for accidentally duplicated directories
  (e.g., `tests/integration/integration/conftest.py`). Run
  `find . -name conftest.py -not -path "*/.venv/*"` to locate
  duplicates.

**`jubilant.TaskError` propagating from `context.wait()`**
- The `ready` function only catches `AssertionError`, but
  `juju.exec()` raises `TaskError` on non-zero exit codes. Change
  `except AssertionError:` to `except Exception:` in the `ready`
  function.

**`command not found` after charm reaches `active` status**
- Charm `active` status means reconciliation completed, but installed
  binaries may not be on `PATH` yet. Add a `context.wait()` polling
  loop that checks for the binary (e.g.,
  `juju.exec("apptainer --version")`) before using it. This replaces
  any `sleep(N)` from the original tests.
