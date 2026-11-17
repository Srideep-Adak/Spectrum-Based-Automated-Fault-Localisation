# Spectrum-Based Automated Fault Localisation

A machine-learning-assisted fault localisation system that ranks
suspicious Java source-code lines using test coverage information. The
project combines **Spectrum-Based Fault Localisation (SBFL)** with a
**LightGBM classifier** and **isotonic regression calibration** to help
developers narrow down where a software defect is likely to be located.

> **Implementation note:** The current implementation described in this
> repository's project report is an SBFL + LightGBM approach. Despite
> the project title, an LLM is not part of the implemented prediction
> pipeline.

## Overview

A failing test tells a developer that something is wrong, but usually
does not identify the faulty line. This project analyses line coverage
from failing and passing tests, calculates suspiciousness scores, and
uses a trained model to re-rank the most suspicious candidates.

The system also calibrates model outputs into interpretable
probabilities and provides a Streamlit interface for exploring and
downloading predictions.

## Key Features

-   **Five SBFL formulas:** Ochiai, Tarantula, DStar, Jaccard and Op2.
-   **Machine-learning re-ranking:** LightGBM classifier re-ranks the
    top 100 candidate lines selected using Ochiai (with DStar
    tie-breaking).
-   **28 engineered features:** coverage counts, formula scores and
    ranks, tie-group information, and source-file position/context
    features.
-   **Probability calibration:** isotonic regression converts model
    scores into calibrated line-level probabilities and bug-level top-k
    probabilities.
-   **Project-independent evaluation:** leave-one-project-out validation
    across 17 Defects4J projects.
-   **Prediction interface:** Streamlit app for inspecting suspicious
    lines, probabilities, confidence levels and downloadable JSON
    results.
-   **Command-line prediction support:** prediction modules are also
    available for programmatic use.

## How It Works

``` text
Defects4J buggy version
        |
        v
Checkout, compile and run tests
        |
        v
Per-test line coverage
(failing tests + sampled passing tests)
        |
        v
Coverage caching and faulty-line ground truth
        |
        v
SBFL suspiciousness scoring
Ochiai | Tarantula | DStar | Jaccard | Op2
        |
        v
Select top 100 candidate lines
        |
        v
Feature engineering (28 features per line)
        |
        v
LightGBM classifier
        |
        v
Isotonic probability calibration
        |
        v
Ranked suspicious lines and probabilities
        |
        v
Streamlit interface / CLI
```

### 1. Dataset and coverage collection

The system was evaluated using real Java bugs from **Defects4J**. Buggy
versions are checked out and compiled, then selected failing and passing
tests are executed individually to collect line coverage.

The reported experiment processed 346 bug entries across 17 projects. Of
these, 331 had per-test coverage, and 293 had the faulty line within the
inspected search space and were used for evaluation.

### 2. SBFL scoring

For each line, the system calculates suspiciousness using five formulas:

-   Ochiai
-   Tarantula
-   DStar (with exponent 2)
-   Jaccard
-   Op2

The top 100 candidates are selected using Ochiai scores, with DStar used
to break ties.

### 3. Feature engineering

Each candidate line is represented by 28 features, grouped broadly into:

-   **Spectrum features:** execution counts from failing and passing
    tests and related ratios.
-   **Formula features:** suspiciousness scores, normalised scores and
    score ranks from the five formulas.
-   **Context features:** tie-group size, search-space size, file
    length, relative line position, and suspiciousness of nearby lines.

### 4. LightGBM classification

A LightGBM gradient-boosted decision-tree classifier predicts whether
each candidate line is faulty.

Reported model settings:

  Parameter                 Value
  ----------------------- -------
  Estimator count             200
  Learning rate              0.05
  Number of leaves              7
  Minimum child samples        10
  Row subsampling             0.8
  Column subsampling          0.8
  Random state                  0

### 5. Probability calibration

Isotonic regression calibrates the classifier's raw scores into
line-level probabilities. Additional calibration is used to estimate the
probability that the fault appears within the top 1, 3 or 5 ranked
candidates.

## Technology Stack

  Area                                Technologies
  ----------------------------------- ------------------------
  Programming language                Python 3
  Data processing                     NumPy, pandas
  Machine learning                    LightGBM, scikit-learn
  Model persistence                   joblib
  Visualisation                       Matplotlib
  Web interface                       Streamlit
  Benchmark and coverage collection   Defects4J, Cobertura
  Development environment used        Ubuntu on WSL

## Evaluation

The reported evaluation uses **leave-one-project-out validation**. In
each fold, the model is trained on bugs from 16 projects and evaluated
on the remaining project. This tests whether the model generalises to a
project it has not seen during training.

### Results on 293 Defects4J bugs

  --------------------------------------------------------------------------------------------------
  Method               Top-1       Top-3       Top-5      Top-10         MFR         MAR      EXAM %
  -------------- ----------- ----------- ----------- ----------- ----------- ----------- -----------
  Ochiai                9.2%       33.1%       44.4%       60.4%       25.47       62.64       12.80

  Tarantula             8.2%       31.7%       43.0%       59.7%       25.59       62.49       12.99

  DStar                 9.6%       32.4%       43.7%       59.0%       25.35       62.50       12.81

  Jaccard               8.5%       32.8%       44.7%       60.4%       25.55       62.67       12.93

  Op2                   9.6%       31.1%       42.7%       57.7%       26.19       67.86       12.81

  **LightGBM**     **15.7%**   **37.5%**   **45.1%**   **60.4%**   **24.27**   **60.67**   **12.37**
  --------------------------------------------------------------------------------------------------

The LightGBM model improves Top-1 accuracy from **9.2% to 15.7%**
compared with Ochiai. Top-3 improves from **33.1% to 37.5%**, and Top-5
from **44.4% to 45.1%**. Top-10 remains at 60.4%, partly because the
learning stage re-ranks the top 100 SBFL candidates rather than
introducing candidates excluded by the first stage.

**Metrics:** Top-N is the percentage of bugs whose first faulty line
appears within the first N ranked lines. MFR is Mean First Rank, MAR is
Mean Average Rank, and EXAM is the average proportion of the search
space that must be examined to find a fault. Lower MFR, MAR and EXAM
values indicate better ranking performance.

## Installation and Usage

### Prerequisites

-   Python 3
-   pip
-   The project source code
-   The dependencies listed in `requirements.txt`

Defects4J and its Java/Perl-related prerequisites are required if you
intend to reproduce the benchmark coverage-collection pipeline. They are
not needed merely to launch the Streamlit interface with an already
prepared model and input data.

### 1. Clone the repository

``` bash
git clone https://github.com/Srideep-Adak/LLM-Based-Automated-Fault-Localisation.git
cd LLM-Based-Automated-Fault-Localisation
```

### 2. Create and activate a virtual environment

``` bash
python -m venv .venv
```

On Linux / macOS:

``` bash
source .venv/bin/activate
```

On Windows PowerShell:

``` powershell
.venv\Scripts\Activate.ps1
```

### 3. Install dependencies

``` bash
pip install -r requirements.txt
```

### 4. Run the Streamlit application

``` bash
streamlit run app.py
```

Open the local URL printed by Streamlit in your browser.

### Using the application

The reported Streamlit interface provides two main tabs:

1.  **Results overview:** load result files and inspect aggregate
    Top-1/3/5/10 metrics, mean first rank and per-bug results.
2.  **Fault prediction:** provide a coverage JSON file or select a bug
    from the available cache. Inspect the predicted file and line,
    calibrated probability, confidence level, top-k probabilities and
    the most suspicious lines. Predictions can be downloaded as JSON.

The repository also includes prediction modules for command-line or
programmatic use. Refer to the scripts' help output and documentation
for the exact arguments supported by the checked-out version.

## Project Modules

The implementation report describes the following main scripts and
modules:

  -----------------------------------------------------------------------
  Module / script                     Responsibility
  ----------------------------------- -----------------------------------
  `run_defects4j_all.py`              Runs the benchmark data-collection
                                      workflow across bugs.

  `fl_ochiai.py`                      Collects coverage and creates
                                      initial fault-localisation results.

  `preprocess_d4j_full.py`            Preprocesses benchmark and patch
                                      data.

  `check_direction.py`                Checks the detected direction of
                                      developer patches.

  `fl_common.py`                      Shared coverage processing, SBFL
                                      scoring and evaluation utilities.

  `evaluate.py`                       Compares the five SBFL formulas.

  `fl_features.py`                    Builds the feature table for
                                      candidate lines.

  `train_model.py`                    Trains and validates the LightGBM
                                      model and fits calibration models.

  `fl_input.py`                       Accepts and normalises supported
                                      coverage input formats.

  `fl_predict.py`                     Produces ranked fault predictions
                                      and calibrated probabilities.

  `predict_all.py`                    Runs prediction and reports results
                                      across bugs.

  `app.py`                            Provides the Streamlit user
                                      interface.
  -----------------------------------------------------------------------

File names above are taken from the project report; consult the
repository for the exact current directory layout and available
command-line arguments.

## Limitations

-   The search space in the reported evaluation is restricted to classes
    modified by the developer's fix, rather than the entire codebase.
-   The approach depends on usable test coverage and cannot localise
    faults when the required test information is unavailable.
-   The model re-ranks the SBFL-selected candidates; a faulty line
    omitted from that candidate set cannot be recovered by the
    re-ranking stage.
-   The reported benchmark covers Java projects from Defects4J.
    Performance on other languages and unrelated real-world codebases
    has not been established.
-   The current implementation is coverage- and feature-based; deeper
    source-code semantics, stack traces and error logs are potential
    extensions.

## Future Improvements

-   Expand the search space from modified classes to the full program.
-   Evaluate on additional projects, larger datasets and other
    programming languages.
-   Incorporate program semantics, source-code context, stack traces and
    error logs.
-   Explore integration with IDEs such as VS Code and IntelliJ IDEA.
-   Investigate LLM-based explanations of likely root causes and
    suggested fixes.
-   Use developer feedback to improve future model versions.


## References

-   [Defects4J](https://github.com/rjust/defects4j)
-   Abreu et al. --- Ochiai spectrum-based fault localisation.
-   Jones and Harrold --- Tarantula spectrum-based fault localisation.
-   Wong et al. --- DStar suspiciousness metric.
-   Research works reviewed in the project report on SBFL,
    machine-learning-based fault localisation and automated debugging.

## License MIT
