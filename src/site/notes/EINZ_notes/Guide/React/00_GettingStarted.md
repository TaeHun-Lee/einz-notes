---
{"dg-publish":true,"permalink":"/einz-notes/guide/react/00-getting-started/","title":"00. 리액트란?"}
---


# 00. 리액트란?

## 1. 핵심 철학

### 1.1. 라이브러리를 넘어선 프레임워크 생태계

과거의 React는 단순히 UI를 만들기 위한 라이브러리였으나, 현재는 **Next.js**, **Remix**와 같은 메타 프레임워크와 결합하여 **풀스택 아키텍처**로 변화했습니다.

### 1.2. 렌더링 전략의 변화 (RSC)
- **과거:** 모든 로직과 렌더링을 브라우저(Client)에서 처리 (CSR).
- **현재:** **React Server Components (RSC)**를 표준으로 채택.
    - 데이터 페칭과 무거운 로직은 **서버**에서 수행.
    - 인터랙션(클릭, 입력 등)이 필요한 부분만 `'use client'`를 선언하여 **클라이언트**로 전송.
    - 목표: 초기 로딩 속도(FCP) 개선 및 번들 사이즈 최소화.

---

## 2. 기술 스택 표준

일반적으로 아래의 툴 체인을 사용합니다.

| 분류 | 표준 기술 (Standard) | 비고 |
| :--- | :--- | :--- |
| Language | TypeScript | `Strict` 모드 필수, `any` 사용 지양 |
| Framework | Next.js (App Router) | SEO 및 SSR 필요 시 필수 |
| Build Tool | Vite | 순수 SPA(어드민 등) 혹은 가벼운 앱 개발 시 사용 |
| Styling | Tailwind CSS | Zero-runtime, 유틸리티 클래스 기반 |
| Server State | TanStack Query | API 데이터 캐싱 및 동기화 |
| Client State | Zustand | 전역 UI 상태 관리 (Redux 대체) |
| Form | React Hook Form | 폼 유효성 검사 및 성능 최적화 |

> [!warning] **Legacy React와 달라진 점**
> - `Create React App (CRA)`은 더 이상 사용하지 않습니다.
> - `Redux`는 복잡한 비동기 상태 관리가 필요한 특수한 경우를 제외하고는 도입을 지양합니다.
> - `useEffect` 내부에서 직접 데이터를 `fetch`하는 것을 금지합니다. (TanStack Query 사용 권장)

### Linting & Formatting
 
- ESLint + Prettier: 대표적으로 많이 사용됩니다.
- 엄격한 규칙: `eslint-plugin-react-hooks`를 반드시 활성화하여 의존성 배열(dependency array) 누락을 엄격하게 잡습니다.
- Biome: 최근 ESLint와 Prettier를 합친 고성능 도구인 Biome으로 넘어가는 프로젝트들도 늘어나고 있습니다.

---

## 3. 폴더 구조 및 아키텍처

기능(Feature) 기반 아키텍처를 지향합니다. 기술적 분류(components, hooks)보다 도메인(Business Logic) 기준의 응집도를 우선합니다.

### 디렉토리 구조 예시
```text
src/
├── app/                  # Next.js App Router (라우팅 진입점)
├── components/           # 전역 공통 UI (Button, Input, Modal...)
├── hooks/                # 전역 유틸리티 훅 (useDebounce, useToggle...)
├── lib/                  # 외부 라이브러리 설정 (axios, queryClient...)
├── features/             # ✨ 핵심: 기능 단위 모듈
│   ├── auth/             # 인증 관련 기능
│   │   ├── api/          # 로그인, 회원가입 API
│   │   ├── components/   # 로그인 폼, 회원가입 버튼
│   │   ├── hooks/        # useAuth, useLogin
│   │   └── types/        # User 인터페이스
│   └── dashboard/        # 대시보드 기능
└── utils/                # 순수 자바스크립트 헬퍼 함수
```

## 4. 코드 컨벤션

### 4.1. 컴포넌트 작성

- 선언 방식: 디버깅 용이성을 위해 `function` 키워드 사용을 권장합니다. (컴포넌트 명 파악 용이)
- Props 파라미터에서 즉시 구조 분해 할당합니다.

```typescript
export function UserCard({ name, role }: UserCardProps) {
  return <div>{name} ({role})</div>;
}
```

### 4.2. Custom Hooks 분리

- 컴포넌트 내부의 로직이 길어지면 반드시 **Custom Hook**으로 분리합니다.
- UI 컴포넌트는 "어떻게 보여줄지"에만 집중하고, 훅은 "어떻게 동작할지"를 담당합니다.
-  `use[Feature]Logic` 형태의 커스텀 훅으로 추출하여 컴포넌트는 렌더링에만 집중하게 합니다.

### 4.3. 파일명 규칙

- **컴포넌트:** `PascalCase.tsx` (예: `UserProfile.tsx`)
- **훅/함수:** `camelCase.ts` (예: `useAuth.ts`, `formatDate.ts`)

### 4.4 상태 관리의 분리

과거에는 Redux 하나에 모든 상태를 담았지만, 지금은 **상태의 성격**에 따라 도구를 나누어 씁니다.

|**상태 종류**|**설명**|**추천 도구**|
|---|---|---|
|Server State|API 데이터 (캐싱, 동기화 필요)|TanStack Query (React Query), SWR|
|Client State|UI 상태 (모달 열림, 테마 등)|Zustand, Jotai, Recoil|
|Form State|입력 폼 데이터 및 유효성 검사|React Hook Form|
|Context API|단순한 전역 설정 (복잡한 로직 지양)|React Built-in Context|

### 4.5 Barrel Files 지양

 `index.ts`를 만들어 폴더 내 모든 것을 Re-export 하는 방식은 트리쉐이킹 성능 문제를 일으킬 수 있어, 최근 툴체인에서는 꼭 필요한 경우에만 사용하거나 지양하는 추세입니다.

---

## 5. 참고 자료

아래는 작성 시 참고했거나 참고 할 만한 자료들입니다.
업무 투입 전 아래 자료들을 순차적으로 확인하는 것을 권장합니다.

### 아키텍처 및 패턴

- [ ] **[Bulletproof React](https://github.com/alan2207/bulletproof-react)**: 확장 가능한 프로젝트 구조의 교과서 (GitHub)
- [ ] **[Patterns.dev](https://www.patterns.dev/)**: 모던 웹 앱 디자인 패턴 가이드

### 실무 적용 사례 (기술 블로그)

- [ ] **[토스 기술 블로그](https://toss.tech/)**: "Effective Component", "선언적인 코드 작성하기"
- [ ] **[우아한형제들 기술 블로그](https://techblog.woowahan.com/)**: "React Query & Zustand 도입기"
- [ ] **[카카오 엔터테인먼트 FE](https://www.google.com/search?q=https://fe-developers.kakaoent.com/)**: Next.js 및 아토믹 디자인 적용 사례

### 심화 학습

- [ ] **[React.dev - Thinking in React](https://react.dev/learn/thinking-in-react)**: 리액트 공식 설계 사고방식
- [ ] **[TkDodo's Blog](https://tkdodo.eu/blog/)**: React Query의 모든 것 (Best Practices)

---
*Last Updated: 2026-02-12*