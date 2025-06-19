### **Task 1. Create a BigQuery AWS connection**
BigQuery Omni accesses Amazon S3 data through authorized connections from Google Cloud. Each connection has its own unique Amazon Web Services (AWS) Identity and Access Management (IAM) user. 
You grant permissions to users through AWS IAM roles. The policies within the AWS IAM roles determine what data BigQuery can access for each connection.

![Screenshot 2025-06-19 9 07 01 AM - Display 1](https://github.com/user-attachments/assets/4e0db186-3ac3-43c0-af43-29060235e1cd)


```Note: In this lab, you will be using both Google Cloud and AWS consoles, and switch between the two tabs/windows based on task descriptions.```

**Create an AWS IAM policy for BigQuery**
Sign in to the AWS Management Console. Click the Open AWS Console button on the lab pane, and log in with the provided username and password.
![Screenshot 2025-06-19 9 15 59 AM](https://github.com/user-attachments/assets/1f2d0f48-efb1-4f5b-8d00-6e4ec3109de5)

![Screenshot 2025-06-19 9 20 05 AM](https://github.com/user-attachments/assets/53901232-cffd-4897-a92a-40b83193cdc5)


Search for Amazon S3 in the Search bar at the top and select S3. A regional bucket named ```qwiklabs-gcp-01-106feaa24b71``` with data pre-populated is already available for this lab.
Copy this bucket name for subsequent steps.

Search for AWS Identity and Access Management (IAM) in the Search bar at the top and select IAM.

From the left pane select Policies and click bigquery-omni-connection-policy.

Click Edit policy > JSON and paste the following into the editor. Replace all instances of <BUCKET_NAME> with your S3 bucket name copied from Step 2.
```json
{
    "Statement": [
        {
            "Action": [
                "s3:ListBucket"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::<BUCKET_NAME>"
            ],
            "Sid": "BucketLevelAccess"
        },
        {
            "Action": [
                "s3:GetObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::<BUCKET_NAME>",
                "arn:aws:s3:::<BUCKET_NAME>/*"
            ],
            "Sid": "ObjectLevelGetAccess"
        },
        {
            "Action": [
                "s3:PutObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::<BUCKET_NAME>",
                "arn:aws:s3:::<BUCKET_NAME>/*"
            ],
            "Sid": "ObjectLevelPutAccess"
        }
    ],
    "Version": "2012-10-17"
}
```
Click Save changes.
![Screenshot 2025-06-19 9 10 07 AM](https://github.com/user-attachments/assets/ea8bc3f4-d730-4b98-844c-7fa08a8cde21)

Validate the AWS IAM for BigQuery
From the left pane, select Roles.

Click the bigquery-omni-connection role.
![Screenshot 2025-06-19 9 15 06 AM](https://github.com/user-attachments/assets/fd9e91b7-65ef-444f-b41f-8b5ee073fa7f)

Copy the Role ARN, it will be in the following format, where <ACCOUNT_ID> is your AWS Account ID.

```arn:aws:iam::<ACCOUNT_ID>:role/bigquery-omni-connection```


Create the BigQuery AWS connection
In the Google Cloud Console, from the Navigation Menu, go to BigQuery > BigQuery Studio.

Click +ADD, then select Connections to external data sources.


``` Note: Alternatively, if you do not see the option for + Add followed by Connections to external data sources,
you can click + Add data, and then use the search bar for data sources to search for Vertex AI. 
Click on the result for Vertex AI.
```
In the External data source pane, enter the following information:
```
For Connection type, select BigLake on AWS (via BigQuery Omni).
For Connection ID, type bq-omni-aws-connector for an identifier for the connection resource.
For Connection location, select aws-us-east-1.
Optional: For Friendly name, enter a user-friendly name for the connection. The friendly name can be any value that helps you identify the connection resource if you need to modify it later.
Optional: For Description, enter a description for this connection resource.
For AWS role id, enter the full IAM Role ARN that you copied in the previous step in this format: arn:aws:iam::AWS_ACCOUNT_ID:role/ROLE_NAME
Click Create connection.
```
In the BigQuery Explorer, click the dropdown next to your project name and navigate to the newly created connection in the External Connections list.

bigquery connection resource info
![Screenshot 2025-06-19 9 15 37 AM](https://github.com/user-attachments/assets/22035cee-4253-45dd-8a3b-d8a05ead9dcd)
Note the BigQuery Google identity. This is a Google principle that is specific to each connection. Copy this BigQuery Google identity, it will be used in the next section.
Your BigQuery Google Identity should resemble the following:

BigQuery Google identity: ```114999259451445753095```
Click Check my progress to verify the objective.

**Create the BigQuery AWS connection**

Add a Trust Relationship to the AWS role
The trust relationship lets the BigQuery AWS connection assume the role and access the S3 data as specified in the roles policy.

Navigate back to the AWS IAM console.

From the left pane, select Roles.
![Screenshot 2025-06-19 9 15 59 AM](https://github.com/user-attachments/assets/69518b88-64fe-4288-b81c-1b508bc194a9)

Select the bigquery-omni-connection role.

Click Edit and then do the following:
```
Verify if Maximum session duration is set to 12 hours. As each query can run for up to six hours, this duration allows for one additional retry. 
Increasing the session duration beyond 12 hours will not allow for additional retries. 
For more information, see the query/multi-statement query execution-time limit.
```
Click Save changes.
Select Trust Relationships tab and click Edit trust policy.

Replace the policy content with the following, replacing "00000" with the BigQuery Google identity you copied in the previous section.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "accounts.google.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "accounts.google.com:sub": "12672533623633"
        }
      }
    }
  ]
}
```
Click Update Policy.
The connection is now ready to use.
![Screenshot 2025-06-19 9 20 05 AM](https://github.com/user-attachments/assets/e324717f-b418-4567-bb37-576081763cae)

Note: There may be a propagation delay for role assignment in AWS. If you receive an error of this type when using a new connection, waiting and trying again later may resolve the issue.

### **Task 2. Run queries on the AWS S3 external table**
BigQuery Omni does not manage data stored in Amazon S3. To access S3 data, define an external table. This table is called an external table because the data is not stored in BigQuery managed storage. For more information about external tables, see External tables.

**Create a BigQuery dataset**
In this section, you will create a BigQuery dataset in the same region as your AWS S3 bucket.

In the Google Cloud Console, go to the BigQuery page to create a dataset.

Click the 3 dots next to your project name and select Create dataset.

On the Create dataset page, enter the following information:

For Dataset ID, enter ```bq_omni_demo```.
For Location type, select Region.
For Data location, choose ```aws-us-east-1```.
Click Create dataset.
![Screenshot 2025-06-19 9 21 25 AM](https://github.com/user-attachments/assets/91fd1bdf-d0da-4291-9ffe-afb10b69a499)

Create an external table
In this section you will create an external table in the above dataset.

In this BigQuery explorer, expand your project and select the bq_omni_demo dataset created.

In the details panel, click Create table.

On the Create table page, in the Source section, do the following:

For Create table from, select **Amazon S3**.
For Select S3 path, enter ```s3://[S3 bucket name]/taxi-data_green_trips_table.csv```.
Replace ```[S3 bucket name]``` with ```qwiklabs-gcp-01-106feaa24b71```.
For File format, ```select CSV```.
Note: supported formats are ```AVRO, PARQUET, ORC, CSV, NEWLINE_DELIMITED_JSON, and Google Sheets```.
Important: make sure to remove any spaces in the S3 path as this will cause errors!
On the Create table page, in the Destination section, do the following:

For Dataset name, choose ```bq_omni_demo```.
In the Table name field, use ```bq-omni-table```.
Verify that Table type is set to External table.
For Connection ID, choose the appropriate Connection ID from the dropdown.
In the Schema section, select the Auto detect checkbox.
Click Create table.
![Screenshot 2025-06-19 9 23 38 AM](https://github.com/user-attachments/assets/5d8690c6-292e-4b63-a735-28741e8d0f69)

Note: You do not need to specify table schema since it is autodetected from the source file.
Click Check my progress to verify the objective.

![Screenshot 2025-06-19 9 24 02 AM](https://github.com/user-attachments/assets/9267e02e-0e6f-4789-9002-52ecced9f42b)

Create the BigQuery Dataset and External Table


### **Task 3. Create an external table and query AWS S3 data**
```
BigQuery Omni lets you query the external table like any BigQuery table. The maximum result size for interactive queries is 10 GB (preview).
For more information, see Limitations. If your query result is larger than 10 GB, 
then we recommend that you export it to Amazon S3. The query result is stored in a BigQuery temporary table.
```
**Query the external table**
From the bq-omni-table details page, click on Query a new editor will open.

In the Query editor, execute the following statement:
```mysql
SELECT * FROM `qwiklabs-gcp-01-106feaa24b71.bq_omni_demo.bq-omni-table`
```
Click Run.
You should see the following output:
![Screenshot 2025-06-19 9 26 05 AM](https://github.com/user-attachments/assets/0702ecf9-6776-4655-b951-d25b0ebd60a1)


Click Check my progress to verify the objective.

Query the External Table


### **Task 4. Export query results to AWS S3**
BigQuery Omni lets you export the result of a query against a BigQuery external table to Amazon S3.

**Export Query Results**
BigQuery Omni writes to the specified Amazon S3 location irrespective of any existing content. The export query can overwrite existing data or mix the query result with existing data. In the Query editor field, you will need to run a Google Standard SQL export query. Google Standard SQL is the default syntax in the Google Cloud console. The following is the template for what you will need to write:

```bigquery
EXPORT DATA WITH CONNECTION `CONNECTION_REGION.CONNECTION_NAME` \
OPTIONS(uri="s3://BUCKET_NAME/PATH", format="FORMAT", ...) \
AS QUERY
```
You will need to replace the following:
```
CONNECTION_REGION: the region where the connection was created.
CONNECTION_NAME: the connection name that you created with the necessary permission to write to the S3 bucket.
BUCKET_NAME: the Amazon S3 bucket where you want to write the data.
PATH: the path where you want to write the exported file to
FORMAT: supported formats are JSON, AVRO, and CSV.
QUERY: the query to analyze the data that is stored in a BigQuery external table.
```
For this lab, the query has been pre-populated for you. Paste this query into the editor:

```mysql
EXPORT DATA WITH CONNECTION `aws-us-east-1.bq-omni-aws-connector`
OPTIONS(uri="s3://qwiklabs-gcp-01-106feaa24b71/exports/*", format="CSV")
AS SELECT * FROM `qwiklabs-gcp-01-106feaa24b71.bq_omni_demo.bq-omni-table`
```
Click Run.
![Screenshot 2025-06-19 9 43 35 AM](https://github.com/user-attachments/assets/1608050e-cd49-4fbc-856c-c036ed62ab7a)

You should see the following output:

```Successfully exported 15417 rows into 1 files.```

Navigate to your S3 bucket and verify the data has been exported in the exports directory.
AWS console export query to s3 bucket
![Screenshot 2025-06-19 9 44 36 AM](https://github.com/user-attachments/assets/a3196d6b-4952-4284-96e3-a95506dd666c)

![Screenshot 2025-06-19 9 44 46 AM](https://github.com/user-attachments/assets/1b24c4e2-20db-4908-8beb-147033ba7349)


Great! You have successfully executed an export query and created a file in your S3 bucket.
