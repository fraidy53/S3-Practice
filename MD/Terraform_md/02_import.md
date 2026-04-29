
단계 1에서 Terraform으로 만든 S3 버킷을 destroy 후 다시 콘솔에서 만든 상황을 가정합니다.  
이 버킷을 `terraform import`로 다시 Terraform 관리 하에 두는 실습입니다.

---

## 왜 import가 필요한가?

```
콘솔 또는 CLI로 만든 리소스
       ↓
Terraform은 이 리소스의 존재를 모름 (state에 없음)
       ↓
terraform apply 하면 "새로 만들려고" 충돌 발생
       ↓
terraform import → state에 등록 → Terraform으로 관리 시작
```

---

## 1. 기존 S3 버킷 import

### 1단계: 빈 리소스 블록 선언
```hcl
# main.tf
resource "aws_s3_bucket" "frontend" {
  # import 후 내용을 채울 예정
}
```

### 2단계: import 실행
```powershell
terraform import aws_s3_bucket.frontend myapp-frontend-dev-20260429
```

### 3단계: 현재 state 속성 확인
```powershell
terraform state show aws_s3_bucket.frontend
```

### 4단계: 출력된 속성을 main.tf에 반영
```hcl
resource "aws_s3_bucket" "frontend" {
  bucket = "myapp-frontend-dev-20260429"
  tags = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

### 5단계: plan으로 drift 확인
```powershell
terraform plan
# "No changes." 가 나오면 코드와 실제 상태가 일치한 것
```

---

## 2. import 가능한 주요 리소스 식별자

| 리소스 타입 | import 식별자 |
|------------|--------------|
| `aws_s3_bucket` | 버킷 이름 |
| `aws_instance` | 인스턴스 ID (`i-xxx`) |
| `aws_vpc` | VPC ID (`vpc-xxx`) |
| `aws_subnet` | 서브넷 ID (`subnet-xxx`) |
| `aws_security_group` | 보안 그룹 ID (`sg-xxx`) |
| `aws_internet_gateway` | IGW ID (`igw-xxx`) |

---

## 3. Terraform 1.5+ import 블록 (최신 방식)

명령어 대신 코드로 import를 선언합니다.

```hcl
# import.tf
import {
  to = aws_s3_bucket.frontend
  id = "myapp-frontend-dev-20260429"
}
```

```powershell
terraform plan   # import 포함한 계획 확인
terraform apply  # import 실행
```

---

## 4. terraform state 명령어

```powershell
# state에 등록된 리소스 목록
terraform state list

# 특정 리소스 상세 정보
terraform state show aws_s3_bucket.frontend

# 리소스를 state에서만 제거 (실제 인프라 유지)
terraform state rm aws_s3_bucket.frontend

# 리소스 이름 변경
terraform state mv aws_s3_bucket.frontend aws_s3_bucket.web
```
