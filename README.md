# Evolutionary Reinforcement Learning — Ackley & Littman (1991)

A from-scratch replication of:

> Ackley, D. H., & Littman, M. L. (1991). *Interactions Between Learning and
> Evolution.* In C. G. Langton et al. (Eds.), *Artificial Life II* (pp.
> 487–509). Addison-Wesley.

The paper's question: does combining genetic evolution with within-lifetime
reinforcement learning ("ERL") produce more effective behavior than either
mechanism alone, or than chance? Their answer is yes, demonstrated in a
concrete artificial-life world ("World AL") populated by adaptive agents and
a separate, hard-coded predator species. This repo rebuilds that world and
architecture directly, then runs their own comparative-study design at their
own scale (5 conditions × 100 seeds × 1,000,000-step ceiling) to test whether
the result replicates.

**It does — with real statistical power (p < 0.00001, n = 100 seeds per
condition).** See [Results](#results) below.

This is a sibling project to
[HintonNowlan1987](https://github.com/doesburg11/HintonNowlan1987) (a replication
of Hinton & Nowlan 1987, the theoretical precursor this paper builds on) and
feeds a further-diverging module,
[`eco_evolutionary_erl_baldwin`](https://github.com/doesburg11/PredPreyGrass/tree/main/predpreygrass/evolutionary/eco_evolutionary_erl_baldwin),
in the [PredPreyGrass](https://github.com/doesburg11/PredPreyGrass) project.
This repo is the faithful-replication baseline; that module is where the
architecture gets pushed toward a different environment.

## The mechanism

Each agent carries two single-layer networks:

- **Evaluation network** (`genome.eval_weights`, `eval_bias`) — fixed for the
  agent's entire life. Maps observation → a scalar "goodness" value. This
  *is* the genetically inherited goal, never touched by learning.
- **Action network** (`genome.action_weights`, `action_bias`) — only the
  *initial* weights are genetic. A live copy is made at birth and adjusted
  every step by reinforcement learning.

Reinforcement signal: `R_t = E_t - E_{t-1}` — "am I better off now than a
moment ago, by my own inherited sense of good." No externally supplied
reward function.

**Reproduction copies the genome record, never the live action network.**
Whatever an agent learned during its life is discarded at reproduction; only
the pre-learning genome (plus mutation/crossover) is passed on. This is what
makes the mechanism Darwinian, not Lamarckian — not a convention, a hard
architectural fact: reproduction reads from a record that learning never
writes to. `erl/world.py`'s `_handle_reproduction` enforces this explicitly
(always `.genome`, never `.action_weights`), and
`tests/test_erl_baldwin.py::test_offspring_genome_does_not_inherit_parents_learned_weights`
asserts it directly.

The paper's headline finding: combined evolution+learning (ERL) produced far
more long-surviving populations than evolution alone, learning alone, or no
adaptation — and evolution alone did *surprisingly badly*. Their explanation:
it's much easier to genetically specify a compact **goal** (one
evaluation-network weight: "food is good") than to specify the full
**behavior** needed to act on it (many action-network weights). Genes encode
*what*; learning fills in *how*.

They also detected genetic assimilation (the Baldwin effect itself) via
**functional-constraint analysis**: track, per genome site, how much that
site's value changes across a lineage over generations. Sites that matter for
survival get purged of mutations (low change rate = "constrained"); sites
that don't matter drift freely. Early in a run, evaluation-network sites were
constrained (the learned goal was doing the work); later, action-network
sites became constrained instead (the behavior had been assimilated — agents
approached food instinctively, no learning required). `erl/metrics.py`'s
`FunctionalConstraintTracker` implements the same method, tracked separately
for eval-weight sites vs. action-weight sites — this repo runs it, but the
paper's own deeper longitudinal (millions-of-steps, single-population)
version of that analysis was not attempted here; see
[Known gaps](#known-gaps-from-the-paper).

They also found a second phenomenon not in Hinton & Nowlan's original
theoretical Baldwin effect: **shielding**. When an innate ability (e.g.
instinctive predator-avoidance) is survival-critical enough that agents must
be born with it, the corresponding *evaluation*-network genes for that domain
stop mattering for fitness and can drift freely — some agents can genuinely
evolve to prefer the sight of danger while remaining fit, because their
action network avoids it reflexively regardless of what the evaluation
network says.

## What's matched from the paper vs. necessarily adapted

**Matched, including every exact number the paper actually publishes:**

- 100×100 grid, non-toroidal (`grid_size=100`).
- Two distinct populations, not two adaptive ones: a single ADAPTIVE species
  (`Agent` — genome + learning, omnivorous: eats plants, dead agents, dead
  carnivores) and a permanently NON-adaptive species (`Carnivore` — no
  genome, no network, no learning, hard-coded "seek nearest visible agent"
  rule, regardless of `strategy`).
- Agents sense 4 cells in each compass direction, carnivores 6
  (`agent_sense_range=4`, `carnivore_sense_range=6` — exact paper values).
- A new carnivore spawns every 200 steps (`carnivore_spawn_interval=200` —
  exact paper value, Figure 4).
- `min_plants=50` reseed floor (exact paper value).
- Trees (shelter, one occupant, carnivores can't climb or attack a sheltered
  agent), walls (permanent, damage on collision), corpses (persistent,
  partially edible over multiple bites, decay over time) — all present per
  Figure 4/5, mechanics implemented as described.
- Action semantics exactly matching Figure 5's table: 4 directions (no
  "stay"), effect determined by target-cell contents (Enter / Eat all /
  Climb / Damage self / Damage other / Eat some), including that carnivores
  structurally cannot target a wall or occupied tree ("as programmed").
- Observation vector matches Figure 4's input panel: visual appearance in 4
  directions + in-tree binary + health + energy (`OBS_DIM=7`; the paper's
  explicit "bias" input unit is instead a standard network bias term —
  behaviorally equivalent, not an extra input feature).
- The five comparative-study conditions (`--strategy`): **ERL** (evolution +
  learning), **E** (evolution only), **L** (learning only, no heritable
  variation), **F** (fixed, no evolution and no learning), **B** (behave
  randomly — pure luck).

**Not the same, and can't be, because the paper doesn't say:** damage
amounts, energy thresholds, growth/birth/death probabilities, wall density,
and reproduction costs are never published as numbers — only described
qualitatively ("minor damage", "geometric growth", "sufficiently
nourished"). Every such constant in `erl/config.py` is my own chosen value,
clearly marked there. No amount of rebuilding recovers numbers the paper
never printed.

**Deliberate simplifications:** the learning rule (`erl/networks.py`'s
`reinforce_update`) is a standard REINFORCE policy-gradient step, not the
paper's exact CRBP backprop-through-stochastic-threshold algorithm (Figure
3) — same complementary-reinforcement logic, different implementation.
Genome encoding is real-valued weights with Gaussian mutation, not their
redundant 4-bit-per-weight bit-string.

## Running it

```bash
pip install -r requirements.txt
python run_erl_simulation.py --steps 200000 --seed 41 --log-every 500 --constraint-window 5000
```

Useful flags:

- `--strategy {ERL,E,L,F,B}` — override the config's default strategy.
- `--steps` — step ceiling (the paper's own comparative study uses
  1,000,000).
- `--out-dir` — where to write `progress.csv` (defaults to
  `~/erl_results/ERL_BALDWIN_<timestamp>/`).

Run the test suite with:

```bash
pip install -r requirements.txt pytest
pytest tests/ -q
```

Performance note: each agent's network is tiny (single-layer) and learns
locally with plain NumPy — no Ray, no gymnasium multi-agent API, no GPU. But
the full World AL mechanics (100×100 grid, carnivores, trees, walls,
corpses) run at roughly 30 steps/sec single-threaded. A run to the paper's
own 1,000,000-step comparative-study ceiling takes on the order of hours per
seed that survives that long.

## Results

Full detail, including the boom-bust collapse that had to be diagnosed and
fixed before this result held up, is in [`RESULTS.md`](RESULTS.md). Summary:

500/500 runs complete, 100 seeds per condition, run to the paper's own
1,000,000-step comparative-study ceiling.

| strategy | median survival | mean survival | reached 1M-step cap |
|---|---|---|---|
| **ERL** | **1,000,000** | 832,097 | **83%** |
| L | 40,413 | 407,919 | 37% |
| E | 3,221 | 335,594 | 33% |
| F | 1,746 | 43,256 | 2% |
| B | 1,242 | 302,136 | 23% |

**ERL (evolution + learning combined) significantly outperforms evolution
alone, learning alone, no adaptation, and pure random behavior — p < 0.00001
against all four (Mann-Whitney U, two-sided, n = 100 per condition).**

The internal structure substantially reproduces the paper's own findings
too: learning-alone significantly beats evolution-alone (p = 0.004,
matching their own "surprising" finding), and no-adaptation is statistically
indistinguishable from luck (p = 0.97).

One honest discrepancy from the paper, not resolved here: evolution-alone
significantly *beats* luck in this replication (p = 0.0007); the paper found
the reverse. See `RESULTS.md` for the discussion of which direction is
actually the surprising one and the candidate explanations considered.

## Known gaps from the paper

- Learning rule is a REINFORCE approximation of the paper's exact CRBP
  algorithm; genome encoding is real-valued weights, not their redundant
  bit-string.
- Overall survival difficulty isn't calibrated to their reported rate — 83%
  of ERL runs reach the step ceiling here vs. their ~7-18%. The *ranking*
  matches; the *absolute difficulty* doesn't.
- The paper's deeper result — a single-population longitudinal study
  (millions of steps, functional-constraint genetic-assimilation analysis,
  the "shielding"/"goal regression" findings) — was never attempted here.
  This comparative study only measures survival time.
- The evolution-alone-vs-luck discrepancy above is unexplained.

## References

- Ackley, D. H., & Littman, M. L. (1991). *Interactions Between Learning and
  Evolution.* In C. G. Langton, C. Taylor, J. D. Farmer, & S. Rasmussen
  (Eds.), *Artificial Life II* (pp. 487–509). Addison-Wesley.
- Hinton, G. E., & Nowlan, S. J. (1987). *How Learning Can Guide Evolution.*
  Complex Systems, 1(3), 495–502. See
  [HintonNowlan1987](https://github.com/doesburg11/HintonNowlan1987) for a direct
  replication of this paper, the theoretical precursor to Ackley & Littman's
  work.
