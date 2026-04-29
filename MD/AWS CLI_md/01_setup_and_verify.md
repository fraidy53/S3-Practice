
## 1. AWS CLI 설치

### 1.1 Windows (MSI 설치 프로그램 — 권장)

1. 아래 링크에서 설치 파일 다운로드

   **[AWSCLIV2.msi 다운로드](https://awscli.amazonaws.com/AWSCLIV2.msi)**

2. 다운로드된 `.msi` 파일 실행 → Next → Next → Install → Finish

3. PowerShell(또는 CMD)을 **새로 열고** 설치 확인

```powershell
aws --version
# 출력 예시: aws-cli/2.x.x Python/3.x.x Windows/10
```

### 1.2 Windows (Chocolatey 패키지 매니저)

Chocolatey가 설치된 환경이라면 한 줄로 설치 가능합니다.

```powershell
choco install awscli
```

### 1.3 macOS

```bash
# Homebrew
brew install awscli

# 확인
aws --version
```

### 1.4 Linux (Ubuntu/Debian)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# 확인
aws --version
```

---

## 2. AWS CLI 설치 확인

```powershell
aws --version
# 출력 예시: aws-cli/2.x.x Python/3.x.x Windows/10
```

버전이 `1.x.x` 로 나오면 v1이 설치된 것입니다. v2로 재설치하세요.

---

## 3. 자격증명 설정 (`aws configure`)

```powershell
aws configure
```

입력 항목:

| 항목 | 예시 |
|------|------|
| AWS Access Key ID | `AKIA...` |
| AWS Secret Access Key | `wJalr...` |
| Default region name | `ap-northeast-2` |
| Default output format | `json` |

설정 파일 저장 위치:
```
C:\Users\[사용자명]\.aws\credentials   # 키 저장
C:\Users\[사용자명]\.aws\config        # 리전/포맷 저장
```

---

## 4. 연결 상태 확인

```powershell
# 현재 로그인된 IAM 사용자 확인
aws sts get-caller-identity
```

정상 출력 예시:
```json
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/my-username"
}
```

---

## 5. 다중 프로파일 관리

AWS 계정이 여러 개일 때 `~/.aws/credentials` 파일에 프로파일별로 키를 등록하고 전환할 수 있습니다.

### 5.1 프로파일 추가 방법

**방법 A — 명령어로 추가 (대화형 입력)**

```powershell
aws configure --profile welabs
# AWS Access Key ID 입력:
# AWS Secret Access Key 입력:
# Default region name 입력: ap-northeast-2
# Default output format 입력: json

aws configure --profile twig
# 동일하게 입력
```

**방법 B — credentials 파일 직접 편집**

`C:\Users\[사용자명]\.aws\credentials` 파일을 메모장 등으로 열어 직접 추가합니다.

```ini
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = xxxxxx

[welabs]
aws_access_key_id = AKIA...
aws_secret_access_key = xxxxxx

[twig]
aws_access_key_id = AKIA...
aws_secret_access_key = xxxxxx
```

> 키가 이미 있다면 방법 B가 더 빠릅니다. 방법 A는 키를 새로 발급받았을 때 사용합니다.

---

### 5.2 프로파일 선택(전환) 방법

**방법 1 — 명령어마다 `--profile` 옵션 (일회성)**

```powershell
aws s3 ls --profile welabs    # 이 명령만 welabs로 실행
aws s3 ls --profile twig      # 이 명령만 twig로 실행
aws s3 ls                     # default로 실행
```

**방법 2 — 환경변수로 세션 전체 전환**

터미널을 닫기 전까지 해당 프로파일로 고정됩니다.

```powershell
$env:AWS_PROFILE = "profile1"   # welabs로 전환
aws sts get-caller-identity   # welabs 계정으로 실행됨

$env:AWS_PROFILE = "profile2"     # twig로 전환
aws sts get-caller-identity   # twig 계정으로 실행됨

Remove-Item Env:AWS_PROFILE   # 해제 → default로 복귀
```
```cmd
set AWS_PROFILE=profile1
aws sts get-caller-identity

rem 해제
set AWS_PROFILE=
```

---

### 5.3 config 파일에서 리전 별도 관리

`~/.aws/config` 파일에서 프로파일별 리전과 출력 형식을 따로 설정합니다.  
credentials와 달리 `[profile 이름]` 형식으로 작성합니다.

```ini
# C:\Users\[사용자명]\.aws\config

[default]
region = ap-northeast-2
output = json

[profile welabs]
region = ap-northeast-2
output = json

[profile twig]
region = us-east-1
output = table
```

### 5.4 현재 어느 계정인지 확인

프로파일 전환 후 항상 아래 명령으로 현재 계정을 확인합니다.

```powershell
aws sts get-caller-identity
```

출력의 `Arn` 필드에서 계정 번호와 사용자 이름을 확인할 수 있습니다.

```json
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/welabs-user"
}
```

> **보안 주의**: `credentials` 파일에는 실제 키가 평문으로 저장됩니다.  
> 해당 파일을 Git에 커밋하거나 외부에 공유하지 않도록 주의하세요.  
> `.gitignore`에 `.aws/` 또는 `credentials`를 추가하는 것을 권장합니다.

---

## 6. 빠른 점검 체크리스트

```powershell
# 1. CLI 버전
aws --version

# 2. 자격증명 확인
aws sts get-caller-identity

# 3. 리전 확인
aws configure get region

# 4. S3 기본 권한 테스트
aws s3 ls
```

모두 오류 없이 실행되면 준비 완료입니다.
