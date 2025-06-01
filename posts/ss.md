## ✨ 목적

Ubuntu 20.04 서버 환경에서 **ZooKeeper 없이 Kafka (KRaft 모드)**와 Redis를 설치하고, GitLab CI/CD 배포 환경에 적절히 반영하며, 데이터 저장 경로까지 파악하는 가이드입니다.

---

## ✅ 1. Redis 설치 및 설정

### 👉 설치

```jsx
sudo apt update
sudo apt install redis -y
```

### 👉 서비스 실행 및 확인

```
sudo systemctl enable redis-server
sudo systemctl start redis-server
redis-cli ping  # → PONG
```

### 👉 Redis 설정 파일 경로

- 설정 파일: `/etc/redis/redis.conf`
- 데이터 저장 위치:
    
    ```
    grep ^dir /etc/redis/redis.conf
    # 예시 출력: dir /var/lib/redis/
    ```
    
- 로그 파일: `/var/log/redis/redis-server.log`

---

## ✅ 2. Kafka (KRaft Mode) 설치 및 설정 (ZooKeeper 없이)

### 📁 설치 경로 추천

```
cd /opt
sudo curl -L -O https://archive.apache.org/dist/kafka/3.5.1/kafka_2.13-3.5.1.tgz
sudo tar -xzf kafka_2.13-3.5.1.tgz
sudo mv kafka_2.13-3.5.1 kafka
```

### 📅 KRaft ID 생성 (초기 1회)
⇒ eKunQsxuTvW1qx6ZMYThPQ

```
sudo /opt/kafka/bin/kafka-storage.sh random-uuid
# 출력 예: 0d9a7f71-d017-4b68-a5d5-dbd8e11823cd
```

### 🔧 로그 디렉토리 초기화

```
sudo /opt/kafka/bin/kafka-storage.sh format \
  --config /opt/kafka/config/kraft/server.properties \
  --cluster-id QrCFCWdEQxWpFOhk1oR10Q
```

### 🚀 Kafka 실행 (KRaft 모드)

```
sudo /opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/kraft/server.properties
```

### 🔧 시스템 서비스 등록 (선택)

`/etc/systemd/system/kafka.service`

```
[Unit]
Description=Apache Kafka Server
After=network.target

[Service]
Type=simple
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/kraft/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reexec
sudo systemctl enable kafka
sudo systemctl start kafka
```

### 📂 Kafka 데이터 저장 위치

- 기본 저장 경로: `/tmp/kraft-combined-logs`
- 설정 파일 내 확인:
    
    ```
    grep log.dirs /opt/kafka/config/kraft/server.properties
    ```
    
    원한다면 `/var/lib/kafka-logs` 같은 경로로 변경 가능
    

---

## ⚖️ 3. GitLab CI/CD 배포 연동 시 고려사항

### 📋 `.gitlab-ci.yml` 예시

```
deploy-to-prod:
  stage: deploy
  script:
    - echo "Deploying to production..."
    - ssh user@your-server-ip 'bash ~/deploy.sh'
  only:
    - main
```

### 🔧 서버에 필요한 설정

- Kafka와 Redis가 **백그라운드에서 항상 실행 중**이어야 함
- 시스템 재시작 시 자동 실행되도록 systemd 등록 (`redis-server`, `kafka.service`)
- Kafka 포트(9092), Redis 포트(6379) 열려 있어야 함 (필요시 방화벽 `ufw allow`)

```
sudo ufw allow 6379
sudo ufw allow 9092
```

### 🌌 CI/CD로 배포할 때 유의사항

- `application.yml`이나 환경 변수에 Kafka/Redis 접속 정보를 반드시 명시해야 함
- 배포 스크립트 안에서 Kafka 토픽 생성 스크립트를 별도 실행할 수도 있음:
    
    ```
    /opt/kafka/bin/kafka-topics.sh --create --topic stock.inbound \
      --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
    ```
    

---

## ✅ 정리 요약

| 구성 요소 | 데이터 저장 위치 | 설정 파일 위치 | 서비스 실행 |
| --- | --- | --- | --- |
| Redis | `/var/lib/redis/` | `/etc/redis/redis.conf` | `systemctl start redis-server` |
| Kafka (KRaft) | `/tmp/kraft-combined-logs` → 수정 가능 | `/opt/kafka/config/kraft/server.properties` | `systemctl start kafka` (등록 시) |

---

이 구성을 바탕으로 Spring Boot + Kafka + Redis 연동이 가능합니다. 필요 시 systemd 등록/삭제, 모니터링 설정, 로그 순환까지 이어서 안내해드릴 수 있습니다.
