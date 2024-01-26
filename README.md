
# Accessing CloudSQL with Private Service Connect enabled from Cloud Run

This approach ensures that your database connections remain private and isolated, reducing the risk of unauthorized access and meeting stringent regulatory requirements. It also offers improved performance, lower latency, and cost savings, while giving you greater control over network access policies. By leveraging Private Service Connect, you can build robust and reliable systems that prioritize data privacy and network efficiency, making it an essential practice in modern cloud development.

## Environment Variables

Assuming your using Cloud Shell or a terminal on your PC with Google Cloud SDK installed

`export PROJECT_ID=$(gcloud config get-value project)`

`export PROJECT_USER=$(gcloud config get-value core/account)`

`export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")`

`export IDNS=${PROJECT_ID}.svc.id.goog`

`export GCP_REGION="us-central1" `

`export GCP_ZONE="us-central1-a"`

`export NETWORK_NAME="default"`

`export SQL_NETWORK_NAME="cloudsql"`

`export SUBNET_NAME="sql-psc"`

`export DB_INSTANCE_NAME="instance01"`

`export DB_INSTANCE_VERSION="MYSQL_8_0"`

`export CONNECTOR=run2csql`



## APIs

Enabled the APIs for your project
```bash
  gcloud services enable compute.googleapis.com \
    storage.googleapis.com \
    sqladmin.googleapis.com \
    dns.googleapis.com
```
    
## Create VPC for CloudSQL MySQL

Create a network

```bash
  gcloud compute networks create $SQL_NETWORK_NAME --subnet-mode=auto
```

Default firewall rules

```bash
gcloud compute firewall-rules create ${SQL_NETWORK_NAME}-allow-internal \
    --action=ALLOW \
    --direction=INGRESS \
    --network=${SQL_NETWORK_NAME} \
    --priority=1000 \
    --rules=tcp:0-65535,udp:0-65535,icmp \
    --source-ranges=10.128.0.0/9
```

```bash
gcloud compute firewall-rules create ${SQL_NETWORK_NAME}-allow-ssh \
    --action=ALLOW \
    --direction=INGRESS \
    --network=${SQL_NETWORK_NAME} \
    --priority=1000 \
    --rules=tcp:22 \
    --source-ranges=0.0.0.0/0
```

```bash
gcloud compute firewall-rules create ${SQL_NETWORK_NAME}-allow-icmp \
    --action=ALLOW \
    --direction=INGRESS \
    --network=${SQL_NETWORK_NAME} \
    --priority=1000 \
    --rules=icmp \
    --source-ranges=0.0.0.0/0
```

```bash
gcloud compute firewall-rules create ${SQL_NETWORK_NAME}-allow-db \
    --action=ALLOW \
    --direction=INGRESS \
    --network=${SQL_NETWORK_NAME} \
    --priority=1000 \
    --rules=tcp:5432 \
    --source-ranges=10.10.0.0/24
```

Add a subnet
```bash
gcloud compute networks subnets create $SUBNET_NAME \
    --network=$SQL_NETWORK_NAME \
    --range=10.10.0.0/24 \
    --region=$GCP_REGION
```
## Creating CloudSQL MySQL

```bash
gcloud sql instances create $DB_INSTANCE_NAME \
    --project=$PROJECT_ID \
    --region=$GCP_REGION \
    --enable-private-service-connect \
    --allowed-psc-projects=$PROJECT_ID \
    --availability-type=regional \
    --no-assign-ip \
    --cpu=2 \
    --memory=7680MB \
    --database-version=$DB_INSTANCE_VERSION
    --ebable-bin-log 
```

Change root password
```bash
gcloud sql users set-password root \
    --instance=$DB_INSTANCE_NAME \
    --prompt-for-password
```

## Get the service attachment URI

```bash
SVC_ATTACH=$(gcloud sql instances describe $DB_INSTANCE_NAME --format='value(pscServiceAttachmentLink)')
```

## Create PSC endpoint for VPC

```bash
gcloud compute addresses create psc-sql-ep \
    --project=$PROJECT_ID \
    --region=$GCP_REGION \
    --subnet=$SUBNET_NAME \
    --purpose=GCE_ENDPOINT
```

Verify the address and take note of it as it'll be used in creating DNS record

```bash
gcloud compute addresses list --filter="name=( 'NAME' 'psc-sql-ep')" --project=$PROJECT_ID
```

Create forwarding-rule
```bash
gcloud compute forwarding-rules create forward-sql-rule-1 \
    --project=$PROJECT_ID \
    --region=$GCP_REGION \
    --network=$SQL_NETWORK_NAME \
    --target-service-attachment=$SVC_ATTACH \
    --address=psc-sql-ep
```
List PSC endpoint to validate
```bash
gcloud compute forwarding-rules list --filter="name=( 'NAME' 'forward-sql-rule-1')" \
    --regions=$GCP_REGION
```

## Configure DNS managed zone
Get Cloud SQL DNS name
```bash
SQL_DNS_NAME=$(gcloud sql instances describe $DB_INSTANCE_NAME --project=$PROJECT_ID --format='value(dnsName)')
```

Create DNS zone in each VPC
```bash 
gcloud dns managed-zones create sql-zone \
    --project=$PROJECT_ID \
    --description="zone to connect to cloud sql in different vpc" \
    --dns-name=$SQL_DNS_NAME \
    --networks=$SQL_NETWORK_NAME \
    --visibility=private
```

Create DNS record for zone. IP address is from step above
```bash
gcloud dns record-sets create $SQL_DNS_NAME \
    --project=$PROJECT_ID \
    --type=A \
    --rrdatas=10.10.0.4 \
    --zone=sql-zone
```

## VPC connector to send Cloud Run request through
```bash
gcloud compute networks vpc-access connectors create $CONNECTOR \
    --region=$GCP_REGION \    
    --network=$SQL_NETWORK_NAME \
    --range=10.9.0.0/28 \
    --min-instances=2 \
    --max-instances=10 \
    --machine-type=f1-micro
```

At this point your CloudSQL should be up and running with Private Service Connect enabled
## Deploying Cloud Run with VPC Connector

My current directory structure

```bash
.
├── Dockerfile
├── app.py
├── requirements.txt
```

Dockerfile
```bash
FROM python:3.10-slim

ENV PYTHONUNBUFFERED True

WORKDIR /var/www/html
COPY . /var/www/html/

RUN pip install --no-cache-dir -r requirements.txt

CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 app:app
```

My requirements.txt file
```bash
Flask==2.0.2
gunicorn==20.1.0
Werkzeug==2.0.2
cloud-sql-python-connector
pymysql
sqlalchemy
```

My app.py file - For Testing purposes we added the password in the code but should [Secret Manager](https://cloud.google.com/security/products/secret-manager) in production
```python
from flask import Flask, jsonify
from google.cloud.sql.connector import Connector, IPTypes
import pymysql
import sqlalchemy
import os

app = Flask(__name__)

@app.route('/')
def index():
    return "Welcome to the Flask App!"

@app.route('/query')
def query_db():
    # Initialize Cloud SQL Connector
    connector = Connector()

    def init_connection_pool(connector: Connector) -> sqlalchemy.engine.Engine:
        # Function to create a new connection
        def getconn() -> pymysql.connections.Connection:
            conn = connector.connect(
                "your-project-name:us-central1:instance01",
                "pymysql",
                ip_type=IPTypes.PSC,
                user="alex",
                password="passwd01", #Use Secret Manager in production
                db="mysql"
            )
            return conn

        # Create a connection pool
        pool = sqlalchemy.create_engine(
            "mysql+pymysql://",
            creator=getconn,
        )
        return pool

    # Initialize connection pool
    pool = init_connection_pool(connector)

    with pool.connect() as db_conn:
        try:
            result = db_conn.execute(sqlalchemy.text("SELECT user, host FROM user")).fetchall()
            response_data = [{'user': row[0], 'host': row[1]} for row in result]
            return jsonify(response_data)  # Return a JSON response
        except Exception as e:
            print("==========", e)
            return jsonify({'error': 'An error occurred'}), 500  # Return an error JSON response with HTTP status 500

if __name__ == '__main__':
    app.run(debug=True, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

Build the image container
```bash
gcloud builds submit --tag gcr.io/$PROJECT_ID/run2csqlpsc
```

Deploy Cloud Run
```bash
gcloud run deploy run2csqlpsc \
     --image gcr.io/$PROJECT_ID/run2csqlpsc \
     --allow-unauthenticated \
     --region $GCP_REGION \
     --platform managed \
     --max-instances 1 \
     --vpc-connector $CONNECTOR
```

Calling the Cloud Run URL followed by /query [https://---.a.run.app/query] should return something like:

[{"host":"","user":"root"},{"host":"%","user":"cloudiamgroup"},{"host":"%","user":"cloudsqlreplica"},{"host":"%","user":"cloudsqlsuperuser"},{"host":"%","user":"root"},{"host":"%","user":"root@cloudsqlproxy~10.230.7.2"},{"host":"127.0.0.1","user":"cloudsqlexport"},{"host":"127.0.0.1","user":"cloudsqlimport"},{"host":"127.0.0.1","user":"cloudsqloneshot"},{"host":"127.0.0.1","user":"root"},{"host":"::1","user":"cloudsqlexport"},{"host":"::1","user":"cloudsqlimport"},{"host":"::1","user":"cloudsqloneshot"},{"host":"::1","user":"root"},{"host":"cloudsqlproxy~10.230.7.2","user":"alex"},{"host":"localhost","user":"cloudsqlapplier"},{"host":"localhost","user":"cloudsqlimport"},{"host":"localhost","user":"cloudsqlobservabilityadmin"},{"host":"localhost","user":"mysql.infoschema"},{"host":"localhost","user":"mysql.session"},{"host":"localhost","user":"mysql.sys"},{"host":"localhost","user":"root"}]

Which is the result of the query defined in our app.py file

## Acknowledgements

 - I would like to extend my heartfelt gratitude to Aarmir Haroon for his invaluable assistance. Your support has been immensely appreciated.



## Documentation

[Configure Private Service Connect](https://cloud.google.com/sql/docs/postgres/configure-private-service-connect)

