\# Wind Turbine Anomaly Detection



A personal project: can I spot early signs of trouble in wind turbine SCADA data when there are no failure labels? I trained an LSTM autoencoder on normal operation, flagged windows it could not reconstruct well, and turned that into a daily health score per turbine.



\*\*Data:\*\* Engie La Haute Borne wind farm, 4 turbines, Dec 2012 to Jan 2018, 10-minute readings (about 0.8M rows after cleaning). The data is not in this repo. Get it from the original source and check its license.



\## What I found



\- The model's validation error is 0.043, compared with 0.888 for just predicting the mean.

\- One turbine (R80711) drifts away from the rest from mid-2017. Its monthly health score drops from about 96 (May 2017) to about 27 (Jan 2018), while the other three stay between 93 and 100. Gearbox oil temperature sensors contribute most of the error.

\- On that turbine the LSTM flags 52% of the windows after June 2017. Z-score, PCA and Isolation Forest flag 10%, 2.5% and 1.8%. On the three healthy turbines, false alarms are 2.3% for the LSTM and 0.7 to 1.7% for the others.



!\[Monthly reconstruction error](results/monthly\_error.png)



\## How it works



1\. PySpark cleans the raw CSVs and stores them as Parquet. About 20 to 32% of the 10-minute timestamps are missing, so everything downstream is gap-aware.

2\. Spark SQL builds hourly and daily aggregates and lag features (plain GROUP BY and JOIN, no window functions).

3\. The LSTM autoencoder reads 6-hour windows of 16 sensors. Train, validation and test are split by time, and the scaler is fit on training data only.

4\. The threshold for each turbine is the 99th percentile of its validation error.

5\. A `pandas\_udf` scores windows in Spark. The scores match single-machine inference (max difference 2e-5).

6\. The daily health score is 100 times the share of that day's windows under the threshold.



\## Read the results carefully



\- \*\*There are no failure labels.\*\* I picked R80711 after seeing the model output, so this is not a verified fault detection. I could not confirm the cause (gearbox or cooling problem, sensor drift, or a change in how the turbine was run).

\- I assumed the early part of the data is healthy. I did not verify that.

\- The baselines are basic. A model that predicts temperature from power and wind speed would be a tougher comparison.

\- The LSTM raises more false alarms than the simple baselines.

\- Spark ran locally on 2 CPU cores and was slower than scoring on one GPU (25 s vs 0.6 s). I make no speedup claim. The Spark part checks correctness and is built for larger fleets.



\## Run it



Everything lives in the notebook in `notebooks/`. I ran it in Google Colab (T4 GPU, Java 21, PySpark 4.0.4) with the data in Google Drive. Paths assume `/content/drive/MyDrive/scada\_project`.

