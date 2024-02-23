# # AWS-Lambda-Python-SQLAlchemy-RDS
Simple connection to a public RDS with SQLAlchemy and Secrets Manager. NOTE this code expects that you created RDS using a secret.

## Code
```py
import sqlalchemy as sa
from sqlalchemy import create_engine, MetaData, Table, Column, Integer, String
import boto3
import urllib.parse
import json
from botocore.exceptions import ClientError

def get_secret(secret_name, region_name):
    # Create a Secrets Manager client
    session = boto3.session.Session()
    client = session.client(service_name='secretsmanager', region_name=region_name)
    try:
        get_secret_value_response = client.get_secret_value(SecretId=secret_name)
    except ClientError as e:
        raise e
    else:
        return json.loads(get_secret_value_response['SecretString'])

def lambda_handler(event, context):
    # Get database password from AWS Secrets Manager
    secret_name = "your_secret_name"
    region_name = "us-east-1"
    db_secret = get_secret(secret_name, region_name)

    # Construct the database connection string
    db_username = db_secret["username"]
    db_password = urllib.parse.quote_plus(db_secret["password"]) # secrets use special chars
    db_host = "your_db_host"
    db_port = "3306"
    db_database = "your_database"
    db_string = f"mysql+pymysql://{db_username}:{db_password}@{db_host}:{db_port}/{db_database}"

    # Create the database engine and connect
    engine = create_engine(db_string)

    # Create a Hello World table (if it doesn't exist)
    metadata = MetaData()
    if not engine.dialect.has_table(engine, "hello_world"):
        hello_world = Table('hello_world', metadata,
                            Column('id', Integer, primary_key=True),
                            Column('message', String(255))
                            )
        metadata.create_all(engine)

    return {
        'statusCode': 200,
        'body': 'Hello World table created or already exists!'
    }
```

## Inline Policy

Create this inline policy for the AWS Lambda Role.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "secretsmanager:GetSecretValue",
            "Resource": "arn:aws:secretsmanager:region:account-id:secret:secret-name"
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt",
                "kms:DescribeKey"
            ],
            "Resource": "arn:aws:kms:region:account-id:key/key-id"
        }
    ]
}
```

## Conclusion
Enjoy SQLAlchemy, the best Python SQL ORM, in your Lambda functions! 
