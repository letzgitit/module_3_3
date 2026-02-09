---
name: fe-code-review
description: LogWatch Admin 프론트엔드 코드 리뷰 및 테스트 코드 검증
---

# LogWatch Admin - 프론트엔드 테스트 가이드

## 프로젝트 정보
- **Next.js 16 (App Router)** + TypeScript + Tailwind CSS v4
- **테스트 도구**: Jest + React Testing Library + MSW

## 빠른 설정

```bash
# 설치
npm install -D jest @testing-library/react @testing-library/jest-dom
npm install -D jest-environment-jsdom msw@latest

# 실행
npm run test              # 전체 테스트
npm run test:watch        # watch 모드
npm run test:coverage     # 커버리지
```

### jest.config.js
```javascript
const nextJest = require('next/jest')
module.exports = nextJest({ dir: './' })({
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
  testEnvironment: 'jest-environment-jsdom',
  moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1' },
})
```

### jest.setup.js
```javascript
import '@testing-library/jest-dom'
import { server } from './src/mocks/server'
beforeAll(() => server.listen())
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

## 테스트 전략

### 1. Server Components
로직을 Client Component로 분리하여 테스트

```typescript
// ❌ 테스트 불가
export default async function Page() {
  const data = await fetchData()
  return <div>{data}</div>
}

// ✅ 분리하여 테스트
export default async function Page() {
  const data = await fetchData()
  return <DataView data={data} />
}

// DataView.test.tsx
test('데이터를 렌더링한다', () => {
  render(<DataView data={mockData} />)
  expect(screen.getByText('1000')).toBeInTheDocument()
})
```

### 2. Client Components

```typescript
import { render, screen, fireEvent } from '@testing-library/react'

jest.mock('next/navigation', () => ({
  useRouter: () => ({ push: jest.fn() }),
  usePathname: () => '/',
}))

test('필터 선택 시 onChange 호출', () => {
  const onChange = jest.fn()
  render(<LogFilter onChange={onChange} />)
  fireEvent.click(screen.getByLabelText('ERROR'))
  expect(onChange).toHaveBeenCalledWith({ level: 'ERROR' })
})
```

### 3. API 라우트

```typescript
import { GET } from './route'
import { NextRequest } from 'next/server'

test('로그 목록 반환', async () => {
  const req = new NextRequest('http://localhost:3000/api/logs')
  const res = await GET(req)
  expect(res.status).toBe(200)
  expect(await res.json()).toHaveProperty('logs')
})
```

### 4. 커스텀 훅

```typescript
import { renderHook, act, waitFor } from '@testing-library/react'

test('로그인 성공', async () => {
  const { result } = renderHook(() => useAuth())

  await act(async () => {
    await result.current.login('user@test.com', 'pass')
  })

  await waitFor(() => {
    expect(result.current.isAuthenticated).toBe(true)
  })
})
```

## MSW 설정

```typescript
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/logs', () =>
    HttpResponse.json({ logs: [/* mock data */] })
  ),
  http.post('/api/alerts', async ({ request }) => {
    const body = await request.json()
    return HttpResponse.json({ id: 'new', ...body }, { status: 201 })
  }),
]

// src/mocks/server.ts
import { setupServer } from 'msw/node'
import { handlers } from './handlers'
export const server = setupServer(...handlers)
```

## 기능별 테스트 체크리스트

### 대시보드
- [ ] 실시간 로그 스트림 렌더링
- [ ] 로그 레벨 필터링 (ERROR, WARN, INFO, DEBUG)
- [ ] 차트 데이터 표시
- [ ] 핵심 지표 카드

### 로그 검색 (/logs)
- [ ] 키워드 검색
- [ ] 날짜/시간 범위 필터
- [ ] 페이지네이션
- [ ] 로그 상세 뷰

### 알림 관리 (/alerts)
- [ ] 알림 규칙 폼 유효성 검증
- [ ] 임계값 설정
- [ ] 알림 채널 선택

### 레이아웃
- [ ] Sidebar 네비게이션 링크
- [ ] 현재 경로 활성화 표시

## 코드 리뷰 체크리스트

### 필수
- [ ] 새 컴포넌트/페이지에 테스트 코드 작성
- [ ] Server/Client Component 구분 테스트
- [ ] API 라우트 테스트 포함
- [ ] 모든 import 문 작성
- [ ] `jest.mock()` 파일 최상단 위치
- [ ] 모든 테스트 통과
- [ ] 커버리지 목표 달성

### 권장
- [ ] 에러/로딩 상태 테스트
- [ ] 엣지 케이스 처리
- [ ] AAA 패턴 (Arrange-Act-Assert)
- [ ] 명확한 테스트 설명

### LogWatch 특화
- [ ] 실시간 로그 스트림 업데이트
- [ ] 로그 검색/필터링 로직
- [ ] 알림 규칙 생성/수정
- [ ] 차트 렌더링

## 커버리지 목표

| 영역 | 목표 |
|------|------|
| UI 컴포넌트 | 90% |
| 기능 컴포넌트 | 80% |
| API 라우트 | 85% |
| 훅 | 90% |
| 유틸리티 | 95% |

## 피해야 할 것

```typescript
// ❌ 구현 세부사항 테스트
expect(component.state.count).toBe(1)

// ✅ 사용자 관점 테스트
expect(screen.getByText('1')).toBeInTheDocument()

// ❌ 비동기 처리 없음
fireEvent.click(button)
expect(screen.getByText('Success')).toBeInTheDocument()

// ✅ 비동기 처리
fireEvent.click(button)
await waitFor(() => {
  expect(screen.getByText('Success')).toBeInTheDocument()
})

// ❌ jest.mock 테스트 내부
test('...', () => {
  jest.mock('next/navigation')
})

// ✅ jest.mock 파일 최상단
jest.mock('next/navigation')
test('...', () => {})
```

## 파일 구조

```
src/
├── app/
│   ├── (dashboard)/page.tsx + page.test.tsx
│   ├── (auth)/login/page.tsx + page.test.tsx
│   └── api/logs/route.ts + route.test.ts
├── components/
│   ├── ui/Button.tsx + Button.test.tsx
│   ├── layout/Sidebar.tsx + Sidebar.test.tsx
│   └── features/LogTable.tsx + LogTable.test.tsx
├── hooks/useAuth.ts + useAuth.test.ts
├── lib/utils.ts + utils.test.ts
└── mocks/
    ├── handlers.ts
    └── server.ts
```

## 참고 문서
- [Jest](https://jestjs.io/)
- [React Testing Library](https://testing-library.com/react)
- [MSW](https://mswjs.io/)
- [Next.js Testing](https://nextjs.org/docs/app/building-your-application/testing)
