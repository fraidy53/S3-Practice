
단계 4의 `vpc.tf` 코드를 모듈로 추출합니다.  
"같은 네트워크 구성을 dev/prod 환경에서 재사용"하는 것이 목표입니다.

---

## 왜 모듈이 필요한가?

단계 4의 `vpc.tf`는 VPC, 서브넷, IGW, 라우팅 테이블, 보안 그룹이 한 파일에 뒤섞여 있습니다.  
prod 환경에 동일한 구성을 만들려면 코드를 통째로 복사해야 합니다.

```
# 모듈 없이 (현재)
lab04/vpc.tf  ← VPC 코드 50줄

# 모듈 사용 후
lab05/
├── main.tf          ← module "network" 호출 3줄
└── modules/
    └── network/     ← 재사용 가능한 VPC 코드
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

## 파일 구조

```
lab05/
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── main.tf          ← 모듈 호출
├── outputs.tf
└── modules/
    └── network/
        ├── main.tf      ← 단계 4의 vpc.tf에서 추출
        ├── variables.tf
        └── outputs.tf
```

---

## modules/network/variables.tf

```hcl
variable "project" {
  type = string
}

variable "environment" {
  type = string
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "public_subnet_cidr" {
  type    = string
  default = "10.0.1.0/24"
}

variable "private_subnet_cidr" {
  type    = string
  default = "10.0.2.0/24"
}

variable "region" {
  type    = string
  default = "ap-northeast-2"
}
```

---

## modules/network/main.tf (단계 4의 vpc.tf 추출)

```hcl
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags = merge(local.common_tags, { Name = "${var.project}-vpc" })
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = "${var.region}a"
  map_public_ip_on_launch = true
  tags = merge(local.common_tags, { Name = "${var.project}-public-subnet" })
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidr
  availability_zone = "${var.region}a"
  tags = merge(local.common_tags, { Name = "${var.project}-private-subnet" })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(local.common_tags, { Name = "${var.project}-igw" })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
  tags = merge(local.common_tags, { Name = "${var.project}-public-rtb" })
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

---

## modules/network/outputs.tf

```hcl
output "vpc_id" {
  value = aws_vpc.this.id
}

output "public_subnet_id" {
  value = aws_subnet.public.id
}

output "private_subnet_id" {
  value = aws_subnet.private.id
}
```

---

## 루트 main.tf (모듈 호출)

단계 4의 vpc.tf 50줄이 아래 10줄로 줄어듭니다.

```hcl
# 네트워크 모듈 호출
module "network" {
  source = "./modules/network"

  project             = var.project
  environment         = var.environment
  region              = var.region
  vpc_cidr            = "10.0.0.0/16"
  public_subnet_cidr  = "10.0.1.0/24"
  private_subnet_cidr = "10.0.2.0/24"
}

# EC2 - 모듈 output으로 subnet_id 참조
resource "aws_security_group" "backend" {
  name   = "${var.project}-backend-sg"
  vpc_id = module.network.vpc_id     # 모듈 output 참조

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "backend" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t2.micro"
  subnet_id              = module.network.public_subnet_id  # 모듈 output 참조
  vpc_security_group_ids = [aws_security_group.backend.id]
  key_name               = var.key_pair_name

  tags = {
    Name        = "${var.project}-backend"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

---

## provider.tf (Remote State 연결)

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "myapp-terraform-state-20260429"
    key            = "training/lab05/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

provider "aws" {
  region = var.region
}
```

---

## outputs.tf

```hcl
output "vpc_id" {
  value = module.network.vpc_id
}

output "backend_public_ip" {
  value = aws_instance.backend.public_ip
}
```

---

## 실행

```powershell
terraform init    # 모듈 다운로드 (.terraform/modules/ 에 저장됨)
terraform plan
terraform apply

terraform output
```

---

## 단계 4와 비교

| 항목 | 단계 4 (vpc.tf 직접 작성) | 단계 5 (모듈 사용) |
|------|------------------------|-----------------|
| vpc.tf 코드 줄 수 | 약 60줄 | 삭제됨 |
| main.tf에서 네트워크 선언 | 없음 | `module "network"` 10줄 |
| prod 환경 추가 시 | vpc.tf 복사 후 값 변경 | `module "network"` 블록 하나 더 추가 |
| 네트워크 구조 변경 시 | 환경별로 각각 수정 | `modules/network/main.tf` 한 곳만 수정 |
