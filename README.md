# Knee MRI model evaluation

Two models can show the same validation score to three decimal places and still behave differently on new examinations. This notebook investigates that problem for knee MRI abnormality detection. It combines a historical record of rounded scores with a simulation where the model components and study labels are known.

## The imaging task

Each knee MRI examination is paired with its radiology report. The underlying task is to assign one confidence score to each of twelve findings on every examination. A scan can have several findings, including ligament tears, meniscal damage, cartilage loss, or fractures. The notebook focuses on evaluating those predictions. It runs with embedded scores and synthetic studies, without loading scans, processing reports, or training a detector.

The metric is **macro-averaged ROC AUC**. For each finding, AUC measures how often a study with the finding receives a higher score than a study without it, with half credit for a tie. Averaging the twelve AUCs gives each finding equal weight, even when some findings are rare. Choosing a cutoff for diagnosis is a separate decision.

## What the analysis found

| Question | Result | How to read it |
| --- | --- | --- |
| What happened to records with validation AUC **0.958**? | Of 1,638 records, 1,239 (75.6%) had holdout AUC **0.942**. Holdout values ranged from **0.899 to 0.946**. | The same three-decimal score can conceal a range of holdout outcomes. |
| When one component changed, how often did a small difference carry over? | In the primary simulation, validation and holdout changes had the same sign in **80%** of comparisons with an absolute validation change of at most 0.001. | The cutoff uses full-precision scores. It can include two scores that round to different values. |
| What did a second candidate add? | The mean retrospective gain in holdout AUC was **0.000016** for a repeat run and **0.000082** for a run differing by one component. | Gain counts the better holdout result of the two. It measures what the pair covered. A deployed model would still need a selection rule. |

## How it works

1. **Examine the historical scores.** An embedded table preserves the frequencies of all 6,425 paired validation and holdout records. Each pair was matched by an identifier that the source removed. Its holdout value was the best of up to two selected runs, so the scores in a pair may describe different configurations. The notebook reconstructs the distribution and computes median score differences from the recorded precision.
2. **Generate controlled comparisons.** Each trial creates 1,300 synthetic examinations and assigns 390 to validation and 910 to holdout. A shared baseline produces scores for twelve findings. Switching three optional components on or off gives eight configurations, with four noisy runs of each. All configurations are evaluated on the same studies within a trial.
3. **Check sensitivity.** The notebook repeats the experiment 600 times for each of three assumed component-effect standard deviations (0.0005, 0.001, and 0.002 AUC). It reports split-to-split agreement, score variability, rank correlations, and the holdout coverage of candidate pairs. The findings are independent in the simulation. Their prevalences, baseline AUCs, and component effects are assumptions. Dependence between findings in real scans could change how much AUC varies. Measuring performance in clinical practice would require patient-level evaluation.

## Run the notebook

From the folder containing `knee_mri_evaluation.ipynb`, run the following in a Python environment. The analysis was tested with Python 3.12, NumPy 2.3.5, and SciPy 1.17.0.

```bash
python -m pip install numpy==2.3.5 scipy==1.17.0 jupyterlab
jupyter lab knee_mri_evaluation.ipynb
```

In JupyterLab, choose **Run → Run All Cells**. No separate dataset download or GPU is required. For a quicker exploratory run, reduce `N_TRIALS` in the simulation settings cell. The saved notebook outputs use the default settings and fixed random seeds.

The notebook displays the historical frequency table and simulation comparisons. Its last cell writes `evaluation_summary.json` in the working directory with the observed counts, simulation settings, and metrics by effect size. Running the cell again replaces the previous summary.
