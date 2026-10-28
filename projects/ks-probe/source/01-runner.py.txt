"""
AsyncOrchestrator — rate-limited, checkpointing async dispatcher.

Accepts a list of RunSpec objects, skips already-completed ones,
dispatches the rest concurrently (bounded by max_concurrent semaphore),
and collects results.
"""
from __future__ import annotations

import asyncio
import logging
from typing import Any, Awaitable, Callable, Dict, List, Optional

from ks_probe.orchestrator.checkpoint import CheckpointManager
from ks_probe.orchestrator.planner import RunSpec
from ks_probe.orchestrator.rate_limiter import TokenBucketRateLimiter

logger = logging.getLogger(__name__)

RunFn = Callable[[RunSpec], Awaitable[Dict[str, Any]]]


class AsyncOrchestrator:
    """
    Async, rate-limited, checkpointing experiment runner.

    Usage:
        orchestrator = AsyncOrchestrator(
            run_fn=my_experiment.execute_run,
            rate_limiters={"claude-sonnet-4-6": TokenBucketRateLimiter(rpm=20)},
            max_concurrent=4,
            checkpoint_manager=CheckpointManager(),
        )
        results = asyncio.run(orchestrator.run_all(specs))
    """

    def __init__(
        self,
        run_fn: RunFn,
        rate_limiters: Optional[Dict[str, TokenBucketRateLimiter]] = None,
        max_concurrent: int = 4,
        checkpoint_manager: Optional[CheckpointManager] = None,
        mock_mode: bool = False,
    ) -> None:
        self._run_fn = run_fn
        self._limiters = rate_limiters or {}
        self._sem = asyncio.Semaphore(max_concurrent)
        self._checkpoint = checkpoint_manager
        self._mock_mode = mock_mode

    async def run_all(self, specs: List[RunSpec]) -> List[Dict[str, Any]]:
        """
        Dispatch all specs. Returns list of result dicts (errors included).
        Specs already recorded in the checkpoint DB are skipped.
        """
        to_run: List[RunSpec] = []
        skipped = 0

        for spec in specs:
            if self._checkpoint and self._checkpoint.is_done(
                experiment_name=spec.experiment_name,
                model_id=spec.model_id,
                seed=spec.seed,
                threshold_tokens=spec.context_tokens,
                probe_position=spec.probe_position,
            ):
                skipped += 1
            else:
                to_run.append(spec)

        logger.info(
            "[Orchestrator] %d specs to run, %d skipped (checkpoint).",
            len(to_run), skipped,
        )

        tasks = [self._run_one(spec) for spec in to_run]
        all_results = await asyncio.gather(*tasks, return_exceptions=True)

        results: List[Dict[str, Any]] = []
        for spec, outcome in zip(to_run, all_results):
            if isinstance(outcome, Exception):
                logger.error("[Orchestrator] %s FAILED: %s", spec.run_key, outcome)
                results.append({"run_key": spec.run_key, "error": str(outcome)})
            else:
                results.append(outcome)

        return results

    async def _run_one(self, spec: RunSpec) -> Dict[str, Any]:
        limiter = self._limiters.get(spec.model_id)
        async with self._sem:
            if limiter and not self._mock_mode:
                await limiter.acquire(estimated_tokens=spec.context_tokens or 0)
            try:
                return await self._run_fn(spec)
            except Exception as exc:
                raise exc  # will be caught by gather + return_exceptions
