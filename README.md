# ISE Global Market: Association Rule Mining & Sequential Pattern Mining

How do global stock markets move together with the **Istanbul Stock Exchange (ISE 100)**?
This project uses **Association Rule Mining (ARM)** and **Sequential Pattern Mining (SPM)** to find out which global market conditions happen together with ISE moves, whether yesterday's global markets can hint at ISE today, and how market sentiment travels from one region to another.

Every rule and pattern in this project must pass a **statistical test**, not only a support/confidence threshold. Every claim about prediction is also checked on a **time period that was not used for mining**.

---

## Table of Contents
1. [Research Questions](#1-research-questions)
2. [Dataset](#2-dataset)
3. [Repository Structure](#3-repository-structure)
4. [How to Run](#4-how-to-run)
5. [Methodology](#5-methodology)
6. [Key Findings](#6-key-findings)
7. [Answers to the Research Questions](#7-answers-to-the-research-questions)
8. [Output File](#8-output-file)
9. [Limitations](#9-limitations)
10. [References](#10-references)

---

## 1. Research Questions

| # | Question | Method |
|---|---|---|
| RQ1 | Which combinations of global market conditions often happen **on the same day** as ISE going up or down? | ARM (same day) |
| RQ2 | Can global market conditions **on the previous day** give a hint about the direction of ISE today? | ARM with lag (t-1) |
| RQ3 | Which **sequences** of market moves happen often over a few days, and how does sentiment **spread between regions**? | SPM + sequential spread rules |

---

## 2. Dataset

| Item | Detail |
|---|---|
| File | `data/data_akbilgic.xlsx` |
| Source | Akbilgic, Bozdogan & Balaban, *ISTANBUL STOCK EXCHANGE* dataset, UCI Machine Learning Repository |
| Period | 5 January 2009 to 22 February 2011 |
| Size | 536 trading days, no missing values, no duplicate dates |
| Values | Daily returns |

**Columns**

| Column | Index | Region |
|---|---|---|
| `ISE_TL` | ISE 100 in Turkish Lira (**main target**) | Turkey |
| `ISE_USD` | ISE 100 in US Dollar (removed from mining) | Turkey |
| `SP` | S&P 500 | America |
| `BOVESPA` | Bovespa (Brazil) | America |
| `DAX` | DAX (Germany) | Europe |
| `FTSE` | FTSE 100 (UK) | Europe |
| `EU` | MSCI Europe | Europe |
| `NIKKEI` | Nikkei 225 (Japan) | Asia |
| `EM` | MSCI Emerging Markets | Global EM |

**Data quality notes**
- The calendar follows Istanbul trading days. 21 missing weekdays are Turkish holidays, so ISE is never zero.
- Zero returns in other indices (NIKKEI 34 days, BOVESPA 29, SP 18, FTSE 14, DAX 10, EU 5) are days when that market was closed. They are **not imputed**, because there really was no price change. They become `Flat` after discretization.
- Extreme returns (for example ISE -6.2% and +6.9%) are real market events and are kept.

---

## 3. Repository Structure

```
ISE-Global-Market-ARM-SPM/
├── data/
│   └── data_akbilgic.xlsx            # raw dataset
├── notebooks/
│   └── analisis_arm_spm_ise.ipynb    # full analysis (with saved outputs and charts)
├── outputs/
│   └── tabel_ringkasan_arm_spm.csv   # summary table of all significant rules and patterns
├── docs/
│   └── presentasi_arm_spm.pptx       # project presentation slides
├── requirements.txt                  # Python libraries
├── .gitignore
└── README.md
```

> Note: the notebook text and the slides are written in Indonesian. Code, variable names, and charts can be followed without knowing the language, and this README explains everything in English.

---

## 4. How to Run

**Requirements:** Python 3.9 or newer.

```bash
# 1. Clone the repository
git clone https://github.com/zahraychana/ISE-Global-Market-ARM-SPM.git
cd ISE-Global-Market-ARM-SPM

# 2. (Optional) create a virtual environment
python -m venv .venv
# Windows: .venv\Scripts\activate
# macOS/Linux: source .venv/bin/activate

# 3. Install libraries
pip install -r requirements.txt

# 4. Open the notebook
cd notebooks
jupyter notebook analisis_arm_spm_ise.ipynb
```

Then choose **Kernel → Restart & Run All**. The notebook reads data from `../data/` and saves the summary table to `../outputs/`.

**Main libraries:** `pandas`, `numpy`, `scipy`, `matplotlib`, `seaborn`, `networkx`, `mlxtend`, `openpyxl`.

### Main settings (Section 1 of the notebook)

All parameters are in one place, so they are easy to change.

| Parameter | Value | Meaning |
|---|---|---|
| `K_SIGMA` | 0.5 | Up if return > 0.5σ, Down if return < -0.5σ, otherwise Flat |
| `ALPHA` | 0.05 | Significance level after Benjamini-Hochberg correction |
| `MIN_IMPROVEMENT` | 0.02 | A rule must beat its simpler "parent" rules by at least 2 percentage points of confidence |
| `ARM_SAME_MINSUP`, `ARM_SAME_MAXLEN` | 0.05, 4 | Same-day ARM: min support 5%, up to 3 conditions + 1 target |
| `ARM_LAG_MINSUP`, `ARM_LAG_MAXLEN` | 0.03, 3 | Lag ARM: min support 3%, up to 2 conditions + 1 target |
| `SPM_WINDOW` | 3 | Length of each sequence (trading days) |
| `SPM_MINSUP` | 0.20 | Minimum support for sequential patterns |
| `SPM_MAX_ITEMS`, `SPM_MAX_ELEMS` | 4, 3 | Maximum pattern size |
| `N_PERM` | 200 | Number of permutations for the SPM significance test |
| `TRAIN_FRAC` | 0.70 | First 70% of days for training, last 30% for testing |
| `SEED` | 42 | Random seed for reproducible results |

---

## 5. Methodology

The notebook follows this flow:

```
Setup → Load data → Data quality check → EDA → Preprocessing
      → ARM (same day) → ARM (lag t-1) → SPM → Sensitivity test → Answers
```

### 5.1 Exploratory Data Analysis (EDA)
- Descriptive statistics and return distribution for every index.
- Same-day correlation matrix and lag-1 correlation with ISE.
- Results:
  - On the same day, ISE moves most closely with **Europe** (EU 0.66, FTSE 0.62, DAX 0.60) and **EM** (0.60). It is weaker with America (SP 0.44, BOVESPA 0.43) and weakest with NIKKEI (0.26).
  - DAX, FTSE, and EU are very similar to each other (correlation 0.87 to 0.95).
  - With a 1-day lag, only **BOVESPA(t-1) (0.21)** and **SP(t-1) (0.17)** have a clear correlation with ISE today.

### 5.2 Preprocessing

**Target choice.** `ISE_TL` measures the Turkish stock market itself. `ISE_USD` mixes stock moves with the USD/TRY exchange rate and has a 0.94 correlation with `ISE_TL`, so it is removed to avoid useless rules like `ISE_USD↑ ⇒ ISE↑`.

**Discretization.** ARM and SPM need categories, so each daily return becomes one of three states:

| State | Rule |
|---|---|
| **Up** (↑) | return > K·σ |
| **Down** (↓) | return < -K·σ |
| **Flat** | in between |

σ is the standard deviation of each index, and K = 0.5. A σ-based threshold is fair across indices with different volatility (for example σ of ISE ≈ 1.6%, σ of EM ≈ 1.05%). It also removes small "noise" moves and puts holidays (return 0) into Flat.

**Trading session order.** Markets close at different times on the same calendar day. This order is used for the lag and SPM analysis:

| Session | Indices | Approx. closing time (UTC) |
|---|---|---|
| S1 Asia | NIKKEI | ~06:00 |
| S2 Europe & Turkey | ISE, DAX, FTSE, EU | ~15:30 to 16:30 |
| S3 America & Global | SP, BOVESPA, EM | ~20:00 to 21:00 |

Because SP, BOVESPA, and EM close **after** Istanbul, same-day rules with these indices show **association**, not cause and effect.

### 5.3 Association Rule Mining (ARM)
- **Transactions:** one trading day = one transaction, one-hot encoded (`SP_Up`, `DAX_Down`, ...).
- **Algorithm:** Apriori (`mlxtend`). Only rules with ISE on the right side (`conditions ⇒ ISE↑/↓`) are kept.
- **Three layers of evaluation for every rule:**
  1. **Classic metrics:** support, confidence, lift, leverage, conviction.
  2. **Significance test:** one-sided Fisher exact test on a 2×2 table, then **Benjamini-Hochberg (BH)** correction, because hundreds of rules are tested at once (controls the false discovery rate).
  3. **Improvement:** the rule's confidence minus the best confidence of its simpler parent rules (including the base rate). Rules that do not add anything are removed as redundant.
- A rule is **significant** if q-value < 0.05, lift > 1, and improvement ≥ 0.02.
- **Two ARM versions:**
  - **Same day:** global conditions on day t ⇒ ISE on day t (RQ1).
  - **Lag:** global conditions and ISE itself on day t-1 ⇒ ISE on day t (RQ2).
- **Validation:**
  - Train/test split: mine on the first 70% of days, test on the last 30%.
  - Sub-period stability: recalculate each significant rule separately in the two periods.

### 5.4 Sequential Pattern Mining (SPM)
- **Element (itemset):** one trading session. Each day is split into three ordered elements S1 → S2 → S3, so the order **within a day** is kept. Indices that close in the same session are in one itemset.
- **Item:** the direction of an index in that session (for example `DAX_Down`). Flat is not used as an item.
- **Sequence:** a sliding window of 3 trading days (9 elements), moving one day at a time.
- **Two levels:**
  1. **Index level:** 8 individual indices.
  2. **Region level:** indices grouped by region. ASIA = NIKKEI, TURKEY = ISE, EUROPE = DAX + FTSE + EU, AMERICA = SP + BOVESPA, EM = MSCI EM. Returns are divided by σ, averaged within the region, then discretized with the same K·σ rule.
- **Algorithm:** a custom **PrefixSpan** written from scratch, because `mlxtend` has no SPM and the `prefixspan` package on PyPI does not support itemsets. It supports:
  - **S-extension:** add an item as a new element after the last one.
  - **I-extension:** add an item into the last element (same session).
- **Significance with two null models** (200 permutations each, BH-corrected):

  | Null model | How data is shuffled | What it destroys | A pattern that passes means... |
  |---|---|---|---|
  | **Independent** | each market column shuffled separately | all links between markets | there is a real link between markets (may be same-day co-movement) |
  | **Temporal** | order of days shuffled, each day kept whole | only the order between days | there is a real **cross-day** structure |

  Patterns are labeled **Cross-day transmission**, **Same-day co-movement**, or **Not meaningful**.
- **Sequential spread rules between regions:** `A(s) ⇒ B(next available session)`. Confidence = P(B moves in the same direction | A moved). Tested with a one-sided binomial test + BH. Regions that close in the same session (Turkey & Europe, America & EM) are not paired, because their order cannot be known.

### 5.5 Sensitivity Test
The whole core analysis is repeated with **K = 0.25** and **K = 0.75**. A finding is called **robust** only if its direction stays the same for all K values.

---

## 6. Key Findings

Base rate: ISE goes **up on 32.1%** of days and **down on 21.8%** of days.

### 6.1 Same-day ARM (RQ1)
- **53 of 94** rules are significant (26 for ISE↑, 27 for ISE↓).
- **Single indices:** Europe gives the strongest signal. `DAX↑` or `EU↑` raises the chance of ISE↑ to 65% (lift 2.02). `DAX↓` raises the chance of ISE↓ to 52% (lift 2.39). NIKKEI is the weakest.
- **Strongest combinations** (one European index + one American index + EM, all in the same direction):

  | Rule | Days | Confidence | Lift |
  |---|---|---|---|
  | `EM↑ + EU↑ + SP↑ ⇒ ISE↑` | 51 | 81% | 2.52 |
  | `BOVESPA↓ + DAX↓ + EM↓ ⇒ ISE↓` | 34 | 74% | 3.39 |
  | `EM↑ + EU↑ ⇒ ISE↑` | 70 | 75% | — |
  | `DAX↓ + EM↓ ⇒ ISE↓` | 48 | 68% | — |

- **Asymmetry:** down rules (lift ≈ 3) are stronger than up rules (lift ≈ 2.5). Bad global news carries over to ISE more strongly than good news.
- **Stability:** all 52 significant rules from the training period still have lift > 1 in the test period (median confidence 0.665 → 0.600).

### 6.2 Lag ARM (RQ2)
- Only **6 of 102** rules are significant. Maximum lift drops from 3.39 to 1.83.

  | Rule | Confidence vs. base rate | Lift | Status |
  |---|---|---|---|
  | `BOVESPA(t-1)↓ ⇒ ISE↓` | 35.0% vs 21.9% | 1.60 | ✅ Robust (all K, both periods) |
  | `SP(t-1)↓ ⇒ ISE↓` | 33.6% vs 21.9% | 1.54 | ⚠️ Fails in the test period |
  | `NIKKEI(t-1)↓ ⇒ ISE↑` | 44.4% vs 32.0% | 1.39 | ⚠️ Stable over time, but only at K = 0.5 |
  | `BOVESPA(t-1)↑ + NIKKEI(t-1)↓ ⇒ ISE↑` | 58.6% | — | ⚠️ Only 17 days |

- Signals from America only work for the **down** direction.
- In the train/test check, SP-based rules **fail** in the test period (lift 0.52 to 0.58).

### 6.3 SPM (RQ3)
- **Most "frequent" patterns are only same-day co-movement.** 201 of 270 index patterns pass only the independent null model. Example: `{DAX↓, EU↓, FTSE↓} → {SP↓}` has support 31.6% but temporal lift 1.02. It looks like "Europe leads America", but it is the same day seen in closing-time order.
- **Real cross-day transmission flows from the West to Asia.** 26 of 33 cross-day patterns end in NIKKEI. Examples: `{DAX↑} → {SP↑} → {NIKKEI↑}` (temporal lift 1.93), `{SP↓} → {NIKKEI↓}` (1.86).
- **Strongest region chain:** `{EUROPE↓} → {AMERICA↓} → {ASIA↓}` (support 20%, temporal lift 2.24). Selling pressure "follows the sun".
- **ISE behaves as part of the European block.** Patterns `{ISE↓} → {NIKKEI↓}` (1.56) and `{ISE↑} → {NIKKEI↑}` (1.31).

**Spread rules between regions**

| Path | Direction | Confidence | Lift |
|---|---|---|---|
| EUROPE → AMERICA (same day) | Down | 56.7% | 2.76 |
| EUROPE → AMERICA (same day) | Up | 55.3% | 2.15 |
| AMERICA(t) → ASIA(t+1) | Down | 60% (base 25%) | 2.40 |
| AMERICA(t-1) → TURKEY | Down | 36.4% (base 21.9%) | 1.66 |
| ASIA(t) → TURKEY(t) | Down | 32.8% | 1.50 |
| AMERICA → TURKEY / ASIA → TURKEY | Up | — | 1.22 / 1.13 (not significant) |

- **Asymmetry:** 13 of 16 down paths are significant, but only 9 of 16 up paths. Negative sentiment spreads wider and stronger.

### 6.4 Sensitivity (K = 0.25 / 0.5 / 0.75)
- **Same-day ARM is robust:** the number of rules changes (ISE↑: 33/26/24, ISE↓: 48/27/18), but the strongest rules are always Europe + America + EM in the same direction.
- **Lag ARM is fragile for "up":** all ISE↑ lag rules disappear at K = 0.75. `BOVESPA(t-1)↓ ⇒ ISE↓` stays significant at every K, and its lift grows from 1.41 to 1.85 with a stricter threshold.
- **Down spread into Turkey is robust:** AMERICA → TURKEY Down (1.50 / 1.66 / 1.65) and ASIA → TURKEY Down (1.35 / 1.50 / 1.68).

---

## 7. Answers to the Research Questions

**RQ1: Which global conditions happen together with ISE moves on the same day?**
ISE moves with global markets when the move is **global and at the same time** across regions. The strongest mix is one European index, one American index, and EM in the same direction: `EM↑ + EU↑ + SP↑` → ISE up in 81% of cases (base 32%), and `BOVESPA↓ + DAX↓ + EM↓` → ISE down in 74% of cases (base 22%). Europe and EM are the best single indicators. NIKKEI is the weakest. Down links are stronger than up links. These rules are stable over time and across thresholds.

**RQ2: Can yesterday's global markets hint at ISE today?**
Yes, but only **weakly**. Only `BOVESPA(t-1)↓ ⇒ ISE↓` passes every check. It raises the chance of ISE falling from about 22% to about 35%, but ISE still does not fall in 65% of those cases. It is a **risk warning**, not a reliable direction predictor. Same-day co-movement (RQ1) is much stronger than any previous-day signal.

**RQ3: Which sequences happen often, and how does sentiment spread?**
Most frequent sequences are only same-day co-movement that looks ordered because of closing times. Real spread follows trading hours: **Europe → America on the same day, then America → Asia the next day**, with `{EUROPE↓} → {AMERICA↓} → {ASIA↓}` as the strongest pattern. ISE moves as part of Europe. Sentiment that reaches ISE comes from America (previous day) and Asia (same morning), but **only negative sentiment** arrives in a significant and robust way.

---

## 8. Output File

`outputs/tabel_ringkasan_arm_spm.csv` contains **362 significant rules and patterns** from all methods (column names are in Indonesian):

| Column | Meaning |
|---|---|
| `metode` | Method: same-day ARM (53), lag ARM (6), index SPM (234), region SPM (47), region spread rules (22) |
| `aturan / pola` | The rule or pattern (for example `BOVESPA↓ + DAX↓ + EM↓ ⇒ ISE↓`) |
| `n_antecedent` | Number of times the left side happened |
| `n_kejadian` | Number of times the whole rule/pattern happened |
| `n_total` | Total transactions or windows |
| `support`, `confidence`, `lift` | Standard metrics |
| `signifikan` | `True` if the rule/pattern passed the statistical test |
| `kategori SPM` | SPM label (cross-day transmission / same-day co-movement) |
| `lift_temporal` | Lift against the temporal null model (SPM only) |

For SPM, a pattern `<α1, ..., αk>` is read as a rule `α1 ... αk-1 ⇒ αk`.

---

## 9. Limitations
- Only 536 days, during the recovery after the 2008 crisis. Some lag rules are based on fewer than 50 days, so they should be tested on other periods.
- Session closing times are estimates. Istanbul closes at the same time as Europe, so the direction of influence between ISE and Europe cannot be found from daily data.
- All findings are **associations**. ARM and SPM do not prove cause and effect.
- Discretization removes information about the size of returns. The threshold was tested with only three K values.
- The temporal null model also removes volatility clustering (bad days tend to come together). Some cross-day patterns may reflect volatile periods, not pure direction transmission. The one-step spread rules (Section 5.4) are less affected by this.

---

## 10. References
- Akbilgic, O., Bozdogan, H., & Balaban, M. E. (2014). A novel Hybrid RBF Neural Networks model as a forecaster. *Statistics and Computing*, 24, 365–375. Dataset: [UCI Machine Learning Repository: ISTANBUL STOCK EXCHANGE](https://archive.ics.uci.edu/dataset/247/istanbul+stock+exchange).
- Agrawal, R., & Srikant, R. (1994). Fast algorithms for mining association rules. *VLDB*.
- Pei, J., Han, J., et al. (2001). PrefixSpan: Mining sequential patterns efficiently by prefix-projected pattern growth. *ICDE*.
- Benjamini, Y., & Hochberg, Y. (1995). Controlling the false discovery rate. *Journal of the Royal Statistical Society B*, 57(1), 289–300.
- Raschka, S. (2018). MLxtend. *Journal of Open Source Software*, 3(24), 638.
