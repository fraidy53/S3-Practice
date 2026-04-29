
---

## 왜 Remote State가 필요한가?

| 항목 | 로컬 State | Remote State (S3) |
|------|-----------|------------------|
| 협업 | 불가 (파일 공유 필요) | 팀 전체 공유 가능 |
| 안전성 | 로컬 삭제 시 분실 | S3 버전 관리로 보존 |
| 동시 작업 | 충돌 위험 | DynamoDB Lock으로 방지 |
| CI/CD 연동 | 어려움 | GitHub Actions에서 바로 사용 |

---

## 1. State 저장용 S3 버킷 + DynamoDB 테이블 생성

```powershell
# S3 버킷 생성 (버전 관리 활성화)
aws s3api create-bucket `
    --bucket myapp-terraform-state-20260429 `
    --region ap-northeast-2 `
    --create-bucket-configuration LocationConstraint=ap-northeast-2

aws s3api put-bucket-versioning `
    --bucket myapp-terraform-state-20260429 `
    --versioning-configuration Status=Enabled

# DynamoDB Lock 테이블 생성
aws dynamodb create-table `
    --table-name terraform-lock `
    --attribute-definitions AttributeName=LockID,AttributeType=S `
    --key-schema AttributeName=LockID,KeyType=HASH `
    --billing-mode PAY_PER_REQUEST `
    --region ap-northeast-2
```

---

## 2. provider.tf에 backend 추가

이후 모든 실습 폴더의 `provider.tf`에 아래 backend 블록을 포함합니다.

```hcl
# provider.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "myapp-terraform-state-20260429"
    key            = "training/lab03/terraform.tfstate"   # 단계별로 key 변경
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-lock"
  }
}

provider "aws" {
  region = "ap-northeast-2"
}
```

### 단계별 State key 구조
```
myapp-terraform-state-20260429/
├── training/lab03/terraform.tfstate   ← 이번 단계
├── training/lab04/terraform.tfstate   ← 3-Tier 실습
├── training/lab05/terraform.tfstate   ← Module 실습
└── training/lab06/terraform.tfstate   ← GitOps 실습
```

---

## 3. 기존 로컬 State를 S3로 마이그레이션

이미 로컬에 state가 있다면 아래 명령으로 S3로 이관합니다.

```powershell
terraform init -migrate-state
# "yes" 입력 → 로컬 terraform.tfstate가 S3로 이동됨
```

처음 시작하는 경우:
```powershell
terraform init
terraform apply
# → 로컬에 terraform.tfstate 파일 없이 바로 S3에 저장됨
```

---

## 4. 동작 확인

### S3에서 state 파일 확인
```powershell
aws s3 ls s3://myapp-terraform-state-20260429/training/ --recursive
```

### DynamoDB Lock 동작 확인
`terraform apply` 실행 중 콘솔에서 확인:  
**DynamoDB → 테이블 → terraform-lock → 항목 탐색**  
→ apply 중 LockID 항목 생성됨, 완료 후 자동 삭제됨

---

## 5. State 관련 유용한 명령어

```powershell
# 현재 state 목록
terraform state list

# 특정 리소스 상세
terraform state show aws_s3_bucket.frontend

# S3에 저장된 state 버전 목록 (버전 관리 확인)
aws s3api list-object-versions `
    --bucket myapp-terraform-state-20260429 `
    --prefix training/lab03/terraform.tfstate
```

---

> **다음 단계 예고**: Remote State 설정이 완료됐습니다.  
> 단계 4 (3-Tier 인프라)부터는 이 S3 backend를 `key`만 바꿔서 이어받아 사용합니다.
