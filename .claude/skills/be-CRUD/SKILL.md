---
name: be-CRUD
description: RESTful CRUD API 구현
---

# CRUD API 구현 가이드

## RESTful API 패턴

| 동작 | HTTP 메서드 | 엔드포인트 | 상태 코드 |
|------|------------|-----------|----------|
| 목록 조회 | GET | /api/logs | 200 |
| 단일 조회 | GET | /api/logs/:id | 200, 404 |
| 생성 | POST | /api/logs | 201 |
| 수정 | PUT | /api/logs/:id | 200, 404 |
| 삭제 | DELETE | /api/logs/:id | 204, 404 |

## 표준 구현 템플릿

### 1. Create (생성)
```python
@app.route('/api/logs', methods=['POST'])
@jwt_required()
def create_log():
    """로그 생성"""
    data = request.get_json()

    # 유효성 검증
    if not data.get('level') or not data.get('message'):
        return jsonify({'error': 'level과 message는 필수입니다'}), 400

    # 생성
    log = Log(
        level=data['level'],
        message=data['message'],
        source=data.get('source'),
        timestamp=datetime.utcnow()
    )

    db.session.add(log)
    db.session.commit()

    return jsonify(log.to_dict()), 201
```

### 2. Read (조회)
```python
@app.route('/api/logs', methods=['GET'])
@jwt_required()
def get_logs():
    """로그 목록 조회"""
    # 쿼리 파라미터
    level = request.args.get('level')
    page = request.args.get('page', 1, type=int)
    limit = request.args.get('limit', 20, type=int)

    # 필터링
    query = Log.query
    if level:
        query = query.filter_by(level=level)

    # 페이지네이션
    pagination = query.order_by(Log.timestamp.desc()).paginate(
        page=page, per_page=limit, error_out=False
    )

    return jsonify({
        'logs': [log.to_dict() for log in pagination.items],
        'total': pagination.total,
        'page': page,
        'pages': pagination.pages
    }), 200

@app.route('/api/logs/<int:log_id>', methods=['GET'])
@jwt_required()
def get_log(log_id):
    """로그 단일 조회"""
    log = Log.query.get_or_404(log_id)
    return jsonify(log.to_dict()), 200
```

### 3. Update (수정)
```python
@app.route('/api/logs/<int:log_id>', methods=['PUT'])
@jwt_required()
def update_log(log_id):
    """로그 수정"""
    log = Log.query.get_or_404(log_id)
    data = request.get_json()

    # 수정 가능한 필드만 업데이트
    if 'level' in data:
        log.level = data['level']
    if 'message' in data:
        log.message = data['message']

    log.updated_at = datetime.utcnow()

    db.session.commit()

    return jsonify(log.to_dict()), 200
```

### 4. Delete (삭제)
```python
@app.route('/api/logs/<int:log_id>', methods=['DELETE'])
@jwt_required()
def delete_log(log_id):
    """로그 삭제"""
    log = Log.query.get_or_404(log_id)

    db.session.delete(log)
    db.session.commit()

    return '', 204
```

## 고급 패턴

### 1. 벌크 삭제
```python
@app.route('/api/logs/bulk-delete', methods=['POST'])
@jwt_required()
def bulk_delete_logs():
    """여러 로그 한번에 삭제"""
    data = request.get_json()
    ids = data.get('ids', [])

    if not ids:
        return jsonify({'error': 'ids는 필수입니다'}), 400

    deleted = Log.query.filter(Log.id.in_(ids)).delete(synchronize_session=False)
    db.session.commit()

    return jsonify({'deleted': deleted}), 200
```

### 2. 부분 수정 (PATCH)
```python
@app.route('/api/logs/<int:log_id>', methods=['PATCH'])
@jwt_required()
def patch_log(log_id):
    """로그 부분 수정"""
    log = Log.query.get_or_404(log_id)
    data = request.get_json()

    # 제공된 필드만 수정
    for key, value in data.items():
        if hasattr(log, key):
            setattr(log, key, value)

    db.session.commit()
    return jsonify(log.to_dict()), 200
```

### 3. 중첩 리소스
```python
@app.route('/api/alerts/<int:alert_id>/logs', methods=['GET'])
@jwt_required()
def get_alert_logs(alert_id):
    """특정 알림의 로그 조회"""
    alert = Alert.query.get_or_404(alert_id)
    logs = alert.logs.order_by(Log.timestamp.desc()).all()

    return jsonify({
        'alert': alert.to_dict(),
        'logs': [log.to_dict() for log in logs]
    }), 200
```

## 에러 핸들링

```python
from werkzeug.exceptions import HTTPException

@app.errorhandler(404)
def not_found(error):
    return jsonify({'error': 'Not found'}), 404

@app.errorhandler(400)
def bad_request(error):
    return jsonify({'error': 'Bad request'}), 400

@app.errorhandler(500)
def internal_error(error):
    db.session.rollback()
    return jsonify({'error': 'Internal server error'}), 500

@app.errorhandler(Exception)
def handle_exception(e):
    if isinstance(e, HTTPException):
        return e

    app.logger.error(f"Unhandled exception: {e}")
    db.session.rollback()
    return jsonify({'error': 'Internal server error'}), 500
```

## 유효성 검증

```python
from marshmallow import Schema, fields, validate, ValidationError

class LogSchema(Schema):
    level = fields.Str(required=True, validate=validate.OneOf(['ERROR', 'WARN', 'INFO', 'DEBUG']))
    message = fields.Str(required=True, validate=validate.Length(min=1, max=1000))
    source = fields.Str(validate=validate.Length(max=100))

def create_log():
    schema = LogSchema()
    try:
        data = schema.load(request.get_json())
    except ValidationError as err:
        return jsonify({'errors': err.messages}), 400

    # ... 생성 로직
```

## CRUD 구현 체크리스트
- [ ] 적절한 HTTP 메서드 사용 (GET, POST, PUT, DELETE)
- [ ] 올바른 상태 코드 반환 (200, 201, 204, 400, 404, 500)
- [ ] 입력 유효성 검증
- [ ] 에러 핸들링
- [ ] 인증/인가 확인 (@jwt_required)
- [ ] 페이지네이션 (목록 조회)
- [ ] 필터링/정렬 (쿼리 파라미터)
- [ ] 트랜잭션 관리 (commit/rollback)
- [ ] API 문서화 (docstring)
- [ ] 테스트 코드 작성

## 성능 최적화 팁
- Eager loading으로 N+1 문제 해결
- 인덱스 추가 (자주 조회하는 컬럼)
- 대용량 데이터는 yield_per() 사용
- 캐싱 활용 (Redis)
- 페이지네이션 필수
