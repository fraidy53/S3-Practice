VPC부터 EC2까지 전체 네트워크 인프라를 CLI로 직접 구성합니다.  
오후 Terraform 실습에서 이 과정을 코드 한 줄로 대체하는 것이 핵심 포인트입니다.

---

## 전체 구성 순서

```
VPC 생성
  └─ 서브넷 생성
  └─ 인터넷 게이트웨이 생성 & VPC 연결
  └─ 라우팅 테이블 생성 & 서브넷 연결
  └─ 보안 그룹 생성 (SSH + HTTP 허용)
  └─ EC2 인스턴스 실행
```

---

## 1. VPC 생성

```powershell
# VPC 생성
$VPC_ID = aws ec2 create-vpc `
    --cidr-block 10.0.0.0/16 `
    --query "Vpc.VpcId" `
    --output text

Write-Host "VPC ID: $VPC_ID"

# 이름 태그 추가
aws ec2 create-tags `
    --resources $VPC_ID `
    --tags Key=Name,Value=MyCliVPC
```

---

## 2. 서브넷 생성

```powershell
$SUBNET_ID = aws ec2 create-subnet `
    --vpc-id $VPC_ID `
    --cidr-block 10.0.1.0/24 `
    --availability-zone ap-northeast-2a `
    --query "Subnet.SubnetId" `
    --output text

Write-Host "Subnet ID: $SUBNET_ID"

aws ec2 create-tags `
    --resources $SUBNET_ID `
    --tags Key=Name,Value=MyCliPublicSubnet
```

---

## 3. 인터넷 게이트웨이 생성 & VPC 연결

```powershell
# IGW 생성
$IGW_ID = aws ec2 create-internet-gateway `
    --query "InternetGateway.InternetGatewayId" `
    --output text

Write-Host "IGW ID: $IGW_ID"

# VPC에 연결
aws ec2 attach-internet-gateway `
    --internet-gateway-id $IGW_ID `
    --vpc-id $VPC_ID
```

---

## 4. 라우팅 테이블 설정

```powershell
# 라우팅 테이블 생성
$RTB_ID = aws ec2 create-route-table `
    --vpc-id $VPC_ID `
    --query "RouteTable.RouteTableId" `
    --output text

Write-Host "Route Table ID: $RTB_ID"

# 인터넷으로 향하는 경로 추가 (0.0.0.0/0 → IGW)
aws ec2 create-route `
    --route-table-id $RTB_ID `
    --destination-cidr-block 0.0.0.0/0 `
    --gateway-id $IGW_ID

# 서브넷과 라우팅 테이블 연결
aws ec2 associate-route-table `
    --subnet-id $SUBNET_ID `
    --route-table-id $RTB_ID
```

---

## 5. 보안 그룹 생성 (SSH + HTTP 허용)

```powershell
# 보안 그룹 생성
$SG_ID = aws ec2 create-security-group `
    --group-name MyCliSG `
    --description "CLI practice security group" `
    --vpc-id $VPC_ID `
    --query "GroupId" `
    --output text

Write-Host "Security Group ID: $SG_ID"

# SSH (22) 허용
aws ec2 authorize-security-group-ingress `
    --group-id $SG_ID `
    --protocol tcp --port 22 --cidr 0.0.0.0/0

# HTTP (80) 허용
aws ec2 authorize-security-group-ingress `
    --group-id $SG_ID `
    --protocol tcp --port 80 --cidr 0.0.0.0/0

# Spring Boot 기본 포트 (8080) 허용
aws ec2 authorize-security-group-ingress `
    --group-id $SG_ID `
    --protocol tcp --port 8080 --cidr 0.0.0.0/0
```

---

## 6. EC2 인스턴스 실행

```powershell
# 키페어 생성 (없는 경우)
aws ec2 create-key-pair `
    --key-name MyCliKeyPair `
    --query "KeyMaterial" `
    --output text | Out-File -FilePath MyCliKeyPair.pem -Encoding ascii

# EC2 실행
$INSTANCE_ID = aws ec2 run-instances `
    --image-id ami-0c031a79ffb01a803 `
    --count 1 `
    --instance-type t2.micro `
    --key-name MyCliKeyPair `
    --subnet-id $SUBNET_ID `
    --security-group-ids $SG_ID `
    --associate-public-ip-address `
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=MyCliEC2}]" `
    --query "Instances[0].InstanceId" `
    --output text

Write-Host "Instance ID: $INSTANCE_ID"
```

---

## 7. 인스턴스 실행 확인

```powershell
# 상태 확인 (running 될 때까지 대기)
aws ec2 describe-instances `
    --instance-ids $INSTANCE_ID `
    --query "Reservations[0].Instances[0].[State.Name, PublicIpAddress]" `
    --output table

# 퍼블릭 IP 확인
$PUBLIC_IP = aws ec2 describe-instances `
    --instance-ids $INSTANCE_ID `
    --query "Reservations[0].Instances[0].PublicIpAddress" `
    --output text

Write-Host "Public IP: $PUBLIC_IP"
Write-Host "SSH 접속: ssh -i MyCliKeyPair.pem ec2-user@$PUBLIC_IP"
```

---

## 8. 리소스 정리 (실습 후 반드시 실행)

```powershell
# 1. EC2 종료
aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# 2. 보안 그룹 삭제 (EC2 종료 후)
aws ec2 delete-security-group --group-id $SG_ID

# 3. 서브넷 삭제
aws ec2 delete-subnet --subnet-id $SUBNET_ID

# 4. 라우팅 테이블 삭제
aws ec2 delete-route-table --route-table-id $RTB_ID

# 5. IGW 분리 후 삭제
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# 6. VPC 삭제
aws ec2 delete-vpc --vpc-id $VPC_ID

Write-Host "모든 리소스 삭제 완료"
```

