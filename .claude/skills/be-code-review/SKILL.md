---
name: be-code-review
description: LogWatch Admin 백엔드 코드 리뷰 및 테스트 코드 검증
---

# LogWatch Admin - 백엔드 테스트 가이드

## 프로젝트 정보
- **Python 3.11+** + Flask + SQLAlchemy
- **테스트 도구**: pytest + pytest-flask + pytest-cov

## 빠른 설정

```bash
# 설치
pip install pytest pytest-flask pytest-cov pytest-mock
pip install flask-testing factory-boy faker

# 실행
pytest                           # 전체 테스트
pytest --cov=app --cov-report=html  # 커버리지
pytest -v                        # 상세 출력
pytest tests/test_logs.py        # 특정 파일만
pytest -k "test_create"          # 특정 테스트만
```

### pytest.ini
```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    -v
    --strict-markers
    --disable-warnings
markers =
    unit: 단위 테스트
    integration: 통합 테스트
    slow: 느린 테스트
```

### conftest.py
```python
import pytest
from app import create_app, db
from app.models import User, Log, Alert

@pytest.fixture
def app():
    """테스트용 Flask 앱"""
    app = create_app('testing')
    with app.app_context():
        db.create_all()
        yield app
        db.session.remove()
        db.drop_all()

@pytest.fixture
def client(app):
    """테스트 클라이언트"""
    return app.test_client()

@pytest.fixture
def auth_headers(client):
    """인증 헤더"""
    response = client.post('/api/auth/login', json={
        'email': 'test@example.com',
        'password': 'password'
    })
    token = response.json['token']
    return {'Authorization': f'Bearer {token}'}
```

## 테스트 전략

### 1. API 엔드포인트 테스트

```python
def test_get_logs(client, auth_headers):
    """로그 목록 조회"""
    response = client.get('/api/logs', headers=auth_headers)

    assert response.status_code == 200
    assert 'logs' in response.json
    assert isinstance(response.json['logs'], list)

def test_get_logs_with_filter(client, auth_headers):
    """로그 필터링"""
    response = client.get(
        '/api/logs?level=ERROR&start_date=2026-01-01',
        headers=auth_headers
    )

    assert response.status_code == 200
    logs = response.json['logs']
    assert all(log['level'] == 'ERROR' for log in logs)

def test_create_alert_rule(client, auth_headers):
    """알림 규칙 생성"""
    data = {
        'name': '에러 알림',
        'condition': {'level': 'ERROR', 'count': 10},
        'channels': ['email', 'slack']
    }
    response = client.post('/api/alerts', json=data, headers=auth_headers)

    assert response.status_code == 201
    assert response.json['id'] is not None
    assert response.json['name'] == '에러 알림'

def test_unauthorized_access(client):
    """인증 없이 접근"""
    response = client.get('/api/logs')
    assert response.status_code == 401
```

### 2. 비즈니스 로직 테스트

```python
from app.services.log_service import LogService

def test_filter_logs_by_level():
    """로그 레벨별 필터링 로직"""
    service = LogService()
    logs = service.filter_logs(level='ERROR', limit=10)

    assert len(logs) <= 10
    assert all(log.level == 'ERROR' for log in logs)

def test_calculate_statistics():
    """통계 계산 로직"""
    service = LogService()
    stats = service.calculate_stats(start_date='2026-02-01')

    assert 'total_logs' in stats
    assert 'error_count' in stats
    assert stats['total_logs'] >= stats['error_count']

def test_alert_rule_evaluation():
    """알림 규칙 평가"""
    from app.services.alert_service import AlertService

    service = AlertService()
    rule = {'level': 'ERROR', 'count': 10, 'window': '5m'}

    should_alert = service.evaluate_rule(rule)
    assert isinstance(should_alert, bool)
```

### 3. 데이터베이스 테스트

```python
from app.models import Log, Alert

def test_create_log(app):
    """로그 생성"""
    log = Log(
        level='ERROR',
        message='Database connection failed',
        source='api-server',
        timestamp='2026-02-09T00:00:00Z'
    )
    db.session.add(log)
    db.session.commit()

    assert log.id is not None
    assert Log.query.filter_by(level='ERROR').count() == 1

def test_log_relationships(app):
    """모델 관계"""
    alert = Alert(name='테스트 알림')
    log = Log(level='ERROR', message='Test', alert=alert)

    db.session.add_all([alert, log])
    db.session.commit()

    assert log.alert_id == alert.id
    assert alert.logs[0].id == log.id

def test_query_performance(app):
    """쿼리 성능"""
    # 대량 데이터 생성
    logs = [Log(level='INFO', message=f'Log {i}') for i in range(1000)]
    db.session.bulk_save_objects(logs)
    db.session.commit()

    import time
    start = time.time()
    result = Log.query.filter_by(level='INFO').limit(100).all()
    duration = time.time() - start

    assert len(result) == 100
    assert duration < 0.1  # 100ms 이내
```

### 4. Mock을 사용한 외부 의존성 테스트

```python
from unittest.mock import patch, Mock

def test_send_slack_notification(client):
    """슬랙 알림 전송 (모킹)"""
    with patch('app.services.notification.slack_client') as mock_slack:
        mock_slack.send_message.return_value = {'ok': True}

        from app.services.notification import send_slack_alert
        result = send_slack_alert('Error occurred', channel='#alerts')

        assert result['ok'] is True
        mock_slack.send_message.assert_called_once()

def test_email_service_failure(client):
    """이메일 전송 실패 처리"""
    with patch('app.services.notification.smtp_client') as mock_smtp:
        mock_smtp.send.side_effect = Exception('SMTP connection failed')

        from app.services.notification import send_email_alert
        result = send_email_alert('test@example.com', 'Alert')

        assert result['success'] is False
        assert 'error' in result
```

### 5. Fixture를 사용한 테스트 데이터 생성

```python
import pytest
from factory import Factory, Faker, SubFactory

class LogFactory(Factory):
    class Meta:
        model = Log

    level = Faker('random_element', elements=['ERROR', 'WARN', 'INFO', 'DEBUG'])
    message = Faker('sentence')
    source = Faker('slug')
    timestamp = Faker('date_time_this_month')

@pytest.fixture
def sample_logs():
    """샘플 로그 데이터"""
    return [LogFactory.build() for _ in range(10)]

def test_with_factory(sample_logs):
    """Factory를 사용한 테스트"""
    assert len(sample_logs) == 10
    assert all(hasattr(log, 'level') for log in sample_logs)
```

## 기능별 테스트 체크리스트

### 로그 관리 API
- [ ] GET /api/logs - 로그 목록 조회
- [ ] GET /api/logs/:id - 로그 상세 조회
- [ ] POST /api/logs - 로그 생성
- [ ] 로그 레벨별 필터링 (ERROR, WARN, INFO, DEBUG)
- [ ] 날짜/시간 범위 필터링
- [ ] 페이지네이션 (page, limit)
- [ ] 로그 소스별 필터링

### 알림 관리 API
- [ ] GET /api/alerts - 알림 규칙 목록
- [ ] POST /api/alerts - 알림 규칙 생성
- [ ] PUT /api/alerts/:id - 알림 규칙 수정
- [ ] DELETE /api/alerts/:id - 알림 규칙 삭제
- [ ] 알림 규칙 평가 로직
- [ ] 알림 전송 (이메일, 슬랙)

### 통계 API
- [ ] GET /api/stats - 대시보드 통계
- [ ] 시간대별 로그 집계
- [ ] 로그 레벨별 카운트
- [ ] 소스별 로그 분포

### 인증/인가
- [ ] POST /api/auth/login - 로그인
- [ ] POST /api/auth/register - 회원가입
- [ ] JWT 토큰 생성/검증
- [ ] 권한 체크 (관리자, 일반 사용자)

### 데이터베이스
- [ ] 모델 생성/수정/삭제
- [ ] 관계 (1:N, N:M) 동작
- [ ] 인덱스 성능
- [ ] 트랜잭션 롤백

## 코드 리뷰 체크리스트

### 필수
- [ ] 모든 API 엔드포인트에 테스트 작성
- [ ] 성공 케이스 + 실패 케이스 테스트
- [ ] 인증/인가 테스트 포함
- [ ] 데이터베이스 트랜잭션 격리
- [ ] 외부 의존성 모킹
- [ ] 모든 테스트 통과
- [ ] 커버리지 목표 달성

### 권장
- [ ] 엣지 케이스 테스트 (빈 데이터, 잘못된 입력)
- [ ] 에러 핸들링 검증
- [ ] 성능 테스트 (응답 시간, 쿼리 최적화)
- [ ] AAA 패턴 (Arrange-Act-Assert)
- [ ] 테스트 설명 명확 (docstring)
- [ ] Fixture 재사용

### LogWatch 특화
- [ ] 대량 로그 처리 테스트
- [ ] 실시간 로그 스트림 처리
- [ ] 알림 규칙 복잡한 조건 테스트
- [ ] 통계 계산 정확성 검증
- [ ] 로그 보존 정책 테스트

## 커버리지 목표

| 영역 | 목표 |
|------|------|
| API 엔드포인트 | 90% |
| 비즈니스 로직 | 85% |
| 데이터베이스 모델 | 80% |
| 유틸리티 함수 | 95% |

## 피해야 할 것

```python
# ❌ 테스트 간 데이터 공유
logs = []  # 전역 변수
def test_create():
    logs.append(Log())

# ✅ Fixture 사용
@pytest.fixture
def logs():
    return []

# ❌ 실제 외부 서비스 호출
def test_slack():
    slack_client.send('message')  # 실제 슬랙 호출

# ✅ Mock 사용
def test_slack(mocker):
    mock = mocker.patch('app.services.slack_client')
    mock.send.return_value = {'ok': True}

# ❌ 하드코딩된 타임스탬프
def test_log():
    log = Log(timestamp='2026-02-09')

# ✅ 동적 시간 생성
from datetime import datetime
def test_log():
    log = Log(timestamp=datetime.utcnow())

# ❌ 트랜잭션 롤백 없음
def test_create():
    db.session.add(Log())
    db.session.commit()  # 다음 테스트에 영향

# ✅ Fixture로 자동 롤백
@pytest.fixture
def app():
    yield app
    db.session.rollback()
```

## 디렉토리 구조

```
app/
├── __init__.py
├── models/
│   ├── __init__.py
│   ├── log.py
│   └── alert.py
├── api/
│   ├── __init__.py
│   ├── logs.py
│   └── alerts.py
├── services/
│   ├── __init__.py
│   ├── log_service.py
│   └── alert_service.py
└── utils/
    ├── __init__.py
    └── validators.py

tests/
├── conftest.py
├── factories.py
├── test_api/
│   ├── test_logs.py
│   └── test_alerts.py
├── test_services/
│   ├── test_log_service.py
│   └── test_alert_service.py
└── test_models/
    ├── test_log.py
    └── test_alert.py
```

## 유용한 pytest 플러그인

```bash
pip install pytest-flask          # Flask 테스트 헬퍼
pip install pytest-cov             # 커버리지
pip install pytest-mock            # 모킹
pip install pytest-xdist           # 병렬 실행
pip install pytest-timeout         # 타임아웃
pip install factory-boy            # 테스트 데이터 생성
pip install faker                  # 가짜 데이터 생성
```

## 참고 문서
- [pytest](https://docs.pytest.org/)
- [pytest-flask](https://pytest-flask.readthedocs.io/)
- [Flask Testing](https://flask.palletsprojects.com/en/latest/testing/)
- [factory-boy](https://factoryboy.readthedocs.io/)
