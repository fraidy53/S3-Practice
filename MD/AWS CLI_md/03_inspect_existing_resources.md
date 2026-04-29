
콘솔로 직접 만든 S3, EC2, VPC 리소스를 CLI로 조회합니다.  
"콘솔에서 클릭한 것이 명령어로 어떻게 보이는지" 연결하는 것이 핵심입니다.

---

## 1. S3 버킷 조회

```powershell
# 전체 버킷 목록
aws s3 ls

# 특정 버킷 내용 확인
aws s3 ls s3://[버킷이름] --recursive --human-readable

# 버킷 상세 정보
aws s3api get-bucket-location --bucket [버킷이름]

# 정적 웹호스팅 설정 확인
aws s3api get-bucket-website --bucket [버킷이름]

# 퍼블릭 액세스 차단 설정 확인
aws s3api get-public-access-block --bucket [버킷이름]
```

---

## 2. EC2 인스턴스 조회

```powershell
# 전체 인스턴스 목록 (이름 + 상태 + IP)
aws ec2 describe-instances --query "Reservations[*].Instances[*].[Tags[?Key=='Name'].Value|[0], State.Name, PublicIpAddress, InstanceId]" --output table

# 실행 중인 인스턴스만
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" --query "Reservations[*].Instances[*].[InstanceId, PublicIpAddress, InstanceType]" --output table

# 특정 인스턴스 상세 조회
aws ec2 describe-instances --instance-ids [인스턴스_ID] --output table
```

---

## 3. 보안 그룹 조회

```powershell
# 전체 보안 그룹 목록
aws ec2 describe-security-groups --query "SecurityGroups[*].[GroupId, GroupName, Description]" --output table

# 특정 보안 그룹의 인바운드 규칙 확인
aws ec2 describe-security-groups --group-ids [SG_ID] --query "SecurityGroups[0].IpPermissions" --output table
```

---

## 4. VPC / 네트워크 조회

```powershell
# VPC 목록
aws ec2 describe-vpcs  --query "Vpcs[*].[VpcId, CidrBlock, Tags[?Key=='Name'].Value|[0]]" --output table

# 서브넷 목록
aws ec2 describe-subnets --query "Subnets[*].[SubnetId, CidrBlock, AvailabilityZone, Tags[?Key=='Name'].Value|[0]]" --output table

# 인터넷 게이트웨이 목록
aws ec2 describe-internet-gateways --query "InternetGateways[*].[InternetGatewayId, Attachments[0].VpcId]" --output table
```

---

## 5. 실전 시나리오 : CI/CD 파이프라인에서 쓰는 S3 버킷 현황 파악

이전 수업에서 GitHub Actions로 배포한 프론트엔드 버킷을 CLI로 확인합니다.

```powershell
# 배포된 React 빌드 파일 목록 확인
aws s3 ls s3://[프론트엔드_버킷명]/ --recursive --human-readable

# index.html 마지막 수정 시각 확인 (배포 시점 파악)
aws s3api head-object --bucket [프론트엔드_버킷명] --key index.html --query "[LastModified, ContentLength]" --output table

# 버킷 총 용량 및 파일 수 확인
aws s3 ls s3://[프론트엔드_버킷명] --recursive --summarize --human-readable
```

---

## 6. 전체 리소스 현황 한 번에 보기 (요약 스크립트)

```powershell
Write-Host "=== S3 버킷 목록 ==="
aws s3 ls

Write-Host "`n=== EC2 인스턴스 목록 ==="
aws ec2 describe-instances `
    --query "Reservations[*].Instances[*].[Tags[?Key=='Name'].Value|[0], State.Name, InstanceId]" `
    --output table

Write-Host "`n=== VPC 목록 ==="
aws ec2 describe-vpcs `
    --query "Vpcs[*].[VpcId, CidrBlock]" `
    --output table
```
