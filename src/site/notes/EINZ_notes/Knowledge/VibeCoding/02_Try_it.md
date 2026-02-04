---
{"dg-publish":true,"permalink":"/einz-notes/knowledge/vibe-coding/02-try-it/","title":"02. 직접 해보기"}
---


# 02. 직접 해보기

이 가이드에서는 **바이브 코딩(Vibe Coding)** 을 통해 React와 NestJS를 활용한 '간단한 메모 앱'을 구축합니다. 직접 코드를 타이핑하기보다 AI에게 의도를 전달하고 결과를 조율하는 데 집중하세요.

---

## Step 1: Frontend Proto (React + Tailwind CSS)

백엔드 없이 브라우저의 `LocalStorage`를 활용해 데이터를 저장하는 프로토타입을 만듭니다.

### 목표
* AI를 활용해 프로젝트 초기 구조 잡기
* Tailwind CSS로 빠르게 스타일링 하기
* LocalStorage 연동 로직 구현하기

### 프롬프트 시도해보기

> Vite + React + Tailwind CSS 스택으로 메모 앱을 만들어줘.
> 1. 구조: 비즈니스 로직은 `useMemoStore`라는 커스텀 훅으로 분리하고, UI 컴포넌트와 분리해줘.
> 2. 기능: 메모 추가, 삭제가 가능해야 하며 `title`, `content`, `id(uuid)`, `createdAt` 데이터를 포함해줘.
> 3. 저장: LocalStorage를 사용하되, 에러 핸들링(용량 초과 등)을 포함한 유틸리티 함수를 만들어줘.
> 4. 디자인: Shadcn UI 스타일의 다크 모드를 지원하는 깔끔한 레이아웃으로 부탁해.

> [!tip] Antigravity의 AgentManager 화면에서 시도해보기
> Antigravity의 경우 CLI 통합 환경인 Claude Code나 Gemini CLI와 다르게 AI Agent가 IDE에 통합되어 있는 환경입니다. `Ctrl + E` 키를 눌러 AgentManager 화면을 열고 프롬프트를 입력해보세요.

![Pasted image 20260204142247.png|AgentManager](/img/user/EINZ_notes/Knowledge/VibeCoding/images/Pasted%20image%2020260204142247.png)

> [!tip] Implementation Plan에 Comment 기능 사용하기
> 최초 프롬프트로 지시할 때 Antigravity가 Implemenation Plan을 제안할 경우, 이 Plan 내에 의도하지 않았거나 의도했는데 누락된 기능 및 지시 사항이 있는 경우 해당 부분을 Drag 하거나 Add Comment 버튼을 클릭하여 Plan을 수정할 수 있습니다.

---

## Step 2: Fullstack Integration (React + NestJS + MySQL)

이제 메모를 서버에 저장하고 데이터베이스로 관리하는 풀스택 앱으로 확장합니다.

### 목표
* NestJS 백엔드 구축 및 CRUD API 생성
* MySQL 테이블 스키마 설계 및 연동
* 프론트엔드 API 호출 로직 수정

### 프롬프트 시도해보기

> **(Antigravity 또는 Cursor의 @Codebase 활용)**

> 이제 이 프로젝트에 NestJS 백엔드를 추가할 거야.
> 1. Backend: MySQL을 DB로 쓰고, TypeORM을 사용해줘. `Memo` 테이블(id, title, content, createdAt)을 생성하고 CRUD API를 만들어줘. CORS 설정을 활성화해서 프론트엔드 호출을 허용해줘.
> 2. Frontend: 기존 LocalStorage 로직을 제거하고 `Axios`와 `React Query`를 사용해 서버와 통신하도록 바꿔줘.
> 3. Type: 백엔드의 Entity와 프론트엔드의 Interface를 일치시켜서 타입 안정성을 확보해줘.
> 4. Infrastructure: `docker-compose.yml`을 만들어서 NestJS와 MySQL을 한 번에 띄울 수 있게 해줘.

---

## TIPS

### 1. "Refactor this" 기법
기능이 동작한다면, 코드를 드래그하고 AI에게 이렇게 물어보세요.
> "이 컴포넌트의 가독성을 높이고 싶어. 비즈니스 로직을 커스텀 훅으로 분리해줄 수 있어?"

> [!tips]
> Antigravity일 경우 코드를 드래그 하고 바로 AgentManager 혹은 Command Pallete을 통해 AI에 질의가 가능합니다.

### 2. "Explain like I'm five" (5살에게 설명하듯이 설명해줘)
생성된 코드가 이해되지 않는다면 주저 말고 설명을 요구하세요.
> "이 NestJS의 데코레이터들이 각각 어떤 역할을 하는지 초보자도 이해하기 쉽게 설명해줘."

### 3. 에러 발생 시 대처법
터미널의 에러 메시지를 그대로 복사해서 붙여넣으세요.
> "서버 실행 중에 `EntityNotFoundError`가 발생했어. `memos.service.ts` 파일에서 발생 가능한 잠재적 오류를 탐색해줘.

---

## 완료 체크리스트
- [ ] React 앱에서 메모 저장 및 삭제가 가능한가?
- [ ] 페이지 새로고침 후에도 데이터가 유지되는가?
- [ ] NestJS 서버를 통해 DB에 데이터가 정상적으로 들어가는가?
- [ ] (선택) AI에게 요청해 '검색 기능'을 추가해 보았는가?

---

**Next Step:**
축하합니다! 직접 앱을 만들어보며 AI와 협업하는 감을 잡으셨나요?

---
*Last Updated: 2026-02-04*