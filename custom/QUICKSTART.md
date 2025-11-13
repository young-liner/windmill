# Windmill 커스텀 빌드 - 빠른 시작

## 🎯 현재 상태

✅ **완료된 작업:**
- Google SSO 10명 제한 제거 완료
- 백엔드 코드 수정 완료 (`backend/windmill-api/src/oauth2_oss.rs`)
- 빌드 스크립트 준비 완료 (`./custom/build_and_push.sh`)

⏭️ **다음 단계:**
- Docker 이미지 빌드 및 GCR 푸시

---

## 🚀 빌드 및 배포

### 1단계: 전제 조건 확인

```bash
# Docker 실행 확인 (Colima 사용 시)
docker info

# gcloud 인증 확인
gcloud auth list

# 디스크 공간 확인 (20GB 이상 필요)
df -h .
```

### 2단계: 빌드 실행

```bash
# custom 디렉토리로 이동
cd /Users/younghan/Projects/ln/windmill/custom

# 빌드 및 푸시 (자동)
./build_and_push.sh
```

### 3단계: GKE에 배포

빌드가 완료되면 다음 이미지가 생성됩니다:
```
us.gcr.io/liner-219011/windmill/omni:custom-1
```

Kubernetes 배포 YAML에서 이미지를 업데이트하세요:

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
        image: us.gcr.io/liner-219011/windmill/omni:custom-1
        # ... 기타 설정
```

```bash
# 배포 적용
kubectl apply -f your-deployment.yaml

# 롤아웃 재시작 (기존 배포 업데이트)
kubectl rollout restart deployment/windmill-server
kubectl rollout restart deployment/windmill-worker
```

---

## 📊 빌드 정보

| 항목 | 내용 |
|------|------|
| **이미지 경로** | `us.gcr.io/liner-219011/windmill/omni:custom-1` |
| **Edition** | Community Edition (CE) |
| **활성화된 Features** | oauth2, static_frontend, all_languages, prometheus |
| **플랫폼** | linux/amd64 |
| **예상 빌드 시간** | 20-40분 (최적화됨) |
| **필요 디스크 공간** | ~20GB |
| **수정 사항** | SSO 사용자 제한 10명 → 무제한 |
| **빌드 최적화** | 패키지 캐싱 건너뛰기 (30-90분 절약) |

---

## 🔧 스크립트 옵션

```bash
# 빌드만 수행 (푸시 안함)
./build_and_push.sh --build-only

# 푸시만 수행 (이미 빌드된 이미지)
./build_and_push.sh --push-only

# 도움말
./build_and_push.sh --help
```

---

## 🐛 문제 해결

### Docker 데몬 실행 안됨
```bash
# Colima 시작
colima start
```

### 디스크 공간 부족
```bash
# Docker 이미지 정리
docker system prune -a

# 빌드 캐시 정리
docker builder prune
```

### 빌드 실패
```bash
# 로그 확인
docker build --no-cache --progress=plain ...

# BuildKit 비활성화하고 재시도
DOCKER_BUILDKIT=0 docker build ...
```

### 인증 오류
```bash
# GCR 인증 재설정
gcloud auth configure-docker us.gcr.io

# gcloud 로그인 확인
gcloud auth login
```

---

## ✅ 검증

빌드 후 다음을 확인하세요:

```bash
# 로컬 이미지 확인
docker images | grep windmill

# GCR 이미지 확인
gcloud container images list --repository=us.gcr.io/liner-219011/windmill

# 이미지 테스트 (로컬)
docker run -p 8000:8000 us.gcr.io/liner-219011/windmill/omni:custom-1
```

---

## 📚 추가 참고 자료

- 상세 가이드: `./README.md`
- 빌드 스크립트: `./build_and_push.sh`
- 원본 Dockerfile: `../Dockerfile`
- 수정된 소스: `../backend/windmill-api/src/oauth2_oss.rs`

---

**작성일**: 2025-11-12  
**Windmill 버전**: 1.574.3  
**이미지 태그**: custom-1

