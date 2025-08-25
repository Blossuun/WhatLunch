# 통합 크롤러

EC2에서 여러 사이트 크롤러를 하나의 이미지로 실행하기 위한 디렉터리입니다.
`CRAWLER_NAME` 환경변수에 따라 실행할 스크립트를 전환합니다.

## 구조

```
/opt/crawler
├─ compose.yaml       # 사이트별 서비스 정의 (단일 이미지 공유)
├─ crawl_plan.sh      # 서비스를 순차 실행하는 배치 스크립트
├─ register_cron.sh   # 크론 등록 스크립트 (매일 02:00)
├─ unregister_cron.sh # 크론 해제 스크립트
└─ app/
   ├─ Dockerfile      # Playwright 기반 이미지
   ├─ requirements.txt# 크롤링 라이브러리 (playwright 제외)
   ├─ run.py          # CRAWLER_NAME으로 엔트리 포인트 전환
   ├─ daily_crawler.sh# 온비드 베이스 → 디테일 일괄 실행
   ├─ sites/
   │  ├─ autoinside_daily_ec2.py
   │  ├─ autohub_daily_ec2.py
   │  └─ automart_daily_ec2.py
   └─ onbid/
      ├─ crawl_base.py
      └─ crawl_detail.py
```

## 사용 방법

### 이미지 빌드

```bash
docker build -t crawler app
```

### 개별 크롤러 실행

`compose.yaml`에서 각 서비스는 `CRAWLER_NAME`을 전달하여 단일 이미지를 재사용합니다.
직접 실행할 경우 다음과 같이 지정합니다.

```bash
CRAWLER_NAME=autoinside docker compose run --rm autoinside
```

### 배치 실행

`crawl_plan.sh`는 `docker compose run`을 이용해 모든 서비스를 순차 실행하고
로그를 `/var/log/<서비스명>` 디렉터리에 저장합니다.
`register_cron.sh`를 통해 매일 02:00에 실행하도록 크론에 등록할 수 있습니다.

## 헬스체크

컨테이너는 `/health` 엔드포인트를 통해 동작 여부를 노출합니다.
`compose.yaml`에는 이를 주기적으로 확인하는 `healthcheck`가 정의돼 있습니다.

`run.py`에서 헬스 서버를 구동하는 코드:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import threading

class HealthHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/health":
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b"OK")
        else:
            self.send_response(404)
            self.end_headers()

def start_health_server():
    HTTPServer(("0.0.0.0", 8080), HealthHandler).serve_forever()

threading.Thread(target=start_health_server, daemon=True).start()
```

`compose.yaml`의 각 서비스에는 다음과 같은 설정이 포함됩니다:

```yaml
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 1m
      timeout: 10s
      retries: 3
```

`curl http://localhost:8080/health`로 수동 점검할 수 있으며,
헬스체크 실패 시 Docker의 재시작 정책을 통해 자동 복구할 수 있습니다.

