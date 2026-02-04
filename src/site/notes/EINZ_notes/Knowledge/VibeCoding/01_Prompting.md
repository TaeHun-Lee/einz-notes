---
{"dg-publish":true,"permalink":"/einz-notes/knowledge/vibe-coding/01-prompting/","title":"01. 프롬프팅 표준 가이드"}
---


# 01. 프롬프팅 표준 가이드

바이브 코딩(Vibe Coding)의 핵심은 코드를 직접 치는 것이 아니라, AI에게 **의도(Intent)** 를 정확히 전달하고 **맥락(Context)** 을 동기화하는 것입니다. 이 문서에서는 사내 개발 시 AI와 소통하는 표준 방식을 정의합니다.

---

## 1. 프롬프팅 3대 원칙

> [!important] 1. 맥락을 명확히 명시한다.
> AI는 내가 보고 있는 화면을 다 알지 못합니다. 관련 파일, 로그, 구조를 명시적으로 제공하세요.

> [!important] 2. 구체적으로 명령하라.
> "로그인 만들어줘" 보다는 "NestJS에서 Passport를 사용해 JWT 기반 로그인 로직을 작성해줘"가 정확합니다.

> [!important] 3. 반복적으로 개선하라.
> 한 번에 완벽한 결과물을 기대하기보다, 대화를 통해 코드를 다듬어 나가는 과정이 중요합니다. 예를 들어 "NestJS의 Pipeline으로 로그인 보안 검증 절차를 만들어줘" 라고 명령 했다면, "만들어진 소스에 대해 오류가 없는지 점검해줘" 와 같이 재검증 절차를 거치는 게 좋습니다.

---

## 2. 도구별 컨텍스트 주입법

각 도구의 기능을 활용해 AI에게 '맥락'을 전달하는 방법입니다.

### 1. Claude Code / Cursor 스택
* **Symbol 활용**: `@`를 입력하여 AI가 참조해야 할 대상을 지정하세요.
    * `@Files`: 특정 파일의 코드를 읽게 할 때
    * `@Folders`: 프로젝트 구조를 파악하게 할 때
    * `@Web`: 최신 라이브러리 공식 문서를 참조할 때

### 2. Gemini / Antigravity 스택
* **Global Rules 활용**: 프로젝트 루트의 `GEMINI.md`나 Antigravity의 Global Rules설정을 통해 사내 컨벤션 및 규칙을 상시 주입하세요.
* **이미지 활용**: UI 구현 시 스케치나 캡처본을 첨부하면 훨씬 정확한 코드가 생성됩니다.

### 3. Gemini CLI + VSCode 스택
* **Pipe 활용**: 터미널의 출력을 직접 전달하세요.
    ```bash
    cat error.log | gemini "이 에러의 원인과 해결책을 알려줘"
    ```

---

## 3. 표준 프롬프트 패턴 (Templates)

상황 별로 아래 패턴을 복사하여 수정해 사용하세요.

### A. 신규 기능 구현 (New Feature)
> **Role**: 너는 Senior [기술스택] 개발자야.
> **Context**: 현재 `@src/services` 구조를 참고해줘.
> **Task**: [기능명] 기능을 추가하려고 해.
> **Constraint**: 외부 라이브러리 없이 순수 SDK만 사용하고, 에러 핸들링은 `Result` 패턴을 적용해줘.

### B. 코드 리뷰 및 리팩토링 (Refactoring)
> **Task**: 첨부한 코드의 시간 복잡도를 개선해줘.
> **Focus**: 가독성보다 성능 최적화에 집중하고, 변경된 부분에 주석을 달아줘.

### C. 트러블슈팅 (Debugging)
> **Problem**: [에러 메시지]가 발생했어.
> **Context**: 관련 있는 `@파일명`과 현재 환경 정보를 보낼게.
> **Request**: 발생 가능한 원인 3가지를 우선순위대로 나열하고 해결 코드를 작성해줘.

---

## 4. 기본 컨텍스트

Claude Code의 경우 Claude.md, Gemini CLI 및 Antigravity의 경우 GEMINI.md에 기본 컨텍스트 (프롬프트 시작 이전 사전에 AI Model이 숙지해야 할 사항) 제공이 가능합니다.

### 추천 컨텍스트 프롬프트

> Mandatory Language Rule: The output language must always be Korean. Even if the input is in English or other languages, do not translate your response back to that language. Use Korean exclusively unless a different language is explicitly requested for a specific task.

- 한글 대답을 강제합니다.

> Coding Principles & Guidelines:  Strictly adhere to the following principles when generating or refactoring code:
> 1. Descriptive & Meaningful Naming
> 1.1 Use highly descriptive and self-documenting names for all functions, variables, and classes.
> 1.2 Avoid abbreviations (e.g., use `fetchUserAuthenticationStatus` instead of `authStat`).
> 1.3 Names should clearly communicate the "Intent" and "Action" of the code.
> 2. Preference for Native Features (Vanila/Pure)
> 2.1 Minimize reliance on third-party libraries. 
> 2.2 Prioritize built-in JavaScript (ES6+) and TypeScript features unless an external library is essential for performance or security.
> 2.3 Aim for lightweight, dependency-free implementations to reduce bundle size and maintenance overhead.
> 3. Robust Error Handling & User Feedback
> 3.1 Do not use simple `console.log` for error management.
> 3.2 Implement structured error handling (e.g., try-catch blocks, Result patterns) that provides meaningful context.
> 3.3 Design code to return or emit user-friendly feedback/notifications so the UI can inform the user of the current state or failure.

- 코드 컨벤션을 주입합니다. 명확하고 직관적인 변수 명, 함수 명을 강제하고 가능하면 외부 라이브러리 대신 Vanila 코드를 작성하도록 하게 하며 번들 사이즈를 최소화 하려고 노력하도록 지시합니다. 에러 처리를 보다 고도화 하도록 강제합니다. 

> Response Format
> 1. Provide a brief explanation of the architectural choices made.
> 2. Ensure all code snippets are TypeScript-first with proper type definitions.
> 3. If a trade-off is required between these principles, explain the reasoning.

- 답변 사항에 대해 더 자세하게 설명하도록 지시합니다.

---

## ⚠️ 주의 사항

1. **보안 주의**: 프롬프트에 고객 개인정보, 비밀번호, API Key를 절대 포함하지 마세요.
2. **맹신 금지**: AI가 생성한 코드는 반드시 로컬에서 빌드 및 테스트를 거쳐야 합니다.
3. **바이브 유지**: AI가 엉뚱한 답을 한다면, 이전 대화를 수정(Edit)하거나 맥락을 초기화하여 다시 시작하세요.

---

> [!tips] 프롬프트 가이드: 영문 시스템 프롬프트 활용하기
> Q: 왜 시스템 프롬프트는 영문이 유리한가요?
> A: 현재 대부분의 대형 언어 모델(LLM)은 영문 데이터셋으로 가장 밀도 있게 학습되었습니다. 특히 코딩 컨벤션, 아키텍처 설계, 기술적 제약 사항은 영문으로 전달할 때 AI가 훨씬 더 정교하고 논리적인 코드를 생성합니다. 답변은 한글로 받더라도, AI의 '페르소나'와 '규칙'은 위 영문 템플릿을 사용하는 것을 권장합니다.

---

**Next Step:**
- 이제 [[EINZ_notes/Knowledge/VibeCoding/02_Try_it\|02. 직접 해보기]] 문서로 이동하여 우리 프로젝트의 구조를 학습해보세요.

---
*Last Updated: 2026-02-04*