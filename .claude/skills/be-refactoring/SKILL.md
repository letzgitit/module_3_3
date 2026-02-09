---
name: be-refactoring
description: 백엔드 코드 리팩토링 및 최적화
---

# 백엔드 리팩토링 가이드

## 리팩토링 원칙
1. **작동하는 코드를 먼저** - 리팩토링 전 테스트 작성
2. **작은 단계로** - 한 번에 하나씩 변경
3. **자주 커밋** - 각 단계마다 커밋
4. **테스트 실행** - 변경 후 항상 테스트 확인

## 일반적인 리팩토링 패턴

### 1. 긴 함수 분리

**Before (안 좋은 예)**
```python
@app.route('/api/logs', methods=['POST'])
def create_log():
    data = request.get_json()

    # 유효성 검증
    if not data.get('level'):
        return jsonify({'error': 'level required'}), 400
    if data['level'] not in ['ERROR', 'WARN', 'INFO', 'DEBUG']:
        return jsonify({'error': 'invalid level'}), 400
    if not data.get('message'):
        return jsonify({'error': 'message required'}), 400

    # 로그 생성
    log = Log(level=data['level'], message=data['message'])
    db.session.add(log)
    db.session.commit()

    # 알림 체크
    if log.level == 'ERROR':
        alert_rules = AlertRule.query.filter_by(level='ERROR').all()
        for rule in alert_rules:
            if should_trigger_alert(rule, log):
                send_notification(rule, log)

    return jsonify(log.to_dict()), 201
```

**After (개선된 예)**
```python
@app.route('/api/logs', methods=['POST'])
def create_log():
    data = request.get_json()

    # 유효성 검증
    validation_error = validate_log_data(data)
    if validation_error:
        return jsonify({'error': validation_error}), 400

    # 로그 생성
    log = create_log_record(data)

    # 알림 체크
    check_and_send_alerts(log)

    return jsonify(log.to_dict()), 201

def validate_log_data(data):
    """로그 데이터 유효성 검증"""
    if not data.get('level'):
        return 'level required'
    if data['level'] not in ['ERROR', 'WARN', 'INFO', 'DEBUG']:
        return 'invalid level'
    if not data.get('message'):
        return 'message required'
    return None

def create_log_record(data):
    """로그 레코드 생성"""
    log = Log(level=data['level'], message=data['message'])
    db.session.add(log)
    db.session.commit()
    return log

def check_and_send_alerts(log):
    """알림 규칙 체크 및 전송"""
    if log.level != 'ERROR':
        return

    alert_rules = AlertRule.query.filter_by(level='ERROR').all()
    for rule in alert_rules:
        if should_trigger_alert(rule, log):
            send_notification(rule, log)
```

### 2. 비즈니스 로직을 서비스 레이어로 분리

**Before**
```python
@app.route('/api/stats', methods=['GET'])
def get_stats():
    total = Log.query.count()
    errors = Log.query.filter_by(level='ERROR').count()
    warns = Log.query.filter_by(level='WARN').count()

    return jsonify({
        'total': total,
        'error_count': errors,
        'warn_count': warns,
        'error_rate': errors / total if total > 0 else 0
    }), 200
```

**After**
```python
# app/services/log_service.py
class LogService:
    @staticmethod
    def get_statistics():
        """로그 통계 계산"""
        total = Log.query.count()
        errors = Log.query.filter_by(level='ERROR').count()
        warns = Log.query.filter_by(level='WARN').count()

        return {
            'total': total,
            'error_count': errors,
            'warn_count': warns,
            'error_rate': errors / total if total > 0 else 0
        }

# app/api/logs.py
@app.route('/api/stats', methods=['GET'])
def get_stats():
    stats = LogService.get_statistics()
    return jsonify(stats), 200
```

### 3. 중복 코드 제거

**Before**
```python
@app.route('/api/logs/<int:log_id>', methods=['GET'])
def get_log(log_id):
    log = Log.query.get(log_id)
    if not log:
        return jsonify({'error': 'Log not found'}), 404
    return jsonify(log.to_dict()), 200

@app.route('/api/alerts/<int:alert_id>', methods=['GET'])
def get_alert(alert_id):
    alert = Alert.query.get(alert_id)
    if not alert:
        return jsonify({'error': 'Alert not found'}), 404
    return jsonify(alert.to_dict()), 200
```

**After**
```python
def get_or_404(model, id, name):
    """모델 조회 또는 404"""
    obj = model.query.get(id)
    if not obj:
        return jsonify({'error': f'{name} not found'}), 404
    return obj

@app.route('/api/logs/<int:log_id>', methods=['GET'])
def get_log(log_id):
    log = get_or_404(Log, log_id, 'Log')
    return jsonify(log.to_dict()), 200

@app.route('/api/alerts/<int:alert_id>', methods=['GET'])
def get_alert(alert_id):
    alert = get_or_404(Alert, alert_id, 'Alert')
    return jsonify(alert.to_dict()), 200
```

### 4. 쿼리 최적화

**Before (N+1 문제)**
```python
def get_logs_with_alerts():
    logs = Log.query.all()
    result = []
    for log in logs:
        result.append({
            'log': log.to_dict(),
            'alert': log.alert.to_dict() if log.alert else None  # N번 쿼리 발생
        })
    return result
```

**After (Eager Loading)**
```python
from sqlalchemy.orm import joinedload

def get_logs_with_alerts():
    logs = Log.query.options(joinedload(Log.alert)).all()  # 1번 쿼리
    result = []
    for log in logs:
        result.append({
            'log': log.to_dict(),
            'alert': log.alert.to_dict() if log.alert else None
        })
    return result
```

### 5. 설정 값 추출

**Before**
```python
def send_email(to, subject, body):
    smtp_server = 'smtp.gmail.com'
    smtp_port = 587
    from_email = 'noreply@example.com'
    password = 'hardcoded_password'  # ❌
    # ...
```

**After**
```python
# config.py
class Config:
    SMTP_SERVER = os.getenv('SMTP_SERVER', 'smtp.gmail.com')
    SMTP_PORT = int(os.getenv('SMTP_PORT', 587))
    FROM_EMAIL = os.getenv('FROM_EMAIL')
    EMAIL_PASSWORD = os.getenv('EMAIL_PASSWORD')

# service.py
from config import Config

def send_email(to, subject, body):
    smtp_server = Config.SMTP_SERVER
    smtp_port = Config.SMTP_PORT
    from_email = Config.FROM_EMAIL
    password = Config.EMAIL_PASSWORD
    # ...
```

### 6. 매직 넘버/문자열 제거

**Before**
```python
def filter_logs(level):
    if level not in ['ERROR', 'WARN', 'INFO', 'DEBUG']:
        raise ValueError('Invalid level')

    logs = Log.query.filter_by(level=level).limit(50).all()
    return logs
```

**After**
```python
# app/constants.py
class LogLevel:
    ERROR = 'ERROR'
    WARN = 'WARN'
    INFO = 'INFO'
    DEBUG = 'DEBUG'

    ALL = [ERROR, WARN, INFO, DEBUG]

DEFAULT_PAGE_SIZE = 50

# app/services/log_service.py
from app.constants import LogLevel, DEFAULT_PAGE_SIZE

def filter_logs(level):
    if level not in LogLevel.ALL:
        raise ValueError('Invalid level')

    logs = Log.query.filter_by(level=level).limit(DEFAULT_PAGE_SIZE).all()
    return logs
```

## 리팩토링 체크리스트
- [ ] 테스트 코드 작성 완료
- [ ] 긴 함수 분리 (10줄 이상 → 분리 고려)
- [ ] 중복 코드 제거 (DRY 원칙)
- [ ] 비즈니스 로직 서비스 레이어로 이동
- [ ] 매직 넘버/문자열 상수화
- [ ] 하드코딩된 설정값 환경변수로 추출
- [ ] N+1 쿼리 문제 해결
- [ ] 의미 있는 변수/함수명 사용
- [ ] 주석 최소화 (코드 자체가 설명)
- [ ] 리팩토링 후 테스트 실행

## 리팩토링 순서
1. **안전망 구축** - 테스트 코드 작성
2. **작은 변경** - 한 가지씩 개선
3. **테스트 실행** - 매 변경마다 확인
4. **커밋** - 작동하는 상태로 커밋
5. **반복** - 다음 개선 사항으로 이동

## 언제 리팩토링하는가?
- 새 기능 추가 전 (코드 정리)
- 버그 수정 후 (근본 원인 제거)
- 코드 리뷰 중 (중복/복잡도 발견)
- 성능 문제 발견 시
- 테스트 추가가 어려울 때

## 피해야 할 것
- ❌ 테스트 없이 리팩토링
- ❌ 한 번에 여러 가지 변경
- ❌ 기능 추가와 리팩토링 동시 진행
- ❌ 잘 모르는 코드 대규모 변경
- ❌ 리팩토링만 하는 큰 PR
