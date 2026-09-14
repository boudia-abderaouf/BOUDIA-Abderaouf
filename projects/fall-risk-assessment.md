# Fall Risk Assessment Using Gait Analysis and Wearable Sensors

**CDTA / École Nationale Polytechnique, Algiers · 2022 – 2023 · Research**
**Publication status:** manuscript submitted to an IEEE journal — under review.
**Authorship:** W. Dib, **A. Boudia**, F. Boukhedimi, O. Kerdjidj — *second author*.
**Role:** signal processing, gait-cycle segmentation, feature extraction and the
machine-learning benchmark.

📄 [Read the manuscript](../articles/Fall_Risk_Assessment_Using_Gait_Analysis.pdf)

---

## Problem

Falls are a leading cause of injury and death among older adults. Identifying who is at
risk *before* a fall happens is what enables prevention, but the clinical instruments
that do it — Timed Up and Go, Berg Balance Scale, supervised gait assessment — are
time-consuming, resource-heavy and observe a person for minutes in a controlled room.

Wearable inertial sensors can observe gait continuously instead. The open question the
work addresses: **how much of that clinical discrimination survives a single sensor and
a classifier light enough to run in real time?**

## Context

Most of the published literature buys accuracy with instrumentation — five, ten sensors
across the body, or a deep network that cannot run on an embedded device. Both choices
defeat the purpose: an obtrusive rig is not worn daily, and a model that needs a GPU is
not a wearable.

The study therefore constrains itself to a **single lower-back accelerometer** and
evaluates the accuracy/cost trade-off explicitly rather than reporting accuracy alone.

## Dataset

The **Long Term Movement Monitoring (LTMM) database** from PhysioNet — a public dataset
of tri-axial accelerometer and gyroscope signals recorded from the lower back of **71
community-dwelling older adults** (mean age 78.4 ± 4.7, range 65–87), worn on a belt.

Participants are labelled fallers or non-fallers from self-reported falls over the
preceding year, with two or more falls defining a faller. Only the laboratory-recorded
portion was used, for measurement standardisation and reproducibility.

## Approach

```
raw accelerometer signal
   → gait-cycle segmentation (4 algorithms compared)
   → time, frequency and spatio-temporal feature extraction per cycle
   → PCA (99 % retained variance)
   → classifier benchmark
```

**Segmentation.** Everything downstream depends on correctly locating two events in each
gait cycle — Initial Contact (heel strike) and Final Contact (toe-off). Four published
algorithms were implemented and compared:

| Algorithm | Principle |
|---|---|
| Gonzalez | Zero-crossings in the filtered anteroposterior signal, then heuristic peak association |
| Ghersi | Models acceleration as a modified triangle wave via dynamic time warping |
| CWT | Detrend, low-pass, trapezoidal integration, then continuous wavelet differentiation with a first-order Gaussian wavelet |
| Shin | Sliding-window summation for denoising, differential acceleration to remove gravity, then zero-crossing detection |

**Features.** Per gait cycle: statistical descriptors (mean, signal energy, standard
deviation, variance, skewness, min/max) and spatio-temporal gait parameters (cadence,
step and stride duration, single- and double-support duration, stance and swing phase
duration, symmetry, and step length via the Weinberg estimator).

**Dimensionality reduction.** PCA retaining ≥ 99 % of total variance — reached with 10
principal components.

**Classification.** SVM, decision tree, decision tree + AdaBoost, random forest, random
forest + AdaBoost, KNN, Gaussian naive Bayes and a deep neural network, each tuned by
`GridSearchCV`, on a 70 / 10 / 20 train / validation / test split (2,112 / 302 / 604
gait cycles).

## Results

Reported in the manuscript, on the held-out test set:

| Model | Test accuracy | ROC-AUC | Execution time |
|---|---|---|---|
| **KNN (k = 3)** | **89.7 %** | 90 % | **0.008 s** |
| DNN | 89.4 % | 89 % | 29 s |
| SVM (RBF) | 87.6 % | 88 % | 0.25 s |
| DT + AdaBoost | 85.8 % | 86 % | 1.8 s |
| RF / RF + AdaBoost | 84.9 % | 85 % | ~1.1 s |
| Decision tree | 77.2 % | 78 % | 0.05 s |

KNN and the DNN are statistically comparable in accuracy — and separated by more than
three orders of magnitude in execution time. That is the paper's actual finding: on this
task, the deep model buys nothing it can keep once you care about deployment.

For fall risk specifically, recall on the faller class is the metric that matters —
a missed faller is the costly error. SVM, KNN and the DNN all reach ≈ 91 % F1 on the
fall class. The Gonzalez segmentation algorithm gave the best balance of discrimination
and simplicity.

## My contribution

- Implementation and comparative evaluation of the four gait-cycle segmentation
  algorithms on the LTMM signals.
- Signal pre-processing from a single inertial measurement unit (detrending, filtering,
  gravity removal).
- Per-cycle feature extraction — statistical, frequency-domain and spatio-temporal gait
  parameters.
- The classifier benchmark: dimensionality reduction, hyperparameter search and the
  accuracy/latency evaluation.

**Earlier related work (2022).** In a preceding internship at the same lab I built a
**1D CNN** for human-activity recognition and fall detection, converted it to
**TensorFlow Lite** and deployed it on an **Arduino Nano 33 BLE** with onboard inertial
sensors — the embedded counterpart to the question this study answers offline.

## Technologies

Python · scikit-learn · NumPy · SciPy · pandas · Matplotlib · TensorFlow / Keras ·
TensorFlow Lite · MATLAB (signal prototyping) · C/C++ and Arduino (embedded deployment) ·
PhysioNet LTMM dataset

## What I learned

- **The segmentation step sets the ceiling.** Four algorithms on the same signal produce
  measurably different gait parameters; choosing one is a modeling decision, not
  preprocessing hygiene.
- **Report the cost next to the score.** The most useful column in the results table is
  execution time, and it is the one most papers omit.
- **Class-conditional metrics, not accuracy.** In fall risk, a false negative and a false
  positive are not the same mistake.

## Links

- 📄 [Manuscript (PDF)](../articles/Fall_Risk_Assessment_Using_Gait_Analysis.pdf)
- Dataset: Long Term Movement Monitoring (LTMM) database, PhysioNet — publicly available.
- Implementation code is not currently published.
