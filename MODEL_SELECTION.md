# Model selection

## The question
The model predicts which customers are likely to leave (churn), so the company can contact them with an offer.
A missed churner (false negative) is a lost customer; a false alarm only costs one unnecessary offer.

## Candidates
All three use `class_weight="balanced"` because only 25% of customers churn: without it the models miss many churners.

| Candidate | Why I tried it | AUC | F1 | F1 per slice (lowest) | p95 latency (ms) | Size (KB) | MLflow run ID |
|---|---|---|---|---|---|---|---|
| logreg_balanced | Simple linear baseline: fast, tiny, easy to explain | 0.798 | 0.589 | 0.333 (two-year) | 6.2 | 4 | c9266ed7 |
| random_forest_balanced | Many trees (200, depth 6): catches non-linear patterns | 0.773 | 0.566 | 0.375 (two-year) | 52.3 | 1579 | 83288ca7 |
| hist_gboost_balanced | Boosted trees (depth 3): usually strong on tabular data | 0.769 | 0.565 | 0.400 (two-year) | 20.8 | 101 | 7f42fd1d |

## The champion
**logreg_balanced.** It has the best F1 overall (0.589) and the best AUC (0.798), it is the fastest (6.2 ms, 3x faster
than boosting and 8x faster than the forest) and the smallest (4 KB vs 1.6 MB for the forest).
What I accepted losing: it has the lowest F1 on two-year customers (0.333 vs 0.400 for boosting). The test set has
only 7 two-year customers who churned, so this gap is about one customer, while logreg is better on the biggest
group (month-to-month, 140 customers: 0.626 vs 0.587).

## My promotion rules
- Latency budget (p95): 50 ms, because the API should answer well under 100 ms and the model gets half of that.
  GitHub runners are slower than my laptop, so 50 ms leaves a safety margin. The random forest (52 ms) is outside it.
- Minimum gain to replace the champion (F1 points): 0.01, because the test set is small (240 customers, 60 churners),
  so a smaller gain could be just noise. A new model must be clearly better before we replace a working one.

## What I would check next
The two-year slice is very small (7 churners in the test set), so its F1 is unreliable. I would collect more data
or use cross-validation before trusting slice numbers. Also, the data can change: churn_v2.csv has a different
hash, so the champion should be re-evaluated on new data instead of compared only with its old F1.
