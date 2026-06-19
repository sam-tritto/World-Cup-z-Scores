# 🏆 Grouped LOO Z-Score World Cup Analysis

A possession-chain-based **Leave-One-Out (LOO)** player evaluation framework for the FIFA World Cups of 2018 and 2022. Instead of raw box-score statistics, it measures how a team's offensive structural efficiency shifts when each player is removed from the system — then standardizes those scores within **positional peer groups** so that a world-class center back can actually compete with a world-class striker on the same leaderboard.

---

## Why This Approach?

Traditional metrics like goals, assists, and pass completion rate are heavily position-dependent. A defensive midfielder who completes three key passes in a possession chain that results in a 0.5 xG shot is doing exceptional work — but their raw stats look modest next to a striker who finishes that same chain.

This framework answers a better question:

> **How exceptional is this player compared to others playing the same role?**

It does this with two innovations:

1. **Possession Chain LOO** — credits every player involved in a dangerous possession chain with the chain's Expected Goal (xG) value, not just the shooter.
2. **Grouped Z-Scores** — standardizes ratings within positional peer cohorts (e.g., all Fullbacks, all Center Backs) rather than against all players globally.

---

## Methodology

### 1. Data Source
Event-level data from **[StatsBomb Open Data](https://github.com/statsbomb/open-data)** via the `statsbombpy` library, covering every match of the **2018 Russia** and **2022 Qatar** World Cups. Events and player minutes are cached to Parquet on first run (~3 min), so subsequent runs are instant.

### 2. LOO Engine (Offensive)
For each match:
1. Identify all possession chains that contained at least one shot.
2. Value each chain as `chain_xg = Σ shot_statsbomb_xg`.
3. Credit every player from the possession team who had any event in that chain.
4. A player's **Offensive LOO Influence Delta** = sum of `chain_xg` across all chains they participated in, normalized per 90 minutes.

### 3. LOO Engine (Defensive) — v2
The engine also measures how players **disrupt opponent possession chains**. Defensive disruption events — Interceptions, Blocks, Clearances, Ball Recoveries, and Duels — that occur during an opponent's possession chain are credited to the defending player.

### 4. Possession Chain Value (PCV) — v2
A second target metric that adds a small baseline value to every possession chain, rewarding defensive stops even when they don't prevent an outright shot:

$$\text{PCV} = \text{chain\\_xg} + 0.01 \times \text{event\\_count}$$

### 5. Tournament Aggregation
Each player's per-match metrics are summed to tournament level, and per-90 rates are re-derived from the aggregated totals. Players with fewer than a minimum minutes threshold are excluded.

### 6. Grouped Peer Standardization
For each of 6 grouping strategies, sub-metric Z-scores are computed within the group:

$$Z_i = \frac{x_i - \mu_g}{\sigma_g}$$

If a cohort has fewer than 10 players, a pooled σ (size-weighted across all cohorts) is used to prevent extreme scores from tiny groups.

| # | Key | Cohorts | Purpose |
|---|-----|---------|---------| 
| 1 | `group_global` | 1 | Baseline — no grouping |
| 2 | `group_tier` | 4 (GK/DEF/MID/FWD) | Coarse positional |
| 3 | `group_cohort` | 7 | **Recommended** |
| 4 | `group_position_raw` | 18 | Fine-grained StatsBomb positions |
| 5 | `group_tenure` | 2 | First WC vs. Returning |
| 6 | `group_age` | 3 | Young/Prime/Veteran (requires supplemental CSV) |

The 7 cohorts used in `group_cohort` are: **Forward, Winger, Central Mid, Defensive Mid, Fullback, Center Back, Goalkeeper**.

### 7. Position-Weighted Composite Z-Score
Each player's final rating is a weighted sum of their cohort-level sub-Z-scores, where the weights shift by positional role:

| Cohort | Offensive Weight | Defensive Weight | Disruption Weight |
|--------|:-:|:-:|:-:|
| Forward / Winger | 0.70 | 0.15 | 0.15 |
| Central / Defensive Mid | 0.40 | 0.30 | 0.30 |
| Fullback / Center Back | 0.15 | 0.45 | 0.40 |
| Goalkeeper | 0.00 | 1.00 | 0.00 |

Three composite scores are computed for side-by-side model comparison:

| Model | Description |
|-------|-------------|
| `composite_zscore` | Original — team-level opponent xG conceded |
| `composite_zscore_xg` | xGD LOO — player-specific defensive chain LOO xG |
| `composite_zscore_pcv` | PCV LOO — Possession Chain Value for both offensive and defensive |

### 8. Match Outcome Validation
To validate the predictive signal of the framework, player composite Z-scores are aggregated to a team level for each match using a **minutes-weighted average** of the players on the field. 

These aggregated team-level Z-scores are compared to actual tournament match outcomes (excluding draws, $N=100$) to evaluate winner prediction accuracy:
* **xGD LOO Model**: **66.00%** winner prediction accuracy
* **Original Model**: **62.00%** winner prediction accuracy
* **PCV LOO Model**: **57.00%** winner prediction accuracy

This validates that isolating individual players' disruptions of opponent possession chains (as done in the xGD LOO model) captures a more predictive signal of team success than relying on team-level defensive averages.

### 9. Next-Match Predictive Modeling & ML Comparison
We also implement a true **ex-ante prediction test** to evaluate if expanding averages of team Z-scores from preceding matches can forecast the winner of the *next* match. 

We compare Z-score differentials against traditional baselines (prior Goal Differential and xG Differential) and train machine learning models (**Logistic Regression** and **LightGBM Classifier**) using Leave-One-Tournament-Out cross-validation:
* **PCV LOO Z-Score (Rule)**: **56.58%** accuracy
* **Logistic Regression (xGD LOO)**: **56.58%** accuracy
* **Goal Diff Baseline (Rule)**: **55.26%** accuracy
* **xG Diff Baseline (Rule)**: **55.26%** accuracy
* **Original Z-Score Model (Rule)**: **55.26%** accuracy
* **LightGBM (All Features)**: **55.26%** accuracy
* **xGD LOO Z-Score Model (Rule)**: **48.68%** accuracy

Due to the extremely short history size of tournament data (1 to 6 matches) and lack of opponent-strength adjustments, ex-ante match prediction is highly volatile and struggles to outperform simple historical baselines.

---


## Project Structure

```
.
├── Grouped LOO z-Score World Cup Analysis.ipynb  # Main analysis notebook
├── data/
│   ├── events_raw.parquet     # Cached StatsBomb event data (auto-generated)
│   └── player_minutes.parquet # Cached player minutes data (auto-generated)
├── pyproject.toml             # Python dependencies (managed by uv)
└── uv.lock                    # Locked dependency versions
```

---

## Getting Started

### Prerequisites
- Python 3.13+
- [`uv`](https://docs.astral.sh/uv/) — fast Python package manager

### Install & Run

```bash
# Install dependencies
uv sync

# Launch the notebook
uv run jupyter lab
```

The first run will fetch all event data from StatsBomb's open data API and cache it locally. Subsequent runs load directly from the Parquet cache.

---

## Key Outputs

- **Top 25 Global Leaderboard** — ranked by each of the three composite models.
- **Positional Top 10 Comparisons** — side-by-side rankings for Forwards, Central Mids, and Center Backs across all three models, showing how model choice shifts who surfaces at the top.
- **Lionel Messi Spotlight** — match-by-match Z-score evolution across both the 2018 and 2022 World Cups, tracking his Offensive LOO, Defensive LOO, and Defensive Disruptions Z-scores relative to the Forward/Winger cohort average.
- **Match Outcome Validation** — aggregated team-level Z-score differential comparisons against actual match scores and winners to validate ecological framework validity, demonstrating **66.00% predictive accuracy** for the player-specific xGD LOO model.
- **Next-Match Prediction & ML Comparison** — ex-ante sports forecasting evaluating expanding average Z-score differentials against traditional baselines, Logistic Regression, and a LightGBM Classifier, analyzing tournament-specific small sample predictability and simulating predictions for the 2018 and 2022 Finals.
- **Player Value Evolution** — dual-panel visualization showing Lionel Messi's match-by-match sub-metric Z-scores and comparing cumulative Z-score sums for Messi, Kylian Mbappé, and Antoine Griezmann in the 2022 World Cup, illustrating individual tournament impact.


---

## Interpreting Z-Scores

| Z-Score | Meaning |
|---------|---------|
| `> +2.0` | Exceptional — top ~2% of the positional peer group |
| `+1.0 to +2.0` | Above average — top ~16% |
| `0.0` | Exactly average for the cohort |
| `-1.0 to 0.0` | Below average |
| `< -2.0` | Well below average — bottom ~2% |

A Z-score of `0.0` does **not** mean the player is bad — it means they are average for their positional role, which for a World Cup squad is already elite by any global standard.

---

## Data & Limitations

- **Source:** StatsBomb Open Data (free, research-grade). Commercial StatsBomb 360 data (not used here) would add more spatial context.
- **Scope:** FIFA World Cup 2018 (Russia) and 2022 (Qatar). Club-level or other tournament data is not included.
- **Small samples:** Players with limited minutes are filtered out. Rare positional cohorts use pooled σ to guard against extreme Z-scores.
- **2026 stub:** A CSV ingestion hook is in place for when StatsBomb releases 2026 World Cup data.

---

## Roadmap

| Iteration | Enhancement |
|-----------|-------------|
| **v3** | Integrate 2026 World Cup live data via the CSV stub |
| **v3** | Build an interactive dashboard (Streamlit / Panel) for exploring rankings by group, tournament, and position |
| **v4** | Apply Bayesian shrinkage for small-sample groups instead of pooled-σ |
| **v4** | Implement bootstrap confidence intervals on Z-Scores |
