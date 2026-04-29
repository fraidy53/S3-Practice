
단계 3에서 설정한 Remote State를 이어받아 VPC + EC2 + S3 전체 구성을 코드화합니다.  
오전 CLI 실습과 동일한 인프라를 `terraform apply` 한 번으로 재현합니다.

---

## 목표 아키텍처

```
Internet
   │
   ▼
S3 (React 프론트엔드 정적 호스팅)

Internet
   │
   ▼
EC2 (Spring Boot 백엔드)
   │
VPC (10.0.0.0/16)
├── Public Subnet  (10.0.1.0/24)  ← EC2
└── Private Subnet (10.0.2.0/24)
```

---

## 파일 구조

```
lab04/
├── provider.tf      ← Remote State backend 포함
├── variables.tf
├── terraform.tfvars
├── vpc.tf
├── ec2.tf
├── s3.tf
└── outputs.tf
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

  # 단계 3에서 만든 S3 backend - key만 lab04로 변경
  backend "s3" {
    bucket         = "myapp-terraform-state-20260429"
    key            = "training/lab04/terraform.tfstate"
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

## variables.tf

```hcl
variable "region" {
  default = "ap-northeast-2"
}

variable "project" {
  default = "myapp"
}

variable "environment" {
  default = "dev"
}

variable "key_pair_name" {
  description = "EC2 접속용 키페어 이름"
  type        = string
}
```

## terraform.tfvars

```hcl
project       = "myapp"
environment   = "dev"
key_pair_name = "MyCliKeyPair"
```

---

## vpc.tf

```hcl
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = merge(local.common_tags, { Name = "${var.project}-vpc" })
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.region}a"
  map_public_ip_on_launch = true
  tags = merge(local.common_tags, { Name = "${var.project}-public-subnet" })
}

resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "${var.region}a"
  tags = merge(local.common_tags, { Name = "${var.project}-private-subnet" })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = merge(local.common_tags, { Name = "${var.project}-igw" })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  tags = merge(local.common_tags, { Name = "${var.project}-public-rtb" })
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "backend" {
  name   = "${var.project}-backend-sg"
  vpc_id = aws_vpc.main.id

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

  tags = merge(local.common_tags, { Name = "${var.project}-backend-sg" })
}
```

---

## ec2.tf

```hcl
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
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.backend.id]
  key_name               = var.key_pair_name

  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y java-17-amazon-corretto docker
    systemctl start docker
    systemctl enable docker
    usermod -aG docker ec2-user
  EOF

  tags = merge(local.common_tags, { Name = "${var.project}-backend" })
}
```

---

## s3.tf

```hcl
resource "aws_s3_bucket" "frontend" {
  bucket = "${var.project}-frontend-${var.environment}-20260429"
  tags   = merge(local.common_tags, { Name = "${var.project}-frontend" })
}

resource "aws_s3_bucket_website_configuration" "frontend" {
  bucket = aws_s3_bucket.frontend.id
  index_document { suffix = "index.html" }
  error_document { key    = "index.html" }
}

resource "aws_s3_bucket_public_access_block" "frontend" {
  bucket                  = aws_s3_bucket.frontend.id
  block_public_acls       = false
  ignore_public_acls      = false
  block_public_policy     = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_policy" "frontend" {
  bucket     = aws_s3_bucket.frontend.id
  depends_on = [aws_s3_bucket_public_access_block.frontend]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = "*"
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.frontend.arn}/*"
    }]
  })
}
```

---

## outputs.tf

```hcl
output "backend_public_ip" {
  value = aws_instance.backend.public_ip
}

output "backend_api_url" {
  value = "http://${aws_instance.backend.public_ip}:8080/api/hello"
}

output "frontend_website_url" {
  value = "http://${aws_s3_bucket_website_configuration.frontend.website_endpoint}"
}
```

---

## 실행

```powershell
terraform init    # S3 backend 연결 확인
terraform plan    # 생성될 리소스 목록 확인
terraform apply

terraform output  # URL 및 IP 확인
```

---

## 오전 CLI와 비교

| 작업 | 오전 (CLI) | 지금 (Terraform) |
|------|-----------|-----------------|
| 전체 구성 | 명령어 15줄 이상, 변수 수동 복사 | `terraform apply` 1번 |
| 삭제 | 역순으로 명령어 6줄 | `terraform destroy` 1번 |
| 재현성 | 스크립트 저장 필요 | 코드 자체가 문서 |
| State 저장 | 없음 | S3에 자동 저장 (단계 3 설정) |

---