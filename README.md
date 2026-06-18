# GPHH-CARP NSGA-II Test — Elite & Sparse Selection Strategy

> **Genetic Programming Hyper-Heuristic for the Capacitated Arc Routing Problem (CARP)**
>
> This script (`main_gphh_metro_nsga2_test.py`) implements an NSGA-II-based multi-objective GPHH with an **Elite & Sparse** parent selection strategy, designed to improve coverage of the Pareto front in the objective space.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Algorithm Pipeline](#algorithm-pipeline)
- [Elite & Sparse Strategy](#elite--sparse-strategy)
- [Configuration](#configuration)
- [Dependencies](#dependencies)
- [Usage](#usage)
- [Output Structure](#output-structure)
- [Output Files](#output-files)
- [Project Structure](#project-structure)

---

## Overview

This program evolves routing heuristics for the **Capacitated Arc Routing Problem (CARP)** using Genetic Programming Hyper-Heuristic (GPHH). It optimizes two objectives simultaneously:

- **F1 (total cost)** — minimize total routing cost
- **F2 (mileage difference)** — minimize the mileage imbalance among vehicles

The NSGA-II algorithm maintains a population of GP individuals (heuristic rules) and evolves them through crossover, mutation, and reproduction over multiple generations.

---

## Key Features

- **Multi-objective optimization** via NSGA-II non-dominated sorting and crowding distance
- **Elite & Sparse selection strategy** — HDBSCAN clustering in normalized objective space to identify and preserve diverse, high-quality individuals
- **Rotating evaluation model** — instances are rotated each generation to improve generalization
- **Parallel evaluation** support for fitness computation
- **Train/Test split** — training on seeded instances, testing on held-out instances
- **Automatic visualization** of population distribution across generations
- **Archive-based testing** — Pareto fronts from all generations are archived and merged into the final test population

---

## Algorithm Pipeline

Each generation executes the following steps:

1. **Pareto extraction** — extract the non-dominated front from the current parent population; archive it
2. **Breeding** — generate offspring via crossover / mutation / reproduction (with Elite & Sparse or tournament selection)
3. **Offspring evaluation** — evaluate offspring on a *rotated* set of instances (offset from parent evaluation seeds)
4. **Combine & select** — merge parent + offspring populations, apply NSGA-II survivor selection
5. **Deduplication** — remove individuals with identical (F1, F2) pairs; fill empty slots via crossover from top performers
6. **Re-evaluation** — evaluate the selected population on the current generation's instances
7. **Visualization** — save population distribution plots (parent, offspring, combined, selected)

After all generations complete, a **testing phase** runs:
- Merge archived Pareto fronts with the final parent population
- Evaluate on held-out test instances
- Save test Pareto front and population distribution comparisons

---

## Elite & Sparse Strategy

The Elite & Sparse strategy addresses the problem of "gaps" in the Pareto front by intentionally selecting parents from sparsely populated regions.

### Workflow

| Step | Description |
|------|-------------|
| **1. Non-dominated sorting** | Collect individuals up to rank `elite_max_rank` (default: 6) into a candidate pool |
| **2. Normalize objectives** | Min-max normalize F1 and F2 to [0, 1] |
| **3. Compute distances** | Calculate pairwise Euclidean distances; compute average distance per individual |
| **4. HDBSCAN clustering** | Cluster individuals in normalized (F1, F2) space using HDBSCAN |
| **5. Filter clusters** | Remove clusters whose average normalized F1 > 0.5 or F2 > 0.5 |
| **6. Select parents** | With probability `elite_sparse_prob` (default: 0.1), select from clusters: |
| | - **Mutation**: pick 1 random cluster → individual with max average distance |
| | - **Crossover**: pick 2 random clusters → extract non-dominated fronts → find closest pair between fronts |

### RNG Isolation

All Elite & Sparse random decisions use a dedicated `random.Random` instance seeded with `elite_sparse_rng_seed` (default: `20260610`), ensuring reproducibility without interfering with the main RNG flow.

---

## Configuration

Parameters are defined in `configure_parameters_test.py`, which extends `configure_parameters.py`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `elite_sparse_enabled` | `True` | Enable Elite & Sparse strategy |
| `elite_max_rank` | `6` | Non-dominated rank cutoff for candidate pool |
| `hdbscan_min_cluster_size` | `5` | Minimum cluster size for HDBSCAN |
| `hdbscan_min_samples` | `None` | Core point threshold (defaults to `min_cluster_size`) |
| `hdbscan_metric` | `'euclidean'` | Distance metric for HDBSCAN |
| `elite_sparse_prob` | `0.1` | Probability of selecting parents from clusters |
| `archive_test_fronts_keep` | `4` | Number of Pareto fronts to keep from final parent population |
| `dedup_crossover_parent_ratio` | `0.1` | Ratio of top performers used as crossover parent pool |
| `elite_sparse_rng_seed` | `20260610` | Seed for Elite & Sparse RNG isolation |

Base GPHH parameters (population size, generations, crossover/mutation probabilities, etc.) are inherited from `configure_parameters.py`.

---

## Dependencies

| Package | Purpose |
|---------|---------|
| `deap` | Genetic programming framework (gp, creator, tools) |
| `hdbscan` | Density-based clustering for Elite & Sparse strategy |
| `numpy` | Numerical computation, pairwise distance matrix |
| `matplotlib` | Population distribution visualization |

Install missing dependencies:

```bash
pip install deap hdbscan numpy matplotlib
```

---

## Usage

### Direct execution

```bash
python main_gphh_metro_nsga2_test.py
```

### Programmatic usage

```python
from main_gphh_metro_nsga2_test import main

main(
    SEED=42,              # Random seed (None uses configure_parameters.seed)
    PROBLEM="CARP",       # Problem name
    PATH="data_CARP/egl-e1-A.dat",  # Instance file path
    INSTANCE_SET="egl",   # Instance set name
    INSTANCE_ELE="e1"     # Instance element name
)
```

---

## Output Structure

All outputs are written under `output_test/`:

```
output_test/
├── train/
│   ├── {instance}-{seed}-{problem}.out.stat       # Human-readable stats
│   ├── {instance}-{seed}-{problem}.stat.csv       # Per-generation statistics
│   └── plots_nor_elite_sparse/
│       └── {instance_set}/{instance_ele}/
│           ├── {problem}-{instance}-{seed}-gen_0000.png  # Population plots
│           └── ...
│           └── {problem}-{instance}-{seed}-train_vs_test_gen_XXXX.png
├── test/
│   └── {instance}-{seed}-{problem}.stat.csv       # Test results (full population)
└── pareto_fronts/
    └── {instance_set}-elite-sparse/
        └── {instance}-{seed}-elite-sparse-NSGA2-pareto.csv
```

---

## Output Files

| File | Content |
|------|---------|
| `*.out.stat` | Best individual per generation with fitness values and GP rule |
| `train/*.stat.csv` | CSV: Gen, BestFit1, BestFit2, mean/std of fitness, program size, population size, Pareto front size |
| `train/plots_nor_elite_sparse/*.png` | 2x2 population distribution plots per generation |
| `train/*train_vs_test*.png` | Train vs. test population comparison (full population + Pareto front) |
| `test/*.stat.csv` | Test evaluation: Index, TestFit1, TestFit2, ProgramSize, UniqueTerminals, Individual |
| `pareto_fronts/*-pareto.csv` | Test-based Pareto front: Individual, F1, F2 |

---

## Project Structure

| File | Purpose |
|------|---------|
| `main_gphh_metro_nsga2_test.py` | **This script** — main entry point with Elite & Sparse NSGA-II |
| `configure_parameters_test.py` | Test-specific configuration (Elite & Sparse params) |
| `configure_parameters.py` | Base GPHH configuration |
| `breeding.py` | Parent selection, crossover, mutation operations |
| `evaluate.py` | Fitness evaluation (sequential & parallel) |
| `Instance.py` | CARP instance loading and seeded sampling |
| `psetHoldCARP.py` | GP primitive set definition |
| `toolboxRegister.py` | DEAP toolbox initialization |
| `utils.py` | Utility functions (deduplication, etc.) |
| `toolBoxCARP.py` | CARP-specific toolbox operations |
| `main_gphh_metro_nsga2.py` | Standard NSGA-II (without Elite & Sparse, no archive) |
