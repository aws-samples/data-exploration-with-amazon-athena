# Data Exploration with Apache Spark in Amazon Athena

This lab is provided as part of [AWS Innovate Data Edition](https://aws.amazon.com/events/aws-innovate/data/). Click [here](https://github.com/phonghuule/aws-innovate-data-edition-2023) to explore the full list of hands-on labs.

:information_source: You will run this lab in your own AWS account and running this lab will incur some costs. Please follow directions at the end of the lab to remove resources to avoid future costs.

## Overview

Amazon Athena is a serverless, interactive analytics service built on open-source frameworks, supporting open-table and file formats. Athena provides a simplified, flexible way to analyze petabytes of data where it lives. Analyze data or build applications from an Amazon Simple Storage Service (S3) data lake and 30 data sources, including on-premises data sources or other cloud systems using SQL or Python. Athena is built on open-source Trino and Presto engines and Apache Spark frameworks, with no provisioning or configuration effort required.

In this lab, we'll explore how to combine the Amazon Athena with the power of Apache Spark to quickly and easily explore and transform data.

## Architecture

In order to explore the data with Amazon Athena and Apache Spark, we'll create the following architecture.

![Architecture diagram](./images/architecture.png)

## Before You Begin
Before you start, there are a few things you should know:

- The resources created in this lab will have some costs associated with them. 
- You will need an IAM user or role with appropriate permissions to execute this lab. This includes permissions to create the resources defined in the CloudFormation template, as well as the manual activities performed through the AWS Management Console. 
- This lab does not rely on AWS Lake Formation for Glue catalog permissions; However, if Lake Formation is configured for the account, you will need to grant appropriate permissions to allow access to the catalog resources.

## Step 1 - Setup 
First, we need to set up a data lake in the account that we'll use with Athena.

1.1. Download the CloudFormation template file here. This template will create the following in your account:
- S3Bucket - <accountid>-<stackname>-athenaresults - this bucket will be used to store the results of Athena queries.
- S3Bucket - <accountid>-<stackname>-datalake-raw - this bucket will store our raw customer and sales data as CSV.
- S3Bucket - <accountid>-<stackname>-datalake-processed - this bucket will store our processed customer and sales data as Parquet.
- S3Bucket - <accountid>-<stackname>-datalake-presentation - this bucket will store the results of our aggregation operations.
- Athena Workgroup - AmazonAthenaLakeFormationWorkgroup - this workgroup will be used to run simple queries on the data.
- Glue Database - raw - This database will store the sales and customer metadata for our raw data layer.
- Glue Database - processed - This database will store the sales and customer metadata for our processed data layer.
- Glue Database - presentation - This database will store the  metadata for our aggregated data presentation data layer.
- Glue Crawler - <stackname>-crawler-raw - this crawler will parse our raw data and create the sales and customer metadata in the AWS Glue catalog.
- Glue Crawler - <stackname>-crawler-processed - this crawler will parse our processed data and create the sales and customer metadata in the AWS Glue catalog.
- Glue Crawler - <stackname>-crawler-presentation - this crawler will parse our aggregated data and metadata in the AWS Glue catalog.
- StepFunctionsExecutionRole
- IAM Role - <stackname>-GlueExecutionRole - this IAM role will be used by the Glue crawlers and ETL jobs.
- IAM Role - <stackname>-StepFunctionsExecutionRole - this IAM role will be used by the Step Functions state machine.
- IAM Role - <stackname>-AthenaWorkgroupRole - this IAM role will be used by the Athena workgroup for Apache Spark.
- Step Functions State Machine - <stackname>-DataPipelineOrchestration - This will orchestrate Glue crawlers and ETL jobs to prepare the data lake. 
- Glue ETL job - <stackname>_process_customer_job - This job will transform the customer data from the raw layer to the processed layer in the data lake.
- Glue ETL job - <stackname>_process_sales_job - This job will transform the sales data from the raw layer to the processed layer in the data lake.

- AthenaWorkgroupRole

1.2. Log into your AWS Management Console, and navigate to AWS CloudFormation.

1.3. Click **Create stack**

1.4. Select **Upload a template file**, then choose the file you downloaded earlier and click **Next**.

1.5. Enter ```athenalab``` as the **Stack name** and click **Next**.

1.6. On the **Configure stack options** page, leave the default values and click **Next**.

1.7. On the final page, scroll to the end and mark the checkbox next to **I acknowledge that AWS CloudFormation might create IAM resources with custom names.**, then click **Submit**.

1.8. Wait until the stack has completed successfully.


## Step 2 - Load Data and Run Pipeline
Next, we need to load customer and sales data into our data lake.

2.1. Navigate to AWS CloudShell in the AWS Management Console.

2.2. Execute the following commands, replacing the value of **<accountid>**. This copies **sales** and **customer** data into the raw data buckets in the Data Lake.
```
aws s3 cp s3://moderndataarchitecture-immersionday-assets/v4_2/data/customers.csv .
aws s3 cp s3://moderndataarchitecture-immersionday-assets/v4_2/data/sales.csv .
aws s3 cp ./customers.csv s3://<accountid>-athenalab-datalake-raw/customers/customers.csv
aws s3 cp ./sales.csv s3://<accountid>-athenalab-datalake-raw/sales/sales.csv
```
2.3. Navigate to AWS Step Functions in the AWS Management Console.

2.4. Navigate to **State machines**, then select the **athenalab-DataPipelineOrchestration** state machine and click **Start execution**.

2.5. Leave the default input, and click **Start execution**.

2.6. Wait for the state machine to complete. This function will do the following:
- Runs a crawler on the Raw data bucket to create **raw_sales** and **raw_customer** tables in the Raw database in the Glue catalog. 
- Runs 2 Glue ETL jobs in parallel. These jobs transform the Sales and Customer data, removing some unnessary characters from the data and writing the data to the **Processed** data bucket in Parquet format with partitions.
- Runs a crawer on the **Processed** data to create **processed_sales** and **processed_customers** tables in the Processed database in the glue catalog.

## Step 3 - Configure Amazon Athena
Next, we'll configure Amazon Athena to create a notebook to use Apache Spark. 

3.1. Navigate to Amazon Athena in the AWS Management Console. 

3.2. Open the nagivation pane on the left, and select the **Query editor**. In the workgroup dropdown, select the **AmazonAthenaLakeFormation** workgroup.

3.3. In the configuration popup, click **Acknowledge**.

3.4. In the **Database** dropdown, select the **processed** database.

Let's have a quick look at the data.

3.5. Execute the following:
```
SELECT * FROM "processed"."processed_customers" limit 10;
```
3.6. Execute the following:
```
SELECT * FROM "processed"."processed_sales" limit 10;
```
Now let's use Apache Spark in Amazon Athena to explore and transform our data.
3.7. Open the nagivation pane on the left, and select **Workgroups**.

3.8. Click **Create workgroup**.

3.9. Enter ```SparkWorkgroup``` as the **Workgroup name**.

3.10. Select **Apache Spark** as the engine.

3.11. Expand **IAM role configurations**, select **Choose an existing service role** the select the **athenalab-AthenaWorkgroupRole**.

3.12. Expand **Calculation result settings**, select **Choose an existing S3 location**, then browse to the **-athenalab-athenaresults** S3 bucket. 

3.13. Click **Create workgroup**, then click on the workgroup you just created.

3.14. Click **Create notebook**, enter **AthenaLabExploration** name for the notebook and click **Create**.

3.15. Open the newly created notebook.

## Step 4 - Explore Data with Apache Spark

Now that we have a notebook, let's run some commands. Copy the commands below, and paste into a new cell in the notebook (click the + button to add cells). Press **ctrl+enter** or click the **Run** button to execute the statement.

4.1. Let's load some libraries we'll need.
```
import sys
from datetime import datetime
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.functions import col
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd
```
4.2. Load customer and sales data into a Spark dataframe.
```
df_customers = spark.sql("SELECT * FROM processed.processed_customers")
df_sales = spark.sql("SELECT * FROM processed.processed_sales")
```
4.3. Observe a subset of the customer data
```
df_customers.show(10)
```
4.4. Observe a subset of the sales data
```
df_sales.show(10)
```
4.5. Create a simple profile of the customer data
```
df_customers.describe().show()
```
4.6. Create a simple profile of the sales data
```
df_sales.describe().show()
```
4.7. Join sales and customer data on customer_id as a new data frame.
```
df_join=df_customers.join(df_sales, "customer_id").filter(col('timestamp').isNotNull())
df_join.show(10)
```
4.8. Group and aggregate the data into another data frame.
```
df_group = df_join.groupBy("customer_id").agg(
    F.sum("price").alias("total_revenue"),
    F.count("price").alias("count_sales"),
    F.max("price").alias("max_price"),
    F.min("price").alias("min_price"),
    F.avg("price").alias("avg_price"),
    F.max("timestamp").alias("min_timestamp"),
    F.min("timestamp").alias("max_timestamp")
    )
df_group.show(10)
```
4.9. Create a simple profile of the aggregated data
```
df_group.describe().show()
```
4.10. Create a visualization of the data
```
plt.clf()
pdf=df_group.toPandas()
plt.bar(pdf.customer_id, pdf.total_revenue)
%matplot plt
```
4.11. Optional - Write the aggregated data back to s3. Make sure you replace the **<account_id>** below.
```
df_group.write.mode("overwrite").format("parquet").save("s3://<account_id>-athenalab-datalake-presentation/group_by_customer")
```
4.12. Optional - Navigate to AWS Glue and execute the **presentation** data crawler to create a new table in the Glue catalog for the aggregated data.

## Conclusion

In this lab, we've created a simple data lake with Amazon Glue and Amazon Simple Storage Service (S3), and used Amazon Athena to run some simple queries on the data.  We've then use Apache Spark in a notebook in Amazon Athena to further explore the data, join data sets, create an aggregation and a visualization, and optionally saved the results back to the data lake.

## Survey

Please complete the following survey to let us know how you found this lab and how we can improve.

## Clean Up

To clean up the resources created here, complete the following.

1. Navigate to Amazon Athena, and delete the **AthenaLabExploration** notebook and the **SparkWorkgroup** workgroup.
2. Navigate to Amazon S3, and click **Empty** for each of the following buckets.
- <accountid>-athenalab-athenaresults
- <accountid>-athenalab-datalake-raw
- <accountid>-athenalab-datalake-processed
- <accountid>-athenalab-datalake-presentation
3. Navigate to CloudFormation, and delete the **athenalab** stack. 

## Survey

Let us know what you thought of this session and how we can improve the presentation experience for you in the future by completing this event session poll. Participants who complete the surveys from AWS Innovate Online Conference will receive a gift code for USD25 in AWS credits (1, 2 & 3). AWS credits will be sent via email by September 29, 2023.
Note: Only registrants of AWS Innovate Online Conference who complete the surveys will receive a gift code for USD25 in AWS credits via email.
1. AWS Promotional Credits Terms and conditions apply: https://aws.amazon.com/awscredits/
2. Limited to 1 x USD25 AWS credits per participant.
3. Participants will be required to provide their business email addresses to receive the gift code for AWS credits.

Click [here](https://amazonmr.au1.qualtrics.com/jfe/form/SV_1U4cxprfqLngWGy?Session=HOL06) to complete the survey.


## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

