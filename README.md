# 📦 MySQL Dump & Cron Backup

MySQL 자동 백업 시스템을 설계·구현하여, 운영 환경에서 데이터의 안정성과 신뢰성을 보장하는 체계를 직접 구축한 프로젝트입니다.

---

## 👥 팀원
<table>
  <tr>
    <!-- 이름 (링크) -->
    <td align="center">
      <a href="https://github.com/songhajang"><strong>장송하</strong></a>
    </td>
    <td align="center">
      <a href="https://github.com/Minkyoungg0"><strong>문민경</strong></a>
    </td>
  </tr>
  <tr>
    <!-- 프로필 사진 -->
    <td align="center">
      <img src="https://github.com/songhajang.png" width="100"/>
    </td>
    <td align="center">
      <img src="https://github.com/Minkyoungg0.png" width="100"/>
    </td>
  </tr>
</table>

<br/>

---

## 🎯 목표 

- 정기적인 **백업을 자동화**하여 데이터 손실에 대비한다.
- MySQL 데이터베이스를 **덤프하고 압축**한 뒤, **cron으로 주기적으로 실행**하여 운영 환경에서도 안정성과 신뢰성을 보장한다.


---

## 🔧 구현 내용  
```
📦 MySQL Backup Automation
├── 🗄️ DB 준비
│   └── sqld # 테이블 및 샘플 데이터 입력
│       ├── EMP
│       └── DEPT
│
├── 📜 백업 스크립트
│   ├── .sql 추출
│   ├── tar.gz 압축
│   └── 무결성 검사 + 삭제
│
├── ⏰ 자동화 (cron)
│    └── 매일 09:00
│   
│
├── 🗑️ 보관 정책
│   └── 14일이 지난 백업 자동 삭제
│
└── 📝 로그 관리
    ├── backup.log # 백업 기록
    └── cron.log # 자동 실행 로그
```

---

## 🖥️ 프로젝트 환경  
- OS : Ubuntu 22.04 LTS  
- DBMS : MySQL Server 8.0.43
- 언어/도구 : Bash Script, cron, mysqldump, tar  
- 백업 저장소 : `/home/ubuntu/db-backups`

---

## 🔄 데이터베이스 백업 자동화 과정

### 1) 백업 계정 인증
- 전용 계정(backup)을 만들어 최소 권한만 부여
```bash
CREATE USER 'backup'@'localhost' IDENTIFIED BY '비밀번호';
GRANT SELECT, SHOW VIEW, TRIGGER, EVENT, LOCK TABLES ON sqld.* TO 'backup'@'localhost';
FLUSH PRIVILEGES;
```

- 비밀번호와 접근 설정을 별도 관리 파일에 저장해 보안성을 강화
```bash
cat > /home/ubuntu/.my.cnf <<'EOF'
[client]
user=backup
password='비밀번호'
host=localhost
EOF
chmod 600 /home/ubuntu/.my.cnf
```

---

### 2) 백업 스크립트(`shell script`)

<br>

**A. 데이터베이스 전체를 SQL 파일로 추출**
- mysqldump를 활용해 데이터베이스 전체를 SQL 파일로 추출
```bash
"${MYSQLDUMP}" \
  --no-tablespaces --single-transaction --routines --triggers --events \
  --databases "${DB_NAME}" > "${OUT_SQL}"
```

**B. 덤프 파일을 tar.gz 형식으로 압축하고 무결성을 확인**
- 생성된 .sql 파일을 압축하고, tar -tzf로 정상 압축 여부를 검증
```bash
"${TAR}" -czf "${OUT_TAR}" -C "${BACKUP_ROOT}" "$(basename "${OUT_SQL}")"
"${TAR}" -tzf "${OUT_TAR}" >/dev/null
```

**C. 오래된 백업은 자동으로 삭제되도록 보관 정책 적용**
- find 명령으로 14일 이상 된 .tar.gz 파일은 자동 삭제
```bash
"${FIND}" "${BACKUP_ROOT}" -name "${DB_NAME}_*.tar.gz" -type f -mtime +14 -delete
```
**D. 실행 결과 로그 기록**
- 백업이 실행될 때마다 타임스탬프와 백업 파일명을 로그에 남깁니다.
```bash
echo "$(${DATE} -Is) OK: ${OUT_TAR}" >> "${LOG}"
```

---

### 3. 수동 테스트
- 스크립트를 직접 실행하여 백업 파일이 정상 생성되는지 확인
```bash
/home/ubuntu/db-backups/backup.sh
ls -lh /home/ubuntu/db-backups
tail -n 5 /home/ubuntu/db-backups/backup.log
```
---

### 4. 크론 등록(자동화)
- 매일 오전 9시에 자동 백업이 수행되도록 설정
```cron
*/5 * * * * /home/ubuntu/db-backups/backup.sh >> /home/ubuntu/db-backups/cron.log 2>&1
0 9 * * * /home/ubuntu/db-backups/backup.sh >> /home/ubuntu/db-backups/cron.log 2>&1
```
---

### 5. 복원 절차
- 압축된 백업 파일을 해제하여 .sql 파일로 복원

- 데이터베이스에 주입해 테이블과 데이터가 정상적으로 복원되는지 확인
```bash
tar -xzf /home/ubuntu/db-backups/sqld_YYYYmmdd_HHMMSS.tar.gz -C /tmp
mysql < /tmp/sqld_YYYYmmdd_HHMMSS.sql
```

---
## 🔒 운영 환경 고려사항
> 이번 프로젝트에서는 실습 위주로 자동화 백업을 구축했지만,
> 실제 운영 환경에서는 주기, 벤더사 특성, 보안까지 종합적으로 고려해야 한다.  
> 이를 위해 참고한 운영 가이드와 권장 사항은 다음과 같다.

### 1️⃣ 권장 덤프 주기 & 파일명 규칙  
- **운영**: 매일 1회 전체 덤프 + 7~30일 보관  
- **파일명 규칙**: `<DB>_<YYYYmmdd_HHMMSS>.tar.gz`  
  - 예) `sqld_20250908_090000.tar.gz`


### 2️⃣ 보안 및 운영 팁
- `~/.my.cnf` 권한 `600` 유지 (비밀번호 노출 방지)  
- 백업 디렉토리 접근 권한 최소화  
- 원격/다중 보관소에 2차 백업 권장  
- 주기적 **복원 리허설**로 실제 복구 가능성 검증

### 🔍 참고 문서
- [MySQL 공식 매뉴얼 – Backup and Recovery](https://dev.mysql.com/doc/refman/8.0/en/backup-and-recovery.html)  
- [Kakao Cloud DB 백업 가이드](https://cloud.kakao.com/docs)  
- [AWS RDS – Backup and Restore](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_CommonTasks.BackupRestore.html)  
- [Google Cloud SQL 백업 문서](https://cloud.google.com/sql/docs/mysql/backup-recovery)  
- [Naver Cloud DB 서비스 가이드](https://guide.ncloud-docs.com/docs/database-mysql-overview)  


---

## 🛠️ 트러블 슈팅


<details>
<summary> mysqldump 실행 시 PROCESS 권한 오류</summary>

- 증상 : `mysqldump` 실행 시 다음과 같은 에러 발생  
  ```bash
  ubuntu@myserver00:~$ ./db-backups/backup.sh
  mysqldump: Error: 'Access denied; you need (at least one of) the PROCESS privilege(s) for this operation' when trying to dump tablespaces
<br>
- 원인 : mysqldump 기본 동작에 tablespaces 관련 메타데이터도 포함되는데, 이를 조회하려면 PROCESS 권한이 필요함.
<br>
- 해결 : 운영 환경에서는 PROCESS 권한을 주는 게 부담될 수 있으므로, `--no-tablespaces` 옵션을 추가해 덤프 시 tablespaces 정보를 제외
</details>

---

## 📝 회고

실무에서 안정적 서비스 운영을 위해 반드시 고려해야 할 **백업 체계의 중요성**을 체감했고,<br>
이 경험을 바탕으로 실제 환경에서도 **데이터 안정성과 신뢰성**을 지키는 개발자로 성장하고자 한다.
