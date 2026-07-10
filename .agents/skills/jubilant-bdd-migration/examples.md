# Examples

Concrete before/after migrations and custom-step templates for the
`jubilant-bdd-migration` skill. See `SKILL.md` for the step reference and
`reference.md` for configuration, schema, and rules.

All examples follow the same pipeline:

1. Author YAML test plans in `tests/integration/features/test-plan.yaml`.
2. Run `gherkinator validate` to catch schema errors.
3. Run `gherkinator generate --format gh` to produce `.feature` files.
4. Write a `test_*.py` module that loads the generated scenarios.
5. Run `pytest tests/integration/ -v`.

## Full before/after pipeline

### Before (traditional pytest + jubilant)

```python
# tests/integration/test_charm.py
import pytest
import jubilant
import tenacity


@pytest.mark.order(3)
def test_deploy_and_check(juju):
    juju.deploy("slurmctld", "slurmctld", channel="latest/edge")
    juju.wait(lambda s: s.apps["slurmctld"].status == "active")

    result = juju.run("slurmctld/0", "get-password")

    for attempt in tenacity.Retrying(stop=tenacity.stop_after_attempt(5)):
        with attempt:
            assert result.return_code == 0
            assert result.status == "completed"
```

### After (gherkinator-controlled BDD)

**Step 1 — initialize the YAML test plan directory:**

```bash
gherkinator init tests/integration/features --name test-plan
```

This creates `tests/integration/features/test-plan.yaml`.

**Step 2 — author the YAML test plan:**

```bash
gherkinator edit tests/integration/features/test-plan.yaml
```

In the editor, write the plan (final YAML below):

```yaml
feature: "Slurmctld deployment"
type: functional
status: planned
risk: stable
description: "End-to-end deployment check for slurmctld."
background: |-
  Given I add model 'test'
scenarios:
  - |-
    Deploy slurmctld and run get-password
    Given I deploy 'slurmctld' from channel 'latest/edge'
    When I run action 'get-password' on unit 'slurmctld/0'
    Then the workload status for app 'slurmctld' is 'active'
```

**Step 3 — validate the YAML schema:**

```bash
gherkinator validate tests/integration/features/test-plan.yaml
```

Output:

```
All 1 test plan(s) are valid.
```

**Step 4 — generate the `.feature` file:**

```bash
gherkinator generate --format gh tests/integration/features/test-plan.yaml \
  --output-dir tests/integration/features
```

Generated `tests/integration/features/slurmctld_deployment.feature`:

```gherkin
@functional @planned @stable
Feature: Slurmctld deployment

  End-to-end deployment check for slurmctld.

  Background:
    Given I add model 'test'

  Scenario: Deploy slurmctld and run get-password
    Given I deploy 'slurmctld' from channel 'latest/edge'
    When I run action 'get-password' on unit 'slurmctld/0'
    Then the workload status for app 'slurmctld' is 'active'
```

**Step 5 — load the scenarios:**

`tests/integration/test_deployment.py`:

```python
"""BDD step definitions for slurmctld deployment."""
from pytest_bdd import scenarios

scenarios("features/slurmctld_deployment.feature")

# No custom steps needed here — every step above is provided by
# pytest-jubilant-bdd. Action results are available on the `context`
# fixture (context.action_results) if a later scenario needs to inspect
# them.
```

**Step 6 — run the BDD suite:**

```bash
pytest tests/integration/ -v
```

**Step 7 — flip `status` to `implemented` once green:**

Edit the YAML, change `status: planned` to `status: implemented`, then
re-run `gherkinator generate` so the regenerated `.feature` carries the
`@implemented` tag.

## Local-charm deploy example

The YAML input:

```yaml
feature: "Local charm deployment"
type: functional
status: planned
risk: stable
description: "Deploy a locally built charm and configure it."
background: |-
  Given I add model 'test'
scenarios:
  - |-
    Deploy a locally built charm with config
    Given I deploy 'slurmctld' from a local charm located at '/tmp/slurmctld.charm' with '3' units
    Given I set 'debug' for app 'slurmctld' to 'true'
    Then the workload status for app 'slurmctld' is 'active'
```

Generated `.feature`:

```gherkin
@functional @planned @stable
Feature: Local charm deployment

  Deploy a locally built charm and configure it.

  Background:
    Given I add model 'test'

  Scenario: Deploy a locally built charm with config
    Given I deploy 'slurmctld' from a local charm located at '/tmp/slurmctld.charm' with '3' units
    Given I set 'debug' for app 'slurmctld' to 'true'
    Then the workload status for app 'slurmctld' is 'active'
```

Without `located at '...'`, the `deploy_local` step reads the
`SLURMCTLD_CHARM_PATH` environment variable (see `reference.md`).

## Multi-model integration example

The YAML input:

```yaml
feature: "Cross-model integration"
type: reliability
status: planned
risk: candidate
description: "Integrate slurmctld and slurmd across two models."
scenarios:
  - |-
    Integrate slurmctld and slurmd across models
    Given I add model 'controller'
    Given I add model 'compute'
    Given I deploy 'slurmctld' in model 'controller'
    Given I deploy 'slurmd' in model 'compute'
    Given I integrate 'slurmctld' with 'slurmd'
    Then all agents are 'idle' in models 'controller', 'compute'
```

Note this plan uses `type: reliability` (multi-unit / multi-model
integration test) instead of `functional`. Pick the type that matches the
nature of the test.

## Parametrised scenario (Scenario Outline) example

The YAML input — when `examples` is present and the scenario steps
reference `<param>` tokens, gherkinator emits a `Scenario Outline:`:

```yaml
feature: "Multi-user access"
type: functional
status: planned
risk: beta
description: "Same access flow exercised for several users."
background: |-
  Given I add model 'test'
scenarios:
  - |-
    Grant access by role
    Given I deploy 'access-manager' from channel 'latest/edge'
    When I run action 'grant' on unit 'access-manager/0' with parameters 'username=<username> role=<role>'
    Then the workload status for app 'access-manager' is 'active'
examples:
  - - alice
    - admin
  - - bob
    - viewer
```

Generated `.feature`:

```gherkin
@functional @planned @beta
Feature: Multi-user access

  Same access flow exercised for several users.

  Background:
    Given I add model 'test'

  Scenario Outline: Grant access by role
    Given I deploy 'access-manager' from channel 'latest/edge'
    When I run action 'grant' on unit 'access-manager/0' with parameters 'username=<username> role=<role>'
    Then the workload status for app 'access-manager' is 'active'

    Examples:
      | username | role |
      | alice | admin |
      | bob | viewer |
```

## Multi-document YAML (one file, many features)

A single `test-plan.yaml` can hold many features separated by `---`. Each
document becomes one `.feature` file on generation:

```yaml
feature: "Slurmctld deployment"
type: functional
status: planned
risk: stable
background: |-
  Given I add model 'test'
scenarios:
  - |-
    Deploy slurmctld
    Given I deploy 'slurmctld' from channel 'latest/edge'
    Then the workload status for app 'slurmctld' is 'active'
---
feature: "Slurmd deployment"
type: functional
status: planned
risk: stable
background: |-
  Given I add model 'test'
scenarios:
  - |-
    Deploy slurmd
    Given I deploy 'slurmd' from channel 'latest/edge'
    Then the workload status for app 'slurmd' is 'active'
---
feature: "Cross-model integration"
type: reliability
status: planned
risk: candidate
scenarios:
  - |-
    Integrate slurmctld and slurmd
    Given I add model 'controller'
    Given I add model 'compute'
    Given I deploy 'slurmctld' in model 'controller'
    Given I deploy 'slurmd' in model 'compute'
    Given I integrate 'slurmctld' with 'slurmd'
    Then all agents are 'idle' in models 'controller', 'compute'
```

Running:

```bash
gherkinator validate tests/integration/features/test-plan.yaml
gherkinator generate --format gh tests/integration/features/test-plan.yaml \
  --output-dir tests/integration/features
```

produces three `.feature` files in one shot:
`slurmctld_deployment.feature`, `slurmd_deployment.feature`,
`cross_model_integration.feature`. Each `test_*.py` module loads its own.

## Incremental migration with `--risk` / `--status`

During a multi-day migration, generate only the plans you intend to ship:

```bash
# Newly migrated (planned) and only the highest-criticality (edge) plans.
gherkinator generate --format gh tests/integration/features/test-plan.yaml \
  --output-dir tests/integration/features \
  --status planned --risk edge

# Bump to candidate once edge-risk plans are green; this adds beta + edge.
gherkinator generate --format gh tests/integration/features/test-plan.yaml \
  --output-dir tests/integration/features \
  --status planned --risk candidate
```

After all `status: planned` plans in a risk band are green, edit each
YAML document's `status: planned` → `status: implemented` and re-run
without filters to regenerate the full set.

## Removing a plan you no longer want

If a plan turns out to be the wrong granularity, drop it via
`gherkinator delete`:

```bash
# Interactive confirmation
gherkinator delete -i tests/integration/features/test-plan.yaml \
  "Slurmctld deployment"

# Non-interactive
gherkinator delete -y -i tests/integration/features/test-plan.yaml \
  "Slurmctld deployment"
```

Then regenerate to drop the matching `.feature` file:

```bash
gherkinator generate --format gh tests/integration/features/test-plan.yaml \
  --output-dir tests/integration/features
```

If the directory is cluttered with stale `.feature` files from old plans,
nuke and regenerate:

```bash
gherkinator clean -d tests/integration/features
gherkinator generate --format gh tests/integration/features/test-plan.yaml \
  --output-dir tests/integration/features
```

## Custom step templates

Write custom steps only for behavior the framework doesn't cover. Custom
steps take the `context` fixture and use `context.wait()` for any polling.
For the exact `Context` API (`get_juju`, `get_app`, `action_results` /
`exec_results` stacks, `wait()` signatures, `assertions` namespaces),
defer to the **`reusable-step-handler`** skill in the pytest-jubilant-bdd
repo (`.agents/skills/reusable-step-handler/SKILL.md`). The templates
below show the shape a custom step takes.

### Inspecting action / exec results

```python
"""Custom steps for action-result assertions."""
from pytest_bdd import then, parsers

from pytest_jubilant_bdd import Context


@then(parsers.parse("the action should succeed"))
def assert_action_succeeded(context: Context) -> None:
    """Assert the most recent action completed with return code 0."""
    task = context.action_results.peek()
    assert task.status == "completed"
    assert task.return_code == 0
```

### Custom status / command check with polling

Prefer `context.wait()` over `tenacity`. `context.wait()` polls `ready`
until it returns `True` three times in a row (configurable), then returns;
it raises `TimeoutError` on timeout.

**Important**: The `ready` function must catch `Exception` (not just
`AssertionError`). `juju.exec()` raises `jubilant.TaskError` on non-zero
exit codes — these are transient during node re-registration or service
restarts. `tenacity.Retrying` caught all exceptions by default;
`context.wait()` only retries when `ready` returns `False`, so any
unhandled exception aborts the wait.

```python
from pytest_bdd import then, parsers
from pytest_jubilant_bdd import Context


@then(parsers.parse("the unit '{unit}' workload message contains '{substring}'"))
def assert_message_contains(context: Context, unit: str, substring: str) -> None:
    """Wait until the unit's workload status message contains substring."""
    context.wait(
        ready=lambda ctx: substring in ctx.get_unit(unit).workload_status.message,
    )
```

For `juju.exec`-based polling (where the command may fail transiently),
wrap the call in a try/except:

```python
@then(parsers.parse("a slurm gpu job submitted from unit '{login_unit}' runs on unit '{compute_unit}'"))
def gpu_job_submission(context: Context, login_unit: str, compute_unit: str) -> None:
    """Submit a GPU-requesting job and verify it runs on the compute node."""
    juju = context.get_juju()
    slurmd_result = juju.exec("hostname -s", unit=compute_unit)

    def ready(_ctx: Context) -> bool:
        try:
            sackd_result = juju.exec(
                f"srun --partition compute --gres gpu:1 hostname -s",
                unit=login_unit,
            )
            assert sackd_result.stdout == slurmd_result.stdout
            return True
        except Exception:
            return False

    context.wait(ready=ready)
```

### Shared state within a scenario

Use a function-scoped fixture for state that must cross step boundaries
within a single scenario. **This fixture must live in `conftest.py`** —
pytest does not discover fixtures from helper modules like `bdd_utils.py`
or from `test_*_bdd.py` modules that aren't collected as test files.

```python
import pytest
from pytest_bdd import when, then, parsers
from pytest_jubilant_bdd import Context


@pytest.fixture
def scenario_state() -> dict:
    """State shared between steps in one scenario."""
    return {}


@when(parsers.parse("I capture the output of '{command}' on unit '{unit}'"))
def capture_output(
    context: Context, scenario_state: dict, command: str, unit: str
) -> None:
    result = context.get_juju().exec(command, unit=unit)
    scenario_state["stdout"] = result.stdout


@then(parsers.parse("the captured output should be '{expected}'"))
def assert_captured(scenario_state: dict, expected: str) -> None:
    assert scenario_state["stdout"] == expected
```

### Custom step shared across feature files

pytest-bdd resolves step definitions only from the module where
`scenarios()` is called, or from `conftest.py` files. A step defined in
`test_node_operations_bdd.py` is invisible to `test_job_submission_bdd.py`.
If a custom step is used by scenarios in multiple `.feature` files, define
it in `conftest.py`:

```python
# conftest.py
from pytest_bdd import parsers, when
from pytest_jubilant_bdd import Context


@when(parsers.parse("I reset the node configuration on unit '{unit}'"))
def reset_node_config(context: Context, unit: str) -> None:
    """Custom step available to ALL test_*_bdd.py modules."""
    juju = context.get_juju()
    juju.run(unit, "set-node-config", params={"reset": True})
```

### Binary availability after `active` status

A charm reaching `active` status means its reconciliation loop has
completed, but binaries it installs may not be immediately available on
the unit's `PATH`. When migrating tests that had `sleep(N)` after
`juju.wait()`, replace the fixed sleep with a `context.wait()` polling
loop that checks for the binary:

```python
@given("I deploy 'apptainer' from channel 'latest/edge'")
def deploy_apptainer(context: Context) -> None:
    juju = context.get_juju()
    juju.deploy("apptainer", channel="latest/edge")


@then(parsers.parse("an apptainer container job submitted from unit '{login_unit}' runs on unit '{compute_unit}'"))
def apptainer_oci_scheduling(context: Context, login_unit: str, compute_unit: str) -> None:
    """Pull an OCI image and run a Slurm job inside an Apptainer container."""
    juju = context.get_juju()

    def apptainer_ready(_ctx: Context) -> bool:
        try:
            juju.exec("apptainer --version", unit=compute_unit)
            return True
        except Exception:
            return False

    context.wait(ready=apptainer_ready)

    juju.exec(
        "apptainer pull /tmp/jammy.sif docker://ghcr.io/charmed-hpc/ubuntu-test:jammy",
        unit=compute_unit,
    )
    # ... rest of the test
```
