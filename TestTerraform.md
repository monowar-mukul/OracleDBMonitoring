```markdown
# Terraform Script to Provision an EC2 Instance and Install Oracle Database 19c

## Provider Configuration
# Specifies the AWS region to use for provisioning resources.
```hcl
provider "aws" {
  region = "us-west-2"  # Change to your preferred region
}
```

## EC2 Instance Resource
# Provisions an EC2 instance using the Amazon Linux 2 AMI.
# The instance type is set to `t2.medium`.
# Replace `"your-key-pair"` with your actual key pair name.
# Tags the instance with the name "OracleDBInstance".
```hcl
resource "aws_instance" "oracle_db" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2 AMI
  instance_type = "t2.medium"

  key_name = "your-key-pair"  # Replace with your key pair name

  tags = {
    Name = "OracleDBInstance"
  }

  ## User Data Script
  # A script that runs on instance startup to:
  # - Update the system.
  # - Install necessary packages (`wget` and `unzip`).
  # - Download and unzip the Oracle Database 19c installation files.
  # - Run the Oracle installer in silent mode.
  # - Create a database using the Database Configuration Assistant (DBCA).
  # - Check the status of the Oracle listener and start it if it is not running.
  # - Check if the database is up and running using SQL.
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y wget unzip
              # Download and install Oracle Database 19c
              wget https://example.com/path/to/oracle19c.zip -O /tmp/oracle19c.zip
              unzip /tmp/oracle19c.zip -d /tmp/oracle19c
              /tmp/oracle19c/runInstaller -silent -responseFile /tmp/oracle19c/response/db_install.rsp
              # Create a database
              /u01/app/oracle/product/19.0.0/dbhome_1/bin/dbca -silent -createDatabase -responseFile /tmp/oracle19c/response/dbca.rsp
              # Check Oracle listener status and start if not running
              LISTENER_STATUS=$(/u01/app/oracle/product/19.0.0/dbhome_1/bin/lsnrctl status | grep -i "listener is not running")
              if [ -n "$LISTENER_STATUS" ]; then
                /u01/app/oracle/product/19.0.0/dbhome_1/bin/lsnrctl start
              fi
              # Check if the database is up and running
              su - oracle -c "echo 'SELECT status FROM v\\$instance;' | sqlplus -s / as sysdba" | grep -i "OPEN"
              if [ $? -ne 0 ]; then
                echo "Database is not running"
              else
                echo "Database is up and running"
              fi
              EOF
}
```

## Outputs
# Outputs the instance ID and public IP of the provisioned EC2 instance.
```hcl
output "instance_id" {
  value = aws_instance.oracle_db.id
}

output "public_ip" {
  value = aws_instance.oracle_db.public_ip
}
```

### Notes
- Replace `"your-key-pair"` with your actual key pair name.
- Update the URL in the `wget` command to point to your Oracle Database 19c installation files.
- Ensure the response files (`db_install.rsp` and `dbca.rsp`) are correctly configured for your environment.

Feel free to ask if you need further customization or have any questions!
```
