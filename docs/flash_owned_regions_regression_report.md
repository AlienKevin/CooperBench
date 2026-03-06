# Flash subset regression report: exact-owned-regions plus acknowledgement gate

## Experiment

This report compares two `CooperBench` `flash` runs using the same setup:

- agent: `openhands_sdk`
- model: `gemini/gemini-3-flash-preview`
- setting: `coop`
- backend: `modal`
- Modal profile: `marl`
- git collaboration: disabled

The only intended change was a prompt edit in `/Users/kevin/Dev/CooperBench/src/cooperbench/agents/openhands_agent_sdk/openhands-tools/openhands/tools/preset/default.py`:

- before: ask agents to message planned files and functions
- after: ask agents to message exact owned regions, and "Do not edit a shared file until your teammate explicitly acknowledges the split."

Run names:

- baseline: `coop-oh-gemini-3-flash-flash-baseline`
- updated: `coop-oh-gemini-3-flash-flash-owned-regions`

Source summaries:

- baseline: `/Users/kevin/Dev/CooperBench/logs/coop-oh-gemini-3-flash-flash-baseline/eval_summary.json`
- updated: `/Users/kevin/Dev/CooperBench/logs/coop-oh-gemini-3-flash-flash-owned-regions/eval_summary.json`

## Top-line result

### Pass rate

| run | completed | passed | failed | errors | evaluated pass rate | effective pass rate over 50 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| baseline | 49/50 | 15 | 34 | 0 | 30.61% | 30.0% |
| updated | 50/50 | 7 | 43 | 0 | 14.0% | 14.0% |

Change on fixed 50-task denominator:

- `30.0% -> 14.0%`
- absolute delta: `-16.0` points
- relative delta: `-53.3%`

Completion improved slightly:

- `49/50 -> 50/50`

But the solve rate dropped sharply.

## Outcome transitions

Across the union of evaluated flash pairs:

| baseline -> updated | count |
| --- | ---: |
| fail -> fail | 33 |
| fail -> pass | 1 |
| pass -> pass | 6 |
| pass -> fail | 9 |
| missing -> fail | 1 |

Changed runs:

- `dottxt_ai_outlines_task/1655/7,10`: pass -> fail
- `huggingface_datasets_task/6252/4,6`: pass -> fail
- `pallets_click_task/2800/1,7`: pass -> fail
- `pallets_click_task/2800/2,7`: pass -> fail
- `pallets_click_task/2800/6,7`: pass -> fail
- `pallets_jinja_task/1465/2,3`: pass -> fail
- `pallets_jinja_task/1465/2,6`: pass -> fail
- `pallets_jinja_task/1559/4,8`: pass -> fail
- `pallets_jinja_task/1559/6,9`: pass -> fail
- `typst_task/6554/4,9`: fail -> pass
- `typst_task/6554/5,9`: missing -> fail

The regressions are concentrated in tasks that require coordination around shared APIs or shared implementation files, especially:

- `pallets_click_task`: 3 pass -> fail flips
- `pallets_jinja_task`: 4 pass -> fail flips

## Main finding

The prompt change reduced overlap, but it reduced actual execution even more.

The dominant regression pattern is:

1. agents ask for acknowledgement
2. acknowledgement never arrives or arrives too late
3. one agent produces an empty patch or only a repro/test artifact
4. merge stays clean
5. the feature fails because the required integration edit never happened

This is not mainly a merge-conflict regression. It is a feature-completeness regression caused by a coordination gate that the current messaging system cannot reliably enforce.

## Quantitative evidence

### 1. Fewer conflicts, worse results

| metric | baseline | updated |
| --- | ---: | ---: |
| clean naive merges | 38 | 46 |
| clean naive pass rate | 34.2% | 13.0% |
| conflict union merges | 11 | 4 |
| conflict union pass rate | 18.2% | 25.0% |

Interpretation:

- the prompt did reduce overlaps enough to cut union merges from `11` to `4`
- but the extra clean merges were mostly not good merges
- the clean-merge pass rate collapsed from `34.2%` to `13.0%`

So the new prompt helped avoid collisions, but it also prevented necessary shared-file integration work.

### 2. Shared-file overlap dropped sharply

| metric | baseline | updated |
| --- | ---: | ---: |
| runs where both agents touched at least one same file | 34/49 | 20/50 |
| same-file rate | 69.4% | 40.0% |
| average shared files per run | 0.959 | 0.520 |
| median shared files per run | 1 | 0 |

Pass rates by whether both agents touched a common file:

| bucket | baseline | updated |
| --- | ---: | ---: |
| same-file runs | 38.2% | 30.0% |
| disjoint-file runs | 13.3% | 3.3% |

Interpretation:

- disjointness increased a lot
- but disjoint runs became almost entirely non-functional
- on this benchmark, forcing agents apart at the file/region level is often worse than letting them coordinate carefully inside the same file

### 3. Empty patches exploded

| metric | baseline | updated |
| --- | ---: | ---: |
| runs with at least one empty patch | 6/49 | 28/50 |
| pass rate when any patch is empty | 0.0% | 0.0% |
| share of failures involving an empty patch | 17.6% | 65.1% |

This is the strongest signal in the experiment.

The acknowledgement gate appears to have converted many coordination situations into "one agent waits, then never edits."

### 4. Communication volume fell

| metric | baseline | updated |
| --- | ---: | ---: |
| average messages sent | 9.94 | 8.50 |
| median messages sent | 9 | 8 |
| average total steps | 213.1 | 206.1 |
| average combined patch lines | 233.6 | 172.2 |

Conversation size buckets:

| messages in `conversation.json` | baseline pass rate | updated pass rate |
| --- | ---: | ---: |
| 0-3 | 0.0% on 1 run | 0.0% on 8 runs |
| 4-7 | 38.5% on 13 runs | 18.2% on 11 runs |
| 8-11 | 25.0% on 20 runs | 11.1% on 18 runs |
| 12+ | 33.3% on 15 runs | 23.1% on 13 runs |

The updated prompt did not cause more effective coordination. It caused shorter conversations, smaller patches, and many more runs where one side effectively stopped acting.

### 5. Acknowledgement language appears only in the updated condition and correlates with worse outcomes

Runs whose conversation included `acknowledg*`:

- baseline: `0/49`
- updated: `31/50`

Updated pass rate split:

- with acknowledgement language: `9.7%`
- without acknowledgement language: `21.1%`

This is not proof by itself, but it strongly supports the hypothesis that the explicit acknowledgement gate is the harmful part of the prompt change.

## Examples

## Example A: `pallets_jinja_task/1465/2,6`

Paths:

- baseline conversation: `/Users/kevin/Dev/CooperBench/logs/coop-oh-gemini-3-flash-flash-baseline/coop/pallets_jinja_task/1465/f2_f6/conversation.json`
- updated conversation: `/Users/kevin/Dev/CooperBench/logs/coop-oh-gemini-3-flash-flash-owned-regions/coop/pallets_jinja_task/1465/f2_f6/conversation.json`

Baseline:

- 12 messages
- agent coordination converged to a productive ownership transfer:
  - one agent implemented both `reverse` and `key_transform` in `src/jinja2/filters.py`
  - the other agent added tests in `tests/test_filters.py` and `tests/test_async_filters.py`
- result: pass

Updated:

- 3 messages, all from the same agent
- representative messages:
  - "I'm waiting for your acknowledgement on the `src/jinja2/filters.py` split."
  - "please let me know if I can start editing"
- patch outcome:
  - `agent2.patch`: empty
  - `agent6.patch`: touched only `reproduce_issue.py`
- eval outcome:
  - both features failed
  - sample failures:
    - `TypeError: do_groupby() got an unexpected keyword argument 'reverse'`
    - `TypeError: do_groupby() got an unexpected keyword argument 'key_transform'`

Interpretation:

- the exact issue is not a merge conflict
- neither feature landed in the real implementation file
- the acknowledgement gate converted a shared-file coordination problem into a no-op

## Example B: `huggingface_datasets_task/6252/4,6`

Paths:

- baseline conversation: `/Users/kevin/Dev/CooperBench/logs/coop-oh-gemini-3-flash-flash-baseline/coop/huggingface_datasets_task/6252/f4_f6/conversation.json`
- updated conversation: `/Users/kevin/Dev/CooperBench/logs/coop-oh-gemini-3-flash-flash-owned-regions/coop/huggingface_datasets_task/6252/f4_f6/conversation.json`

Baseline:

- 9 messages
- both agents explicitly coordinated edits inside `src/datasets/features/image.py`
- they negotiated exact ordering around `Image.decode_example`
- patch outcome:
  - one agent changed `src/datasets/features/image.py` and tests
  - the other agent also changed `src/datasets/features/image.py`
- result: pass

Updated:

- 1 message total
- patch outcome:
  - `agent4.patch`: empty
  - `agent6.patch`: only `test_cmyk_conversion.py`
- eval outcome:
  - both features failed

Interpretation:

- the prompt removed the overlap but also removed the implementation
- this is a clean example where "avoid shared edits at all cost" is worse than careful same-file coordination

## Example C: `pallets_click_task/2800/1,7`

Paths:

- baseline conversation: `/Users/kevin/Dev/CooperBench/logs/coop-oh-gemini-3-flash-flash-baseline/coop/pallets_click_task/2800/f1_f7/conversation.json`
- updated conversation: `/Users/kevin/Dev/CooperBench/logs/coop-oh-gemini-3-flash-flash-owned-regions/coop/pallets_click_task/2800/f1_f7/conversation.json`

Baseline:

- 7 messages
- one agent explicitly transferred all `shell_completion.py` responsibility to the other
- the second agent implemented the `core.py` side
- patches:
  - `agent1.patch`: `src/click/shell_completion.py`
  - `agent7.patch`: `src/click/core.py`
- result: pass

Updated:

- 3 messages
- representative message:
  - "I'm still waiting for your acknowledgment of my plan ..."
- patch outcome:
  - `agent1.patch`: `src/click/shell_completion.py`
  - `agent7.patch`: empty
- eval outcome:
  - feature 2 failed with `TypeError: Context.__init__() got an unexpected keyword argument 'validate_nesting'`

Interpretation:

- the shared integration edit in `src/click/core.py` never landed
- the run failed despite a clean merge

## Example D: `typst_task/6554/4,9` is the useful counterexample

Paths:

- baseline conversation: `/Users/kevin/Dev/CooperBench/logs/coop-oh-gemini-3-flash-flash-baseline/coop/typst_task/6554/f4_f9/conversation.json`
- updated conversation: `/Users/kevin/Dev/CooperBench/logs/coop-oh-gemini-3-flash-flash-owned-regions/coop/typst_task/6554/f4_f9/conversation.json`

Baseline:

- 18 messages
- ownership negotiation broke down
- both agents argued over who should implement both `first` and `last`
- one patch ended up empty
- result: fail

Updated:

- 19 messages
- the agents used an actual region split inside the same file:
  - one owned `first`
  - the other owned `last`
  - tests were assigned to one side
- both patches were non-empty
- result: pass

Interpretation:

- "exact owned regions" can help
- the problem is the acknowledgement gate, not the idea of more precise ownership by itself

## What most likely caused the regression

Primary cause:

- the explicit "do not edit until acknowledged" rule is too strict for the current OpenHands messaging model

Why:

- messages are asynchronous and injected into the next reasoning step
- there is no hard handshake primitive, no structured lock, and no guaranteed acknowledgement timing
- the system prompt can ask agents to wait, but the runtime cannot enforce a reliable rendezvous
- as a result, the wait becomes a blocking heuristic rather than a coordination protocol

Observed consequences:

- more one-sided conversations
- many more empty patches
- many more clean-but-incomplete merges
- far fewer successful same-file integrations

Secondary cause:

- the wording encourages over-separation
- some CooperBench tasks need coordinated edits in the same implementation file
- removing those shared edits entirely often leaves one feature unintegrated

## What this experiment does and does not show

What it shows:

- the specific prompt change tested here is harmful on `flash`
- the harmful part is very likely the acknowledgement gate
- exact owned regions by themselves may still be useful

What it does not show:

- it does not prove that all stronger coordination prompts are bad
- it does not prove that same-file region planning is bad
- it does not isolate stochastic model variance completely

That said, the empty-patch jump from `6` to `28` is too large to explain as noise.

## Recommended next experiment

Test only this narrower prompt change:

- keep: "list the exact region you will own in each file"
- remove: "Do not edit a shared file until your teammate explicitly acknowledges the split"

Reason:

- `typst_task/6554/4,9` suggests exact region ownership can help
- the aggregate data suggests the acknowledgement gate is what turns coordination into inactivity

Expected outcome:

- some reduction in overlap should remain
- empty-patch rate should fall back toward baseline
- pass rate should recover materially if the hypothesis is correct
