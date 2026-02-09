# LogWatch Admin - 로그 모니터링 웹 어드민 시스템

실시간 로그 수집, 분석 및 모니터링을 위한 웹 기반 어드민 대시보드입니다.

## 기술 스택

| 구분 | 기술 |
|------|------|
| 프레임워크 | Next.js 16 (App Router) |
| 언어 | TypeScript |
| 스타일링 | Tailwind CSS v4 |
| 번들러 | Turbopack |

## 시작하기

### 사전 요구사항

- Node.js 18 이상
- npm

### 설치 및 실행

```bash
# 의존성 설치
npm install

# 개발 서버 시작 (localhost:3000)
npm run dev

# 프로덕션 빌드
npm run build

# 프로덕션 서버 시작
npm run start

# 린트 검사
npm run lint
```

## 프로젝트 구조

```
src/
├── app/                  # Next.js App Router
│   ├── (dashboard)/      # 대시보드 라우트 그룹 (루트 경로 /)
│   ├── (auth)/           # 인증 라우트 그룹 (로그인, 회원가입)
│   └── api/              # API 라우트 핸들러
├── components/
│   ├── ui/               # 재사용 UI 프리미티브 (Button, Input, Card 등)
│   ├── layout/           # 셸 컴포넌트 (Sidebar, Header)
│   └── features/         # 기능별 복합 컴포넌트
├── lib/                  # 공유 유틸리티, 상수, 헬퍼
├── hooks/                # 커스텀 React 훅
├── types/                # TypeScript 타입 정의
├── services/             # API 클라이언트 함수 / 외부 서비스 연동
└── styles/               # 추가 글로벌 스타일
```

## 주요 기능

### 대시보드
- 실시간 로그 스트림 모니터링
- 로그 레벨별 필터링 (ERROR, WARN, INFO, DEBUG)
- 시간대별 로그 발생 추이 차트
- 핵심 지표 요약 카드

### 로그 검색 및 분석
- 키워드 기반 로그 검색
- 날짜/시간 범위 필터
- 로그 소스별 분류
- 상세 로그 뷰어

### 알림 관리
- 임계값 기반 알림 규칙 설정
- 알림 이력 조회
- 알림 채널 관리 (이메일, 슬랙 등)

### 시스템 관리
- 사용자 계정 관리
- 로그 수집 소스 설정
- 보존 정책 관리

## 라우팅 구조

| 경로 | 설명 | 레이아웃 |
|------|------|----------|
| `/` | 대시보드 메인 | Sidebar + Header |
| `/logs` | 로그 목록 및 검색 | Sidebar + Header |
| `/alerts` | 알림 관리 | Sidebar + Header |
| `/settings` | 시스템 설정 | Sidebar + Header |
| `/login` | 로그인 | 중앙 정렬 (인증) |
| `/register` | 회원가입 | 중앙 정렬 (인증) |

## 코딩 컨벤션

- **import 별칭**: `@/*`는 `src/*`에 매핑
  ```typescript
  import { Sidebar } from "@/components/layout/Sidebar";
  ```
- **컴포넌트 export**: named export 사용 (default export 사용하지 않음)
  ```typescript
  export function LogTable() { ... }
  ```
- **API 응답**: `NextResponse.json()`으로 반환
  ```typescript
  import { NextResponse } from "next/server";
  export async function GET() {
    return NextResponse.json({ logs: [] });
  }
  ```

## API 엔드포인트

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/logs` | 로그 목록 조회 |
| `GET` | `/api/logs/:id` | 로그 상세 조회 |
| `GET` | `/api/alerts` | 알림 규칙 목록 |
| `POST` | `/api/alerts` | 알림 규칙 생성 |
| `GET` | `/api/stats` | 대시보드 통계 데이터 |
| `POST` | `/api/auth/login` | 로그인 |
| `POST` | `/api/auth/register` | 회원가입 |
