# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

**LogWatch Admin** - 실시간 로그 수집, 분석 및 모니터링을 위한 웹 기반 어드민 대시보드

## 명령어

```bash
npm install          # 의존성 설치
npm run dev          # 개발 서버 시작 (Turbopack, localhost:3000)
npm run build        # 프로덕션 빌드
npm run start        # 프로덕션 서버 시작
npm run lint         # ESLint 실행
```

## 기술 스택

- **프레임워크**: Next.js 16 (App Router)
- **언어**: TypeScript
- **스타일링**: Tailwind CSS v4

**사전 요구사항**: Node.js 18 이상

## 아키텍처

Next.js 16 App Router 기반의 풀스택 웹 애플리케이션. 프론트엔드와 백엔드 API가 단일 프로젝트에 통합되어 있음.

### 라우팅 구조

- **`src/app/(dashboard)/`** — 대시보드 라우트 그룹. `Sidebar` + `Header` 레이아웃 공유
  - `/` — 대시보드 메인 (실시간 로그 스트림, 차트, 핵심 지표)
  - `/logs` — 로그 목록 및 검색 (키워드 검색, 날짜/시간 필터, 상세 뷰어)
  - `/alerts` — 알림 관리 (임계값 규칙 설정, 알림 이력, 채널 관리)
  - `/settings` — 시스템 설정 (사용자 관리, 로그 소스 설정, 보존 정책)

- **`src/app/(auth)/`** — 인증 라우트 그룹. 중앙 정렬 레이아웃, 사이드바 없음
  - `/login` — 로그인
  - `/register` — 회원가입

- **`src/app/api/`** — API 라우트 핸들러. 각 엔드포인트는 `route.ts` 파일에서 HTTP 메서드 함수를 export

라우트 그룹 `(dashboard)`와 `(auth)`는 URL에 **나타나지 않으며**, 레이아웃 구성에만 사용됨.

### API 엔드포인트

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/logs` | 로그 목록 조회 |
| `GET` | `/api/logs/:id` | 로그 상세 조회 |
| `GET` | `/api/alerts` | 알림 규칙 목록 |
| `POST` | `/api/alerts` | 알림 규칙 생성 |
| `GET` | `/api/stats` | 대시보드 통계 데이터 |
| `POST` | `/api/auth/login` | 로그인 |
| `POST` | `/api/auth/register` | 회원가입 |

### 디렉토리 구조

```
src/
├── app/                  # Next.js App Router (페이지, 레이아웃, API 라우트)
│   ├── (dashboard)/      # 대시보드 라우트 그룹
│   ├── (auth)/           # 인증 라우트 그룹
│   └── api/              # API 라우트 핸들러
├── components/
│   ├── ui/               # 재사용 UI 프리미티브 (Button, Input, Card 등)
│   ├── layout/           # 셸 컴포넌트 (Sidebar, Header)
│   └── features/         # 기능별 복합 컴포넌트 (로그 테이블, 차트, 알림 폼 등)
├── lib/                  # 공유 유틸리티, 상수, 헬퍼
├── hooks/                # 커스텀 React 훅
├── types/                # TypeScript 타입 정의
├── services/             # API 클라이언트 함수 / 외부 서비스 연동
└── styles/               # 추가 글로벌 스타일 (Tailwind 외)
```

### 코딩 컨벤션

- **import 별칭**: `@/*`는 `src/*`에 매핑
  ```typescript
  import { Sidebar } from "@/components/layout/Sidebar";
  ```

- **컴포넌트 export**: named export 사용, default export 사용하지 않음
  ```typescript
  export function LogTable() { ... }
  ```

- **API 라우트**: `next/server`의 `NextResponse.json()`으로 응답 반환
  ```typescript
  import { NextResponse } from "next/server";
  export async function GET() {
    return NextResponse.json({ logs: [] });
  }
  ```

### 주요 기능 영역

- **대시보드**: 실시간 로그 스트림, 로그 레벨별 필터링(ERROR, WARN, INFO, DEBUG), 시간대별 추이 차트
- **로그 검색**: 키워드 검색, 날짜/시간 범위 필터, 로그 소스별 분류
- **알림 관리**: 임계값 기반 알림 규칙, 알림 이력, 채널 관리(이메일, 슬랙)
- **시스템 관리**: 사용자 계정, 로그 수집 소스 설정, 보존 정책
