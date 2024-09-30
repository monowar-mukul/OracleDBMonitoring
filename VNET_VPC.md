Terraform script with named configurations for AWS VPC and Azure VNet. The script uses a variable to select the provider and executes the corresponding configuration.

### Terraform Script

```markdown
# Terraform Script to Create AWS VPC or Azure VNet Based on Provider Selection

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

## AWS VPC Configuration
# Creates a VPC with public and private subnets, NAT Gateway, and Security Group in AWS.
```hcl
resource "aws_vpc" "aws_vpc_configuration" {
  count      = var.provider == "aws" ? 1 : 0
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "aws-vpc"
  }
}

resource "aws_subnet" "aws_public_subnet_1" {
  count                   = var.provider == "aws" ? 1 : 0
  vpc_id                  = aws_vpc.aws_vpc_configuration.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true

  tags = {
    Name = "aws-public-subnet-1"
  }
}

resource "aws_subnet" "aws_public_subnet_2" {
  count                   = var.provider == "aws" ? 1 : 0
  vpc_id                  = aws_vpc.aws_vpc_configuration.id
  cidr_block              = "10.0.2.0/24"
  map_public_ip_on_launch = true

  tags = {
    Name = "aws-public-subnet-2"
  }
}

resource "aws_subnet" "aws_private_subnet_1" {
  count      = var.provider == "aws" ? 1 : 0
  vpc_id     = aws_vpc.aws_vpc_configuration.id
  cidr_block = "10.0.3.0/24"

  tags = {
    Name = "aws-private-subnet-1"
  }
}

resource "aws_subnet" "aws_private_subnet_2" {
  count      = var.provider == "aws" ? 1 : 0
  vpc_id     = aws_vpc.aws_vpc_configuration.id
  cidr_block = "10.0.4.0/24"

  tags = {
    Name = "aws-private-subnet-2"
  }
}

resource "aws_internet_gateway" "aws_igw" {
  count  = var.provider == "aws" ? 1 : 0
  vpc_id = aws_vpc.aws_vpc_configuration.id

  tags = {
    Name = "aws-igw"
  }
}

resource "aws_route_table" "aws_public_route_table" {
  count  = var.provider == "aws" ? 1 : 0
  vpc_id = aws_vpc.aws_vpc_configuration.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.aws_igw.id
  }

  tags = {
    Name = "aws-public-route-table"
  }
}

resource "aws_route_table_association" "aws_public_subnet_1" {
  count          = var.provider == "aws" ? 1 : 0
  subnet_id      = aws_subnet.aws_public_subnet_1.id
  route_table_id = aws_route_table.aws_public_route_table.id
}

resource "aws_route_table_association" "aws_public_subnet_2" {
  count          = var.provider == "aws" ? 1 : 0
  subnet_id      = aws_subnet.aws_public_subnet_2.id
  route_table_id = aws_route_table.aws_public_route_table.id
}

resource "aws_eip" "aws_nat_eip" {
  count = var.provider == "aws" ? 1 : 0
  vpc   = true
}

resource "aws_nat_gateway" "aws_nat_gateway" {
  count         = var.provider == "aws" ? 1 : 0
  allocation_id = aws_eip.aws_nat_eip.id
  subnet_id     = aws_subnet.aws_public_subnet_1.id

  tags = {
    Name = "aws-nat-gateway"
  }
}

resource "aws_route_table" "aws_private_route_table" {
  count  = var.provider == "aws" ? 1 : 0
  vpc_id = aws_vpc.aws_vpc_configuration.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.aws_nat_gateway.id
  }

  tags = {
    Name = "aws-private-route-table"
  }
}

resource "aws_route_table_association" "aws_private_subnet_1" {
  count          = var.provider == "aws" ? 1 : 0
  subnet_id      = aws_subnet.aws_private_subnet_1.id
  route_table_id = aws_route_table.aws_private_route_table.id
}

resource "aws_route_table_association" "aws_private_subnet_2" {
  count          = var.provider == "aws" ? 1 : 0
  subnet_id      = aws_subnet.aws_private_subnet_2.id
  route_table_id = aws_route_table.aws_private_route_table.id
}

resource "aws_security_group" "aws_security_group" {
  count  = var.provider == "aws" ? 1 : 0
  vpc_id = aws_vpc.aws_vpc_configuration.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 1521
    to_port     = 1521
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "aws-security-group"
  }
}
```

## Azure VNet Configuration
# Creates a VNet with public and private subnets, and Network Security Group in Azure.
```hcl
resource "azurerm_resource_group" "azure_resource_group" {
  count    = var.provider == "azure" ? 1 : 0
  name     = "azure-resource-group"
  location = "West US"
}

resource "azurerm_virtual_network" "azure_vnet_configuration" {
  count     = var.provider == "azure" ? 1 : 0
  name      = "azure-vnet"
  address_space = ["10.0.0.0/16"]
  location  = azurerm_resource_group.azure_resource_group.location
  resource_group_name = azurerm_resource_group.azure_resource_group.name

  tags = {
    Name = "azure-vnet"
  }
}

resource "azurerm_subnet" "azure_public_subnet_1" {
  count     = var.provider == "azure" ? 1 : 0
  name      = "azure-public-subnet-1"
  resource_group_name  = azurerm_resource_group.azure_resource_group.name
  virtual_network_name = azurerm_virtual_network.azure_vnet_configuration.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "azure_public_subnet_2" {
  count     = var.provider == "azure" ? 1 : 0
  name      = "azure-public-subnet-2"
  resource_group_name  = azurerm_resource_group.azure_resource_group.name
  virtual_network_name = azurerm_virtual_network.azure_vnet_configuration.name
  address_prefixes     = ["10.0.2.0/24"]
}

resource "azurerm_subnet" "azure_private_subnet_1" {
  count     = var.provider == "azure" ? 1 : 0
  name      = "azure-private-subnet-1"
  resource_group_name  = azurerm_resource_group.azure_resource_group.name
  virtual_network_name = azurerm_virtual_network.azure_vnet_configuration.name
  address_prefixes     = ["10.0.3.0/24"]
}

resource "azurerm_subnet" "azure_private_subnet_2" {
  count     = var.provider == "azure" ? 1 : 0
  name      = "azure-private-subnet-2"
  resource_group_name  = azurerm_resource_group.azure_resource_group.name
  virtual_network_name = azurerm_virtual

