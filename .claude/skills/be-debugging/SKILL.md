---
name: be-debugging
description: 백엔드 디버깅 및 문제 해결
---

# 백엔드 디버깅 가이드

## 디버깅 도구

### 1. Flask 디버그 모드
```python
# config.py
class DevelopmentConfig(Config):
    DEBUG = True
    FLASK_ENV = 'development'

# app.py
if __name__ == '__main__':
    app.run(debug=True, port=5000)
```

### 2. 로깅 설정
```python
import logging

# 기본 로깅 설정
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s'
)

# 파일로 로깅
handler = logging.FileHandler('app.log')
handler.setLevel(logging.DEBUG)
app.logger.addHandler(handler)

# 사용
app.logger.debug(f"Request data: {request.json}")
app.logger.error(f"Error occurred: {str(e)}")
```

### 3. pdb 디버거
```python
# 중단점 설정
import pdb; pdb.set_trace()

# 유용한 명령어
# n (next)      : 다음 줄
# s (step)      : 함수 안으로
# c (continue)  : 계속 실행
# p variable    : 변수 출력
# l (list)      : 코드 보기
# q (quit)      : 종료
```

### 4. Flask-DebugToolbar
```python
from flask_debugtoolbar import DebugToolbarExtension

app.config['SECRET_KEY'] = 'dev'
app.config['DEBUG_TB_INTERCEPT_REDIRECTS'] = False
toolbar = DebugToolbarExtension(app)
```

## 일반적인 문제 해결

### API 응답이 없을 때
```python
# 1. 라우트 확인
@app.route('/api/logs', methods=['GET'])  # methods 확인

# 2. 반환값 확인
return jsonify({'logs': logs}), 200  # jsonify 사용

# 3. CORS 확인
from flask_cors import CORS
CORS(app)
```

### 데이터베이스 오류
```python
# 1. 연결 확인
try:
    db.session.execute('SELECT 1')
    print("DB connected")
except Exception as e:
    print(f"DB error: {e}")

# 2. 쿼리 디버깅
from sqlalchemy import event
from sqlalchemy.engine import Engine

@event.listens_for(Engine, "before_cursor_execute")
def receive_before_cursor_execute(conn, cursor, statement, params, context, executemany):
    print(f"SQL: {statement}")
    print(f"Params: {params}")

# 3. 트랜잭션 롤백
try:
    db.session.commit()
except Exception as e:
    db.session.rollback()
    app.logger.error(f"DB Error: {e}")
```

### 성능 문제
```python
# 1. 쿼리 최적화 (N+1 문제)
# ❌ N+1 쿼리
logs = Log.query.all()
for log in logs:
    print(log.alert.name)  # 각 로그마다 쿼리

# ✅ Eager loading
logs = Log.query.options(joinedload(Log.alert)).all()

# 2. 실행 시간 측정
import time

start = time.time()
result = expensive_operation()
print(f"Took {time.time() - start:.2f}s")

# 3. 프로파일링
from werkzeug.middleware.profiler import ProfilerMiddleware
app.wsgi_app = ProfilerMiddleware(app.wsgi_app)
```

### 메모리 누수
```python
# 1. 커넥션 정리
@app.teardown_appcontext
def shutdown_session(exception=None):
    db.session.remove()

# 2. 대용량 데이터 처리
# ❌ 메모리 부족
logs = Log.query.all()  # 모든 데이터 로드

# ✅ 배치 처리
for log in Log.query.yield_per(1000):
    process(log)
```

## 디버깅 체크리스트
- [ ] 로그 확인 (app.log, error.log)
- [ ] HTTP 상태 코드 확인
- [ ] 요청/응답 데이터 확인
- [ ] 데이터베이스 쿼리 확인
- [ ] 예외 스택 트레이스 분석
- [ ] 환경 변수 확인 (.env)
- [ ] 의존성 버전 확인 (requirements.txt)

## 유용한 명령어
```bash
# 로그 실시간 확인
tail -f app.log

# 에러 로그만 필터링
grep ERROR app.log

# 프로세스 확인
ps aux | grep python

# 포트 사용 확인
lsof -i :5000

# 디버그 모드로 실행
FLASK_ENV=development flask run --debug
```

## 디버깅 팁
- 문제를 재현 가능하게 만들기
- 로그를 적극 활용 (logger.debug, logger.error)
- 한 번에 하나씩 변경하며 테스트
- 테스트 코드로 버그 재현 후 수정
- Git bisect로 문제 도입 시점 찾기
