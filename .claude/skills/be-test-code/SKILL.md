---
name: be-test-code
description: 백엔드 테스트 코드 작성 (pytest 기반)
---

# 백엔드 테스트 코드 작성 가이드

## 사용 시점
- 새로운 API 엔드포인트 구현 후
- 비즈니스 로직 추가 후
- 버그 수정 후 회귀 테스트 작성
- 리팩토링 전 안전망 확보

## 테스트 작성 순서

### 1. API 엔드포인트 테스트
```python
def test_get_logs_api(client, auth_headers):
    """GET /api/logs 테스트"""
    response = client.get('/api/logs', headers=auth_headers)

    assert response.status_code == 200
    assert 'logs' in response.json

def test_create_log_api(client, auth_headers):
    """POST /api/logs 테스트"""
    data = {
        'level': 'ERROR',
        'message': 'Test error',
        'source': 'test-service'
    }
    response = client.post('/api/logs', json=data, headers=auth_headers)

    assert response.status_code == 201
    assert response.json['id'] is not None
```

### 2. 비즈니스 로직 테스트
```python
def test_log_filtering_logic():
    """로그 필터링 로직 테스트"""
    from app.services.log_service import LogService

    service = LogService()
    logs = service.filter_logs(level='ERROR', limit=10)

    assert len(logs) <= 10
    assert all(log.level == 'ERROR' for log in logs)
```

### 3. 데이터베이스 테스트
```python
def test_log_model_creation(app):
    """Log 모델 생성 테스트"""
    log = Log(level='ERROR', message='Test', source='test')
    db.session.add(log)
    db.session.commit()

    assert log.id is not None
    assert Log.query.count() == 1
```

### 4. 예외 처리 테스트
```python
def test_invalid_log_level(client, auth_headers):
    """잘못된 로그 레벨"""
    data = {'level': 'INVALID', 'message': 'Test'}
    response = client.post('/api/logs', json=data, headers=auth_headers)

    assert response.status_code == 400
    assert 'error' in response.json

def test_unauthorized_access(client):
    """인증 없이 접근"""
    response = client.get('/api/logs')
    assert response.status_code == 401
```

## 테스트 코드 체크리스트
- [ ] 성공 케이스 테스트
- [ ] 실패 케이스 테스트 (400, 404, 500)
- [ ] 인증/인가 테스트
- [ ] 입력 유효성 검증 테스트
- [ ] 엣지 케이스 테스트 (빈 값, null, 경계값)
- [ ] 외부 의존성 모킹
- [ ] 트랜잭션 격리 확인

## 빠른 명령어
```bash
pytest tests/test_logs.py -v        # 특정 파일 테스트
pytest -k "test_create" -v          # 특정 테스트만
pytest --cov=app --cov-report=html  # 커버리지
pytest -x                            # 첫 실패 시 중단
```

## 테스트 작성 팁
- AAA 패턴 사용 (Arrange-Act-Assert)
- 테스트 이름은 명확하게 (`test_동작_예상결과`)
- Fixture를 적극 활용하여 중복 제거
- Mock은 최소한으로, 실제 DB 사용 권장
