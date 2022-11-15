### PG-Analyze

Step 1. Configure RDS Instance

In order to setup pganalyze monitoring for Amazon RDS or Amazon Aurora for Postgres you'll need to enable the pg_stat_statements extension for collecting query statistics.

Enabling pg_stat_statements
Connect to your database as an RDS superuser (usually the credentials you created the database with), e.g. using psql.

Run the following SQL commands to enable the extension, and make sure it was installed correctly:

    CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
    SELECT * FROM pg_stat_statements LIMIT 1;

Step 2. Create the monitoring user

We recommend creating a separate monitoring user on your PostgreSQL database for pganalyze.

#### Run these as the Postgres superuser
    CREATE USER pganalyze WITH PASSWORD 'pkypvBdEtalwgTBv' CONNECTION LIMIT 5;
    GRANT pg_monitor TO pganalyze;

    CREATE SCHEMA pganalyze;
    GRANT USAGE ON SCHEMA pganalyze TO pganalyze;
    GRANT USAGE ON SCHEMA public TO pganalyze;

    CREATE OR REPLACE FUNCTION pganalyze.get_stat_replication() RETURNS SETOF pg_stat_replication AS
    $$
    /* pganalyze-collector */ SELECT * FROM pg_catalog.pg_stat_replication;
    $$ LANGUAGE sql VOLATILE SECURITY DEFINER;

Then, connect to each database that you plan to monitor on this server as a superuser (or equivalent) and run the following to enable the collection of additional column statistics:

    CREATE SCHEMA IF NOT EXISTS pganalyze;
    GRANT USAGE ON SCHEMA pganalyze TO pganalyze;
    CREATE OR REPLACE FUNCTION pganalyze.get_column_stats() RETURNS SETOF pg_stats AS
    $$
    /* pganalyze-collector */ SELECT schemaname, tablename, attname, inherited, null_frac, avg_width,
    n_distinct, NULL::anyarray, most_common_freqs, NULL::anyarray, correlation, NULL::anyarray,
    most_common_elem_freqs, elem_count_histogram
    FROM pg_catalog.pg_stats;
    $$ LANGUAGE sql VOLATILE SECURITY DEFINER;


Step 3: Setup IAM Policy

We now need to set up an IAM policy and user that the collector can use to access RDS information.

To start, go to Create IAM policy, select JSON and then paste the following policy:

    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "cloudwatch:GetMetricStatistics"
            ],
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Action": [
                "logs:GetLogEvents"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:logs:*:*:log-group:RDSOSMetrics:log-stream:*"
        },
        {
            "Action": [
                "rds:DescribeDBParameters"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:rds:*:*:pg:*"
        },
        {
            "Action": [
                "rds:DescribeDBInstances",
                "rds:DownloadDBLogFilePortion",
                "rds:DescribeDBLogFiles"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:rds:*:*:db:*"
        },
        {
            "Action": [
                "rds:DescribeDBClusters"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:rds:*:*:cluster:*"
        }
    ]
}

This policy grants the following access:

1) RDS metadata used to discover general instance information
2) Cloudwatch metrics to show CPU utilization and other system metrics in pganalyze
3) RDS log file download (for pganalyze Log Insights)

Create an IAM role for pganalyze, and assign the policy to this new role.

Step 4: Install and configure collector

### Deploying pganalyze-collector for postgres monitoring

```bash
NAMESPACE="monitoring"
PG_ANALYZE_USER="pganalyze"
PG_ANALYZE_USER_PASSWORD=""
PG_ANALYZE_API_KEY=""
DB_HOST=""

kubectl create secret generic pganalyze-collector-secrets \
    --from-literal=DB_PASSWORD=${PG_ANALYZE_USER_PASSWORD} \
    --from-literal=PGA_API_KEY=${PG_ANALYZE_API_KEY} \
    --namespace ${NAMESPACE}

helm install pganalyze-collector-aurora ./pganalyze-collector \
    -f ./pganalyze-collector/values.yaml \
    --namespace ${NAMESPACE} \
    --set env.DB_USERNAME=$PG_ANALYZE_USER \
    --set env.DB_HOST=$DB_HOST
```
