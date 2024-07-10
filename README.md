
# Quality_Movie_Data_Analysis

It is an ETL pipeline to find out quality movies using imdb_rating with the help of below AWS services

1.S3 as datalake\
2.Glue Crawler\
3.Glue Catalog\
4.Glue Catalog data quality(transformation)\
5.Redshift(Data warehhouse)\
6.Event Bridge rule\
7.SNS for email notification


## Architecture of ETL pipeline

![Architecture_of_ETL_pipeline](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/1aead0fe-9ec3-416c-b611-5918994e2c7d)



## Steps to create data pipeline

**1. Creation of bucket**: Create S3 bucket with name of movie-data-analysis-storage  and upload imdb_movies_rating.csv

![image](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/00f54767-4119-45b4-8ee0-2e8301d709d1)


**2. Creation of Crawler**:Before creating cralwer, create movie-data database under database section present under AWS Glue. Create the crawler,crawl-movie-dataset for data store in S3 bucket so that it will create meta data table under the movie-database

![image](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/dc94aadb-d9da-4530-8a21-63e52645e574)

After running crawlers  below meta data tables get created

![image](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/0bc8f29e-ab53-4ceb-9efa-950f0575b622)


**3. Data Quality checks**: Go to meta table present under movie-data database, click on imdb_movies_rating_csv meta data table , you will find data quality section in left side, click on recommendation rules and scan the data based on default parameter it will suggest some rules add those rules and update values as per your need. Will add rules  for imdb-rating to check if it is present or not and it's value should be in between 8.5 to 10.3.Save it as as movie_data_quality_check\
Run the data-quality check to see how much data following that rule , create new folder in S3 bucket of name as historical-data-rule-outcome where after running data rules output will get store

![image](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/6453fbfd-5daa-41b8-8a61-664f003c322c)


**4. Create destination table in redshift**: Create schema as movies and under that create table movies-imdb-movies-rating\
Create the crawler as redshift-destination-table-crawler which will create the meta table as redshift_dev_movies_imdb_movies_rating for redshift destination table

![image](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/2b57b5fc-8cc1-4036-a975-104b6ad73b0c)


**5. Design ETL pipeline using AWS GLUE**:
1. Go to AWS Glue-> Select ETL jobs-> Visual ETL, select source as AWS Glue Catalog from where it will pick up the table meta data and rename as S3-data-source and select database as movie-data , attach glue role
   
   ![Glue_data_source](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/e9389773-7180-4ed4-922c-0b27d7a572d5)

2. Need to apply data quality checks on input data for that go to the add icon  in GLUE ETL-> transform, 
select Evaluate Data Quality rename it as a data quality checks, scroll down you will see the data quality actions, check in the publish result to amazon cloudwatch even if rule get failed continue with job

![Data_quality_check](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/c4cb39a8-fc2d-4eb8-8676-58f7c449dfff)

3. If we want new column in original data recordwise which will include  value as record is passing or failing data quality check or not,so for that enable the "add new columns to indicate data quality errors"\
To get the result of "ruleoutcome" create new folder under existing S3 bucket as rule-outcome\
4. Go to GLUE ETL visual code-> ruleoutcome->select target as Amazon S3->edit properties->choose formating  as JSON->choose target as rule outcome in S3\
5. Click on add nodes->transform-> search for Conditional router->add the condition under output-group-1 -> choose key as "DataQualityEvaluation" result-> add condition to match value with "failed" -> rename output group as "Failed records"
6. We need to redirect the failed records on s3 bucket
so that we can later analyze the data using athena so for that add destination as S3 bucket.\
Create"bad_records" directory under  movie-data-analysis-storage bucket to store failed records in JSON format\
7. Now have to add successfull records into redshift table so will need to update the schema as per already created destination table , so for that select default group->add "change schema" transformation->rename it to "drop-columns"->change data type as per schema of redshift table->select target as AWS data catalog->rename to redshift-load-table->choose db,redshift meta data table name,temp location from S3 bucket(create new directory as redshift-temp-data under existing S3 bucket)->select redshift-role->rename GLUE job as "ETL_Movie_Data_Analysis_Using_Glue"->go to 
job details->add parameter as --Job Name, update property, select 2 workers and glue role->create 2 VPC 1 for glue and 1 for cloudwatch log to avoid errors after running Glue job->run the ETL_Movie_Data_Analysis_Using_GLUE job ->once job runs successfully do the query on redshift table, will find some records where out of 1000 records only 33 records get loaded into table which means others records are of below average movies\
8. To check bad records go to S3 bucket path movie-data-analysis-storage/bad_records, download it will see failed records in JSON format where "DataQualityEvaluationResult" value is "failed"\
9. Now have to create event bridge rule so that once data quality processing check is done will get result as failed or succeeded records  capture through event bridge which is integrate with SNS topic on email, to do that search event bridge on AWS console->click on eventbridgerule and create rule as "MovieDQCheckStatus_Rule"->select rule with event pattern ->next ->select "Glue Data Quality" as AWS Service->select Data Quality Evaluation Results available as event type-> select specific state option for event type specification 1,under that select both state succeeded and failed->select 
target as SNS AWS Service ->create topic in SNS and add email as subscriber and create the rule so once run the GLUE ETL job will receive email for failed and succeeded records

![ETL_Movie_Data_Analysis_Using_GLUE](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/878079cc-4ecd-4237-a805-774480003f16)



## OutPut of ETL pipeline after execution


Succeeded records present in redshift table 
![After_glue_job_run_redshift_output](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/75179559-fc4e-43b9-901b-d791d448b9af)

SNS Notification receive on email
![SNS_notification_received_on_email](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/bee4f1e6-2830-4aa8-aa68-16db53b2df45)

Data Quality check failed record present under S3 bucket
![image](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/043d4e04-b011-4231-bf8f-6afcf431fb22)

After run the glue_job no of records present in redshift table

![No_of_records_loaded_into_redshift_table](https://github.com/pritishete/Quality_Movie_Data_Analysis/assets/57429486/ab220108-3363-4ed1-8592-0f3e787e48b6)



































































    








