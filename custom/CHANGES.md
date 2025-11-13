# 커스텀 빌드 변경 사항 요약

## 개요

Windmill Community Edition의 Google SSO 10명 제한을 제거하기 위한 커스텀 빌드입니다.

**날짜**: 2025-11-12  
**Windmill 버전**: 1.574.3  
**변경 방법**: 방법 2 - 제한 완전 제거 (가장 단순)

---

## 변경된 파일

### 1. 백엔드 코드 수정 (User Limit 제거)

**파일**: `backend/windmill-api/src/oauth2_oss.rs`

**변경 내용**:
```rust
// 이전 (172-194번 줄)
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

// 이후 (172-177번 줄)
#[cfg(not(feature = "private"))]
pub async fn check_nb_of_user(_db: &DB) -> error::Result<()> {
    // CUSTOM BUILD: User limit check removed
    // Original CE limits were: 10 SSO users, 50 total users
    // This custom build removes those limitations
    Ok(())
}
```

**효과**:
- SSO/OAuth 사용자 10명 제한 제거
- 전체 사용자 50명 제한 제거
- 사용자 수에 관계없이 항상 성공 반환

**변경 2**: OAuth 로그인 목록 표시 활성화 (133-149번 줄)
```rust
// 이전
#[cfg(not(feature = "private"))]
async fn list_logins() -> error::JsonResult<Logins> {
    // Implementation is not open source
    return Ok(Json(Logins { oauth: vec![], saml: None }));
}

// 이후
#[cfg(all(feature = "oauth2", not(feature = "private")))]
async fn list_logins() -> error::JsonResult<Logins> {
    // CUSTOM BUILD: Return actual OAuth logins configured in the system
    Ok(Json(Logins { 
        oauth: (&OAUTH_CLIENTS.read().await.logins)
            .keys()
            .map(|x| x.to_owned())
            .collect_vec(),
        saml: None 
    }))
}

#[cfg(not(all(feature = "oauth2", not(feature = "private"))))]
async fn list_logins() -> error::JsonResult<Logins> {
    // OAuth not enabled or private feature enabled
    return Ok(Json(Logins { oauth: vec![], saml: None }));
}
```

**효과**:
- 로그인 페이지에서 설정된 OAuth 제공자(Google, Microsoft 등) 버튼이 정상적으로 표시됨

---

### 2. 빌드 스크립트 생성

**파일**: `custom/build_and_push.sh`

**신규 생성** - 자동화된 빌드 및 배포 스크립트

**주요 기능**:
- Docker 이미지 빌드 자동화
- GCR 인증 자동 설정
- 전제 조건 자동 확인
  - Docker 데몬 실행 여부
  - gcloud CLI 설치
  - 디스크 공간 (20GB 이상)
  - 커스텀 수정사항 적용 여부
- 빌드 진행 상황 표시
- 에러 처리 및 로깅

**설정**:
```bash
GCR_REGISTRY="us.gcr.io"
GCP_PROJECT="liner-219011"
IMAGE_NAME="windmill/omni"
IMAGE_TAG="custom-1"
BUILD_FEATURES="oauth2,static_frontend,all_languages,prometheus"
DEFAULT_PLATFORM="linux/amd64"
```

**중요**: `enterprise` feature는 CE를 유지하기 위해 의도적으로 제외했습니다.

**사용법**:
```bash
./build_and_push.sh              # 빌드 + 푸시
./build_and_push.sh --build-only # 빌드만
./build_and_push.sh --push-only  # 푸시만
./build_and_push.sh --help       # 도움말
```

---

### 3. Dockerfile 최적화 (빌드 시간 단축)

**파일**: `Dockerfile`

**변경 내용**:
```dockerfile
# 이전 (224-226번 줄)
COPY ./frontend/src/lib/hubPaths.json ${APP}/hubPaths.json
RUN windmill cache ${APP}/hubPaths.json && rm ${APP}/hubPaths.json && chmod -R 777 /tmp/windmill

# 이후 (224-231번 줄)
# CUSTOM BUILD: Skip package caching to speed up build time
# This means packages will be downloaded on first use instead of being pre-cached
# Original lines (commented out to save 30-90 minutes build time):
# COPY ./frontend/src/lib/hubPaths.json ${APP}/hubPaths.json
# RUN windmill cache ${APP}/hubPaths.json && rm ${APP}/hubPaths.json && chmod -R 777 /tmp/windmill

# Just create the windmill temp directory
RUN mkdir -p /tmp/windmill && chmod -R 777 /tmp/windmill
```

**효과**:
- 빌드 시간: 30-90분 단축 (50-90분 → 20-40분)
- 트레이드오프: 첫 스크립트 실행 시 패키지 다운로드 필요
- 개발/테스트 환경에 적합

---

### 4. 문서 생성

**신규 파일**:
- `custom/README.md` - 상세 가이드 (530줄)
- `custom/QUICKSTART.md` - 빠른 시작 가이드
- `custom/CHANGES.md` - 이 파일

**README.md 주요 섹션**:
1. 빠른 시작 가이드
2. 제한 사항 개요 및 분석
3. 제한이 구현된 위치 (소스 코드 레벨)
4. 해결 방법 (여러 옵션)
5. 커스텀 빌드 방법 (자동/수동)
6. 주의사항 (라이선스, 보안, 성능)
7. 문제 해결 가이드

---

## 빌드 구성

### Features 활성화 (CE Edition)

| Feature | 설명 | 상태 |
|---------|------|------|
| `oauth2` | OAuth2 인증 지원 | ✅ 활성화 (필수) |
| `static_frontend` | 프론트엔드 포함 | ✅ 활성화 (필수) |
| `all_languages` | 모든 언어 런타임 | ✅ 활성화 (권장) |
| `prometheus` | Prometheus 메트릭 | ✅ 활성화 (권장) |
| `parquet` | Parquet 파일 지원 | ❌ 비활성화 (선택 가능) |
| `embedding` | 임베딩/AI 기능 | ❌ 비활성화 (선택 가능) |
| `enterprise` | 엔터프라이즈 기능 | ❌ **의도적으로 제외** |

**참고**: `enterprise` feature를 제외하여 순수 CE를 유지합니다. 필요시 `parquet`, `embedding` 등을 추가할 수 있습니다.

### 이미지 정보

```
Registry: us.gcr.io
Project:  liner-219011
Image:    windmill/omni
Tag:      custom-1
Full:     us.gcr.io/liner-219011/windmill/omni:custom-1
```

---

## 검증

### 코드 수정 확인

```bash
# 커스텀 마커 확인
grep "CUSTOM BUILD" backend/windmill-api/src/oauth2_oss.rs

# 예상 출력:
# // CUSTOM BUILD: User limit check removed
```

### 스크립트 실행 권한

```bash
# 실행 권한 확인
ls -la custom/build_and_push.sh

# 예상 출력:
# -rwxr-xr-x ... build_and_push.sh
```

### 빌드 테스트

```bash
# dry-run (실제 빌드 안함)
cd custom
./build_and_push.sh --help

# 전제 조건만 확인
# (스크립트 시작 후 Ctrl+C로 중단)
```

---

## 배포 프로세스

### 1. 빌드

```bash
cd custom
./build_and_push.sh
```

예상 시간: 30-60분

### 2. 확인

```bash
# GCR에서 이미지 확인
gcloud container images describe \
  us.gcr.io/liner-219011/windmill/omni:custom-1
```

### 3. GKE 배포

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windmill-server
spec:
  template:
    spec:
      containers:
      - name: windmill
        image: us.gcr.io/liner-219011/windmill/omni:custom-1
```

```bash
kubectl apply -f deployment.yaml
kubectl rollout restart deployment/windmill-server
kubectl rollout restart deployment/windmill-worker
```

### 4. 검증

```bash
# 서비스 상태 확인
kubectl get pods -l app=windmill

# 로그 확인
kubectl logs -f deployment/windmill-server

# SSO 로그인 테스트
# 10명 이상의 사용자로 테스트
```

---

## 주의사항

### ⚠️ 라이선스

- 이 커스텀 빌드는 Windmill CE의 라이선스 제한을 우회합니다
- 내부 사용 목적: 일반적으로 문제 없음
- 상업적 재배포: Windmill Labs와 협의 필요
- 원본 라이선스: AGPLv3 + Proprietary

### 🔄 업데이트 관리

```bash
# upstream 추가
git remote add upstream https://github.com/windmill-labs/windmill.git

# 최신 변경사항 가져오기
git fetch upstream
git merge upstream/main

# 충돌 해결 후 재빌드
cd custom
./build_and_push.sh
```

### 🔒 보안

- 사용자 수 증가 시 성능 영향 모니터링
- 액세스 제어 강화 권장
- 정기적인 보안 업데이트 적용
- 감사 로그 활성화 및 모니터링

### 📊 성능

- PostgreSQL 튜닝 권장
- Connection pooling 최적화
- GKE 노드 리소스 증가 고려
- Prometheus 메트릭으로 모니터링

---

## 롤백 계획

### 원본으로 복구

```bash
# 1. 원본 이미지로 전환
kubectl set image deployment/windmill-server \
  windmill=ghcr.io/windmill-labs/windmill:main

# 2. 또는 이전 버전으로
kubectl rollout undo deployment/windmill-server

# 3. 확인
kubectl rollout status deployment/windmill-server
```

### 코드 복구

```bash
# Git에서 원본 파일 복구
git checkout HEAD -- backend/windmill-api/src/oauth2_oss.rs
```

---

## 문제 해결 체크리스트

- [ ] Docker 데몬 실행 중?
- [ ] gcloud 인증 완료?
- [ ] 디스크 공간 충분? (20GB+)
- [ ] 코드 수정 적용됨?
- [ ] features에 oauth2 포함?
- [ ] 네트워크 연결 안정?
- [ ] GCR 권한 있음?
- [ ] Kubernetes 접근 권한?

---

## 연락처 및 리소스

### 내부 문의
- DevOps 팀
- Backend 팀

### 외부 리소스
- [Windmill Docs](https://www.windmill.dev/docs)
- [Windmill GitHub](https://github.com/windmill-labs/windmill)
- [Windmill Discord](https://discord.gg/V7PM2YHsPB)

---

**작성**: 2025-11-12  
**작성자**: AI Assistant  
**검토 필요**: DevOps 팀  
**승인 상태**: 초안

