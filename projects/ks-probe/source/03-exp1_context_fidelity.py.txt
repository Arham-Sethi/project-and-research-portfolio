"""
Exp1: Context Fidelity Decay
RQ1 — Does PRA degrade as context grows from 10K to 200K tokens?

Design:
  len(models) × len(context_sizes) × len(seeds) runs.
  Probe facts injected at random positions throughout the context.
"""
from __future__ import annotations

import json
import logging
import time
from typing import Any, Dict, List

from ks_probe.corpus.builder import ContextBuilder
from ks_probe.core.seed import set_seed
from ks_probe.db.queries import RunData
from ks_probe.experiments.base import BaseExperiment
from ks_probe.orchestrator.planner import RunSpec, RunPlanner
from ks_probe.scorer.composite import score_all_questions, average_pra

logger = logging.getLogger(__name__)


class Exp1ContextFidelity(BaseExperiment):

    experiment_id = "exp1_context_fidelity"

    def build_run_specs(self) -> List[RunSpec]:
        specs = self._planner.plan(self.config)
        # Filter out runs where context_tokens exceeds the model's max
        # (e.g. deepseek-v3.2 at 100K/150K when max is 65K)
        filtered: List[RunSpec] = []
        for spec in specs:
            try:
                mcfg = self.model_registry.get(spec.model_id)
                if spec.context_tokens and spec.context_tokens > mcfg.max_context_tokens:
                    logger.info(
                        "[exp1] Dropping %s at %dk (max %dk)",
                        spec.model_id,
                        spec.context_tokens // 1000,
                        mcfg.max_context_tokens // 1000,
                    )
                    continue
            except KeyError:
                pass
            filtered.append(spec)
        return filtered

    async def execute_run(self, spec: RunSpec) -> Dict[str, Any]:
        set_seed(spec.seed)
        adapter = self._get_adapter(spec.model_id)
        ctx_tokens = spec.context_tokens or 50_000
        n_probes = self.config.probes_per_run

        # Sample probes
        probes = self._sample_probes(n_probes, spec.seed)

        # Build context (filler + injected probes + question block)
        builder = ContextBuilder(
            model_id=spec.model_id,
            seed=spec.seed,
        )
        injected = builder.build(
            target_tokens=ctx_tokens,
            probe_facts=probes,
        )

        # Send to model
        t0 = time.monotonic()
        response = await adapter.send_prompt(
            injected.prompt_text,
            max_output_tokens=self.config.max_output_tokens,
            temperature=self.config.temperature,
        )
        latency_ms = int((time.monotonic() - t0) * 1000)

        # Score
        results = score_all_questions(response.text, injected.scoring_rubric)
        pra = average_pra(results)
        hallucinations = sum(1 for r in results if r.hallucination_flag)
        per_q = {
            r["q_num"]: {
                "pra": s.pra_score,
                "exact_match": s.exact_match,
                "hall": s.hallucination_flag,
            }
            for r, s in zip(injected.scoring_rubric, results)
        }

        # Write to DB
        run_data = RunData(
            experiment_name=self.experiment_id,
            model=spec.model_id,
            seed=spec.seed,
            threshold_tokens=ctx_tokens,
            model_version=response.usage.model_version,
            condition="single_prompt",
            pra_score=pra,
            hallucination_count=hallucinations,
            response_latency_ms=response.usage.latency_ms or latency_ms,
            input_tokens=response.usage.input_tokens,
            output_tokens=response.usage.output_tokens,
            raw_prompt=injected.prompt_text[:2000],  # truncate for storage
            raw_response=response.text[:2000],
            per_question_scores=per_q,
        )
        run_id = self._write_result(run_data)

        logger.info(
            "[exp1] run_id=%d model=%s ctx=%dk pra=%.3f",
            run_id, spec.model_id, ctx_tokens // 1000, pra,
        )
        return {
            "run_key": spec.run_key,
            "run_id": run_id,
            "pra": pra,
            "context_tokens": ctx_tokens,
            "model_id": spec.model_id,
        }
