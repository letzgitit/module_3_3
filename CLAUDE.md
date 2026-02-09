# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 명령어

```bash
npm run dev          # 개발 서버 시작 (Turbopack, localhost:3000)
npm run build        # 프로덕션 빌드
npm run start        # 프로덕션 서버 시작
npm run lint         # ESLint 실행
```

## 아키텍처

**Next.js 16 App Router** 기반의 FE + BE 통합 프로젝트. TypeScript 전체 적용, Tailwind CSS v4로 스타일링.

### 라우팅 구조

- **`src/app/(dashboard)/`** — 대시보드 페이지 라우트 그룹. `Sidebar` + `Header`가 포함된 공유 레이아웃 사용. 루트 경로 `/`가 대시보드를 렌더링함.
- **`src/app/(auth)/`** — 인증 페이지 라우트 그룹 (로그인, 회원가입). 사이드바 없는 중앙 정렬 레이아웃 사용.
- **`src/app/api/`** — 백엔드 API 라우트 핸들러. 각 엔드포인트는 `route.ts` 파일에서 HTTP 메서드 함수(`GET`, `POST` 등)를 export.

라우트 그룹 `(dashboard)`와 `(auth)`는 URL에 **나타나지 않으며**, 어떤 레이아웃으로 감쌀지만 결정함.

### 디렉토리 구조

```
src/
├── app/              # Next.js App Router (페이지, 레이아웃, API 라우트)
├── components/
│   ├── ui/           # 재사용 UI 프리미티브 (Button, Input, Card 등)
│   ├── layout/       # 셸 컴포넌트 (Sidebar, Header)
│   └── features/     # 기능별 복합 컴포넌트
├── lib/              # 공유 유틸리티, 상수, 헬퍼
├── hooks/            # 커스텀 React 훅
├── types/            # TypeScript 타입 정의
├── services/         # API 클라이언트 함수 / 외부 서비스 연동
└── styles/           # 추가 글로벌 스타일 (Tailwind 외)
```

### 코딩 컨벤션

- import 별칭: `@/*`는 `src/*`에 매핑 (예: `import { Sidebar } from "@/components/layout/Sidebar"`)
- 컴포넌트는 named export 사용 (`export function Button`), default export 사용하지 않음
- API 라우트는 `next/server`의 `NextResponse.json()`으로 응답 반환
- 사용자 대면 문자열과 주석에 한국어 사용 가능
