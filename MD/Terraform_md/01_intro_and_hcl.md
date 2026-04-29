Terraform 개념 + HCL 기초 + S3 실습

---

## 1. IaC(Infrastructure as Code)란?

| 방식 | 설명 | 한계 |
|------|------|------|
| 콘솔 클릭 | AWS 콘솔에서 직접 생성 | 재현 불가, 히스토리 없음 |
| AWS CLI | 명령어로 생성 (오전 실습) | 상태 추적 없음, 스크립트 관리 어려움 |
| **Terraform** | **코드로 선언 → 자동 생성/변경/삭제** | 학습 비용 |

Terraform의 핵심 가치:
- **선언형**: "이런 인프라가 있어야 한다"고 선언하면 Terraform이 알아서 맞춤
- **상태 관리**: 현재 인프라 상태를 `terraform.tfstate`로 추적
- **멱등성**: 몇 번을 실행해도 동일한 결과

---

## 2. Terraform 설치

```powershell
# Windows (winget)
winget install HashiCorp.Terraform

# Windows (Chocolatey)
choco install terraform

# 설치 확인
terraform version
```

---

## 3. AWS 인증 방식

Terraform은 별도 로그인 없이 오전에 설정한 `aws configure` 자격증명을 그대로 사용합니다.

```
인증 우선순위:
1. ~/.aws/credentials  ← aws configure 설정값 (가장 일반적)
2. 환경변수 AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
3. IAM Role (EC2/Lambda 위에서 실행 중일 경우)
```

> **절대 금지**: `main.tf` 안에 `access_key`, `secret_key`를 직접 입력하지 마세요.  
> Git에 올라가면 보안 사고로 이어집니다.

---

## 4. 핵심 개념

### Provider

Terraform이 어떤 클라우드/서비스와 통신할지 선언합니다.  
AWS, GCP, Azure 등 다양한 Provider가 있으며 `terraform init` 시 해당 플러그인이 자동 다운로드됩니다.

```hcl
provider "aws" {
  region = "ap-northeast-2"
}
```

### Resource

실제로 생성할 인프라 리소스를 선언합니다.  
형식: `resource "리소스_타입" "로컬_이름"` — 로컬 이름은 같은 `.tf` 파일 안에서 이 리소스를 참조할 때 사용합니다.

```hcl
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name"
}

# 다른 리소스에서 위 버킷을 참조할 때
resource "aws_s3_bucket_policy" "my_policy" {
  bucket = aws_s3_bucket.my_bucket.id   # 리소스_타입.로컬_이름.속성
}
```

### State (`terraform.tfstate`)

Terraform이 관리하는 인프라의 **현재 상태**를 JSON으로 기록하는 파일입니다.  
`terraform plan` 실행 시 이 파일과 실제 AWS 상태를 비교해 변경사항을 계산합니다.

```
실제 AWS 인프라
      ↕ 비교
terraform.tfstate  ← Terraform이 관리하는 상태 기록
      ↕ 비교
.tf 코드           ← 내가 원하는 상태 선언
```

---

### 핵심 명령어 상세 설명

#### `terraform init`

**개념**: 작업 디렉토리를 초기화합니다. Provider 플러그인 다운로드, backend 연결, 모듈 설치를 수행합니다.

- `.tf` 파일을 처음 작성한 후 가장 먼저 실행
- `provider` 또는 `backend` 설정을 변경했을 때
- 새로운 `module`을 추가했을 때
- 팀원의 코드를 git clone한 직후

```powershell
terraform init

# 실행 결과 확인
# .terraform/          ← Provider 플러그인 저장 폴더
# .terraform.lock.hcl  ← Provider 버전 잠금 파일 (git에 커밋)
```

---

#### `terraform plan`

**개념**: 코드와 현재 state를 비교해 **어떤 변경이 일어날지 미리 보여줍니다.** 실제 인프라에는 아무런 영향을 주지 않습니다.

- `apply` 전 반드시 실행해서 의도치 않은 변경이 없는지 확인
- PR 리뷰 시 인프라 변경 내용을 팀원과 공유할 때
- 코드를 수정한 후 영향 범위를 파악할 때

```powershell
terraform plan

# 출력 기호 의미
# + create   → 새로 생성되는 리소스 (초록색)
# ~ update   → 속성이 변경되는 리소스 (노란색)
# - destroy  → 삭제되는 리소스 (빨간색)
# -/+ replace → 삭제 후 재생성 (빨간색+초록색)

# plan 결과를 파일로 저장 (apply 시 정확히 이 계획만 실행)
terraform plan -out=tfplan
terraform apply tfplan
```

---

#### `terraform apply`

**개념**: `plan`에서 확인한 변경사항을 **실제 인프라에 적용**합니다. 실행 전 한 번 더 plan을 보여주고 `yes` 입력을 기다립니다.

- plan 결과를 확인하고 실제로 인프라를 생성/변경할 때
- CI/CD 파이프라인에서 자동 배포할 때 (`-auto-approve` 옵션 사용)

```powershell
terraform apply

# 확인 없이 바로 적용 (CI/CD 환경에서 사용)
terraform apply -auto-approve

# 특정 리소스만 적용
terraform apply -target=aws_s3_bucket.frontend
```

> **주의**: `-target`은 임시 방편입니다. 습관적으로 사용하면 state 불일치가 생길 수 있습니다.

---

#### `terraform destroy`

**개념**: 현재 state에 등록된 **모든 리소스를 삭제**합니다. 삭제 전 plan과 동일하게 삭제 목록을 보여주고 `yes` 입력을 기다립니다.

- 실습 후 비용 발생을 막기 위해 인프라 전체 정리
- dev 환경을 퇴근 전 내렸다가 출근 후 다시 올릴 때
- 특정 환경 전체를 제거할 때

```powershell
terraform destroy

# 확인 없이 바로 삭제 (주의해서 사용)
terraform destroy -auto-approve

# 특정 리소스만 삭제
terraform destroy -target=aws_instance.backend
```

> **주의**: State 버킷, DB처럼 데이터가 있는 리소스는 `lifecycle { prevent_destroy = true }` 로 보호하는 것을 권장합니다.

---

#### 그 외 자주 쓰는 명령어

```powershell
# 코드 포맷 자동 정리 (.tf 파일 들여쓰기 맞춤)
terraform fmt

# 문법 오류 검사 (apply 전 빠른 검증)
terraform validate

# apply 후 output 값 확인
terraform output
terraform output website_url   # 특정 output만

# state에 등록된 리소스 목록
terraform state list

# 특정 리소스 상세 정보
terraform state show aws_s3_bucket.frontend
```

| 명령어 | 인프라 변경 | 주 사용 시점 |
|--------|-----------|------------|
| `init` | X | 프로젝트 시작, provider 변경 후 |
| `plan` | X | apply 전 항상 |
| `apply` | **O** | 실제 적용 시 |
| `destroy` | **O** | 전체 정리 시 |
| `fmt` | X | 코드 작성 중 |
| `validate` | X | 문법 확인 시 |

---

## 5. HCL 핵심 문법

테라폼 코드를 더 유연하고 재사용 가능하게 만드는 4가지 핵심 요소입니다.

---

### 5.1 variable (입력 변수)
*   리전, 인스턴스 타입, 태그 이름 등 **코드의 수정 없이 실행 시점에 값을 바꾸고 싶을 때** 사용합니다.
*   환경별(dev, prod)로 다른 설정값을 적용하고 싶을 때 필수입니다.


*   함수의 '파라미터'와 같습니다.
*   `default` 값을 지정할 수도 있고, 실행 시 터미널에서 직접 입력받을 수도 있습니다.

```hcl
# variables.tf
variable "bucket_name" {
  description = "S3 버킷 이름"
  type        = string
}

variable "environment" {
  type    = string
  default = "dev"

  validation {
    condition     = contains(["dev", "prod"], var.environment)
    error_message = "dev 또는 prod 만 허용됩니다."
  }
}
```

---

### 5.2 locals (내부 계산 값)
*   코드 내에서 **중복되는 긴 문자열이나 복잡한 계산식**을 하나의 이름으로 묶어 가독성을 높이고 싶을 때 사용합니다.
*   변수(`variable`)들을 조합해서 새로운 값을 만들 때 유용합니다. (예: `프로젝트명 + 리전명`)

*   코드 내부에서만 사용되는 '상수'와 같습니다. 외부에서 값을 바꿀 수 없습니다.
*   중복을 제거하여 한 곳만 고쳐도 전체에 반영되게 합니다.

```hcl
locals {
  project     = "myapp"
  common_tags = {
    Project     = local.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

---

### 5.3 output (출력 값)
*   `apply` 성공 후, 생성된 리소스의 **퍼블릭 IP나 접속 주소(URL)** 등을 화면에 바로 띄우고 싶을 때 사용합니다.
*   다른 테라폼 프로젝트나 외부 스크립트에서 이 리소스의 정보를 참조해야 할 때 사용합니다.

*   함수의 '리턴값'과 같습니다.
*   민감한 정보(비밀번호 등)는 `sensitive = true` 설정을 통해 화면 노출을 막을 수 있습니다.

```hcl
# outputs.tf
output "website_url" {
  value       = "http://${aws_s3_bucket_website_configuration.frontend.website_endpoint}"
  description = "웹사이트 접속 주소"
}
```

---

### 5.4 data source (기존 리소스 참조)
*   테라폼으로 관리하지 않는 **이미 존재하고 있는 리소스의 정보**를 가져와야 할 때 사용합니다.
*   최신 AMI ID, 특정 VPC의 ID 등 **실행 시점에 AWS로부터 최신 데이터를 받아오고 싶을 때** 사용합니다.

*   데이터베이스의 'SELECT' 쿼리와 같습니다. 리소스를 생성하는 것이 아니라 '조회'만 합니다.

```hcl
data "aws_ami" "amzn2" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

---

## 6. 실습: S3 정적 웹호스팅 구성

### 파일 구조
```
lab01/
├── provider.tf
├── variables.tf
├── main.tf
├── outputs.tf
└── terraform.tfvars
```

### provider.tf
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

### variables.tf
```hcl
variable "region" {
  default = "ap-northeast-2"
}

variable "bucket_name" {
  type = string
}

variable "environment" {
  default = "dev"
}
```

### main.tf
```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket" "frontend" {
  bucket = var.bucket_name
  tags   = local.common_tags
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

### outputs.tf
```hcl
output "website_url" {
  value = "http://${aws_s3_bucket_website_configuration.frontend.website_endpoint}"
}
```

### terraform.tfvars
```hcl
bucket_name = "myapp-frontend-dev-20260429"
environment = "dev"
```

### 실행
```powershell
terraform init
terraform plan
terraform apply

terraform output website_url
```

