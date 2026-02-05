---
{"dg-publish":true,"permalink":"/einz-notes/guide/vibe-coding/00-getting-started/","title":"00. 바이브코딩이란?"}
---

# 00_GettingStarted: 바이브코딩이란?

> "The hottest new programming language is English." - Andrej Karpathy

**바이브코딩(Vibe Coding)** 은 코드를 한 줄 한 줄 직접 타이핑하는 것이 아니라, AI에게 자연어로 의도(Intent)와 흐름(Vibe)을 전달하며 코드를 작성하는 새로운 개발 패러다임을 의미합니다.

개발자는 이제 '작성자(Writer)'가 아닌 **'관리자(Manager)'** 이자 **'리뷰어(Reviewer)'** 가 되어, AI가 작성한 결과물의 논리를 검증하고 전체적인 아키텍처를 조율하는 데 집중합니다.

---

## 바이브코딩을 위한 3대 요소

바이브코딩을 시작하기 위해서는 다음 3가지 도구의 조합이 필요합니다.

1.  **Brain (LLM)**: 코드를 실제로 생각하고 작성하는 인공지능 모델 (예: Claude 3.5 Sonnet, Gemini 1.5 Pro)
2.  **Body (IDE)**: AI와 개발자가 소통하며 코드를 편집하는 환경 (예: Cursor, VS Code, Windsurf)
3.  **Memory (Context)**: AI에게 프로젝트의 정보를 제공하는 문맥 관리 도구 (예: MCP, RAG, Project Indexing)

---

## ⚔️ 주요 기술 스택 비교: The "Meta" vs The "Challenger"

현재 바이브코딩 커뮤니티에서 가장 주목받는 두 가지 조합의 특징과 장단점을 비교합니다.

### 1️. The "Meta" Stack: Claude + Cursor + MCP
현재 가장 널리 쓰이며, 정교한 코딩 능력에 최적화된 조합입니다.

* **Claude (3.5 Sonnet)**: 현존하는 LLM 중 코딩 로직과 추론 능력이 가장 뛰어납니다.
* **Cursor**: AI Native IDE의 선두주자입니다. `Tab` 키 하나로 코드를 완성하고 수정하는 경험(DX)이 뛰어납니다.
* **MCP (Model Context Protocol)**: Anthropic이 만든 표준 프로토콜로, 로컬 파일뿐만 아니라 Google Drive, Slack, Github 등 외부 데이터 소스를 AI에게 안전하게 연결해 줍니다.

| 장점 (Pros)                           | 단점 (Cons)                                           |
| :---------------------------------- | :-------------------------------------------------- |
| **코딩 품질**: 복잡한 알고리즘 구현에 유리함         | **Context 제한**: 토큰 제한으로 인해 초대형 프로젝트 전체를 한 번에 넣기 어려움 |
| **강력한 생태계**: 다양한 MCP 서버를 통해 확장성 무한대 | **비용**: API 사용량이 많아질수록 비용 부담이 발생할 수 있음              |
| **사용자 경험(UX)**: Cursor의 직관적인 인터페이스  |                                                     |

### 2️. The "Challenger" Stack: Gemini + Anti-Gravity
구글의 생태계와 방대한 문맥 처리 능력을 앞세운 강력한 조합입니다.

* **Gemini (3.0 Pro/Flash)**: **방대한 컨텍스트 윈도우(1M~2M 토큰)** 를 자랑합니다. 수만 개의 파일이 있는 프로젝트 전체를 통째로 읽고 이해할 수 있습니다.
* **Anti-Gravity**: Google에서 Claud AI 및 Cursor 진영에 대항하기 위해 VS Code를 기반으로 Fork하여 만들어낸 Gemini AI 통합 IDE 입니다.
* **Multimodal**: 텍스트 뿐만 아니라 기획서 이미지, 시연 영상 등을 보고 바로 코드로 변환하는 능력이 탁월합니다.

| 장점 (Pros)                                           | 단점 (Cons)                                             |
| :-------------------------------------------------- | :---------------------------------------------------- |
| **Massive Context**: 프로젝트 전체 문서, 로그, 코드를 한 번에 입력 가능 | **디테일 부족**: 로직 생성 시 Claude보다 약간의 지연 및 끊김 현상이 발생할 수 있음 |
| **속도와 비용**: Gemini Flash 모델 사용 시 매우 빠르고 저렴함         | **도구 성숙도**: Cursor에 비해 IDE 통합 경험이 아직 발전 중인 단계         |
| **멀티모달**: UI 스케치만 던져줘도 프론트엔드 코드 생성 가능               |                                                       |

---

## 결론: 무엇을 선택해야 할까?

* **정교한 로직과 복잡한 리팩토링**이 필요하다면? -> **Claude + Cursor**
* **거대한 레거시 코드 분석**이나 **문서 기반의 빠른 프로토타이핑**이 필요하다면? & Claud AI의 토큰 비용이 부담된다면? -> **Gemini 기반 스택**

바이브코딩은 도구보다 **"어떻게 질문하느냐(Prompting)"** 가 더 중요합니다. 지금 바로 시작해봅시다.

---

# 실전 가이드: 설치 및 시작하기

각 스택 별 설치 및 초기 설정 방법입니다.

## 1. Claude + Cursor 시작하기

가장 대중적인 바이브코딩 환경을 구축합니다.

### Claude Code 시작하기

**Claude Code**는 터미널에서 직접 실행되는 Anthropic의 **에이전트형 코딩 도구(Agentic Coding Tool)** 입니다.

> [!INFO] 주요 특징
> * **터미널 통합**: 별도의 창이나 IDE 전환 없이 터미널에서 바로 실행 (`claude`)
> * **직접 행동 (Takes Action)**: 파일 수정, 명령어 실행, 커밋 생성 등을 직접 수행
> * **Unix 철학**: 파이프라인(`|`)을 통해 다른 명령어와 연동 가능 (예: 로그를 `tail`로 읽어 Claude에게 분석 요청)

---

### 1. 준비 사항 (Prerequisites)

Claude Code를 사용하기 위해서는 다음 중 하나가 필요합니다.

* **Claude 구독 계정**: Pro, Max, Teams, 또는 Enterprise 플랜
* **Claude Console 계정**: API 사용을 위한 콘솔 계정

---

### 2️. 설치 방법 (Installation)

운영체제(OS)에 맞는 명령어를 터미널에 입력하여 설치합니다.
*(자동 업데이트가 지원되는 **Native Install** 방식을 권장합니다.)*

#### macOS, Linux, WSL
```bash
curl -fsSL https://claude.ai/install.sh | bash
```
- **Homebrew 사용자**: `brew install --cask claude-code` _(Homebrew 설치 시 자동 업데이트 미지원, `brew upgrade claude-code`로 수동 업데이트 필요)_

#### Windows

**PowerShell**
```
irm https://claude.ai/install.ps1 | iex
```

**CMD**
```
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```
- **WinGet 사용자**: `winget install Anthropic.ClaudeCode` _(WinGet 설치 시 자동 업데이트 미지원, `winget upgrade Anthropic.ClaudeCode`로 수동 업데이트 필요)_

*(참조: 저는 Windows10 환경에서 PowerShell 설치 방식이 동작하지 않아 CMD 방식으로 설치하였습니다.)*

---

### 3. 실행 및 초기 설정 (Get Started)

설치가 완료되었다면 프로젝트 폴더로 이동하여 Claude Code를 실행하세요.

1. **프로젝트 폴더로 이동**
  Bash
```
cd my-project-directory
```

2. **Claude Code 실행**
Bash
```
claude
```

3. **로그인 (최초 1회)**
- 명령어를 실행하면 브라우저가 열리고 로그인 및 권한 승인 절차가 진행됩니다.
- 승인이 완료되면 터미널에서 바로 사용이 가능합니다.

---

## 활용 팁 (Usage Tips)

- **자연어로 명령하기**: "이 프로젝트의 아키텍처를 설명해줘", "메인 페이지의 버튼 색상을 파란색으로 바꿔줘"와 같이 대화하듯 명령하세요.
- **버그 수정**: 에러 메시지를 그대로 복사해 넣거나, "로그인 기능이 작동하지 않아"라고 말하면 Claude가 원인을 분석하고 수정을 제안합니다.
- **자동화**: `claude -p "커밋 메시지 작성해줘"`와 같이 단일 명령으로 특정 작업을 수행할 수 있습니다.

---

## Cursor IDE 설치
* **다운로드**: [Cursor 공식 홈페이지](https://www.cursor.com/) 접속 후 OS에 맞는 버전 다운로드 및 설치.
* **설정**: 기존 VS Code 사용자는 확장 프로그램(Extensions)과 설정을 그대로 가져올 수 있습니다 (Import Settings).

### 모델 설정
* `Ctrl` + `Shift` + `J` (또는 설정)를 눌러 **Settings** 진입.
* **Models** 탭에서 `claude-3-5-sonnet`가 활성화되어 있는지 확인합니다.
* *Tip: Pro 버전을 구독하지 않을 경우, 개인 API Key를 입력하여 사용할 수도 있습니다.*

### MCP (Model Context Protocol) 연결 (선택 사항)
* Cursor 설정 > **Features** > **MCP** 메뉴로 이동.
* `Add New Server`를 클릭하여 로컬 파일이나 외부 도구(Github, Google Drive 등)를 연결할 수 있습니다.

---

## 2. Gemini + Anti-Gravity 시작하기

방대한 문맥(Context) 처리가 필요한 경우 사용합니다.

### 1. API Key 발급
* [Google AI Studio](https://aistudio.google.com/)에 접속하여 구글 계정으로 로그인합니다.
* **Get API key** 버튼을 눌러 Gemini AI 사용을 위한 키를 발급 받습니다.
* 혹은 Google One에서 Gemini Pro 혹은 Gemini Ultra 과금을 결제한 경우 Google AI Studio 계정 대신 Google One 계정을 그대로 사용해도 됩니다.

### 2. 환경 구성 (두 가지 방법 중 선택)

#### 방법 A: Google Anti-Gravity IDE 설치
* [Google AntiGravity](https://antigravity.google/download)에 접속하여 AntiGravity IDE를 설치합니다.
* Google AntiGravity는 현재 무료 Gemini 3.0 Flash 모델 사용을 지원합니다. (향후 과금제로 변경될 수 있습니다.)
* Google One의 Gemini Pro 및 Gemini Ultra 과금 사용자의 경우 Gemini 3.0 Pro 모델 및 기타 Agent의 대량 토큰을 지원합니다.

#### 방법 B: VS Code + Continue 확장 프로그램
* VS Code 확장 마켓플레이스에서 **Continue** 검색 및 설치.
* `config.json` 설정 파일에 위에서 발급받은 Gemini API Key를 입력합니다.
* IDE 내에서 Gemini의 긴 컨텍스트 윈도우를 활용해 코딩할 수 있습니다.

## 2.1 Gemini CLI + IDE (VSCode) 시작하기

 Claude Code 처럼 CLI 환경에서 Gemini AI 모델을 사용할 수 있도록 해줍니다.

### 1. 설치 및 초기 환경 설정
 1. Gemini CLI 설치 (node 및 npm 설치가 되어있어야 합니다.)
 ```bash
 npm install -g @google/gemini-cli
 ```
 2. **프로젝트 폴더로 이동**
  Bash
```
cd my-project-directory
```

2. **Gemini CLI 실행**
Bash
```
gemini
```

3. **Auth 방식 선택**
 Gemini CLI의 토큰 계산 방식에는 총 3가지 방식이 존재하며 각각 과금 방식이 다릅니다.
 
| 사용자 유형 / 시나리오 | 권장 인증 방식 | Google Cloud 프로젝트 필요 여부|
| :--------- | :--------- | :--------- |
| 개인 Google 계정 | Google로 로그인 | 아니요 (단, 예외 있음) |
| 조직 사용자(기업, 학교, Google Workspace 계정) | Google로 로그인 | 예 |
| AI Studio 사용자(Gemini API 키 보유 시) | Gemini API 키 사용 | 아니요 |
| Google Cloud Vertex AI 사용자 | Vertex AI | 예 |
| 헤드리스(Headless) 모드(CI/CD, 서버 등 UI가 없는 환경) | Gemini API 키또는 Vertex AI 사용 | 아니요(Gemini API 키 사용 시)예(Vertex AI 사용 시) |

**(참조: Google AI Studio 사용자의 경우 Gemini CLI 무료 사용이 가능하지만 향후 토큰 제한 및 과금이 발생할 수 있습니다.)**


# 프롬프팅 (Prompting) 작성 방법에 대하여

Claude Code나 Gemini CLI와 같은 AI 기반 코딩 도구를 사용할 때는 단순히 질문을 던지기보다, **AI가 코드 구조를 명확히 파악하고 목적에 맞는 결과를 내놓을 수 있도록 맥락을 설계** 하는 것이 핵심입니다.

효과적인 코딩 프롬프팅을 위한 모범 사례와 구체적인 가이드를 정리해 드립니다.

---

## 1. 프롬프트 구성의 핵심 5요소

좋은 코딩 프롬프트는 아래 5가지 요소를 포함할수록 정확도가 올라갑니다.

|**요소**|**설명**|
|---|---|
|**역할 정의 (Role)**|"너는 시니어 React 개발자야"와 같이 페르소나 설정|
|**명확한 목표 (Goal)**|해결하려는 문제나 구현하려는 기능을 구체적으로 명시|
|**제약 조건 (Constraints)**|라이브러리 버전, 코딩 스타일, 성능 요구사항 등|
|**맥락 제공 (Context)**|현재 프로젝트의 파일 구조, 기존 코드 스니펫 등|
|**출력 형식 (Format)**|코드만 출력할지, 테스트 코드나 설명을 포함할지 정의|

---

## 2. 코딩 프롬프팅 모범 사례

### 1. 구체적인 제약 사항 명시 (Constraints)

"로그인 페이지 만들어줘"라고 하기보다는, 사용하는 스택과 규칙을 미리 정해주는 것이 좋습니다.

> **예시:**
>
> "Next.js 14(App Router)와 Tailwind CSS를 사용해서 로그인 페이지를 만들어줘. 폼 유효성 검사는 `zod`를 사용하고, 디자인은 다크 모드를 기본으로 해줘."

### 2. 단계별 작업 요청 (Chain of Thought)

복잡한 기능은 한 번에 요청하기보다 논리적인 순서로 나누어 요청하세요.

> **예시:**
> 
> 1. "먼저 이 API 응답 데이터를 처리할 TypeScript 인터페이스를 정의해줘."
>
> 2. "정의된 인터페이스를 바탕으로 데이터를 필터링하는 커스텀 훅을 작성해줘."
>
> 3. "그 훅을 사용하는 UI 컴포넌트를 만들어줘."
>

### 3. 에러 해결 시 '맥락' 제공

단순히 "에러 고쳐줘"라고 하면 AI는 추측을 해야 합니다. 에러 메시지와 관련 코드를 함께 전달하세요.

> **예시:**
> 
> "아래 `@tanstack/query` 코드에서 `useQuery` 부분에 'Type undefined is not assignable' 에러가 발생해. 이 에러를 해결하고 왜 발생했는지 짧게 설명해줘. [코드 첨부]"

---

## 3. 실전 프롬프트 템플릿 (예시)

Claude Code나 Gemini CLI에서 바로 활용할 수 있는 구조화된 예시입니다.

```Markdown
### 1. 배경 (Context)
현재 프로젝트는 [Python/FastAPI]를 사용하고 있어. 유저가 업로드한 이미지를 S3에 저장하는 기능을 구현 중이야.

### 2. 목표 (Objective)
이미지 리사이징 기능을 포함한 S3 업로드 유틸리티 함수를 작성해줘.

### 3. 제약 사항 (Constraints)
- 이미지 처리는 `Pillow` 라이브러리를 사용해.
- 가로 길이는 최대 1024px로 고정하고 세로 비율은 유지해야 해.
- 비동기(`async/await`) 방식으로 작성해줘.
- AWS SDK는 `boto3` 대신 `aioboto3`를 사용해.

### 4. 출력 (Output)
- 완성된 함수 코드
- 필요한 라이브러리 설치 명령어
- 사용 예시 코드
```

---

## 4. 유의해야 할 점

- **보안 주의:** 프롬프트에 API 키, 데이터베이스 비밀번호, 개인정보가 포함되지 않도록 절대 주의하세요.
- **버전 확인:** AI는 학습 데이터 시점에 따라 구버전 문법을 제안할 수 있습니다. "최신 버전(v2.0 이상) 기준으로 작성해줘"라고 명시하는 것이 안전합니다.
- **코드 검증:** AI가 생성한 코드는 '그럴듯해 보이지만' 버그가 있을 수 있습니다. 반드시 로컬 환경에서 테스트하고 코드 리뷰를 거쳐야 합니다.
- **CLI 도구의 강점 활용:** Claude Code나 Gemini CLI는 파일 읽기 권한이 있다면 "현재 폴더의 `@auth.ts` 파일을 참고해서 로그아웃 로직을 추가해줘"와 같이 **파일 참조** 기능을 적극적으로 활용하세요.

**Next Step:**
프롬프팅 가이드를 숙지하셨나요? 이제 [[EINZ_notes/Guide/VibeCoding/01_Prompting\|01. 프롬프팅 표준 가이드]] 문서로 이동하여 우리 프로젝트의 구조를 학습해보세요.

---
*Last Updated: 2026-02-04*