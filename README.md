Knee MRI model evaluation

Two models can have the exact same validation score to three decimals and still act differently on new exams. I wanted to see how much that actually happens, so this notebook digs into it for knee MRI abnormality detection. It uses two things: a real record of rounded scores, and a simulation where I control the model parts and the labels.

The task

The data behind this is a set of knee MRI studies from a lot of different sites, scanners and protocols around the world. Each study is one scanning session made up of several MRI series saved as DICOM slices, usually around 30 slices per series. The series come in sagittal, coronal or axial planes, and some are fluid-sensitive or fat-suppressed.

A model has to give a probability for each of 12 findings per study: ACL, MCL, medial meniscus, lateral meniscus, medial OA, lateral OA, patellofemoral OA, effusion, synovitis, Baker's cyst, bone contusion and fracture. One study can have several of these at once.

Only a small part of the training studies come with labels. The rest just have the free-text radiology report, and not always in English, so labels for those have to be pulled out of the report. At test time there's no report at all, only the scans, for about 1,300 studies. How common each finding is also isn't guaranteed to match between training, validation and the final test set, which is one more reason a validation score can move once you hit new data.

This notebook doesn't do any of that modeling. It doesn't load scans, read reports or train anything. It's only about evaluating the predictions once you have them, and everything runs on embedded scores and synthetic studies.

The metric is macro ROC AUC. For each finding, AUC is basically how often a study that has the finding gets a higher score than one that doesn't (ties count as half). Then you average the 12 AUCs, so a rare finding counts as much as a common one. Picking a cutoff to actually call a diagnosis is a separate problem and isn't covered here.

What I found

Same score, different results. There were 1,638 records with a validation AUC of 0.958. On holdout, 1,239 of them (75.6%) landed at 0.942, but in total they spread across 25 different values from 0.899 to 0.946. So the same three-decimal score can hide a wide spread. The median drop from validation to holdout was 0.016, and it was the same inside and outside the 0.958 group.

Small differences are shaky. In the main simulation, switching one component moved validation and holdout in the same direction 87% of the time. When the validation change was 0.001 or less, that dropped to 80%. The 0.001 cutoff is on full-precision scores, so two scores inside it can still round to different numbers.

Rerunning isn't the same as new data. A single run's validation AUC moved with an SD of 0.0030 across new sets of studies. Two reruns of the same setup on the same studies only differed by an SD of 0.000075. The ordering inside a narrow score window was weak too. Runs within 0.001 of each other on validation had a rank correlation of 0.39 with their holdout order, and reruns of the same setup were basically random at 0.01.

Keeping a second candidate. If you keep two candidates and take the better holdout result, a plain rerun of the same model only adds 0.000016 AUC on average. Taking the top two by validation adds 0.000036, but 88% of the time those two are just reruns of the same setup anyway. Pairing the top run with its closest one-component neighbor adds 0.000082. It came out ahead 17% of the time and was almost never worse. That's looking back after the fact. If you were actually deploying, you'd still need a rule for picking one.

How it works

1. The historical scores. There's an embedded table with the counts for all 6,425 paired validation and holdout records. The pairs were matched using an ID that was removed from the source. Each holdout value is the best of up to two selected runs, so the two scores in a pair might come from different setups. The notebook rebuilds the distribution from that table and works out the median score differences at the precision they were recorded.

2. The simulation. Each trial makes 1,300 fake exams, the same size as the real test set, split 390 for validation and 910 for holdout. A shared baseline scores all 12 findings. On top of that there are three optional components you can switch on or off, which gives 8 configurations, and each one gets 4 noisy runs. Every configuration is scored on the same studies within a trial.

3. Sensitivity. I ran the whole thing 600 times for each of three assumed component effect sizes (SD of 0.0005, 0.001 and 0.002 AUC). For each setting it reports how often validation and holdout agree, how much scores bounce around, rank correlations, and how much a pair of candidates covers on holdout. The pattern holds across all three. Small-change agreement goes from 74% at the smallest effect to 80% at the largest, and the neighbor pair beats a plain rerun every time (0.000105, 0.000082 and 0.000055 versus 0.000016).

A few things to keep in mind. The findings are independent in the simulation, and the prevalences, baseline AUCs and component effects are all my assumptions. In real scans findings show up together, which could change how noisy AUC is. None of this tells you how a model does in an actual clinic either. That would need patient-level evaluation.

Running it

Tested with Python 3.12, NumPy 2.3.5 and SciPy 1.17.0. From the folder with knee_mri_evaluation.ipynb:

python -m pip install numpy==2.3.5 scipy==1.17.0 jupyterlab
jupyter lab knee_mri_evaluation.ipynb

Then go to Run → Run All Cells. You don't need to download any data or have a GPU. For a quick run, lower N_TRIALS in the simulation settings cell. The saved outputs use the default settings and fixed seeds.

The last cell writes evaluation_summary.json to your working directory with the observed counts, the simulation settings and the metrics for each effect size. Running it again overwrites the old file.
