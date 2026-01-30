# Azure-Data-Factory---Covid19-Analysis

## **Introduction:**
This project focuses on building an end-to-end data engineering pipeline to analyze COVID-19 data across Europe. \
Azure Data Factory is used for automated data ingestion, while Azure Databricks and Data Flows handle scalable data transformation and processing. \
Azure Blob Storage and Azure Data Lake Gen2 provide reliable storage for raw and curated data. \
The processed data is published to Azure SQL Database and visualized through interactive Power BI reports to enable data-driven insights.

## **Project Architecture:**

![project_architecture](Screenshots/data_arch1.png)

## **Dataset:** 
We are using data from **European Centre for Disease Prevention and Control** through HTTP connector 
and **Population data** through Azure Blob storage.\
We can get the ECDC data from the below link :

    https://www.ecdc.europa.eu/en/covid-19/data


## **Good Practices:**
### **Naming Conventions :**
>**Linked Service:** ls_(ablob/adls)_storagename\
>**Data Set:** ds_datatype_folder\
>**Pipeline:** pl_function

## **Creating Azure Resources:**
### **Create Azure data Factory:**
>Create Azure Data Factory : covid19-sudhanshu-adf  (give meaningful name)

### **Create Azure Blob Storage:**
>Create Azure Blob Storage: sudhanshucovid19 (give meaningful name, Small case) \
>search storage account > give subscription and resource group > create

### **Create Azure Databricks Workspace:**
>search Azure databricks > give subscription and resource group > create

### **Create Azure Data Lake Storage Gen2:**
>same as we created storage account.

> [!CAUTION]
>Just make sure to check Enable hierarchical namespace (under advanced)  this time.

![adls](Screenshots/adls.png)

### **Download storage Explorer:** 
>storage account > storage browser > download Azure Storage Explorer

**Login:**
>Azure portal > users > copy user principal name > and use this as login id to login


![storage_exp](Screenshots/storage_explorer.png)

### **Create Azure SQL Database:**
>Azure SQL databse > create > Db name : covid19-db > server : create new > give unique name

>Use SQL authentication > give admin name and password

give these settings:

![sql_db](Screenshots/sql_db.png)

> [!NOTE]
> select Locally-redundant backup storage.

Networking:
give these settings:

![sql_db2](Screenshots/sql_db2.png)

### **Create a dashboard:**
>Azure Data Factory > Pin > Pin to dashboard > create new > Covid Reporting Dashboard

**pin all resources in the dashboard.**

![dashboard](Screenshots/covid_dashboard.png)

## **Creating Containors for Data Loading:**
- **Create Azure Blob Storage Containors:**\
  Create the containor : population\
  Storage account > storage browser > Blob container > Add > population \
  and upload the below file:\
  [population_by_age.tsv.gz](covid19_data_sets/population_data)
- **Create ADLS Gen2 Containors:**\
  Same as above create : raw

## **Data Ingestion:**
### **Create Pipeline to Ingest Population Data (pl_ingest_population_data):** 
- **Validation Activity:**\
  Check if the file : **population_by_age.tsv.gz** exists on the specific location\
- **Get Metadata Activity:**\
  Get column count for the input file.
- **If Condition Activity:**\
  Check the Count value from the metadata activity and match:

      @equals(activity('Get File Metadata').output.columnCount,13)
  If the match is:
  + **True:**
    * Copy Population Data_copy (Copy Acivity):
      - Copy the data to Sink (ADLS Gen2 Containor - raw)
    * Delete Activity:
      - On copy success delete the file from source.
   + **False:**
     * Fail Activity:
       - Raise error message (File Incompatible)\

Here is the complete pipeline:
![pl_ingest_population_data.png](Screenshots/pl_ingest_population_data.png)

> [!WARNING]
> Publish the pipelines and Data sets ASAP, otherwise all changes will be lost.

- **Create Trigger (tr_se_pl_ingest_population_data):**\
  Create Event based trigger and attach the above pipeline in that.\
  Execute the trigger:
  ![Trigger_pipeline](Screenshots/pipeline_success1.png)

### **Create Pipeline to Ingest ECDC Data (pl_ingest_ecdc_data):** 
- **Lookup Activity:**\
  We have on lookup file having the below details:
  * Relative URL
  * Sink File Name\
So we can Ingest all the files from ECDC website.\
The lookup file is a JSON file : ecdc_file_list_for_2_files2.json\
Give this File as Input to Lookup Activity\
[ecdc_file_list_for_2_files2.json](Lookup_Files/ecdc_file_list_for_2_files2.json)

- **For Each Activity:**
  * Get the output value from the Lookup Activity using the below Item:

          @activity('Look for the ecdc details').output.value
  * Create Dataset for HTTP Source and create a parameter for **relative_url** and use this parameter in connection of this dataset:
    ![ds_http_ecdc_raw_csv](Screenshots/ds_http_ecdc_raw_csv.png)
  * Create Dataset forSink ADLS Gen2 and create a parameter for **file_name** and use this parameter in connection of this dataset:
    ![ds_dl_ecdc_raw_csv](Screenshots/ds_dl_ecdc_raw_csv.png)
    
  * Create Copy Activity : **Copy ECDC Data** inside For Each, Using the output from Lookup Activity
    - Parameter value for Source:
      
      ![source_param1](Screenshots/source_param1.png)
    - Parameter value for Sink:
      
      ![sink_param1](Screenshots/sink_param1.png)

- **Complete Pipeline (pl_ingest_ecdc_data):**

  ![pl_ingest_ecdc_data](Screenshots/pl_ingest_ecdc_data.png)

## **Data Transformation:**
### **Transformation of Cases and Deaths:**
#### **Transformation Requirements:**
1. Filter Data for **Europe** only.
2. Remove columns : continent, rate_14_day
3. Split indicator column (take indicator and daily_count column and pivot them in cases_count and deaths_count)
4. Get country_code_2_digit and country_code_3_digit from the lookup file.

We will perform all the above transformation using **Data Flow:** df_transform_cases_deaths.

#### **Data Flow Transformation Steps:**
1. Select the cases_deaths file as source:

        raw/ecdc/cases_deaths/cases_death.csv
2. Filter the data only for **Europe**. Put this command in the Filter transformation

       continent == 'Europe' && not(isNull(country_code))
3. Select transformation to select only the required columns.
4. Split indicator column (take indicator and daily_count column and pivot them in cases_count and deaths_count).\
   - Select Pivot transformation. 
   - Give Pivot Key as:
     
       ![pivot_key1](Screenshots/pivot_key1.png)
     
   - Give Pivoted Columns as:
     
       ![pivoted_columns2](Screenshots/pivoted_columns2.png)
   - Other then Pivot Key and Pivoted Columns keep all the columns in group by.
5. Get country_code_2_digit and country_code_2_digit from the lookup file.
   - Create one more source of this lookup file : [country_lookup.csv](Lookup_Files/country_lookup.csv)
   - Do Lookup on Country column.
6. After Lookup actvity remove the duplicate columns using Select activity.
7. Create a Sink transformation. Create the Sink on this path:

       processed/ecdc/cases_deaths

Here is the complete Dataflow:\
**df_transform_cases_deaths**

![df_transform_cases_deaths](Screenshots/df_transform_cases_deaths.png)

#### **Create Pipeline for Data Flow execution:** 
pl_df_process_cases_deaths
- In the pipeline use Data Flow activity and select the above Data Flow

### **Transformation of Hospital Admissions:**
#### **Transformation Requirements:**
1. Select required columns only : Remove URL, Rename date to reported_date, year_week to reported_year_week
2. Get country_code_2_digit and country_code_3_digit from the lookup file.
3. Select dim_date lookup file to get the week_start_date and week_end_date.
4. Conditional split the data as Weekly and Daily data.
5. Pivot Weekly data on **Weekly new hospital admissions per 100k** and **Weekly new ICU admissions per 100k** \
   Daily data on **Daily hospital occupancy** and **Daily ICU occupancy**.
6. Sink separately for Weekly and Daily data.

We will perform all the above transformation using **Data Flow:** df_transform_hospital_admissions.

#### **Data Flow Transformation Steps:**
1. Select the cases_deaths file as source:

        raw/ecdc/hospital_admissions/hospital_admissions.csv
2. Select Activity : Remove URL, Rename date to reported_date, year_week to reported_year_week
3. Get country_code_2_digit and country_code_3_digit from the lookup file.
4. Conditional Split Transformation :\
   Use below Split Condition:

   ![split_condition](Screenshots/split_condition.png)

5. Join Transformation:\
   Join with Lookup File : [dim_date.ccv](Lookup_Files/dim_date.csv)
6. Pivot Weekly data on **Weekly new hospital admissions per 100k** and **Weekly new ICU admissions per 100k** \
   Daily data on **Daily hospital occupancy** and **Daily ICU occupancy**.
7. Sink Transformation :\
   **Weekly:**

           processed/ecdc/hospitals_admissions_weekly

   **Daily:**

           processed/ecdc/hospitals_admissions_daily

Complete Data Flow: **df_transform_hospital_admission**

![df_transform_hospital_admission](Screenshots/df_transform_hospital_admission.png)

#### **Create Pipeline for Data Flow execution:** 
pl_df_process_hospital_admissions
- In the pipeline use Data Flow activity and select the above Data Flow

## **Data Transformation Using Databricks:**

Open the databricks workspace we created and create an All Purpose cluster:
> Azure Portal > Dashboard > Databricks Workspace > Launch > Compute > create compute

### **To Enable Datsbricks to access data in ADLS Gen2:**

Create Azure Service Principal:

> Microsoft Intra > New app registration > Covid-reporting-app

Copy these values: **DB_VARIABLE_VALUE**
1. Application(application_id)
2. Directory (directory_id)
3. Certificate and secrets > new client secret. copy the values, Secret value(service_credential)
4. Storage account name (storage_account)

> [!IMPORTANT]
> Save theseabove values somewhere on notebook, we will need them later.

**Grant access for data lake to azure service principal:**
> ADLS storage account > I am > add role assignment > Storage Blob Data Contributor > give access to covid-reporting-app

### **Transformation of population_by_age.csv:**
#### **Transformation Requirements:**
1. Split the country code & age group
2. Exclude all data other than 2019
3. Remove non numeric data from percentage
4. Pivot the data by age group
5. Join to dim_country to get the country, 3 digit country code and the total population.

Here is the Databricks notebook created for this transformation:\
[transform_population_by_age_data](Scripts_notebooks/transform_population_by_age_data.dbc)

### **Transformation of testing.csv:**
#### **Transformation Requirements:**
1. Join to dim_country to get the country, 3 digit country code and the total population.
2. Join to dim_date to get the week start date and week end date.

Here is the Databricks notebook created for this transformation:\
[transform_testing_data](Scripts_notebooks/transform_testing_data.dbc)


           




   


    

      
  
  
  




