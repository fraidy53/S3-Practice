
`--query`, `--filter`, `--output` 옵션을 익혀 원하는 정보만 정확히 뽑아내는 방법을 실습합니다.

---

## 1. `--output` : 출력 형식 선택

AWS CLI 명령 결과를 **어떤 형식으로 출력할지** 지정합니다.  
형식을 지정하지 않으면 `json`이 기본값으로 사용됩니다.

| 값 | 설명 | 사용 시점 |
|----|------|----------|
| `json` | 중첩된 JSON 구조로 출력 (기본값) | jq 등으로 파싱할 때 |
| `table` | 표 형태로 출력 | 눈으로 확인할 때 |
| `text` | 탭 구분 단순 텍스트 출력 | 변수에 저장할 때 |

```powershell
# 동일한 명령어, 형식만 다르게

aws s3api list-buckets --output json

aws s3api list-buckets --output table

aws s3api list-buckets --output text
```

---

## 2. `--query` : 원하는 필드만 추출 (JMESPath)

AWS CLI 명령은 기본적으로 모든 필드를 반환합니다.  
`--query`를 사용하면 **필요한 필드만 골라서** 출력할 수 있습니다.  
JMESPath 문법을 사용하며, 결과를 받은 후 **내 PC(클라이언트)에서** 필터링합니다.

| 문법 | 의미 |
|------|------|
| `필드명` | 해당 필드 값 |
| `[*]` | 배열의 모든 요소 |
| `[0]` | 배열의 첫 번째 요소 |
| `[필드1, 필드2]` | 여러 필드를 동시에 추출 |
| `[?조건]` | 조건에 맞는 요소만 추출 |

### 기본 필드 추출
```powershell
# EC2 인스턴스 ID만 출력
aws ec2 describe-instances --query "Reservations[*].Instances[*].InstanceId" --output text

# 인스턴스 ID + 상태 함께 출력
aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId, State.Name]" --output table
```

### S3 버킷 목록에서 이름만
```powershell
aws s3api list-buckets --query "Buckets[*].Name" --output text
```

### 조건부 추출 (`?` 조건)
```powershell
# 실행 중인(running) 인스턴스만
aws ec2 describe-instances --query "Reservations[*].Instances[?State.Name=='running'].[InstanceId, PublicIpAddress]" --output table
```

---

## 3. `--filters` : 서버 사이드 필터링

`--query`와 비슷해 보이지만 동작 방식이 다릅니다.  
`--filters`는 **AWS 서버에서 미리 걸러서** 결과를 반환하기 때문에  
리소스가 많을수록 `--query`보다 빠르고 효율적입니다.

| 항목 | `--query` | `--filters` |
|------|-----------|-------------|
| 필터 위치 | 내 PC(클라이언트) | AWS 서버 |
| 전체 데이터 수신 | O (받고 나서 자름) | X (서버에서 미리 자름) |
| 사용 가능 대상 | 모든 필드 | AWS가 지원하는 필드만 |
| 주 용도 | 출력 필드 선택 | 조회 대상 리소스 범위 축소 |

```powershell
# Name 태그가 'MyInstance'인 EC2만 조회
aws ec2 describe-instances --filters "Name=tag:Name,Values=sk_kbm_server" --query "Reservations[*].Instances[*].[InstanceId, State.Name]" --output table

# 실행 중인 t2.micro 인스턴스만 (필터 여러 개 동시 적용)
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" "Name=instance-type,Values=t3.micro" --query "Reservations[*].Instances[*].InstanceId" --output text

# 특정 VPC 안의 서브넷만
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxxxxxxx" --output table
```

---

## 4. 결과를 변수에 저장해서 다음 명령에 활용

```powershell
# 인스턴스 ID를 변수에 저장
$INSTANCE_ID = aws ec2 describe-instances --filters "Name=tag:Name,Values=sk_kbm_server" --query "Reservations[0].Instances[0].InstanceId" --output text

Write-Host "Instance ID: $INSTANCE_ID"

# 변수를 다음 명령어에 활용
aws ec2 describe-instances --instance-ids $INSTANCE_ID --output table
```

---

## 5. 실전 예제 : S3 배포 버킷 정보 한 번에 확인

```powershell
# 버킷 이름을 변수로 저장
$BUCKET = "sk5th-kbm-image"

# 버킷 위치 확인
aws s3api get-bucket-location --bucket $BUCKET

# 버킷 안 파일 목록 (최근 수정순)
aws s3 ls s3://$BUCKET --recursive --human-readable
```

---

## 정리

| 옵션 | 역할 | 사용 시점 |
|------|------|----------|
| `--output table` | 보기 좋게 출력 | 탐색/확인 시 |
| `--output text` | 줄바꿈 단순 출력 | 변수 저장 시 |
| `--query` | 클라이언트 필드 추출 | 특정 값만 필요할 때 |
| `--filters` | 서버 사이드 필터 | 대량 리소스 조회 시 |
