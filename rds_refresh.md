Automating the refresh of an Oracle RDS database from one AWS account to another involves several steps and the use of AWS services like AWS Backup, AWS Lambda, and AWS Step Functions. 

Option1:

High level steps: 

1. Set Up AWS Backup:
    * Enable automated backups for your Oracle RDS instance in the source account.
    * Configure cross-account backup policies to copy snapshots to the destination account.
2. Create AWS Lambda Functions:
    * Write Lambda functions to automate the snapshot creation, copying, and restoration processes.
    * Use AWS SDKs (e.g., Boto3 for Python) to interact with RDS and AWS Backup APIs.
3. Set Up AWS Step Functions:
    * Create a state machine in AWS Step Functions to orchestrate the workflow.
    * Define states for creating snapshots, copying snapshots, and restoring them in the destination account.
4. Configure EventBridge:
    * Use Amazon EventBridge to trigger the Lambda functions based on scheduled events or specific conditions.
    * Set up rules to handle the lifecycle of snapshots and database refreshes.
5. Implement Security Measures:
    * Use AWS Key Management Service (KMS) to encrypt snapshots and ensure secure data transfer.
    * Share KMS keys between accounts to enable cross-account access to encrypted snapshots.

Please go through the solution outlined [1] and you can automate this using the solution outlined in [2] using Amazon EventBridge and AWS Lambda. 

[1] https://aws.amazon.com/blogs/database/automate-cross-account-backup-of-amazon-rds-for-oracle-including-database-parameter-groups-option-groups-and-security-groups/ 
[2] https://aws.amazon.com/blogs/database/automate-cross-account-backups-of-amazon-rds-and-amazon-aurora-databases-with-aws-backup/ 

Option2:

AWS Marketplace tool: Aurora and RDS Automated Database Refresh from CirrusHQ
[+]https://aws.amazon.com/marketplace/pp/prodview-kpjltdtykfsj2 
Description: CircuHq  Automated Database Refresh for Aurora and RDS automatically migrates Aurora and RDS database snapshots from a production AWS Account to a non-production AWS Account to securely refresh the non-production environment with production-level data. It provides all infrastructure components by utilizing Infrastructure as Code (IaC) and a CI/CD CodePipeline to deploy an AWS Step Function State Machine to orchestrate database snapshot copy, re-encryption, restore and cleanup of the non-production database with zero impact to the production database. 
Optionally executed on a schedule to periodically refresh non-production databases.
