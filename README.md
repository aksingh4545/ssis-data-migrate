<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>SSIS to Snowflake Data Pipeline via S3</title>
  <style>
    body {
      font-family: Arial, Helvetica, sans-serif;
      line-height: 1.6;
      margin: 40px;
      color: #222;
    }
    h1, h2, h3 {
      color: #0b5394;
    }
    code, pre {
      background: #f4f4f4;
      padding: 6px;
      border-radius: 4px;
      display: block;
      overflow-x: auto;
    }
    ul {
      margin-left: 20px;
    }
    .section {
      margin-bottom: 40px;
    }
  </style>
</head>

<body>

<h1>SSIS → AWS S3 → Snowflake → Power BI Data Pipeline</h1>

<p>
This project demonstrates a complete real-world data engineering pipeline where data
is extracted using <strong>SSIS</strong>, staged in <strong>AWS S3</strong>,
loaded into <strong>Snowflake</strong> as a cloud data warehouse, and finally
visualized using <strong>Power BI</strong>.
</p>

<hr/>

<div class="section">
<h2>1. Architecture Overview</h2>

<p>The pipeline follows these stages:</p>

<pre>
Source CSV File
      ↓
SSIS Data Flow (Flat File Source → Flat File Destination)
      ↓
Local Output File
      ↓
AWS S3 (via AWS CLI + Execute Process Task)
      ↓
Snowflake External Stage
      ↓
Snowflake Warehouse Table
      ↓
Power BI Reporting
</pre>

<p>
This design separates ingestion, storage, and analytics layers,
making the pipeline scalable and production-ready.
</p>
</div>

<hr/>

<div class="section">
<h2>2. Tools and Technologies Used</h2>

<ul>
  <li>SQL Server Integration Services (SSIS)</li>
  <li>AWS S3</li>
  <li>AWS CLI</li>
  <li>Snowflake (Enterprise Edition on AWS)</li>
  <li>Power BI Desktop</li>
</ul>
</div>

<hr/>

<div class="section">
<h2>3. SSIS Configuration</h2>

<h3>3.1 Data Flow Task</h3>

<ul>
  <li><strong>Source:</strong> Flat File Source (Heart Disease CSV)</li>
  <li><strong>Destination:</strong> Flat File Destination</li>
</ul>

<p>
The SSIS package reads the source CSV and writes it to a local directory:
</p>

<pre>
C:\Users\ACER\Downloads\target\ankit.csv
</pre>

<p>
This file acts as the staging artifact for cloud upload.
</p>

<h3>3.2 Execute Process Task (AWS CLI)</h3>

<p>
SSIS uses an Execute Process Task to upload the generated file to S3.
</p>

<p><strong>Executable:</strong></p>
<pre>
C:\Program Files\Amazon\AWSCLIV2\aws.exe
</pre>

<p><strong>Arguments:</strong></p>
<pre>
s3 cp C:\Users\ACER\Downloads\target\ankit.csv s3://s3-ssis-bucket/data_source/ankit.csv
</pre>

<p><strong>Working Directory:</strong></p>
<pre>
C:\Users\ACER\Downloads\target
</pre>

<p>
The task uploads the file to the S3 bucket after the data flow completes.
</p>
</div>

<hr/>

<div class="section">
<h2>4. AWS S3 Setup</h2>

<h3>4.1 S3 Bucket</h3>

<pre>
s3://s3-ssis-bucket/data_source/
</pre>

<p>
The uploaded file is stored as:
</p>

<pre>
ankit.csv
</pre>

<h3>4.2 IAM Policy</h3>

<p>
The IAM role used by Snowflake has the following permissions:
</p>

<pre>
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::s3-ssis-bucket/data_source/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::s3-ssis-bucket",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["data_source/*"]
        }
      }
    }
  ]
}
</pre>
</div>

<hr/>

<div class="section">
<h2>5. Snowflake Configuration</h2>

<h3>5.1 Database and Schema</h3>

<pre>
CREATE DATABASE new_ssis_db;
CREATE SCHEMA new_ssis_db.raw;
</pre>

<h3>5.2 Storage Integration</h3>

<pre>
CREATE STORAGE INTEGRATION s3_ssis_int
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = '<AWS_IAM_ROLE_ARN>'
  STORAGE_ALLOWED_LOCATIONS = ('s3://s3-ssis-bucket/data_source/');
</pre>

<h3>5.3 File Format</h3>

<pre>
CREATE FILE FORMAT heart_csv_ff
  TYPE = CSV
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  SKIP_HEADER = 0
  TRIM_SPACE = TRUE;
</pre>

<h3>5.4 External Stage</h3>

<pre>
CREATE STAGE s3_ssis_stage
  URL = 's3://s3-ssis-bucket/data_source/'
  STORAGE_INTEGRATION = s3_ssis_int
  FILE_FORMAT = heart_csv_ff;
</pre>

<h3>5.5 Target Table</h3>

<pre>
CREATE TABLE new_ssis_db.raw.heart_disease (
  age INT,
  sex INT,
  cp INT,
  trestbps INT,
  chol INT,
  fbs INT,
  restecg INT,
  thalach INT,
  exang INT,
  oldpeak FLOAT,
  slope INT,
  ca INT,
  thal INT,
  status VARCHAR
);
</pre>

<h3>5.6 Load Data</h3>

<pre>
COPY INTO new_ssis_db.raw.heart_disease
FROM @s3_ssis_stage/ankit.csv;
</pre>
</div>

<hr/>

<div class="section">
<h2>6. Power BI Integration</h2>

<h3>6.1 Snowflake Connection Details</h3>

<ul>
  <li><strong>Server:</strong> YVQZWKK-ITB60254.snowflakecomputing.com</li>
  <li><strong>Warehouse:</strong> COMPUTE_WH</li>
  <li><strong>Database:</strong> new_ssis_db</li>
  <li><strong>Schema:</strong> raw</li>
  <li><strong>Username:</strong> SAMSUNG</li>
</ul>

<h3>6.2 Power BI Steps</h3>

<ol>
  <li>Open Power BI Desktop</li>
  <li>Get Data → Snowflake</li>
  <li>Enter server and warehouse</li>
  <li>Select database and schema</li>
  <li>Load table <code>heart_disease</code></li>
</ol>
</div>

<hr/>

<div class="section">
<h2>7. Final Outcome</h2>

<p>
The pipeline successfully ingests data from a CSV source, stages it in AWS S3,
loads it into Snowflake, and exposes it for analytics in Power BI.
</p>

<p>
This design follows real-world data engineering best practices and can be extended
with scheduling, Snowpipe automation, or transformation layers.
</p>
</div>

<hr/>

<div class="section">
<h2>8. Future Enhancements</h2>

<ul>
  <li>Automated loading using Snowpipe</li>
  <li>Incremental data loads</li>
  <li>Analytics views and star schema</li>
  <li>Power BI scheduled refresh</li>
  <li>Data quality checks and monitoring</li>
</ul>
</div>

</body>
</html>
