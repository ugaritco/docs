# Features

- [Introduction](#introduction)
    - [When to Use a Feature](#when-to-use-a-feature)
- [Writing a Feature](#writing-a-feature)
- [Flows](#flows)
    - [Defining Steps](#defining-steps)
    - [Repeating a Step: `each()`](#repeating-a-step)
    - [Required & Optional Steps](#required-and-optional-steps)
    - [Dependencies & Conditions](#dependencies-and-conditions)
    - [The Failure Outcome](#flow-failures)
    - [The Flow State](#flow-state)
    - [Transactions & Side Effects](#flow-transactions)
    - [Execution Contract](#execution-contract)
    - [Testing a Flow](#testing-flows)
- [Generating Features](#generating-features)

<a name="introduction"></a>
## Introduction

A **Feature** is a workflow: it runs several [Capabilities](/docs/{{version}}/capabilities) (or other Features) in order, inside one transaction, and returns one **Outcome**. Like a Capability it receives a typed Parcel, and Controllers run it the same way (see [Running Actions From Controllers](/docs/{{version}}/capabilities#running-actions)).

```text
Controller ─► Feature ─► Capability ─► Repository ─► Model
                  └──────► Capability ─┘
```

<a name="when-to-use-a-feature"></a>
### When to Use a Feature

Use a Feature when one request needs **several** Capabilities, for example saving a branch, then each of its translations, then loading the result. A single operation calls its Capability directly, with no Feature in between. A Feature must not reach into repositories; that is the Capabilities' job.

<a name="writing-a-feature"></a>
## Writing a Feature

A Feature extends `Heritage\Features\Feature` and implements `perform()`. As in a Capability, the type of `perform()`'s parameter declares the accepted Parcel and PHP enforces it: a wrong Parcel throws PHP's `TypeError` as it is (see [The Accepted Parcel](/docs/{{version}}/capabilities#the-accepted-parcel)). The final `handle()`, called by Controllers and flows, only runs `perform()`:

```php
class SaveBranchFeature extends Feature
{
    protected function perform(SaveBranchFeatureParcel $data): Outcome
    {
        return $this->flow(BranchOutcome::class, SaveBranchResult::FAILED)
            // ->step(...)
            ->finish(fn (FlowState $state) => BranchOutcome::fromResult(SaveBranchResult::SUCCESS, $state->branch));
    }
}
```

Unlike a Capability, a Feature has no `failed()`: a failure is handled by its [flow](#flow-failures). The Outcome it returns follows the same [Results & Outcomes](/docs/{{version}}/capabilities#results-and-outcomes) conventions as a Capability's.

<a name="flows"></a>
## Flows

A Feature describes its workflow with `$this->flow()`: named **steps**, each running a Capability (or another Feature), in order, inside one transaction. The flow checks every Outcome, rolls back on failure, records what happened, and returns the final Outcome:

```php
use Heritage\Features\Flow\Call;
use Heritage\Features\Flow\FlowState;

protected function perform(SaveBranchFeatureParcel $data): Outcome
{
    return $this->flow(BranchOutcome::class, SaveBranchResult::FAILED)
        ->step('branch', SaveBranchCapability::class, SaveBranchCapabilityParcel::fromParcel($data),
            failWith: SaveBranchResult::FAILED_MODEL_SAVE)

        ->each('translations', $data->translations ?? [],
            fn (?array $attributes, string $locale, FlowState $state) => $attributes === null
                ? Call::to(DestroyBranchTranslationCapability::class, new DestroyBranchTranslationParcel(branch_id: $state->branch->id, locale: $locale))
                    ->failWith(SaveBranchResult::FAILED_TRANSLATION_DESTROY)
                : Call::to(SaveBranchTranslationCapability::class, SaveBranchTranslationParcel::fromArray([...$attributes, 'branch_id' => $state->branch->id, 'locale' => $locale]))
                    ->failWith(SaveBranchResult::FAILED_TRANSLATION_SAVE),
            errorKey: 'translations.{key}',
            context: fn (FlowState $state, string $locale) => ['entity' => $state->branch->slug, 'locale' => Locales::resolver()->label($locale)])

        ->step('branchWithTranslations', ShowBranchCapability::class,
            fn (FlowState $state) => new ShowBranchParcel(id: $state->branch->id))

        ->finish(fn (FlowState $state) => BranchOutcome::fromResult(SaveBranchResult::SUCCESS, $state->branchWithTranslations));
}
```

Steps are defined first; `finish()` runs them and returns the Outcome. Capabilities given by class name are resolved from the container, so the Feature needs no constructor. `$state->branch` reads the result of the `branch` step.

`flow()` takes two optional arguments:

| Argument | Meaning |
| :--- | :--- |
| `$outcome` | The Feature's Outcome class: a halted flow returns it, built from the failure's Result with its errors ([The Failure Outcome](#flow-failures)) |
| `$failWith` | The failure Result of every step without its own `failWith` |

`SaveBranchCapabilityParcel::fromParcel($data)` copies the provided keys the Capability's Parcel declares (here, not `translations`) and the attached model, so updating a branch stays an update.

<a name="defining-steps"></a>
### Defining Steps

```php
->step(string $name, $capability, $parcel = null,
    optional: false, needs: [], when: null, failWith: null, errorKey: null, context: [])
```

| Argument | Meaning |
| :--- | :--- |
| `$name` | Unique name. The step's data is read with `$state->result($name)` |
| `$capability` | A Capability or Feature class name (resolved from the container), or an instance |
| `$parcel` | The input: a Parcel, `null`, or a closure `fn (FlowState $state) => Parcel` called when the step runs, so it can use earlier results |
| `optional` | `true`: the step's failure does not halt the flow ([Required & Optional Steps](#required-and-optional-steps)) |
| `needs` | A step name or a list of names that must have **succeeded** ([Dependencies & Conditions](#dependencies-and-conditions)) |
| `when` | `fn (FlowState $state) => bool`: the step runs only when it returns `true` |
| `failWith` | The Result of the failure; defaults to the flow's `failWith`, then to the failed Outcome's own Result |
| `errorKey` | Adds the failed Outcome's message to the errors under this key (`'{key}'` is the item key in `each()`) |
| `context` | The context of that message: an array, or `fn (FlowState $state, $key) => array` |

<a name="repeating-a-step"></a>
### Repeating a Step: `each()`

`each()` runs one call per item. The closure receives the item, its key and the state, and returns a `Call`: `Call::to($capability, $parcel)`, optionally `->failWith($result)` (overriding the step's `failWith`). The items may also be a closure `fn (FlowState $state) => iterable`, evaluated when the step runs.

- Every item has its own record (`$state->record('translations')->items()`), keyed by item key.
- A **required** `each()` halts at the first failed item; later items stay `PENDING`.
- An **optional** `each()` runs every item and is `FAILED` when any item failed.
- No items: the step `SUCCEEDED` with an empty result.
- Its result is the succeeded items' data, keyed by item key.

<a name="required-and-optional-steps"></a>
### Required & Optional Steps

Steps are **required** by default. When one fails, the flow **halts**: the transaction is rolled back, later steps stay `PENDING`, `finish()` is not called, and `onFailure()` builds the Outcome.

An **optional** step (`optional: true`) runs inside a **savepoint**: when it fails, only its own changes are rolled back, the failure is recorded, and the flow continues. A failed optional step is never treated as a success: it has no result, and steps that need it are skipped.

```php
->step('notify', NotifyManagersCapability::class, fn (FlowState $s) => ..., optional: true)
```

<a name="dependencies-and-conditions"></a>
### Dependencies & Conditions

For each step, the flow checks, in this order:

1. **`needs`**: every step listed must have `SUCCEEDED`. Otherwise the step is `SKIPPED` with `DEPENDENCY_FAILED` (a need failed) or `DEPENDENCY_SKIPPED` (a need was skipped), and `$record->cause()` names that step.
2. **`when`**: if it returns `false`, the step is `SKIPPED` with `CONDITION_NOT_MET`.
3. The step runs.

Needs are checked first so a condition can safely read a needed result:

```php
->step('notify', NotifyManagersCapability::class, fn (FlowState $s) => ..., optional: true)
->step('audit', AuditNotificationCapability::class, fn (FlowState $s) => ..., needs: 'notify')     // skipped if notify failed
->step('cache', WarmBranchCacheCapability::class, fn (FlowState $s) => ...,
    needs: 'branch', when: fn (FlowState $s) => $s->result('branch')->active)                       // skipped for an inactive branch
```

A skipped step is not a success: a step needing it is skipped too (`DEPENDENCY_SKIPPED`). `needs` must name **earlier** steps; a duplicated name or an unknown need throws a `LogicException` when the flow is defined.

<a name="flow-failures"></a>
### The Failure Outcome

A halted flow returns, in this order of precedence:

1. `onFailure()`'s Outcome, when it is set;
2. the flow's Outcome class (`$this->flow(BranchOutcome::class, ...)`), built from the failure's Result with its errors: `BranchOutcome::fromResult($state->failureResult())->withErrors($state->errors())`;
3. otherwise, the failed step's own Outcome, with `failWith`'s Result and the errors.

The failure's Result is the halting step's `failWith`, else the flow's `failWith`, else the failed Outcome's Result. Most Features need only the second form; use `onFailure(fn (FlowState $state) => Outcome)` to build a different Outcome, with:

- `$state->failureResult()`: the failure's Result;
- `$state->errors()`: its Outcome's errors, plus its message under `errorKey`;
- `$state->failure()`: its record (for an `each()` step, the failed item's record, with `->key`).

**Exceptions:**

- A Capability's unexpected exception is already reported and turned into a failed Outcome by the Capability (see [Handling Failures](/docs/{{version}}/capabilities#handling-failures)): it is a normal step failure.
- An exception raised by the flow's **own callbacks** (a parcel closure, `when`, `context`, `each()` items or call, `finish()`) is a bug: it halts the flow, even in an optional step, is reported, and returns the failure Outcome. Its Result is the step's or the flow's `failWith`; when there is neither `onFailure()` nor a flow Outcome with such a Result, the exception is rethrown after the rollback.
- HTTP, authorization, validation and not-found exceptions are rolled back and **rethrown**, without `onFailure()`, for the framework to render; so is the `TypeError` of a Parcel a step's `perform()` does not accept.

<a name="flow-state"></a>
### The Flow State

`FlowState` keeps the **results** of succeeded steps apart from the **records** of every step, and the flow's own **status**:

| Method | Returns |
| :--- | :--- |
| `result($name)` | The step's data. Throws a `LogicException` if the step did not succeed |
| `resultOr($name, $default)` | The step's data, or the default |
| `outcome($name)` | The Outcome the step returned, successful or not |
| `record($name)` / `records()` | A `StepRecord`: `status()`, `reason()`, `cause()`, `result()`, `errors()`, `exception()`, `items()`, `key` |
| `hasSucceeded()` / `isFailed()` / `isSkipped()` / `isPending()` | The step's status |
| `status()` / `completed()` / `halted()` | The flow's status: `PENDING`, `RUNNING`, `COMPLETED` or `HALTED` |
| `failure()` / `failureResult()` / `errors()` / `exception()` | What halted the flow |
| `toArray()` | The trace of every step (statuses, reasons, errors), without domain data |

Step statuses (`StepStatus`): `PENDING` (not evaluated), `SUCCEEDED`, `FAILED`, `SKIPPED`. A flow may be `COMPLETED` while an optional step `FAILED`. `$state->branch` is a shorthand for `$state->result('branch')`.

The Outcome returned by a flow carries its state: `$outcome->flow()`. It is never part of `toArray()` or the JSON response.

<a name="flow-transactions"></a>
### Transactions & Side Effects

- The flow opens a transaction before the first step and commits after `finish()` has built the Outcome. If `finish()` fails, the flow is rolled back.
- `->connection('tenant')` runs it on another connection; `->withoutTransaction()` runs without a transaction or savepoints, so a halted flow keeps the changes of the steps that ran.
- A rollback only undoes **database** changes. Mail, notifications and calls to other services cannot be undone: dispatch them **after the commit**, as queued jobs or events with `afterCommit`, rather than from a step.

<a name="execution-contract"></a>
### Execution Contract

| Situation | Behavior |
| :--- | :--- |
| Order | `needs`, then `when`, then the step runs |
| A required step fails | Flow halted, transaction rolled back, failure Outcome (`onFailure()`, else the flow's Outcome class); later steps `PENDING` |
| Failure Result | The step's `failWith`, else the flow's `failWith`, else the failed Outcome's Result |
| An optional step fails | Its savepoint rolled back, failure recorded, the flow continues |
| A need failed / was skipped | `SKIPPED`: `DEPENDENCY_FAILED` / `DEPENDENCY_SKIPPED` |
| `when` returns `false` | `SKIPPED`: `CONDITION_NOT_MET` |
| A flow callback throws | Halted, reported, failure Outcome (rethrown when none can be built), even in an optional step |
| HTTP / authorization / validation / not found, a rejected Parcel (`TypeError`) | Rolled back and rethrown |
| `each()` | A record per item; required: halts at the first failed item; empty: `SUCCEEDED` |
| `#[Authorize]` on a step's Capability | Not checked: the Feature's own ability, checked by the controller, covers its steps ([Authorization](/docs/{{version}}/capabilities#authorization)) |
| `result()` of a step that did not succeed | `LogicException` |
| Duplicated name, unknown or later need | `LogicException` when defined |
| `finish()` | Builds the Outcome before the commit; its failure rolls back |
| External side effects | Not undone; dispatch after commit |

<a name="testing-flows"></a>
### Testing a Flow

The state is available from the Feature's Outcome, with assertions:

```php
$outcome = app(SaveBranchFeature::class)->handle($parcel);

$outcome->flow()
    ->assertHalted('translations')
    ->assertSucceeded('branch')
    ->assertFailed('translations')
    ->assertPending('branchWithTranslations');

$outcome->flow()->assertSkipped('audit', SkipReason::DEPENDENCY_FAILED);
$this->assertSame('en', $outcome->flow()->failure()->key);
```

Available: `assertSucceeded()`, `assertFailed()`, `assertSkipped($name, ?SkipReason $reason)`, `assertPending()`, `assertCompleted()`, `assertHalted(?string $by)`.

<a name="generating-features"></a>
## Generating Features

`make:feature` generates a Feature whose `perform()` accepts the given Parcel and runs a flow: `$this->flow(BranchOutcome::class, SaveBranchResult::FAILED)` and a `finish()` building the `SUCCESS` Outcome:

```bash
php scribe make:feature Branch/SaveBranch --artifact=company --parcel=Branch/Features/SaveBranchParcel --result=Branch/Features/SaveBranchResult --outcome=Branch/BranchOutcome --authorize=save --model=Branch
```

`--artifact`, the companion class paths and `--outcome` follow the same rules as the other generators; `--authorize` with `--model` adds the [`#[Authorize]`](/docs/{{version}}/capabilities#authorization) attribute. See [CLI Scaffolding](/docs/{{version}}/capabilities#cli-scaffolding).
