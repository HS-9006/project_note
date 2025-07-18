# Firebase + PlanetScale API 서버 구축 가이드

Firebase Functions(Node.js 기반)로 API 서버를 구축하고,  
PlanetScale(MySQL 클라우드 DB)을 연동하여 회원가입, 로그인, 데이터 저장 등을 처리하는 구조입니다.

DBeaver를 활용하여 DB 테이블을 관리하고, Unity 또는 Postman을 통해 API를 테스트합니다.

---

## 사전 준비

| 항목 | 준비 방법 |
|------|-----------|
| Firebase 계정 | https://firebase.google.com |
| Firebase CLI | `npm install -g firebase-tools` |
| GitHub 계정 | https://github.com |
| PlanetScale 계정 | https://planetscale.com |
| DBeaver 설치 | https://dbeaver.io/download/ |
| Node.js 설치 | https://nodejs.org (LTS 버전 추천) |

| 툴 | 설명 |
|-----|------|
| Postman | REST API 테스트 도구 (권장) |

---

## 1. Firebase Functions 프로젝트 설정

Firebase CLI로 로그인하고 Functions 디렉토리를 초기화합니다.

```bash
firebase login
firebase init functions
```

- Functions 디렉토리: functions  
- 언어: JavaScript  
- ESLint: 선택  
- npm install: Yes

---

## 2. 의존성 설치

```bash
cd functions
npm install express cors dotenv mysql2
```

---

## 3. PlanetScale DB 생성

Trial 상태인 경우 Hobby(무료) 플랜 전환 요청 필요 (요청 완료)

- "Create a new database" 클릭
- Database name: atm-game-db
- Region: asia-northeast3 (Seoul, South Korea)
- 생성 후 Connect 메뉴에서 Host/User/Password 복사

---

## 4. DBeaver로 PlanetScale 연결

DBeaver 실행 → New Connection → MySQL 선택

- Host: aws.connect.psdb.io  
- Port: 3306  
- Username / Password: PlanetScale에서 복사한 값  
- DB name: atm-game-db  
- Use SSL: 체크  
- SSL Mode: require 또는 verify-ca  
- Test Connection → 성공 시 Finish

---

## 5. 테이블 생성 예시 (DBeaver 내 SQL 편집기)

```sql
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(50),
  password VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 6. Firebase Functions API 구현 (functions/index.js)

```js
const functions = require('firebase-functions');
const express = require('express');
const cors = require('cors');
const mysql = require('mysql2');
require('dotenv').config();

const app = express();
app.use(cors());
app.use(express.json());

const db = mysql.createConnection({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  ssl: { rejectUnauthorized: true }
});

app.post('/login', (req, res) => {
  const { username, password } = req.body;
  const sql = 'SELECT * FROM users WHERE username = ? AND password = ?';
  db.query(sql, [username, password], (err, results) => {
    if (err) return res.status(500).json({ error: err });
    if (results.length > 0) {
      res.json({ message: "Login success", user: results[0] });
    } else {
      res.status(401).json({ message: "Invalid credentials" });
    }
  });
});

exports.api = functions.https.onRequest(app);
```

---

## 7. 환경변수 설정 (functions/.env)

```env
DB_HOST=your-host.connect.psdb.io
DB_USER=your-username
DB_PASSWORD=your-password
DB_NAME=atm-game-db
```

> `.gitignore`에 `.env` 포함 필수

---

## 8. Functions 배포

```bash
firebase deploy --only functions
```

- 배포 후 API URL 예시:  
  `https://your-project-id.cloudfunctions.net/api/login`

---

## 9. API 테스트 예시

```json
// 요청 (POST /login)
{
  "username": "testuser",
  "password": "1234"
}
```

```json
// 응답
{
  "message": "Login success",
  "user": {
    "id": 1,
    "username": "testuser"
  }
}
```

---

## 전체 흐름도

```text
[ Unity or Postman ]
         ↓
 [ Firebase API URL ]
         ↓
[ Firebase Functions (index.js) ]
         ↓
[ PlanetScale DB (MySQL) ]
         ↓
[ DBeaver로 확인/관리 ]
```

---

## 현재 상태

- Firebase Functions 배포 및 API 기본 구조 완료  
- PlanetScale DB는 무료 플랜 전환 요청 후 생성 예정  
- 이후 DB 생성, DBeaver 연결, API 완성 및 Unity 연동까지 단계별 진행 예정
