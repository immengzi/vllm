# dyn-pd sleepfix backports (vllm @ v0.18.0)

Source-only backport used by the llm-la dyn-pd warm-standby deployment
(`reg.local:32000/library/vllm-ascend:v0.18.0`). It was previously kept in the
llm-la repo's `patches/vllm/` working directory; this is its proper home.

Archive state: committed on local branch `wip/dynpd-sleepfix-v0.18.0`, **not
pushed**. Baseline: vllm `v0.18.0` = `bcf2be9612`.

## 01-fix-43433-delayed-kv-connector-free-v0.18.0.patch

Upstream vLLM #43433 ("Keep scheduler alive for delayed KV connector frees",
commit `82536acc54`). Backports cleanly to `v0.18.0` (verified with
`git apply --check` against the `v0.18.0` tree).

Why: in the per-card warm-standby topology a prefill (KV producer) engine that
has served requests can transiently fail `/sleep` with HTTP 500. After a
request finishes, its KV blocks are released only once the connector reports
`finished_sending`, which is consumed while the engine keeps stepping. An idle
engine stops stepping, so the blocks stay referenced and
`reset_prefix_cache` refuses to discard them.

What it changes (two spots):

- `vllm/v1/core/sched/scheduler.py::has_finished_requests` also counts finished
  requests that are out of the scheduling queues but still in `self.requests`
  awaiting delayed connector cleanup.
- `vllm/v1/engine/core.py::_process_engine_step` yields the GIL when no model
  execution happened but the scheduler still has requests, so background
  connector transfer threads keep making progress.

Rebalancer-side complement: `pd_rebalancer.py::_sleep_engine` retries `/sleep`
with backoff (`PD_REBALANCER_SLEEP_RETRIES`, default 5;
`PD_REBALANCER_SLEEP_BACKOFF_SECONDS`, default 2) so a transient HTTP 500 does
not abort the transition.

Apply:

```bash
cd <vllm-v0.18.0-source>
git apply patches/dynpd-sleepfix/01-fix-43433-delayed-kv-connector-free-v0.18.0.patch
```
