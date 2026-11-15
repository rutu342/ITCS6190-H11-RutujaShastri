# Hands-on L11: Report

**Name:** Rutuja Hemant Shastri  
**Student ID:** 801484366  
**Email:** rshastri@charlotte.edu

---

## Seed and commands

Seed used (your student ID): 801484366

The existing Spark containers belonged to the A4 workspace, so I replaced those three Spark
services with this repository's H11 services. The base image also needed a compatible NumPy
package installed in the running containers before the jobs would start.

```bash
docker compose up -d spark-master spark-worker-1 spark-worker-2
docker cp main.py spark-master:/opt/spark/work-dir/main.py
docker exec spark-master /opt/spark/bin/spark-submit --master spark://spark-master:7077 /opt/spark/work-dir/main.py task1 /opt/spark/work-dir/shared 801484366
docker exec spark-master /opt/spark/bin/spark-submit --master spark://spark-master:7077 /opt/spark/work-dir/main.py task2 /opt/spark/work-dir/shared 801484366
```

## Results

### Task 1: baseline and full pipeline

```text
seed 801484366: 5704 training rows, 1442 test rows
baseline                         RMSE   272.4   R2  0.140
full                             RMSE   266.4   R2  0.175
+----------------------+---------------+--------+-----+------------------+
|neighbourhood_cleansed|room_type      |bedrooms|price|prediction        |
+----------------------+---------------+--------+-----+------------------+
|Bayview               |Entire home/apt|1.0     |85.0 |82.73056842970254 |
|Bayview               |Entire home/apt|3.0     |250.0|410.0714228207071 |
|Bernal Heights        |Entire home/apt|1.0     |250.0|157.88234530366026|
|Bernal Heights        |Entire home/apt|2.0     |250.0|258.4967343433673 |
|Castro/Upper Market   |Entire home/apt|3.0     |500.0|438.3434856082458 |
+----------------------+---------------+--------+-----+------------------+
Full PipelineModel saved to /opt/spark/work-dir/shared/models/full
Metrics saved to /opt/spark/work-dir/shared/output/task1/metrics
```

The split has 5,704 training rows and 1,442 test rows. The seed-42 split differs because
`randomSplit` assigns rows differently for each seed; the .8/.2 proportions are approximate,
not fixed row counts.

### Task 2: scoring with the saved model

```text
Stages: ['StringIndexerModel', 'OneHotEncoderModel', 'VectorAssembler', 'LinearRegressionModel']
Scored 1440 of 1442 test listings
loaded full model RMSE 266.4
+----------------------+---------------+--------+-----+----------+
|neighbourhood_cleansed|room_type      |bedrooms|price|prediction|
+----------------------+---------------+--------+-----+----------+
|Bayview               |Entire home/apt|1.0     |85.0 |82.7      |
|Bayview               |Entire home/apt|3.0     |250.0|410.1     |
|Bernal Heights        |Entire home/apt|1.0     |250.0|157.9      |
|Bernal Heights        |Entire home/apt|2.0     |250.0|258.5      |
|Castro/Upper Market   |Entire home/apt|3.0     |500.0|438.3      |
+----------------------+---------------+--------+-----+----------+
Predictions saved to /opt/spark/work-dir/shared/output/task2/predictions
```

## Questions

**1. Baseline versus full.** With seed 801484366, the baseline was RMSE 272.4 / R2 0.140
and the full pipeline was RMSE 266.4 / R2 0.175. The added features reduced RMSE by
$6.0 and increased R2 by 0.035. RMSE is in dollars and squares residuals before averaging,
so large errors and high-price outliers have substantial influence; the same dollar RMSE
also represents different relative errors at different price levels. Metrics change with
seed because the train/test listings, prices, and category frequencies change. I would
first tune LinearRegression's `regParam` using cross-validation.

**2. What is in `shared-folder/models/full`?** The saved model contains `metadata/` and
`stages/`, with these stage directories:

```text
metadata/
stages/
  0_StringIndexer_e38d5971887d/
  1_OneHotEncoder_e492064c2230/
  2_VectorAssembler_be574fcd3fdf/
  3_LinearRegression_13f059579170/
```

The metadata records the PipelineModel configuration; each stage directory holds its
saved stage metadata and data. We save the PipelineModel so scoring repeats the same
category indexing, encoding, feature assembly, and regression. StringIndexerModel,
OneHotEncoderModel, and LinearRegressionModel are fitted models. VectorAssembler is a
configured transformer, not a fitted model.

**3. Only transformers at scoring time.** Task 2 wrote 1,440 predictions from 1,442 test
listings. Two rows with null or previously unseen categorical values were dropped by the
StringIndexer stage's `handleInvalid="skip"` setting. To score a stream, read from a
streaming source with `readStream`, apply the loaded PipelineModel to each micro-batch,
and write using `writeStream` to a streaming sink with checkpointing.

## Problems and fixes

The first task 1 attempt used the old A4 cluster mount and failed with
`[PATH_NOT_FOUND] Path does not exist: file:/opt/spark/work-dir/shared/data/sf-airbnb-clean.parquet.`
I replaced the old Spark services with the H11 cluster, resolving that issue. The base
Spark image then failed with `ModuleNotFoundError: No module named 'numpy'`. Ubuntu's
NumPy package was too old and caused `TypeError: 'type' object is not subscriptable`;
installing NumPy 1.26.4 in the running Spark containers resolved it. One worker's first
wheel download failed pip's SHA-256 verification; retrying without cache succeeded.
