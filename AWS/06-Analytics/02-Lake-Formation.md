# AWS Lake Formation

- Built on top of Glue.
- Makes it easy to setup and secure data lake.
- Loading data and monitoring data flows.
- Setting up partitions.
- Encryption and managing keys.
- Defining transformation jobs and monitoring them.
- Access control.
- Auditing.

## Pricing

- No cost for Lake Formation itself.
- But underlying services incur charges -
  - Glue
  - S3
  - EMR
  - Athena
  - Redshift

## Steps for Building a Data Lake

- Create an IAM user for Data Analyst role.
- Create AWS Glue connection to your data source(s).
- Create S3 bucket for the lake.
- Register the S3 path in Lake Formation, and grant permissions.
- Create database in Lake Formation for data catalog, and grant permissions.
- Use a blueprint for a workflow (i.e. database snapshot).
- Run the workflow.
- Grant SELECT permissions to whoever needs to read it (Athena, Redshit Spectrum etc).

## Deep dive

- Cross-account Lake Formation permissions -
  - Recipient must be setup as a data lake administrator.
  - Can use AWS Resource Access Manager for accounts external to your organization.
  - IAM permissions for cross-account access.
- Lake Formation does not support manifests in Athena or Redshift queries.
- IAM permissions on the KMS encryption key are needed for encrypted data catalogs in Lake Formation.
- IAM permissions needed to create blueprints and workflows.

## Governed Tables and Security

- Now supports "Goverened Tables" that support ACID transactions across multiple tables.
  - New type of S3 table.
  - Can't change choice of goverened afterwards.
  - Works with streaming data (Kinesis).
  - Can query with Athena.
- Storage optimization with automatic compaction.
- Granular access control with Row and Cell-level security - both for governed and S3 tables.
- Above features incur additional charges based on usage.
