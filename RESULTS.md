# Results

**Status (2026-08-16): the comparative study (§4) completed — 500/500 runs, 5 conditions ×
100 seeds, matching the paper's own scale. Result: ERL significantly beats every other
condition (p<0.00001 vs. E, L, F, and B), and the internal structure substantially
reproduces the paper's own findings (L beats E, F≈B). One real, honest discrepancy remains
(E beats B here; the paper found the reverse), and overall survival difficulty is not
calibrated to their reported rate (83% of ERL runs reach the step ceiling here vs. their
~7%).** See `README.md` for exactly what's matched vs. adapted from the paper.

---

## 1. World AL construction (2026-08-09)

An earlier prototype of this codebase ran the ERL mechanism on a simpler, generic
predator-prey ecology instead of Ackley & Littman's actual World AL, to get the core
genome/network/learning machinery working first. A comparative study on that prototype
reproduced the paper's headline result (ERL beats luck) but not its internal ranking (E vs.
L). Before sinking more compute into scale alone, the world was rebuilt to match the paper's
described mechanics directly — see `README.md`'s "What's matched vs. necessarily adapted"
section for the full list of what's now matched (including every exact number the paper
actually publishes: grid size, sense ranges, carnivore spawn interval, min_plants) vs. still
necessarily chosen (the paper never publishes damage/threshold/growth-rate constants).

**Structural correction, not just cosmetic:** the prototype had two adaptive species
(predator + prey, both genome+learning). The paper has exactly one adaptive species
("agents") plus a separate, permanently non-adaptive species ("carnivores", hard-coded FSA,
never affected by `strategy`). This is now correctly reflected — `Carnivore` has no genome
field at all (see `test_carnivores_have_no_genome_or_learning`).

**Status after rebuild:**
- 18/18 unit tests passing.
- One full-scale smoke run (`--seed 1 --steps 20000`, 100×100 grid, default population): ran
  365 steps before agent-population extinction (carnivores overwhelmed a population that had
  itself boomed rapidly — final state 0 agents, 351 carnivores). Confirmed the mechanics run
  without crashing; this specific early extinction is not evidence of anything beyond "these
  chosen, unpublished-by-the-paper constants produce a fast boom-bust here."
- **Performance dropped substantially**: ~30 steps/sec on this machine (100×100 grid,
  carnivores, trees, walls, corpses), down from ~240 steps/sec on the earlier, simpler
  ecology. Reaching the paper's 1,000,000-step comparative-study ceiling is estimated at
  **~9 hours per seed** that survives that long.

## 2. First full-scale comparative study on World AL — clean null, root-caused

Full paper-scale study: 5 conditions × 100 seeds × 1,000,000-step ceiling, 24-way parallel.
All 500 runs completed.

| strategy | median | mean | max | reached 1M |
|---|---|---|---|---|
| ERL | 371 | 408 | 2,004 | 0 |
| E | 380 | 513 | 12,285 | 0 |
| L | 384 | 1,344 | 95,065 | 0 |
| F | 391 | 419 | 1,915 | 0 |
| B | 393 | 857 | 17,172 | 0 |

Every median sits in a 371-393 band. **No pairwise comparison was significant** (Mann-
Whitney, n=100 each) — not even ERL vs. B (p=0.056), the one result that had survived every
prior version of this study. Not a single one of 500 runs reached even 10% of the step
ceiling.

**Diagnosis:** direct population trace (seed 1, ERL) showed the founder population of 60
agents exploding to 1,916 by step 120 — fed by abundant plants and an easy reproduction
threshold — which then fed an equally explosive carnivore boom (6→739 by step 280),
collapsing the agent population entirely by ~step 400. Every strategy dies at the same rate
because death arrives via this boom-bust cycle before behavioral differences have any time
window to matter, not because learning/evolution don't work. A structural/parameter problem
in the chosen constants, not a finding about the ERL mechanism.

## 3. Two-stage retune

**Attempt 1 (didn't work):** raised `carnivore_reproduction_energy_threshold` (14→18) and
`carnivore_reproduction_energy_cost` (7→10), reasoning that carnivore population growth was
the runaway variable. Validation (5 seeds, 50k-step budget) showed no improvement —
extinction still at 369-534 steps, carnivore populations still exploding to 300+.

**Root cause, found by tracing the actual population dynamics:** the trigger wasn't
carnivore reproduction, it was agent reproduction being too easy. At
`reproduction_energy_threshold_agent=10.0` (only ~2 plant bites above the starting energy of
5.0) against abundant plants (up to ~2,700 on this grid), the agent population had no real
ceiling before it overshot the environment's actual carrying capacity — and that overshoot
is what fed the carnivore boom, not carnivore-side parameters.

**Attempt 2 (fixed it):** raised `max_energy_agent` (15→30), `reproduction_energy_threshold_agent`
(10→22), and `reproduction_energy_cost_agent` (5→10) — directly targeting the trigger.
Re-traced the same seed: agent population now oscillates in a sustained 100-700 range
(rather than overshooting to ~1,900) and was still healthy at step 3,000, roughly 7.5x
longer than the previous full collapse point. Quick multi-seed check (6 seeds) confirmed the
same pattern — all 6 outlasted a 90-second screening cutoff that previously killed every
seed within it.

A second full 500-run study was launched on the retuned config once this validation held up.
Interrupted once mid-run for a hardware concern — sustained 24-way parallel load pushed CPU
temperature to 95.8°C during a period of high ambient/outside temperature — killed
deliberately, then resumed later from a resume-safe runner (skips any seed/strategy pair
with an already-completed log) once conditions were judged safe again.

## 4. Final comparative study result — ERL significantly beats every other condition

500/500 runs complete, 100 seeds per condition, run to the paper's own 1,000,000-step
comparative-study ceiling, on the retuned World AL (§3).

| strategy | median | mean | reached 1M cap |
|---|---|---|---|
| **ERL** | **1,000,000** | 832,097 | **83%** |
| L | 40,413 | 407,919 | 37% |
| E | 3,221 | 335,594 | 33% |
| F | 1,746 | 43,256 | 2% |
| B | 1,242 | 302,136 | 23% |

ERL's *median* survival time is the step ceiling itself — more than half of all 100 ERL runs
ran the full 1,000,000 steps without agent extinction.

**Significance (Mann-Whitney U, two-sided, n=100 each):**

| comparison | p-value | |
|---|---|---|
| ERL vs. E | <0.00001 | *** |
| ERL vs. L | <0.00001 | *** |
| ERL vs. F | <0.00001 | *** |
| ERL vs. B | <0.00001 | *** |
| E vs. L | 0.00424 | ** |
| E vs. F | <0.00001 | *** |
| E vs. B | 0.00067 | *** |
| L vs. F | <0.00001 | *** |
| L vs. B | <0.00001 | *** |
| F vs. B | 0.97268 | n.s. |

**Headline result:** ERL (nature + nurture combined) significantly outperforms evolution
alone, learning alone, neither, and pure luck — p<0.00001 against all four, with real
statistical power (n=100/condition, matching the paper's own sample size).

**The internal structure substantially reproduces the paper's own findings, not just the
headline claim:**
- **L significantly beats E** (p=0.004) — reproducing Ackley & Littman's own "surprising"
  finding that learning alone outperforms evolution alone (their explanation: it's easier to
  evolve a good *goal*, the compact evaluation network, than to evolve a good *behavior*, the
  full action network — learning fills the gap evolution alone can't).
- **F ≈ B** (p=0.97, no significant difference) — no-adaptation and pure random luck are
  statistically indistinguishable, consistent with the paper's observation that non-learning
  strategies don't meaningfully beat chance.

**One honest, real discrepancy, not glossed over:** the paper reports evolution-alone (E)
doing *worse* than luck (B) in the first ~500,000 steps. Here, **E significantly beats B**
(p=0.0007) — the opposite direction. Not yet explained; a plausible candidate is that this
world's carnivore-avoidance/food-finding dynamics give even an unlearned-but-evolving genome
more traction than in their world, but this is a hypothesis, not a diagnosed cause.

**Worth being precise about which direction is actually the surprising one here.** "Evolution
alone beats pure randomness" is the intuitive, textbook-biology expectation — natural
selection is a real directional force, and even simple hard-wired instincts generally
outperform organisms with no behavioral structure at all. So this result (E beats B) isn't
the odd one out relative to general intuition. **The paper's finding is the counterintuitive
one, and they say so themselves** — their explanation is that pure mutation-and-selection
search, with no within-lifetime correction, can get systematically stuck producing
*confidently bad* behavior (e.g. a mutated action network with a consistent bias toward
danger), not just noisy behavior — worse than Brownian motion, which has no consistent bias
at all, just no intelligence. That failure mode needs a large, rugged, high-dimensional
behavior search space and/or weak selection pressure relative to it; it isn't automatic.
Candidate reasons this world doesn't reproduce that specific trap, none confirmed: (a) a
simpler action space (4-way categorical vs. their exact 2-bit CRBP-driven encoding) may just
be easier for blind mutation-selection to avoid landing on a *confidently wrong* policy; (b)
the retune that fixed the boom-bust collapse (§3) raised reproduction thresholds broadly,
which may have incidentally given every condition, including E, enough population size/
mutation balance to purge bad mutations effectively; (c) different world mechanics
(carnivore threat model, observation encoding) may simply make a confidently-wrong policy
harder to evolve here than in their richer sensory-action space. Distinguishing these would
need looking at what E's evolved policies actually do behaviorally, not just their
survival-time distribution — not done.

**Also unresolved: overall survival difficulty is not calibrated to the paper's reported
rate.** ERL reaches the 1M-step ceiling in 83% of runs here, versus their reported ~7%
(comparative study) to ~18% (a different measure, 10,000-step survival) across their whole
five-strategy population. The *ranking* matches well; the *absolute difficulty* of long-term
survival does not — this world remains easier for a well-adapted population than theirs was,
a direct consequence of retuning away from the boom-bust collapse in §2/§3 using necessarily-
guessed constants, not something recoverable from the paper.

**What would close the remaining gap, if pursued further:** implementing the paper's exact
CRBP learning algorithm (currently a REINFORCE approximation), their redundant bit-string
genome encoding (currently real-valued weights), recalibrating constants toward their
reported ~7-18% survival-difficulty range, and attempting their deeper single-population
longitudinal study (extending a successful ERL population to millions of steps and running
the functional-constraint genetic-assimilation analysis — see `README.md`) — none of which
this comparative study attempted.
