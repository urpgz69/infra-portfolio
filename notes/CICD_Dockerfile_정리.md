# CI/CD & Dockerfile 정리

> 인프라 배포 참고 노트. 개인(DevDesk)·팀 프로젝트 경험 기반.

---

## 1. 핵심 개념 구분

가장 헷갈리는 것부터. **build ≠ push ≠ deploy**, 그리고 **레지스트리 ≠ 배포 서버**.

| 단계 | 하는 일 | 검증 여부 |
| --- | --- | --- |
| **build** | Dockerfile로 이미지 생성 (컴파일·의존성·오류 검출) | ✅ "빌드되나"는 여기서 끝남 |
| **push** | 이미지를 레지스트리에 업로드 (공유용) | ❌ 검증 아님, 배포 준비 |
| **deploy** | 서버에서 이미지 pull → 실행 | 실제 배포 |

- **레지스트리** (Docker Hub / GHCR / ECR) = 이미지를 **어디에 저장**하나
- **배포 서버** (EC2 / Azure VM) = 이미지를 **어디서 실행**하나
- 둘은 독립적. 조합 자유 (EC2+GHCR, VM+Docker Hub 등)

**테스트만 하려면 `push: false`로 build만 해도 됨.** push는 배포/공유할 때만 필요.

---

## 2. CI vs CD

- **CI (Continuous Integration)** = 빌드/테스트 자동화. "코드가 깨지지 않았나" 검증.
- **CD (Continuous Deployment)** = 배포 자동화. 실제 서버에 반영.
- 팀 요청이 "빌드 테스트 자동화"면 → **CI까지**. 배포(CD)는 인프라 담당과 협의.

---

## 3. CI/CD 전체 순서

### 준비 (CI 붙이기 전)
1. 각 서비스 폴더에 **Dockerfile 작성 + git에 push** (만든 것 ≠ 올린 것)
2. **로컬에서 빌드 성공** 확인 (`./gradlew build` / `mvn package` / `npm run build`)
3. **`.env` gitignore 확인** — 시크릿이 git에 없어야 함 (`git ls-files | grep '\.env$'` 비어야 안전)
4. 지갑·인증서 등도 gitignore 확인

### CI 파이프라인 (GitHub Actions)
```
push/PR 발생
  → paths 필터로 해당 서비스만 트리거 (모노레포)
  → 러너(임시 ubuntu) 생성 ── 매번 빈 컴퓨터!
  → checkout (코드 가져오기)
  → 런타임 설치 (setup-java / setup-node)
  → 의존성 설치 + 빌드
  → [선택] 이미지 build & push (레지스트리로)
  → 초록불 / 빨간불
```

### CD 파이프라인 (배포까지)
```
CI 통과 (needs: build)
  → dev/main 브랜치일 때만 (if 조건)
  → SSH로 서버 접속 (appleboy/ssh-action)
  → .env의 이미지 태그 갱신 (sed)
  → [수동] docker compose up -d  ← 재시작은 사람이 (배포 시점 통제)
```

### 브랜치 전략과 트리거
- feature 브랜치 push → **빌드만** (검증), push:false 가능
- dev/main에 반영(머지든 직접 push든) → **build + push + VM 태그 갱신**
- `push: ${{ github.ref == 'refs/heads/dev' || github.ref == 'refs/heads/main' }}` 로 제어
- 실제 컨테이너 재시작(`docker compose up`)은 **여전히 수동**

---

## 4. 모노레포 CI 핵심 (GitHub Actions YAML)

```yaml
name: Backend CI
on:
  push:
    branches: [ main, dev ]        # "**"면 모든 브랜치 (이미지 많이 쌓임 주의)
    paths:
      - "backend/**"               # 해당 폴더 바뀔 때만 (모노레포 필수)
  pull_request:
    paths: [ "backend/**" ]
  workflow_dispatch:               # 수동 실행 버튼

jobs:
  build:
    runs-on: ubuntu-latest         # 러너 = 매번 새 빈 컴퓨터
    defaults:
      run:
        working-directory: backend # 명령을 이 폴더에서 실행
    steps:
      - uses: actions/checkout@v4              # 코드 가져오기 (첫 줄 필수)
      - uses: actions/setup-java@v4            # 런타임 설치
        with: { distribution: temurin, java-version: '17' }
      - run: chmod +x gradlew
      - run: ./gradlew build -x test           # -x test: DB 붙는 통합테스트 제외
```

**핵심 감각 3가지**
1. `on` = **언제** 도나 (트리거 + paths 필터)
2. `runs-on` = **어디서** 도나 (러너 = 빈 컴퓨터라 checkout·setup 매번 필요)
3. `steps` = **뭘** 하나 (위→아래 순서, `uses`=남의 액션 / `run`=직접 명령, 하나 실패시 멈춤)

### 프론트 CI (차이점만)
- `setup-java` → **`setup-node`**
- `gradlew build` → **`npm ci` → `npm run build`**
- `cache: 'npm'` + `cache-dependency-path: frontend/package-lock.json` (빌드 빠르게)
- **Vite 환경변수는 빌드 시점에 박힘** → `build-args`로 주입 (VITE_* 키가 안 박히면 앱 안 뜸)

---

## 5. CI/CD 고급 요소 (팀 프로젝트에서 본 것)

| 요소 | 의미 |
| --- | --- |
| `sha-${GITHUB_SHA::7}` | 커밋 해시 태그 = 배포 버전 **추적성** |
| `type=raw,value=latest,enable={{is_default_branch}}` | latest는 **main만** (아무 브랜치나 덮어쓰기 방지) |
| `docker/metadata-action` | 태그 여러 개 자동 생성 (latest + sha + 수동태그) |
| `cache-from/to: type=gha,scope=backend` | GitHub Actions 캐시, 서비스별 분리 |
| `docker/login-action` | **인증 pull** (익명 10회/시간 → 인증 40회/시간) |
| `needs: build` | build 성공 후에만 다음 job |

---

## 6. Dockerfile 구조

### 명령어 요약
| 명령 | 역할 |
| --- | --- |
| `FROM` | 출발 베이스 이미지 |
| `WORKDIR` | 작업 폴더 |
| `COPY 출처 목적지` | 파일 복사 (`.dockerignore`가 걸러줌) |
| `RUN` | **빌드 시점** 명령 (설치·빌드) |
| `ARG` | **빌드 시점** 변수 (build-args로 주입) |
| `ENV` | 환경변수 (실행 시점까지 유지) |
| `EXPOSE` | 포트 선언 (문서용, 실제 여는 건 compose ports) |
| `ENTRYPOINT`/`CMD` | **실행 시점** 명령 |

### 핵심 원리 3가지
1. **위→아래로 환경을 한 층씩 쌓음** (FROM → WORKDIR → COPY → RUN → CMD)
2. **멀티스테이지** = 빌드용/실행용 분리. 빌드 도구는 버리고 결과물만 → 이미지 경량화
3. **자주 바뀌는 건 아래로** (레이어 캐시) — 의존성 설치를 소스 복사보다 위에

### 백엔드 (Spring Boot, 멀티스테이지)
```dockerfile
# 1단계: JDK로 빌드
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY gradlew . 
COPY gradle gradle
COPY build.gradle settings.gradle ./   # 의존성 관련 먼저 (캐시)
COPY src src                            # 소스는 나중
RUN ./gradlew bootJar -x test           # JAR 생성, 테스트 스킵

# 2단계: JRE로 실행 (가벼움)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar   # 결과물만 가져옴
ENTRYPOINT ["java", "-jar", "app.jar"]
```
- **Maven이면**: `./mvnw package` (Gradle의 `gradlew build`와 다름), 결과물 `target/*.jar`
- 테스트가 DB(@SpringBootTest)에 붙으면 CI/이미지 빌드 시 `-x test`로 스킵

### 프론트 (React+Vite, node→nginx)
```dockerfile
# 1단계: node로 빌드
FROM node:20-alpine AS builder
WORKDIR /app
ARG VITE_API_BASE_URL=/api             # 빌드 시점 변수 (정적 파일에 박힘)
ENV VITE_API_BASE_URL=${VITE_API_BASE_URL}
COPY package*.json ./                   # 의존성 먼저 (캐시)
RUN npm ci
COPY . .                                # 소스 나중
RUN npm run build                       # dist/ 생성

# 2단계: nginx로 정적 서빙
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
CMD ["nginx", "-g", "daemon off;"]      # daemon off = 포그라운드 유지
```
- **주의**: Vite 환경변수(VITE_*)는 빌드 시점에 코드에 박힘. CI build-args와 GitHub Secrets 이름이 코드/Dockerfile/CI 세 곳에서 **정확히 일치**해야 함 (KEY 하나 안 맞으면 앱이 안 뜸)

### AI 서버 (FastAPI) + OpenCV base image 패턴
```dockerfile
# 무거운 의존성은 base 이미지로 (한 번만, 거의 안 바뀜)
FROM python:3.12-slim
RUN apt-get update && apt-get install -y libgl1 libglib2.0-0    # cv2 필요 라이브러리
RUN pip install torch --index-url .../whl/cpu                    # CPU 버전 (가벼움)
RUN pip install opencv-python-headless ultralytics              # headless = 서버용
# → myregistry/opencv-base:py312-yolo 로 push

# 앱 이미지는 base 위에 코드만 (자주 바뀜, 빌드 빠름)
FROM myregistry/opencv-base:py312-yolo
WORKDIR /app
COPY requirements.txt ./
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
- **base image 패턴**: 무거운 것(torch/YOLO)은 base에 한 번, 앱은 `FROM base`로 코드만 얹기 → 코드 변경 시 빌드 시간 대폭 단축
- Gemini만 호출하는 가벼운 AI 서버는 base 분리 불필요 (requirements.txt만)

---

## 7. docker-compose 핵심

```yaml
services:
  backend:
    build: { context: ./backend }
    image: myregistry/backend:${BACKEND_IMAGE_TAG:-004}   # :-004 = 기본값(롤백 안전장치)
    environment:
      DATASOURCE_PASSWORD: ${DATASOURCE_PASSWORD}          # 시크릿은 ${} 참조 (.env에서)
    volumes:
      - ./wallet:/app/wallet:ro                            # :ro = 읽기전용
    networks: [ shiftops-network ]
    restart: unless-stopped

networks:
  shiftops-network:
    driver: bridge      # 같은 네트워크의 컨테이너끼리 서비스명으로 통신 (http://backend:8080)
```

**주의점**
- 서비스들이 **같은 네트워크**여야 서비스명 통신됨 (하나 빠지면 못 닿음)
- 시크릿은 compose에 직접 X → `${VAR}` 참조 + `.env`(gitignore)
- 이미지 태그 `${TAG:-기본값}` = CI가 갱신, 없으면 기본값 (롤백)
- `build` + `image` 둘 다 있으면: 이미지 있으면 사용, 없으면 빌드 (순수 배포용이면 build 불필요)
- WebSocket 프록시엔 Upgrade/Connection 헤더 필수

---

## 8. 자주 밟는 함정 (경험)

- **YAML 들여쓰기**: 탭 금지, 같은 계층 같은 칸. `docker compose config -q`로 검증
- **cp 폴더 중첩**: `cp -r SRC DEST`에서 DEST가 이미 있으면 SRC가 그 **안으로** 들어감 → `rm -rf DEST` 후 복사
- **HikariCP "pool sealed"**: 풀은 시작 후 설정 봉인. 코드가 DataSource 직접 생성 + 환경변수 중복이면 충돌 → 환경변수 정리
- **디스크 부족**: `docker image/container/builder prune` (단 `--volumes`는 DB 날림, 금지). 근본 해결은 EBS 볼륨 확장(growpart + resize2fs, 무중단)
- **Docker Hub pull 제한**: 인증 40회/시간. 많이 걸리면 GHCR 검토
- **레지스트리 선택**: GHCR(GitHub 통합, 전환 쉬움) / ECR(AWS, 취업 유리) / ACR(Azure)

---

## 9. 배포 전 체크리스트

- [ ] Dockerfile이 git에 올라가 있나 (`git ls-files | grep -i dockerfile`)
- [ ] `.env`·지갑·인증서가 gitignore 됐나
- [ ] 로컬 빌드 성공하나
- [ ] 외부 의존성(DB, 외부 API)에 배포 서버에서 접속 되나 (ACL에 서버 IP 등록 등)
- [ ] 시크릿을 배포 서버에 어떻게 넣을지 (.env / Secrets Manager)
- [ ] 서버 스펙이 받쳐주나 (특히 메모리 무거운 서비스)
- [ ] nginx: WebSocket 헤더, 경로 라우팅, TLS
