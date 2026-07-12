
작성일: 2026-07-09

## 배경

EC2, Docker Compose, Apache reverse proxy, Oracle Cloud Autonomous Database, GHCR 이미지를 이용해 VR 전시 서비스 배포를 진행했다. 배포 과정에서 DB wallet, Docker 이미지 빌드 컨텍스트, Apache 프록시, HTTPS/WebSocket, AI 서버 연동 문제가 연쇄적으로 발생했다.

이번 회고의 목적은 단순히 "무엇을 고쳤는지"가 아니라, 왜 문제가 생겼고 다음 배포에서 어떤 순서로 검증해야 하는지 남기는 것이다.

## 주요 문제와 해결

### 1. Oracle Cloud wallet을 컨테이너에서 찾지 못함

증상:

```text
ORA-12263: Failed to access tnsnames.ora in the directory configured as TNS admin
```

원인:

EC2 호스트에는 wallet 폴더가 있었지만 Docker 컨테이너 내부에는 마운트되어 있지 않았다. 컨테이너는 호스트 파일시스템을 자동으로 볼 수 없기 때문에 `TNS_ADMIN=/home/ubuntu/Wallet_...` 경로가 컨테이너 안에서는 존재하지 않았다.

해결:

`docker-compose.yml`의 backend 서비스에 wallet 볼륨을 read-only로 마운트했다.

```yaml
volumes:
  - /home/ubuntu/Wallet_NR2R3OD7FT9HIEO5:/home/ubuntu/Wallet_NR2R3OD7FT9HIEO5:ro
```

재발 방지:

외부 인증서, wallet, 업로드 폴더처럼 런타임에 필요한 파일은 "호스트에 존재하는지"와 "컨테이너 내부에서 보이는지"를 따로 확인한다.

```bash
docker exec -it ai-exhibition-backend ls -al /home/ubuntu/Wallet_NR2R3OD7FT9HIEO5
```

### 2. Oracle wallet SSO keystore 라이브러리 누락

증상:

```text
ORA-17957: Unable to initialize the key store
java.security.KeyStoreException: SSO not found
ClassNotFoundException: oracle.security.crypto.core.AuthenticationException
```

원인:

`ojdbc11`만 포함되어 있었고, Oracle wallet의 `cwallet.sso`를 읽기 위한 companion jar가 부족했다. 처음에는 `oraclepki`만 추가해도 될 것처럼 보였지만 실제로는 `osdt_core`, `osdt_cert`도 필요했다.

해결:

`backend/pom.xml`에 Oracle JDBC와 wallet 관련 런타임 의존성을 명시했다.

```xml
<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc11</artifactId>
    <version>23.5.0.24.07</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>com.oracle.database.security</groupId>
    <artifactId>oraclepki</artifactId>
    <version>23.5.0.24.07</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>com.oracle.database.security</groupId>
    <artifactId>osdt_core</artifactId>
    <version>21.21.0.0</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>com.oracle.database.security</groupId>
    <artifactId>osdt_cert</artifactId>
    <version>21.21.0.0</version>
    <scope>runtime</scope>
</dependency>
```

검증:

빌드된 Spring Boot jar 안에 실제 의존성이 포함됐는지 확인했다.

```text
BOOT-INF/lib/ojdbc11-23.5.0.24.07.jar
BOOT-INF/lib/oraclepki-23.5.0.24.07.jar
BOOT-INF/lib/osdt_core-21.21.0.0.jar
BOOT-INF/lib/osdt_cert-21.21.0.0.jar
```

재발 방지:

의존성 문제는 "빌드 성공"만 보지 말고 최종 fat jar 내부까지 확인한다.

### 3. GHCR 이미지 태그와 owner 문제

증상:

```text
tag does not exist
error from registry: not_found: owner not found
```

원인:

로컬에 push하려는 태그가 없거나, GHCR owner 이름이 실제 GitHub 계정/조직명과 맞지 않았다.

해결:

이미지를 정확한 registry 이름으로 다시 태그하고, 실제 owner 기준으로 push했다.

```bash
docker tag ai-exhibition-backend:wallet-fix ghcr.io/<owner>/ai-exhibition-backend:<tag>
docker push ghcr.io/<owner>/ai-exhibition-backend:<tag>
```

재발 방지:

push 전에 항상 로컬 태그와 로그인 계정을 확인한다.

```bash
docker images | grep ai-exhibition
docker login ghcr.io -u <github-user>
```

### 4. Monorepo build context 문제로 seed 파일 누락

증상:

```text
FileNotFoundException: class path resource [docent-context.json] cannot be opened because it does not exist
```

원인:

백엔드는 `../shared`를 Maven resource로 포함하도록 되어 있었지만, Docker build를 `backend` 폴더 context로 실행하고 있었다. 그 결과 repo 루트의 `shared/docent-context.json`이 이미지에 들어가지 않았다.

해결:

`backend/Dockerfile`을 repo root context 기준으로 수정했다.

```dockerfile
COPY backend/pom.xml ./
COPY backend/src ./src
COPY shared ../shared
```

빌드는 루트에서 실행하도록 변경했다.

```bash
docker build --no-cache -f backend/Dockerfile -t ghcr.io/<owner>/ai-exhibition-backend:<tag> .
```

또한 root `.dockerignore`를 추가해 전체 monorepo가 무제한으로 Docker context에 들어가지 않도록 제한했다.

검증:

```text
BOOT-INF/classes/gallery-seed.json
BOOT-INF/classes/docent-context.json
```

재발 방지:

Monorepo에서 Docker build context는 루트로 잡되, `.dockerignore`로 필요한 경로만 허용한다.

### 5. Apache HTTPS 인증서 마운트 문제

증상:

```text
SSLCertificateFile: file '/etc/letsencrypt/live/.../fullchain.pem' does not exist or is empty
```

원인:

EC2 호스트에는 인증서가 있었지만 컨테이너에 `/etc/letsencrypt` 전체가 마운트되지 않았거나, `live`만 마운트되었을 가능성이 있었다. Let's Encrypt의 `live` 파일들은 실제 파일이 아니라 `archive`를 가리키는 symlink다.

해결:

Apache 컨테이너에 `/etc/letsencrypt` 전체를 read-only로 마운트해야 한다.

```yaml
volumes:
  - /etc/letsencrypt:/etc/letsencrypt:ro
```

재발 방지:

인증서 symlink 대상까지 컨테이너 안에서 확인한다.

```bash
docker exec -it ai-exhibition-apache ls -al /etc/letsencrypt/live/<domain>/
docker exec -it ai-exhibition-apache ls -al /etc/letsencrypt/archive/<domain>/
```

### 6. Apache module 및 reverse proxy 이름 문제

증상:

```text
Invalid command 'Redirect'
DNS lookup failure for: backend returned by /api
```

원인:

`Redirect`는 `mod_alias`가 필요하다. 또한 Apache의 `ProxyPass`가 Docker Compose 서비스명과 맞아야 한다. Compose 네트워크에서는 컨테이너 이름보다 서비스명을 기준으로 접근하는 것이 안전하다.

해결:

Apache 설정에 필요한 모듈을 명시했다.

```apache
LoadModule alias_module modules/mod_alias.so
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule proxy_wstunnel_module modules/mod_proxy_wstunnel.so
```

Compose 서비스명을 확인했다.

```bash
docker compose config --services
```

결과:

```text
ai-server
frontend
backend
apache
```

Apache proxy target을 서비스명 기준으로 유지했다.

```apache
ProxyPass /api http://backend:8080/api
ProxyPass /ws/gallery ws://backend:8080/ws/gallery
ProxyPass / http://frontend:8080/
```

재발 방지:

프록시 오류가 나면 컨테이너 내부 DNS부터 확인한다.

```bash
docker exec -it ai-exhibition-apache getent hosts backend
docker exec -it ai-exhibition-apache getent hosts frontend
```

### 7. AI 서버 연동 점검

증상:

AI 도슨트가 동작하지 않는 것으로 보였고, ai-server 경로 문제가 의심되었다.

확인:

백엔드 기본 AI 서버 URL은 다음과 같다.

```yaml
app:
  ai-server:
    base-url: ${AI_SERVER_BASE_URL:http://ai-server:8010}
```

Compose 서비스명도 `ai-server`였으므로 DNS 이름은 맞았다.

ai-server health 확인:

```bash
docker run --rm --network "$NET" curlimages/curl:8.8.0 http://ai-server:8010/health
```

응답에서 Gemini 키가 정상 주입된 것을 확인했다.

```json
"gemini": {
  "configured": true
}
```

재발 방지:

AI 기능 문제는 다음 순서로 분리해서 확인한다.

1. ai-server 컨테이너가 떠 있는가
2. backend 컨테이너에서 `http://ai-server:8010/health`가 되는가
3. health 응답에서 `gemini.configured`가 `true`인가
4. `/ai/explain` 직접 호출이 성공하는가
5. backend의 `/api/ai/explain` 호출이 성공하는가

### 8. HTTPS 페이지에서 WebSocket Mixed Content 발생

증상:

```text
Mixed Content: The page was loaded over HTTPS, but attempted to connect to the insecure WebSocket endpoint 'ws://...:8080/ws/gallery'
```

원인:

프론트 Dockerfile의 기본 build arg가 HTTP/WS 주소로 고정되어 있었다.

```dockerfile
ARG VITE_API_BASE_URL=http://vr-curatortest002.duckdns.org:8080
ARG VITE_WS_BASE_URL=ws://vr-curatortest002.duckdns.org:8080/ws/gallery
```

HTTPS 배포에서는 브라우저가 `ws://` 연결을 차단한다. Apache가 `/api`, `/ws/gallery`를 HTTPS 도메인에서 프록시하고 있으므로 프론트에는 절대 HTTP/WS 주소를 굳힐 필요가 없었다.

해결:

프론트 Dockerfile의 기본값을 비웠다.

```dockerfile
ARG VITE_API_BASE_URL=
ARG VITE_WS_BASE_URL=
```

프론트 코드는 빈 값이면 현재 origin 기준으로 자동 생성한다.

```text
https://domain  ->  wss://domain/ws/gallery
```

재발 방지:

운영/발표 배포에서는 프론트 번들에 `localhost`, `:8080`, `http://`, `ws://`가 박혀 있는지 검사한다.

```bash
grep -R "ws://\|http://.*8080\|localhost" frontend/dist
```

## 배포 검증 체크리스트

### 이미지 빌드 전

- GHCR owner, image name, tag가 맞는지 확인한다.
- backend 이미지는 repo root에서 `-f backend/Dockerfile`로 빌드한다.
- frontend 빌드에 HTTPS 환경에서 부적절한 `VITE_API_BASE_URL`, `VITE_WS_BASE_URL`이 들어가지 않았는지 확인한다.
- ai-server env에 `GEMINI_API_KEY` 또는 `GEMINI_API_KEYS`가 들어가는지 확인한다.

### 컨테이너 기동 후

```bash
docker compose ps
docker compose config --services
```

### DB 연결 확인

```bash
docker compose logs backend | grep -i "HikariPool"
```

정상 예:

```text
HikariPool-1 - Start completed.
```

### ai-server 확인

```bash
NET=$(docker inspect ai-exhibition-backend --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}{{end}}')
docker run --rm --network "$NET" curlimages/curl:8.8.0 http://ai-server:8010/health
```

### Apache 확인

```bash
docker exec -it ai-exhibition-apache getent hosts backend
docker exec -it ai-exhibition-apache getent hosts frontend
docker exec -it ai-exhibition-apache ls -al /etc/letsencrypt/live/vr-curatortest002.duckdns.org/
```

### 브라우저 확인

- `Mixed Content` 오류가 없어야 한다.
- WebSocket 주소가 `wss://<domain>/ws/gallery`여야 한다.
- `googleads.g.doubleclick.net ... ERR_BLOCKED_BY_CLIENT`는 광고 차단 확장 프로그램에 의한 외부 요청 차단이므로 서비스 오류로 보지 않는다.

## 배운 점

- Docker에서 "호스트에 파일이 있다"와 "컨테이너에서 접근 가능하다"는 완전히 다른 문제다.
- Oracle Autonomous DB wallet은 JDBC driver만으로 충분하지 않고 wallet 보안 companion jar까지 포함해야 한다.
- Monorepo Docker build는 root context와 엄격한 `.dockerignore` 조합이 가장 안정적이다.
- HTTPS 뒤에 Apache reverse proxy를 세운다면 프론트 번들에는 내부 포트나 `ws://` 주소를 박지 않는 편이 안전하다.
- Compose 내부 통신은 서비스명 기준으로 설계하고, 문제 발생 시 컨테이너 내부 DNS부터 확인해야 한다.
- 배포 문제는 한 번에 보려고 하면 헷갈린다. DB, AI 서버, Apache, 브라우저 콘솔을 각각 독립적으로 검증해야 빠르게 좁혀진다.

## 다음 배포에서 지킬 원칙

1. 먼저 `docker compose config --services`로 서비스명을 확정한다.
2. 각 컨테이너 내부에서 필요한 파일과 DNS를 직접 확인한다.
3. backend, ai-server, apache, frontend를 각각 health/API 단위로 검증한다.
4. 프론트 빌드 결과물에 환경별 절대 URL이 박혀 있는지 확인한다.
5. 오류 로그는 최신 로그와 과거 재시작 로그를 구분해서 본다.
