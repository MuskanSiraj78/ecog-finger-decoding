# Decoding Finger Movement from Human ECoG

An end-to-end pipeline that decodes **finger flexion vs. extension** from electrocorticography (ECoG) recorded from a 16×16 subdural electrode grid over human motor cortex.

Built as coursework for a graduate seminar on Human-Computer and Brain-Computer Interfaces. Everything from raw signal preprocessing to classifier validation is implemented and documented across seven notebooks.

---

## Results

| Metric | Value |
| :--- | ---: |
| Classification accuracy (SVM, 10-fold CV) | **75.8%** |
| Area under ROC curve | **0.78** |
| Chance level (400-run permutation test) | 46.6% |
| Statistical significance | p < 0.01 |
| Features used | 52 (down from 310) |

Chance is **not** 50% here — the classes are unequal (163 vs. 151 trials), so always predicting the larger class already yields 51.9%. The permutation test estimates the true baseline empirically, and no shuffled run exceeded 53.8%.

### Classifier comparison

Same features (310), same cross-validation folds:

| Classifier | Accuracy |
| :--- | ---: |
| Linear SVM | **72.0%** |
| Naive Bayes | 69.4% |
| LDA | 60.2% |

LDA underperforms for a structural reason rather than a conceptual one: with 310 features and only 270 training trials, the within-class scatter matrix is rank-deficient and cannot be inverted stably. Reducing to 14 features lifts LDA to 68.8%, above Bayes.

---

## What I found

**Hyperparameter tuning was leaking test data.** The provided SVM routine selected its cost parameter `C` by evaluating each candidate on the *test* fold, then reported accuracy on that same fold. This inflates the result. Re-running it as a proper nested cross-validation — selecting `C` on an inner split of the training data only — gave **72.9%** where the original procedure reported **77.4%**. Two symptoms flagged the problem before the nested run confirmed it: the selected `C` varied by a factor of ~4000 across folds, and the gain vanished under honest validation.

**Fewer features performed better.** Cutting the feature space from 310 to 52 (4 channels, 40–160 Hz sampled every 10 Hz) *raised* cross-validated accuracy from 72.0% to 75.8%. Adjacent frequency bins are highly redundant, so most of the original feature set contributed noise rather than signal — and the smaller model trains an order of magnitude faster.

**Three independent methods converged on the same feature.** Electrode 17 at ~157 Hz emerged as the strongest predictor via a two-sample t-test (|t| = 7.15), via greedy feature selection, and via the linear SVM's own weight vector — averaged over 30 cross-validation folds. These methods share no assumptions, which makes the localisation considerably more credible than any single result.

**A numerical bug in the spectral estimation.** The multitaper routine approximated DPSS tapers by computing them at 1/100 length and interpolating — a reasonable shortcut for continuous recordings of ~500,000 samples. After epoching into 254-sample trials it requested `dpss(2, 3, 5)` and crashed. For inputs that short, `dpss` is exact and cheap, so the correct fix was to skip the shortcut below a length threshold.

---

## Pipeline

| Stage | Notebook | What happens |
| :--- | :--- | :--- |
| Behavioural data | `Session_01` | Identify gestures from sensor-glove traces; hand-label movement onsets |
| Artifact rejection | `Session_02` | Reject bad channels by spectrum and variance; common average reference |
| Filtering | `Session_03` | Butterworth bandpass; epoching; multitaper spectral estimation |
| Dimensionality | `Session_04` | Per-frequency z-scoring; PCA from first principles |
| Feature selection | `Session_05` | Two-sample t-statistics across the channel × frequency grid |
| Classification | `Session_06` | Bayes, LDA and SVM under 10-fold cross-validation |
| Validation | `Session_07` | ROC/AUC, weight-vector analysis, permutation testing |

**Preprocessing details.** Common average referencing removed 76% of total variance and drove mean inter-channel correlation from +0.76 to −0.03 — consistent with the theoretical −1/(N−1) = −0.029 for N = 36 channels, a useful sanity check that the re-referencing behaved as intended.

**Feature construction.** Trials are 0.25 s (254 samples at 1017.25 Hz). Power is estimated with multitaper spectra (TW = 3, K = 5), z-scored per frequency across all channels and trials, then reshaped to `nTrials × (nChannels · nFrequencies)`.

---

## Data and dependencies

**Neither is included.** The recordings are human intracranial data provided by the seminar and are not mine to redistribute; anatomical imagery is excluded for the same reason, and notebook outputs have been stripped. Several helper modules referenced by the notebooks (spectral estimation, the classifiers, plotting utilities) are seminar-provided material and are likewise left out.

What is published here is the analysis itself: the pipeline, the design decisions, and the written interpretation at each stage. The notebooks are meant to be **read** rather than executed.

---

## Repository layout

```
notebooks/    Session_01 … Session_07, one per pipeline stage
```

## Stack

Python · NumPy · SciPy · scikit-learn · Matplotlib
