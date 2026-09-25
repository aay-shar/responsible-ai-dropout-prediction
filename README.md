# Responsible AI: Student Dropout Prediction

Project from Responsible AI, MSc Applied Data Science, Utrecht University, winter 2025 to 2026. A dropout risk model built alongside the governance documentation that a high-risk AI system would need under the EU AI Act.

The point of the project is a model can be accurate overall and still fail the students it matters most for, and that this only becomes visible if you measure it.

## The model

Data is the UCI [Predict Students' Dropout and Academic Success](https://archive.ics.uci.edu/dataset/697/predict+students+dropout+and+academic+success) dataset, 4,424 students from a Portuguese polytechnic. It downloads at runtime through `ucimlrepo`, so nothing needs to be stored here.

Three preprocessing decisions shaped the problem before any model was fitted:

- **Second semester features were dropped.** They leak outcome information and would make the model useless for early intervention, which is the entire use case.
- **Nationality, gender and age were removed** as direct sensitive attributes.
- **Enrolled students were excluded**, reducing the target to a binary dropout versus graduate outcome.

Logistic regression, a decision tree, and a random forest were compared under 5-fold stratified cross-validation, ranked on dropout recall first rather than accuracy. Missing an at-risk student costs more than flagging a student who was never going to drop out, so false negatives were treated as the expensive error.

| Model | Accuracy | Dropout recall | Macro F1 |
|---|---|---|---|
| **Logistic regression** | **0.882** | **0.839** | **0.876** |
| Random forest | 0.881 | 0.783 | 0.872 |
| Decision tree | 0.856 | 0.777 | 0.846 |

Logistic regression won on recall while matching the ensemble on accuracy, and its coefficients are directly readable, which matters for a system a counsellor has to explain to a student.

The strongest predictors were financial rather than academic: unpaid tuition and debtor status dominated. Portuguese law bars students with unpaid tuition from sitting exams, so the model is partly detecting a legal mechanism rather than a behavioural one. That distinction is exactly what an intervention design needs to know.

## Fairness analysis

Group-level metrics were computed across gender, international status, scholarship holding, special educational needs, debtor status, marital status, parental occupation and qualification, and attendance mode.

Equal opportunity was chosen as the primary criterion over demographic parity and equalised odds. Since dropout risk genuinely differs across groups, forcing equal flag rates would under-serve high-risk groups. What should be equal is the chance of being correctly identified when actually at risk.

Removing gender, nationality and age did not remove the bias. Scholarship holders showed a recall of **0.679** against **0.810** for other students: the model caught roughly two thirds of at-risk scholarship students versus four fifths of everyone else. Overall accuracy was near identical between the groups, which is precisely why accuracy alone hides this.

## Bias mitigation and its limits

AIF360's Reweighing was applied, assigning instance weights before training so that group membership and outcome are statistically independent in the reweighted sample.

| | Disadvantaged recall | Other recall | Gap |
|---|---|---|---|
| Before reweighing | 0.679 | 0.810 | 0.131 |
| After reweighing | 0.684 | 0.811 | 0.127 |

The gap closed by 0.4 percentage points. In practical terms, reweighing did not fix it.

This is reported as the finding rather than buried. A pre-processing correction that only adjusts training weights cannot repair a disparity rooted in how well the retained features describe one group versus another. Closing it would require different features, targeted data collection, or a threshold set per group, each of which raises its own legal and ethical questions under the AI Act.

## Governance documentation

| Document | Contents |
|---|---|
| `ethics-assessments/eu_ai_act_classification.pdf` | Classification as high-risk under Annex III(3)(b), why the Article 6 exemptions do not apply given profiling, and Article 5 prohibited-practice risks around exploiting socio-economic vulnerability |
| `ethics-assessments/deda_value_sensitive_design.pdf` | DEDA assessment covering data and privacy, bias mitigation, transparency, and accountability allocation |
| `ethics-assessments/data_inspection_report.pdf` | Population distributions, base rates by group, and classification of variables into personally sensitive attributes and socio-economic proxies |
| `ethics-assessments/ethical_reflection.pdf` | Reflection on the tension between accuracy and fairness across the project |
| `governance/model_card.pdf` | Model card: intended use, out-of-scope use, factors, metrics, ethical considerations, caveats |
| `governance/datasheet.pdf` | Datasheet for the dataset: motivation, composition, collection, preprocessing, distribution |
| `governance/project_proposal.pdf` | Deployment scenario and organisational context |

The deployment design keeps the output advisory. Predictions go to student counsellors only, never to teaching staff, to limit stigma and prevent the prediction becoming self-fulfilling. No automated decision is taken on the basis of a score.

## Running it

```bash
pip install -r requirements.txt
jupyter notebook notebook/dropout_model.ipynb
```

The notebook runs end to end without local data. `aif360` is required for the fairness metrics and reweighing sections.

## Authorship

Group submission for the Responsible AI course. Analysis and documentation were shared across the group.

## Note

This is coursework, published as a record of completed work. It is not intended as reference material for anyone currently enrolled in the course.
