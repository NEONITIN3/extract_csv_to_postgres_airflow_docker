# extract_csv_to_postgres_airflow_docker
## Information

![image](https://github.com/dogukannulu/csv_extract_airflow_docker/assets/91257958/d0c01724-d2c6-4646-8f22-dac0ea3b5931)


```bash
pip install -r requirements.txt

```

## Steps

`write_csv_to_postgres.py`-> gets a csv file from a URL. Saves it into the local working directory as [churn_modelling.csv](https://raw.githubusercontent.com/dogukannulu/datasets/master/Churn_Modelling.csv). After reading the csv file, it writes it to a local PostgreSQL table

`create_df_and_modify.py` -> reads the same Postgres table and creates a pandas dataframe out of it, modifies it. Then, creates 3 separate dataframes.

`write_df_to_postgres.py` -> writes these 3 dataframes to 3 separate tables located in Postgres server. 



## Apache Airflow:

Run the following command to clone the necessary repo on your local

```bash
git clone https://github.com/dogukannulu/docker-airflow.git
```
After cloning the repo, run the following command only once:

```bash
docker build --rm --build-arg AIRFLOW_DEPS="datadog,dask" --build-arg PYTHON_DEPS="flask_oauthlib>=0.9" -t puckel/docker-airflow .
```

Then run the following command:

```bash
docker-compose -f docker-compose-LocalExecutor.yml up -d
```

Now you have a running Airflow container and you can reach out to that on `https://localhost:8080`. If there is `No module: ...` error, you can access to bash with the following command:

```bash
docker exec -it <container_id> /bin/bash 
```

Then:
```bash
pip install <necessary libraries>
```

After all these, we can move all .py files under dags folder in docker-airflow repo.

## PostgreSQL

I am using a local Postgres Server and installed `PgAdmin` to control the tables instead of using `psql`. I defined the necessary info to access to Postgres server in .zshrc (if MacOS).



# DETAILS PROCESS :
Tech Stack
Apache Airflow
PostgreSQL
Pandas
Python
Docker
SQL
Overview
In this article, we are going to get a CSV file from a remote repo, download it to the local working directory, create a local PostgreSQL table, and write this CSV data to the PostgreSQL table with write_csv_to_postgres.py script.

Then, we will get the data from the table. After some modifications and pandas practices, we will create 3 separate data frames with the create_df_and_modify.py script.

In the end, we will get these 3 data frames, create related tables in the PostgreSQL database, and insert the data frames into these tables with write_df_to_postgres.py

All these scripts will run as Airflow DAG tasks with the DAG script.

Think of this project as a practice of pandas and an alternative way of storing the data in the local machine.

Get Remote CSV Data
We have to first download PgAdmin to visually see the created tables and run SQL queries. Using DBeaver is another approach, we can also connect our PostgreSQL instance to DBeaver. Once prompted on the configuration page, we have to define:

Host
Database Name
User name
Password
Port
We will need all these parameters while connecting to the database. You may see the below article to be able to create a new database, user, and password. We are going to use localhost and the port will be 5432 (the default port for Postgres). We can connect to DBeaver using these parameters as well.

How to Create Database, User, and Access to PostgreSQL
In this article, I will explain how to access to our PostgreSQL server using psql via terminal. We should have psql…
medium.com

We need to install all the necessary libraries and packages in the requirements.txt file:

pip install -r requirements.txt
We need to import the necessary libraries and define the environmental variables. For MacOS, we have to define the env variables inside the .zshrc file to get them as credentials (to conclude, we have to add the command directories to the path independent of our OS). We can also define another config.json file.

After all, here comes the main part:

import psycopg2
import os
import traceback
import logging
import pandas as pd
import urllib.request

logging.basicConfig(level=logging.INFO,
                    format='%(asctime)s:%(funcName)s:%(levelname)s:%(message)s')


postgres_host = os.environ.get('postgres_host')
postgres_database = os.environ.get('postgres_database')
postgres_user = os.environ.get('postgres_user')
postgres_password = os.environ.get('postgres_password')
postgres_port = os.environ.get('postgres_port')
dest_folder = os.environ.get('dest_folder')

url = "https://raw.githubusercontent.com/dogukannulu/datasets/master/Churn_Modelling.csv"
destination_path = f'{dest_folder}/churn_modelling.csv'
We have to connect to the Postgres server.

try:
    conn = psycopg2.connect(
        host=postgres_host,
        database=postgres_database,
        user=postgres_user,
        password=postgres_password,
        port=postgres_port
    )
    cur = conn.cursor()
    logging.info('Postgres server connection is successful')
except Exception as e:
    traceback.print_exc()
    logging.error("Couldn't create the Postgres connection")
After connecting to the Postgres server, we will download the remote file into our working directory.

def download_file_from_url(url: str, dest_folder: str):
    """
    Download a file from a specific URL and download to the local direcory
    """
    if not os.path.exists(str(dest_folder)):
        os.makedirs(str(dest_folder))  # create folder if it does not exist

    try:
        urllib.request.urlretrieve(url, destination_path)
        logging.info('csv file downloaded successfully to the working directory')
    except Exception as e:
        logging.error(f'Error while downloading the csv file due to: {e}')
        traceback.print_exc()
Since the second target is uploading the CSV data into a Postgres table, we must create the table.

def create_postgres_table():
    """
    Create the Postgres table with a desired schema
    """
    try:
        cur.execute("""CREATE TABLE IF NOT EXISTS churn_modelling (RowNumber INTEGER PRIMARY KEY, CustomerId INTEGER, 
        Surname VARCHAR(50), CreditScore INTEGER, Geography VARCHAR(50), Gender VARCHAR(20), Age INTEGER, 
        Tenure INTEGER, Balance FLOAT, NumOfProducts INTEGER, HasCrCard INTEGER, IsActiveMember INTEGER, EstimatedSalary FLOAT, Exited INTEGER)""")
        
        logging.info(' New table churn_modelling created successfully to postgres server')
    except:
        logging.warning(' Check if the table churn_modelling exists')
After running the above function, we can check if the table is created successfully via pgAdmin or DBeaver. In the end, we have to insert all the data (pandas data frame) into the newly created table row by row.

def write_to_postgres():
    """
    Create the dataframe and write to Postgres table if it doesn't already exist
    """
    df = pd.read_csv(f'{dest_folder}/churn_modelling.csv')
    inserted_row_count = 0

    for _, row in df.iterrows():
        count_query = f"""SELECT COUNT(*) FROM churn_modelling WHERE RowNumber = {row['RowNumber']}"""
        cur.execute(count_query)
        result = cur.fetchone()
        
        if result[0] == 0:
            inserted_row_count += 1
            cur.execute("""INSERT INTO churn_modelling (RowNumber, CustomerId, Surname, CreditScore, Geography, Gender, Age, 
            Tenure, Balance, NumOfProducts, HasCrCard, IsActiveMember, EstimatedSalary, Exited) VALUES (%s, %s, %s,%s, %s, %s,%s, %s, %s,%s, %s, %s,%s, %s)""", 
            (int(row[0]), int(row[1]), str(row[2]), int(row[3]), str(row[4]), str(row[5]), int(row[6]), int(row[7]), float(row[8]), int(row[9]), int(row[10]), int(row[11]), float(row[12]), int(row[13])))

    logging.info(f' {inserted_row_count} rows from csv file inserted into churn_modelling table successfully')
Modify Data Frames
The second part includes retrieving the data from the Postgres table, modifying it, and creating 3 separate data frames from the main table. This part includes some pandas exercises. Some parts might seem a bit redundant, but they are all for practice. You can modify and change this part.

We have to first get the main data frame and modify it a bit (Assuming we already created the Postgres connection).

def create_base_df(cur):
    """
    Base dataframe of churn_modelling table
    """
    cur.execute("""SELECT * FROM churn_modelling""")
    rows = cur.fetchall()

    col_names = [desc[0] for desc in cur.description]
    df = pd.DataFrame(rows, columns=col_names)

    df.drop('rownumber', axis=1, inplace=True)
    index_to_be_null = np.random.randint(10000, size=30)
    df.loc[index_to_be_null, ['balance','creditscore','geography']] = np.nan
    
    most_occured_country = df['geography'].value_counts().index[0]
    df['geography'].fillna(value=most_occured_country, inplace=True)
    
    avg_balance = df['balance'].mean()
    df['balance'].fillna(value=avg_balance, inplace=True)

    median_creditscore = df['creditscore'].median()
    df['creditscore'].fillna(value=median_creditscore, inplace=True)

    return df
In the end, we will create 3 separate data frames out of the main table.

def create_creditscore_df(df):
    df_creditscore = df[['geography', 'gender', 'exited', 'creditscore']].groupby(['geography','gender']).agg({'creditscore':'mean', 'exited':'sum'})
    df_creditscore.rename(columns={'exited':'total_exited', 'creditscore':'avg_credit_score'}, inplace=True)
    df_creditscore.reset_index(inplace=True)

    df_creditscore.sort_values('avg_credit_score', inplace=True)

    return df_creditscore


def create_exited_age_correlation(df):
    df_exited_age_correlation = df.groupby(['geography', 'gender', 'exited']).agg({
    'age': 'mean',
    'estimatedsalary': 'mean',
    'exited': 'count'
    }).rename(columns={
        'age': 'avg_age',
        'estimatedsalary': 'avg_salary',
        'exited': 'number_of_exited_or_not'
    }).reset_index().sort_values('number_of_exited_or_not')

    return df_exited_age_correlation


def create_exited_salary_correlation(df):
    df_salary = df[['geography','gender','exited','estimatedsalary']].groupby(['geography','gender']).agg({'estimatedsalary':'mean'}).sort_values('estimatedsalary')
    df_salary.reset_index(inplace=True)

    min_salary = round(df_salary['estimatedsalary'].min(),0)

    df['is_greater'] = df['estimatedsalary'].apply(lambda x: 1 if x>min_salary else 0)

    df_exited_salary_correlation = pd.DataFrame({
    'exited': df['exited'],
    'is_greater': df['estimatedsalary'] > df['estimatedsalary'].min(),
    'correlation': np.where(df['exited'] == (df['estimatedsalary'] > df['estimatedsalary'].min()), 1, 0)
    })

    return df_exited_salary_correlation
Write the New Data Frames into the PostgreSQL
At this point, I am assuming we already established the Postgres connection, imported necessary libraries, and defined all variables. We have to first create 3 tables in the Postgres server.

def create_new_tables_in_postgres():
    try:
        cur.execute("""CREATE TABLE IF NOT EXISTS churn_modelling_creditscore (geography VARCHAR(50), gender VARCHAR(20), avg_credit_score FLOAT, total_exited INTEGER)""")
        cur.execute("""CREATE TABLE IF NOT EXISTS churn_modelling_exited_age_correlation (geography VARCHAR(50), gender VARCHAR(20), exited INTEGER, avg_age FLOAT, avg_salary FLOAT,number_of_exited_or_not INTEGER)""")
        cur.execute("""CREATE TABLE IF NOT EXISTS churn_modelling_exited_salary_correlation  (exited INTEGER, is_greater INTEGER, correlation INTEGER)""")
        logging.info("3 tables created successfully in Postgres server")
    except Exception as e:
        traceback.print_exc()
        logging.error(f'Tables cannot be created due to: {e}')
After this stage, we again have to check the successful creation of the tables. In the end, we are going to insert all 3 data frames into the Postgres tables.

def insert_creditscore_table(df_creditscore):
    query = "INSERT INTO churn_modelling_creditscore (geography, gender, avg_credit_score, total_exited) VALUES (%s,%s,%s,%s)"
    row_count = 0
    for _, row in df_creditscore.iterrows():
        values = (row['geography'],row['gender'],row['avg_credit_score'],row['total_exited'])
        cur.execute(query,values)
        row_count += 1
    
    logging.info(f"{row_count} rows inserted into table churn_modelling_creditscore")


def insert_exited_age_correlation_table(df_exited_age_correlation):
    query = """INSERT INTO churn_modelling_exited_age_correlation (Geography, Gender, exited, avg_age, avg_salary, number_of_exited_or_not) VALUES (%s,%s,%s,%s,%s,%s)"""
    row_count = 0
    for _, row in df_exited_age_correlation.iterrows():
        values = (row['geography'],row['gender'],row['exited'],row['avg_age'],row['avg_salary'],row['number_of_exited_or_not'])
        cur.execute(query,values)
        row_count += 1
    
    logging.info(f"{row_count} rows inserted into table churn_modelling_exited_age_correlation")


def insert_exited_salary_correlation_table(df_exited_salary_correlation):
    query = """INSERT INTO churn_modelling_exited_salary_correlation (exited, is_greater, correlation) VALUES (%s,%s,%s)"""
    row_count = 0
    for _, row in df_exited_salary_correlation.iterrows():
        values = (int(row['exited']),int(row['is_greater']),int(row['correlation']))
        cur.execute(query,values)
        row_count += 1

    logging.info(f"{row_count} rows inserted into table churn_modelling_exited_salary_correlation")
You may see the corresponding CSV files for the newly created tables via this link.

Airflow DAG
We have to automate the process a bit even though it is not complicated. We are going to do that using the Airflow DAGs.

We will be running Airflow as a Docker container. I used Puckel’s repo to run, special thanks to Puckel!

Run the following command to clone the necessary repo on your local

git clone https://github.com/dogukannulu/docker-airflow.git
After cloning the repo, run the following command only once so that all dependencies are configured and ready to use.

docker build --rm --build-arg AIRFLOW_DEPS="datadog,dask" --build-arg PYTHON_DEPS="flask_oauthlib>=0.9" -t puckel/docker-airflow .
You can use the below docker-compose.yaml file to run the Airflow as a Docker container.

import os
import sys
from airflow import DAG
from datetime import datetime, timedelta
from airflow.operators.python_operator import PythonOperator

parent_folder = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
sys.path.append(parent_folder)

from write_csv_to_postgres import write_csv_to_postgres_main
from write_df_to_postgres import write_df_to_postgres_main

start_date = datetime(2023, 1, 1, 12, 10)

default_args = {
    'owner': 'dogukanulu',
    'start_date': start_date,
    'retries': 1,
    'retry_delay': timedelta(seconds=5)
}

with DAG('csv_extract_airflow_docker', default_args=default_args, schedule_interval='@daily', catchup=False) as dag:

    write_csv_to_postgres = PythonOperator(
        task_id='write_csv_to_postgres',
        python_callable=write_csv_to_postgres_main,
        retries=1,
        retry_delay=timedelta(seconds=15))

    write_df_to_postgres = PythonOperator(
        task_id='write_df_to_postgres',
        python_callable=write_df_to_postgres_main,
        retries=1,
        retry_delay=timedelta(seconds=15))
    
    write_csv_to_postgres >> write_df_to_postgres 
docker-compose -f docker-compose-LocalExecutor.yml up -d
Now you have a running Airflow container and you can access the UI at https://localhost:8080. If some errors occur with the libraries and packages, we can go into the Airflow container and install them all manually.

docker exec -it <airflow_container_name> /bin/bash
curl -O <https://bootstrap.pypa.io/get-pip.py>
sudo yum install -y python3 python3-devel
python3 get-pip.py --user
pip3 install <list all necessary libraries here>
After running our Airflow DAG, we can see that the initial table is created inside our PostgreSQL server. Then, it is populated with the CSV data. After these, the data will be modified and uploaded into the newly created tables in the Postgres server. We can check all steps by connecting to DBeaver or via pgAdmin.
