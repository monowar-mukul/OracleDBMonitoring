Terraform script that provisions an EC2 instance if the provider is AWS and an Azure VM if the provider is Azure.

### Terraform Script

```markdown
# Terraform Script to Create AWS EC2 or Azure VM Based on Provider Selection

## Provider Configuration
# Specifies the provider to use for provisioning resources.
```hcl
variable "provider" {
  description = "The cloud provider to use (aws or azure)"
  type        = string
  default     = "aws"
}

provider "aws" {
  region = "us-west-2"
  count  = var.provider == "aws" ? 1 : 0
}

provider "azurerm" {
  features {}
  count = var.provider == "azure" ? 1 : 0
}
```

## AWS EC2 Configuration
# Creates an EC2 instance in AWS.
```hcl
resource "aws_instance" "aws_ec2_instance" {
  count           = var.provider == "aws" ? 1 : 0
  ami             = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2 AMI
  instance_type   = "t2.micro"
  key_name        = "your-key-pair"  # Replace with your key pair name

  tags = {
    Name = "aws-ec2-instance"
  }

  ## User Data Script
  # A script that runs on instance startup to install necessary packages.
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              EOF
}

output "aws_instance_id" {
  value = aws_instance.aws_ec2_instance.id
}

output "aws_public_ip" {
  value = aws_instance.aws_ec2_instance.public_ip
}
```

## Azure VM Configuration
# Creates a VM in Azure.
```hcl
resource "azurerm_resource_group" "azure_resource_group" {
  count    = var.provider == "azure" ? 1 : 0
  name     = "azure-resource-group"
  location = "West US"
}

resource "azurerm_virtual_network" "azure_vnet" {
  count     = var.provider == "azure" ? 1 : 0
  name      = "azure-vnet"
  address_space = ["10.0.0.0/16"]
  location  = azurerm_resource_group.azure_resource_group.location
  resource_group_name = azurerm_resource_group.azure_resource_group.name
}

resource "azurerm_subnet" "azure_subnet" {
  count     = var.provider == "azure" ? 1 : 0
  name      = "azure-subnet"
  resource_group_name  = azurerm_resource_group.azure_resource_group.name
  virtual_network_name = azurerm_virtual_network.azure_vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_network_interface" "azure_nic" {
  count               = var.provider == "azure" ? 1 : 0
  name                = "azure-nic"
  location            = azurerm_resource_group.azure_resource_group.location
  resource_group_name = azurerm_resource_group.azure_resource_group.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.azure_subnet.id
    private_ip_address_allocation = "Dynamic"
  }
}

resource "azurerm_virtual_machine" "azure_vm" {
  count               = var.provider == "azure" ? 1 : 0
  name                = "azure-vm"
  location            = azurerm_resource_group.azure_resource_group.location
  resource_group_name = azurerm_resource_group.azure_resource_group.name
  network_interface_ids = [azurerm_network_interface.azure_nic.id]
  vm_size             = "Standard_B1s"

  storage_os_disk {
    name              = "osdisk"
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Standard_LRS"
  }

  storage_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = "18.04-LTS"
    version   = "latest"
  }

  os_profile {
    computer_name  = "hostname"
    admin_username = "adminuser"
    admin_password = "Password1234!"
  }

  os_profile_linux_config {
    disable_password_authentication = false
  }

  tags = {
    Name = "azure-vm"
  }
}

output "azure_vm_id" {
  value = azurerm_virtual_machine.azure_vm.id
}

output "azure_public_ip" {
  value = azurerm_network_interface.azure_nic.private_ip_address
}
```

### Notes
- Replace `"your-key-pair"` with your actual key pair name for the AWS EC2 instance.
- Adjust the configuration as needed for your environment.

This script sets up an EC2 instance if the provider is AWS and a VM if the provider is Azure. 
