# PostgreSQL 백업 및 복구 가이드

## 📋 목차
- [백업 시스템 개요](#백업-시스템-개요)
- [초기 설정](#초기-설정)
- [백업 확인](#백업-확인)
- [데이터 복구 방법](#데이터-복구-방법)
- [트러블슈팅](#트러블슈팅)

---

## 백업 시스템 개요

### 백업 방식
- **자동 백업**: 매일 오전 3시(한국 시간) 자동 실행
- **저장 위치**: GitLab private repository
- **암호화**: AES256으로 암호화되어 저장
- **보관 기간**: 최근 3개의 백업만 유지

### 백업 파일 형식
```
postgres_backup_YYYYMMDD_HHMMSS.sql.gpg
예) postgres_backup_20250104_030000.sql.gpg
```

---

## 초기 설정

### 1. GitLab Private Repository 생성

1. GitLab에 로그인 후 새 프로젝트 생성
   - Project name: `postgres-backups` (또는 원하는 이름)
   - Visibility: **Private** 선택
   - Initialize repository with a README 체크

2. Personal Access Token 생성
   - GitLab → Settings → Access Tokens
   - Token name: `postgres-backup`
   - Scopes: `write_repository` 체크
   - Create personal access token 클릭
   - **생성된 토큰 복사** (다시 볼 수 없음!)

### 2. Secret 파일 생성

```bash
# example 파일을 복사하여 실제 Secret 파일 생성
cd /Users/hch/Files/Dev/project/comp-value-gos/ncloud/backend/db
cp backup-secret.example.yml backup-secret.yml
```

### 3. Secret 파일 편집

`backup-secret.yml` 파일을 열어 다음 정보를 수정:

```yaml
stringData:
  POSTGRES_DB: "COMP_VALUE"
  POSTGRES_USER: "compvalue"
  POSTGRES_PASSWORD: "compvalue"

  # GitLab Personal Access Token
  GITLAB_TOKEN: "glpat-xxxxxxxxxxxxxxxxxxxx"  # 복사한 토큰

  # GitLab 저장소 URL (https:// 제외)
  GITLAB_REPO: "gitlab.com/your-username/postgres-backups.git"

  # 암호화 비밀번호 (강력한 비밀번호로 설정, 복구 시 필요!)
  BACKUP_PASSPHRASE: "your-very-strong-passphrase-123!@#"
```

**⚠️ 중요**: `BACKUP_PASSPHRASE`는 안전하게 별도 보관하세요! 복구 시 필수입니다.

### 4. Kubernetes에 적용

```bash
# Secret 적용 (먼저 적용해야 함)
kubectl apply -f backup-secret.yml

# CronJob 적용
kubectl apply -f backup-cronjob.yml

# 적용 확인
kubectl get cronjob
kubectl get secret postgres-backup-secret
```

### 5. 수동 백업 테스트 (선택사항)

CronJob이 정상 작동하는지 즉시 테스트:

```bash
kubectl create job --from=cronjob/postgres-backup manual-backup-test

# 로그 확인
kubectl logs -f job/manual-backup-test

# 테스트 Job 삭제
kubectl delete job manual-backup-test
```

---

## 백업 확인

### 1. CronJob 상태 확인

```bash
# CronJob 목록 및 마지막 실행 시간 확인
kubectl get cronjob postgres-backup

# 실행된 Job 목록
kubectl get jobs | grep postgres-backup
```

### 2. 백업 로그 확인

```bash
# 최근 실행된 Job의 로그 확인
kubectl logs -l job-name=<job-name>

# 예시
kubectl logs -l job-name=postgres-backup-28471234
```

### 3. GitLab에서 백업 확인

1. GitLab 저장소 접속
2. 백업 파일 확인: `postgres_backup_YYYYMMDD_HHMMSS.sql.gpg`
3. Commit 메시지: `Backup: YYYYMMDD_HHMMSS`

---

## 데이터 복구 방법

### 시나리오 1: Minikube/클러스터는 살아있고 DB만 문제가 있는 경우

#### 방법 A: Pod 내부에서 직접 복구

```bash
# 1. GitLab에서 복구할 백업 파일 다운로드
# GitLab 저장소 → 원하는 백업 파일 → Download

# 2. 백업 파일을 호스트에 저장
# 예: ~/postgres_backup_20250104_030000.sql.gpg

# 3. 백업 파일 복호화 (호스트에서)
gpg --decrypt --batch --passphrase "your-backup-passphrase" \
  postgres_backup_20250104_030000.sql.gpg > restore.sql

# 4. PostgreSQL Pod 이름 확인
kubectl get pods -l app=postgres

# 5. SQL 파일을 Pod로 복사
kubectl cp restore.sql <postgres-pod-name>:/tmp/restore.sql

# 6. Pod에 접속하여 복구
kubectl exec -it <postgres-pod-name> -- bash

# 7. Pod 내부에서 복구 실행
psql -U compvalue -d COMP_VALUE < /tmp/restore.sql

# 8. 복구 확인
psql -U compvalue -d COMP_VALUE -c "\dt"

# 9. 임시 파일 삭제
exit
kubectl exec <postgres-pod-name> -- rm /tmp/restore.sql
```

#### 방법 B: 복구 Job 사용 (편리함)

```bash
# 1. 복구할 백업 파일을 GitLab에서 다운로드하여 호스트에 저장

# 2. ConfigMap 생성
kubectl create configmap postgres-restore-backup \
  --from-file=backup.sql.gpg=./postgres_backup_20250104_030000.sql.gpg

# 3. 복구 Job 실행
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: postgres-restore
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: restore
        image: postgres:15
        command:
          - /bin/bash
          - -c
          - |
            apt-get update && apt-get install -y gnupg
            gpg --decrypt --batch --passphrase "\${BACKUP_PASSPHRASE}" \
              /backup/backup.sql.gpg > /tmp/restore.sql
            PGPASSWORD=\${POSTGRES_PASSWORD} psql -h postgres-service \
              -U \${POSTGRES_USER} -d \${POSTGRES_DB} < /tmp/restore.sql
            echo "Restore completed successfully!"
        env:
        - name: POSTGRES_DB
          value: "COMP_VALUE"
        - name: POSTGRES_USER
          value: "compvalue"
        - name: POSTGRES_PASSWORD
          value: "compvalue"
        - name: BACKUP_PASSPHRASE
          valueFrom:
            secretKeyRef:
              name: postgres-backup-secret
              key: BACKUP_PASSPHRASE
        volumeMounts:
        - name: backup-file
          mountPath: /backup
      volumes:
      - name: backup-file
        configMap:
          name: postgres-restore-backup
EOF

# 4. 복구 로그 확인
kubectl logs -f job/postgres-restore

# 5. 정리
kubectl delete job postgres-restore
kubectl delete configmap postgres-restore-backup
```

---

### 시나리오 2: Minikube/클러스터가 완전히 삭제된 경우

#### 1. Minikube 재설치 및 PostgreSQL 재배포

```bash
# Minikube 시작
minikube start

# PostgreSQL 배포
kubectl apply -f postgresSql.yml

# PostgreSQL Pod가 Ready 될 때까지 대기
kubectl wait --for=condition=ready pod -l app=postgres --timeout=300s
```

#### 2. 데이터 복구

위의 "시나리오 1"의 복구 방법 중 하나를 선택하여 실행

#### 3. 백업 시스템 재설정

```bash
# Secret 재적용
kubectl apply -f backup-secret.yml

# CronJob 재적용
kubectl apply -f backup-cronjob.yml
```

---

### 시나리오 3: 특정 시점으로 복구 (Point-in-Time Recovery)

GitLab에서 원하는 시점의 백업을 선택하여 복구:

```bash
# 1. GitLab에서 백업 파일 목록 확인
# Commits 탭에서 날짜별 백업 확인

# 2. 원하는 시점의 백업 파일 다운로드

# 3. 위의 복구 방법 중 하나 사용
```

---

## 트러블슈팅

### ❌ CronJob이 실행되지 않음

```bash
# CronJob 상태 확인
kubectl describe cronjob postgres-backup

# 최근 Job 실행 여부 확인
kubectl get jobs -l app=postgres-backup

# schedule 시간이 올바른지 확인 (UTC 기준)
kubectl get cronjob postgres-backup -o yaml | grep schedule
```

### ❌ 백업 Job이 실패함

```bash
# 실패한 Job의 로그 확인
kubectl logs -l job-name=<failed-job-name>

# 일반적인 원인:
# 1. GitLab Token이 잘못됨
# 2. GitLab Repo URL이 잘못됨
# 3. PostgreSQL 접속 정보가 잘못됨

# Secret 확인
kubectl get secret postgres-backup-secret -o yaml
```

### ❌ 복구 시 "Invalid passphrase" 오류

- `BACKUP_PASSPHRASE`가 백업 시 사용한 것과 일치하는지 확인
- Secret에서 비밀번호 확인:
  ```bash
  kubectl get secret postgres-backup-secret -o jsonpath='{.data.BACKUP_PASSPHRASE}' | base64 -d
  ```

### ❌ GitLab Push 실패

```bash
# GitLab Token 권한 확인
# - write_repository 권한이 있는지 확인
# - Token이 만료되지 않았는지 확인

# 저장소 접근 테스트 (로컬에서)
git clone https://oauth2:YOUR_TOKEN@gitlab.com/username/postgres-backups.git
```

### ❌ 복구 후 데이터가 이상함

```bash
# 1. 올바른 백업 파일을 사용했는지 확인
# 2. 복구 전 기존 데이터 백업
# 3. 데이터베이스를 완전히 삭제 후 재생성

# PostgreSQL 초기화 (주의!)
kubectl exec -it <postgres-pod> -- psql -U compvalue -d postgres -c "DROP DATABASE COMP_VALUE;"
kubectl exec -it <postgres-pod> -- psql -U compvalue -d postgres -c "CREATE DATABASE COMP_VALUE;"

# 다시 복구 시도
```

---

## 백업 주기 변경

`backup-cronjob.yml` 파일의 `schedule` 수정:

```yaml
spec:
  # 매일 오전 3시 (한국 시간 기준, UTC 18:00)
  schedule: "0 18 * * *"

  # 다른 예시:
  # 매 6시간마다: "0 */6 * * *"
  # 매주 일요일 오전 2시: "0 17 * * 0"
  # 매달 1일 오전 4시: "0 19 1 * *"
```

수정 후:
```bash
kubectl apply -f backup-cronjob.yml
```

---

## 보안 주의사항

1. **backup-secret.yml 파일은 절대 Git에 커밋하지 마세요!**
   - `.gitignore`에 추가 권장: `backup-secret.yml`

2. **BACKUP_PASSPHRASE를 안전하게 보관하세요**
   - 비밀번호 관리자 사용 권장
   - 복구 시 필수!

3. **GitLab Token 주기적 갱신**
   - Token 만료 전 새 토큰 발급
   - Secret 업데이트

4. **Private Repository 유지**
   - GitLab 저장소가 Private인지 정기적으로 확인

---

## 추가 개선 사항 (선택)

### 백업 성공/실패 알림 추가

Slack, Discord, 이메일 등으로 백업 결과 알림을 받고 싶다면
`backup.sh` 스크립트에 webhook 추가 가능

### 백업 파일 크기 모니터링

```bash
# GitLab 저장소 크기 확인
git clone <your-repo>
du -sh postgres-backups/
```

100MB 이상 커지면 Git LFS 사용 고려
