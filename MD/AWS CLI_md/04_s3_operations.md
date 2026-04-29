# S3 운영 관리

이번 실습에서는 AWS CLI를 사용하여 새 S3 버킷을 생성하고, 정적 웹호스팅 설정 및 보안 정책 적용, 그리고 CloudFront 캐시 무효화까지 운영 전반의 과정을 실습합니다.

---

## 1. 새 S3 버킷 생성

S3 버킷 이름은 전 세계에서 유일해야 합니다. `[이름]-cli-web-test` 형식을 권장합니다.

```powershell
# 1.1 버킷 생성 (Make Bucket)
aws s3 mb s3://kbm-cli-web-test --region ap-northeast-2

# 1.2 생성된 버킷 목록 확인
aws s3 ls
```

---

## 2. 정적 웹호스팅 설정

생성한 버킷을 웹사이트 서버처럼 동작하게 설정합니다.

```powershell
# 2.1 정적 웹호스팅 활성화
aws s3 website s3://kbm-cli-web-test/ --index-document index.html --error-document index.html

# 2.2 설정 확인
aws s3api get-bucket-website --bucket kbm-cli-web-test
```

웹호스팅 활성화 후 접근 URL 형식:
```
http://kbm-cli-web-test.s3-website.ap-northeast-2.amazonaws.com
```

---

## 3. 퍼블릭 액세스 차단 해제

정적 웹호스팅으로 외부에서 접속하려면 퍼블릭 읽기 접근이 가능해야 하므로 차단 설정을 해제합니다.

```powershell
# 3.1 퍼블릭 액세스 차단 전체 해제
aws s3api put-public-access-block `
    --bucket kbm-cli-web-test `
    --public-access-block-configuration `
        "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"

# 3.2 확인
aws s3api get-public-access-block --bucket kbm-cli-web-test
```

---

## 4. 버킷 정책 설정 (퍼블릭 읽기 허용)

차단을 해제했더라도, 구체적으로 어떤 객체(파일)를 읽게 해줄지 정책을 추가해야 합니다.

```powershell
# 4.1 버킷 정책 JSON 파일 생성
@"
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::kbm-cli-web-test/*"
        }
    ]
}
"@ | Out-File -FilePath bucket-policy.json -Encoding ascii

# 4.2 정책 적용
aws s3api put-bucket-policy `
    --bucket kbm-cli-web-test `
    --policy file://bucket-policy.json

# 4.3 적용 확인
aws s3api get-bucket-policy --bucket kbm-cli-web-test
```

---

## 5. 파일 업로드 및 배포

```powershell
# 5.1 테스트용 index.html 파일 생성
@"
<html>
<head><title>CLI S3 Test</title></head>
<body>
    <h1>Hello AWS CLI!</h1>
    <p>This page was uploaded via AWS CLI.</p>
</body>
</html>
"@ | Out-File -FilePath index.html -Encoding ascii

# 5.2 단일 파일 업로드
aws s3 cp index.html s3://kbm-cli-web-test/

# 5.3 폴더 전체 업로드 (React build 폴더 등)
aws s3 sync ./build s3://kbm-cli-web-test/ --delete

# 5.4 업로드 확인
aws s3 ls s3://kbm-cli-web-test/ --recursive --human-readable
```

`--delete` 옵션: S3에는 있지만 로컬에는 없는 파일을 자동 삭제하여 동기화합니다.

---

## 6. CloudFront 캐시 무효화 (Invalidation)

S3에 새 파일을 올려도 CloudFront 캐시 때문에 즉시 반영되지 않을 때 사용합니다.

```powershell
# 6.1 CloudFront 배포 목록 확인
aws cloudfront list-distributions `
    --query "DistributionList.Items[*].[Id, DomainName, Origins.Items[0].DomainName]" `
    --output table

# 6.2 전체 캐시 무효화 (/* 경로)
aws cloudfront create-invalidation `
    --distribution-id [DISTRIBUTION_ID] `
    --paths "/*"

# 6.3 무효화 진행 상태 확인
aws cloudfront list-invalidations `
    --distribution-id [DISTRIBUTION_ID] `
    --query "InvalidationList.Items[*].[Id, Status, CreateTime]" `
    --output table
```

---

## 7. GitHub Actions와 연계

배포 자동화 파이프라인에 아래 단계를 추가하여 S3 업로드와 캐시 갱신을 한 번에 처리합니다.

```yaml
- name: Deploy to S3
  run: |
    aws s3 sync ./build s3://${{ secrets.S3_BUCKET }} --delete

- name: Invalidate CloudFront Cache
  run: |
    aws cloudfront create-invalidation \
      --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
      --paths "/*"
```

---

## 8. 리소스 정리 (삭제)

실습이 끝난 후에는 과금을 방지하기 위해 반드시 버킷을 삭제합니다.

```powershell
# 버킷 내 파일 강제 삭제 후 버킷 삭제
aws s3 rb s3://[버킷이름] --force
```

