"""
CompositeScorer — combines exact_match, keyword_rule, constraint, and hallucination
scorers into a single Probe Recall Accuracy (PRA) score per question.

Works with the rubric dict format produced by ks_probe.probes.injector.inject_probes().
"""
from __future__ import annotations

import re
import unicodedata
from dataclasses import dataclass
from typing import Dict, List, Optional

from ks_probe.scorer.base import ScorerResult
from ks_probe.scorer.exact_match import exact_match_score
from ks_probe.scorer.hallucination import HallucinationDetector
from ks_probe.scorer.keyword_rule import KeywordRuleScorer
from ks_probe.scorer.constraint import ConstraintComplianceScorer


_hall_detector = HallucinationDetector()
_kw_scorer = KeywordRuleScorer()
_cs_scorer = ConstraintComplianceScorer()


def score_rubric_entry(response: str, rubric: dict) -> ScorerResult:
    """
    Score a single model response against one rubric entry.

    Args:
        response: The model's text response (may be the full multi-Q response;
                  caller should pre-slice to the per-question answer first).
        rubric: One entry from InjectedContext.scoring_rubric, with keys:
                gold, scoring, keywords, required_conclusion.

    Returns:
        ScorerResult with all sub-scores and final pra_score.
    """
    gold = rubric.get("gold", "")
    scoring = rubric.get("scoring", "exact_match")

    em = bool(exact_match_score(response, gold))
    kw = _kw_scorer.score(response, rubric)
    cs = _cs_scorer.score(response, rubric)
    hall = _hall_detector.detect(response, gold)

    if hall:
        pra = 0.0
    elif scoring == "exact_match":
        pra = 1.0 if em else 0.0
    elif scoring == "keyword_rule":
        pra = kw
    elif scoring == "constraint_compliance":
        pra = cs
    else:
        # Fallback: average of non-null sub-scores
        pra = (float(em) + kw + cs) / 3.0

    return ScorerResult(
        exact_match=em,
        keyword_score=kw,
        constraint_score=cs,
        hallucination_flag=hall,
        pra_score=max(0.0, min(1.0, pra)),
    )


def parse_qa_response(full_response: str, q_nums: List[int]) -> Dict[int, str]:
    """
    Parse a model response that answers multiple questions in Q{n}: format.

    The injector formats questions as:
        Q1: <question>\nAnswer: Q2: <question>\nAnswer: ...

    Models are expected to fill in after each "Answer:" or produce
    "Q1: <answer>\nQ2: <answer>" style output.

    Returns a dict mapping q_num → answer text.
    """
    answers: Dict[int, str] = {}

    # Try "Q{n}: <answer>" pattern (mock adapter format)
    for q_num in q_nums:
        m = re.search(
            rf"Q{q_num}[:\.\)]\s*(.+?)(?=Q\d+[:\.\)]|$)",
            full_response,
            re.DOTALL | re.IGNORECASE,
        )
        if m:
            answers[q_num] = m.group(1).strip()

    if answers:
        return answers

    # Fallback: use the full response for every question (single-Q prompts)
    stripped = full_response.strip()
    for q_num in q_nums:
        answers[q_num] = stripped

    return answers


def score_all_questions(
    full_response: str,
    scoring_rubric: List[dict],
) -> List[ScorerResult]:
    """
    Score all questions in a rubric against the full model response.

    Args:
        full_response: The complete text returned by the adapter.
        scoring_rubric: List of rubric dicts (from InjectedContext.scoring_rubric).

    Returns:
        List of ScorerResult, one per rubric entry, in the same order.
    """
    q_nums = [r["q_num"] for r in scoring_rubric]
    parsed = parse_qa_response(full_response, q_nums)

    results: List[ScorerResult] = []
    for rubric in scoring_rubric:
        q_num = rubric["q_num"]
        answer = parsed.get(q_num, full_response)
        results.append(score_rubric_entry(answer, rubric))

    return results


def average_pra(results: List[ScorerResult]) -> float:
    """Compute mean PRA across a list of ScorerResults."""
    if not results:
        return 0.0
    return sum(r.pra_score for r in results) / len(results)
