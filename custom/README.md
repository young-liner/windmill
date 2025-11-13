# Windmill CE: Google SSO 10명 제한 우회 가이드

> 🎯 **현재 상태**: ✅ 소스 코드 수정 완료, 빌드 스크립트 준비 완료
>
> 이제 `./build_and_push.sh` 실행만 하면 됩니다!

## 빠른 시작 (Quick Start)

```bash
# 1. custom 디렉토리로 이동
cd custom

# 2. 빌드 및 푸시 실행
./build_and_push.sh

# 3. 완료! 이미지가 다음 경로에 생성됩니다:
#    us.gcr.io/liner-219011/windmill/omni:custom-1
```

자세한 내용은 [커스텀 빌드 방법](#커스텀-빌드-방법) 섹션을 참고하세요.

---

## 목차
1. [빠른 시작](#빠른-시작-quick-start)
2. [제한 사항 개요](#제한-사항-개요)
3. [제한이 구현된 위치](#제한이-구현된-위치)
4. [해결 방법](#해결-방법)
5. [커스텀 빌드 방법](#커스텀-빌드-방법)
6. [주의사항](#주의사항)

---

## 제한 사항 개요

Windmill Community Edition (CE)에는 다음과 같은 사용자 제한이 있습니다:

- **SSO/OAuth 사용자**: 최대 10명
- **전체 사용자**: 최대 50명

이 제한은 프론트엔드와 백엔드 모두에서 확인됩니다:
- **프론트엔드**: `frontend/src/lib/components/AuthSettings.svelte` (96-100번 줄)
- **백엔드**: `backend/windmill-api/src/oauth2_oss.rs` (172-194번 줄)

---

## 제한이 구현된 위치

### 1. 백엔드 제한 (핵심)

**파일**: `backend/windmill-api/src/oauth2_oss.rs`

```rust
#[cfg(not(feature = "private"))]
pub async fn check_nb_of_user(db: &DB) -> error::Result<()> {
    let nb_users_sso =
        sqlx::query_scalar!("SELECT COUNT(*) FROM password WHERE login_type != 'password'",)
            .fetch_one(db)
            .await?;
    if nb_users_sso.unwrap_or(0) >= 10 {
        return Err(error::Error::BadRequest(
            "You have reached the maximum number of oauth users accounts (10) without an enterprise license"
                .to_string(),
        ));
    }

    let nb_users = sqlx::query_scalar!("SELECT COUNT(*) FROM password",)
        .fetch_one(db)
        .await?;
    if nb_users.unwrap_or(0) >= 50 {
        return Err(error::Error::BadRequest(
            "You have reached the maximum number of accounts (50) without an enterprise license"
                .to_string(),
        ));
    }
    return Ok(());
}
```

**동작 방식**:
- `password` 테이블에서 `login_type != 'password'`인 사용자 수를 카운트
- 10명 이상이면 에러 반환
- 전체 사용자가 50명 이상이면 에러 반환

### 2. 프론트엔드 경고

**파일**: `frontend/src/lib/components/AuthSettings.svelte`

```svelte
<!-- {#if !$enterpriseLicense || $enterpriseLicense.endsWith('_pro')}
    <Alert type="warning" title="Limited to 10 SSO users">
        Without EE, the number of SSO users is limited to 10. SCIM/SAML is available on EE
    </Alert>
{/if} -->
```

**상태**: ✅ **주석 처리됨** - 이 UI 경고는 주석 처리되어 사용자에게 표시되지 않습니다.

### 3. 데이터베이스 스키마

제한은 PostgreSQL 데이터베이스의 `password` 테이블을 기반으로 합니다:
- `email`: 사용자 이메일 (primary key)
- `login_type`: 로그인 타입 (`'password'`, `'github'`, `'gitlab'`, `'google'`, 등)
- 기타 필드들...

---

## 해결 방법

### 방법 1: 소스 코드 수정 후 커스텀 빌드 (권장)

이 방법이 가장 깔끔하고 유지보수가 용이합니다.

#### 1-1. 백엔드 수정

`backend/windmill-api/src/oauth2_oss.rs` 파일의 `check_nb_of_user` 함수를 수정:

```rust
#[cfg(not(feature = "private"))]
pub async fn check_nb_of_user(db: &DB) -> error::Result<()> {
    // SSO 사용자 제한을 제거하거나 늘림
    let nb_users_sso =
        sqlx::query_scalar!("SELECT COUNT(*) FROM password WHERE login_type != 'password'",)
            .fetch_one(db)
            .await?;
    
    // 원하는 제한 수로 변경 (예: 100명)
    if nb_users_sso.unwrap_or(0) >= 100 {
        return Err(error::Error::BadRequest(
            "You have reached the maximum number of oauth users accounts (100)"
                .to_string(),
        ));
    }

    let nb_users = sqlx::query_scalar!("SELECT COUNT(*) FROM password",)
        .fetch_one(db)
        .await?;
    
    // 전체 사용자 제한도 늘림 (예: 200명)
    if nb_users.unwrap_or(0) >= 200 {
        return Err(error::Error::BadRequest(
            "You have reached the maximum number of accounts (200)"
                .to_string(),
        ));
    }
    return Ok(());
}
```

또는 제한을 완전히 제거:

```rust
#[cfg(not(feature = "private"))]
pub async fn check_nb_of_user(db: &DB) -> error::Result<()> {
    // 제한 없음 - 항상 성공 반환
    return Ok(());
}
```

#### 1-2. 프론트엔드 경고 제거 (선택사항)

`frontend/src/lib/components/AuthSettings.svelte` 파일에서 경고 메시지 제거 또는 수정:

```svelte
{#if !$enterpriseLicense || $enterpriseLicense.endsWith('_pro')}
    <Alert type="info" title="Custom Build">
        This is a custom build with modified user limits.
    </Alert>
{/if}
```

### 방법 2: 제한 완전 제거 (가장 단순)

더 간단한 방법으로, 함수를 완전히 비워두는 것:

```rust
#[cfg(not(feature = "private"))]
pub async fn check_nb_of_user(db: &DB) -> error::Result<()> {
    Ok(())
}
```

---

## 커스텀 빌드 방법

### 사전 요구사항

- Docker 설치 (Colima 권장: macOS용 경량 Docker 런타임)
- gcloud CLI 설치 및 인증 완료
- 충분한 디스크 공간 (최소 20GB)
- 빌드 시간: 약 30-60분 (하드웨어에 따라 다름)

### 빌드 단계

#### 방법 A: 자동화 스크립트 사용 (권장) 🎯

**상태**: ✅ 소스 코드 수정 완료, 스크립트 준비 완료, 빌드 최적화 완료

이 프로젝트에는 빌드와 배포를 자동화하는 `./custom/build_and_push.sh` 스크립트가 포함되어 있습니다.

**⚡ 빌드 최적화**: 패키지 캐싱을 건너뛰도록 설정하여 빌드 시간을 30-90분 단축했습니다. 대신 첫 실행 시 필요한 패키지가 다운로드됩니다.

**1. 스크립트 사용 방법**

```bash
# custom 디렉토리로 이동
cd custom

# 빌드 및 GCR 푸시 (전체 프로세스)
./build_and_push.sh

# 빌드만 수행 (푸시 안함)
./build_and_push.sh --build-only

# 푸시만 수행 (이미 빌드된 이미지)
./build_and_push.sh --push-only

# 도움말 보기
./build_and_push.sh --help
```

**2. 스크립트 구성**

스크립트는 다음을 자동으로 처리합니다:

- **이미지 정보**:
  - Registry: `us.gcr.io`
  - Project: `liner-219011`
  - Image: `windmill/omni:custom-1`
  - 전체 경로: `us.gcr.io/liner-219011/windmill/omni:custom-1`

- **활성화된 Features** (CE 버전):
  ```
  oauth2,static_frontend,all_languages,prometheus
  ```
  
  참고: `enterprise` feature는 의도적으로 제외하여 순수 CE를 유지합니다.

- **자동 체크**:
  - Docker 및 gcloud 설치 확인
  - 커스텀 수정사항 확인
  - 디스크 공간 확인 (20GB 이상 권장)
  - GCR 인증 자동 설정

**3. 빌드 프로세스 모니터링**

빌드 중에는 다음과 같은 정보가 표시됩니다:

```
[INFO] windmill Docker Build & Push Script
======================================

[WARNING] CUSTOM BUILD NOTICE
This build includes custom modifications:
  - User limit check removed (no 10 SSO user limit)
  - Edition: Community Edition (CE)
  - Features enabled: oauth2,static_frontend,all_languages,prometheus

[INFO] Estimated build time: 20-40 minutes (optimized, no package caching)
[INFO] Required disk space: ~20GB

[SUCCESS] Custom modification detected: User limits removed
[INFO] Available disk space: 45GB
[SUCCESS] All prerequisites met

[INFO] Building Docker image...
[INFO] Platform: linux/amd64
[INFO] Features: oauth2,static_frontend,...
...
```

**4. 스크립트 커스터마이징**

필요에 따라 `build_and_push.sh`를 수정할 수 있습니다:

```bash
# 이미지 태그 변경
IMAGE_TAG="custom-2"  # custom-1 → custom-2

# Features 조정 (최소 빌드)
BUILD_FEATURES="oauth2,static_frontend"  # 언어 런타임 제외

# Features 추가 (파일 포맷, AI 기능 등)
BUILD_FEATURES="oauth2,static_frontend,all_languages,prometheus,parquet,embedding"

# 플랫폼 변경
DEFAULT_PLATFORM="linux/arm64"  # M4 Mac에서 실행할 경우
```

---

#### 방법 B: 수동 빌드 (고급 사용자용)

**1. 소스 코드 수정**

위의 "해결 방법" 섹션에 따라 필요한 파일들을 수정합니다.

**2. Docker 이미지 빌드**

프로젝트 루트에서 다음 명령어를 실행:

```bash
# CE 빌드 (권장)
docker build -t windmill-custom:latest \
  --build-arg features="oauth2,static_frontend,all_languages,prometheus" \
  -f Dockerfile .

# 최소 빌드 (언어 런타임 제외)
docker build -t windmill-custom:latest \
  --build-arg features="oauth2,static_frontend" \
  -f Dockerfile .

# 확장 빌드 (추가 기능 포함)
docker build -t windmill-custom:latest \
  --build-arg features="oauth2,static_frontend,all_languages,prometheus,parquet,embedding" \
  -f Dockerfile .
```

주요 feature flags:
- `oauth2`: OAuth2 인증 지원 ⭐ **필수** (Google SSO 사용)
- `static_frontend`: 프론트엔드를 바이너리에 포함 ⭐ **필수**
- `all_languages`: 모든 언어 런타임 지원 (권장)
- `prometheus`: Prometheus 메트릭 (권장)
- `parquet`: Parquet 파일 포맷 지원 (선택)
- `embedding`: AI/임베딩 기능 (선택)

⚠️ **주의**: `enterprise` feature는 CE를 유지하기 위해 의도적으로 제외했습니다.

**3. GKE에 배포**

##### 3-1. 이미지를 Google Container Registry에 푸시

```bash
# 이미지 태그 변경
docker tag windmill-custom:latest gcr.io/[YOUR-PROJECT-ID]/windmill-custom:latest

# GCR에 푸시
docker push gcr.io/[YOUR-PROJECT-ID]/windmill-custom:latest
```

##### 3-2. Kubernetes 배포 설정 업데이트

기존 Windmill 배포 YAML에서 이미지를 커스텀 이미지로 변경:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windmill-server
spec:
  template:
    spec:
      containers:
      - name: windmill-server
        image: gcr.io/[YOUR-PROJECT-ID]/windmill-custom:latest
        # 나머지 설정은 동일...
```

Worker 배포도 동일하게 업데이트합니다.

##### 3-3. 배포 적용

```bash
kubectl apply -f your-windmill-deployment.yaml
kubectl rollout restart deployment/windmill-server
kubectl rollout restart deployment/windmill-worker
```

### 빌드 최적화 팁

1. **빌드 캐시 활용**: Docker BuildKit 사용
   ```bash
   DOCKER_BUILDKIT=1 docker build ...
   ```

2. **Multi-stage 빌드**: Dockerfile은 이미 최적화되어 있음

3. **특정 기능만 빌드**: 필요한 features만 선택하여 빌드 시간 단축

---

## 주의사항

### 1. 법적/라이선스 고려사항

⚠️ **중요**: Windmill의 라이선스를 확인하세요.

- **파일**: `LICENSE`, `LICENSE-AGPL`, `LICENSE-APACHE`
- **Community Edition**: 특정 제한이 있는 proprietary 라이선스
- **Open Source 부분**: AGPLv3 라이선스

`LICENSE` 파일 내용:
```
The "Community Edition" of Windmill available in the docker images hosted under
ghcr.io/windmill-labs/windmill and the github binary releases contains the files
under the AGPLv3 and Apache 2 sources but also includes proprietary and
non-public code and features which are not open source and under the following
terms: Windmill Labs, Inc. grants a right to use all the features of the
"Community Edition" for free without restrictions other than the limits and
quotas set in the software...
```

**주의**: 
- 소스 코드에서 빌드한 바이너리는 AGPLv3 라이선스를 따름
- Community Edition의 제한을 우회하는 것은 Windmill Labs의 라이선스 정책과 상충될 수 있음
- 내부 사용 목적이라면 문제가 적지만, 상업적 사용이나 재배포는 주의 필요

### 2. 업데이트 관리

- 커스텀 빌드를 유지하면 공식 업데이트를 자동으로 받을 수 없음
- 정기적으로 upstream 변경사항을 merge해야 함
- Git을 사용한 버전 관리 권장:

```bash
# upstream 원격 저장소 추가
git remote add upstream https://github.com/windmill-labs/windmill.git

# 최신 변경사항 가져오기
git fetch upstream

# 변경사항 merge
git merge upstream/main

# 충돌 해결 후 커스텀 빌드 재실행
```

### 3. 보안 고려사항

- 사용자 제한은 보안상의 이유로 설정된 것일 수 있음
- 많은 수의 사용자를 지원할 때 고려사항:
  - 데이터베이스 성능
  - 리소스 사용량
  - 액세스 제어 및 권한 관리
  - 감사 로그 용량

### 4. 기술적 고려사항

#### 데이터베이스 성능
- PostgreSQL 설정 최적화 필요
- Connection pooling 설정 확인
- 인덱스 최적화

#### 리소스 할당
- GKE 노드 크기 조정
- 메모리 및 CPU 리소스 증가
- 오토스케일링 설정

#### 모니터링
- Prometheus 메트릭 활용
- 사용자 증가에 따른 성능 모니터링
- 로그 수집 및 분석

### 5. 대안 고려

#### Enterprise Edition 구매
- 공식 지원 및 업데이트
- 추가 기능 (SCIM, SAML, etc.)
- 법적 문제 없음

#### 하이브리드 접근
- 중요 사용자는 SSO
- 나머지는 일반 password 로그인
- 그룹 계정 활용

---

## 변경 이력 추적

커스텀 수정사항을 추적하기 위한 체크리스트:

- [ ] `backend/windmill-api/src/oauth2_oss.rs` - check_nb_of_user 함수 수정
- [ ] `frontend/src/lib/components/AuthSettings.svelte` - 경고 메시지 수정
- [ ] Docker 이미지 빌드 및 테스트
- [ ] GKE 배포 테스트
- [ ] 문서화 완료
- [ ] 팀원들에게 공유

---

## 문제 해결

### 빌드 실패 시

1. **SQLX 오류**:
   - `SQLX_OFFLINE=true` 환경 변수가 설정되어 있는지 확인
   - 이미 Dockerfile에 포함되어 있음

2. **메모리 부족**:
   - Docker Desktop 메모리 할당 증가 (최소 8GB 권장)
   - `docker build --memory=8g ...`

3. **네트워크 문제**:
   - cargo registry 접근 문제 시 재시도
   - VPN 연결 확인

### 런타임 오류

1. **데이터베이스 연결 실패**:
   - PostgreSQL 접속 정보 확인
   - 환경 변수 설정 확인

2. **OAuth 설정 오류**:
   - Google OAuth credentials 확인
   - Redirect URI 설정 확인
   - Base URL 설정 확인

---

## 추가 리소스

- [Windmill 공식 문서](https://www.windmill.dev/docs)
- [Windmill GitHub](https://github.com/windmill-labs/windmill)
- [Windmill Discord](https://discord.gg/V7PM2YHsPB)
- [Self-hosting 가이드](https://www.windmill.dev/docs/advanced/self_host)

---

## 문의

이 가이드에 대한 질문이나 이슈가 있으면:
1. 내부 팀 채널에 문의
2. Windmill Discord 커뮤니티 참고 (라이선스 관련 질문은 주의)
3. 필요시 Windmill Labs에 공식 문의

---

**작성일**: 2025-11-12  
**버전**: 1.0  
**Windmill 버전 기준**: 1.574.3

