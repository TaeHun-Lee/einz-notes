---
{"dg-publish":true,"permalink":"/einz-notes/knowledge/vibe-coding/00-getting-started/","tags":["gardenEntry"]}
---

# 00_GettingStarted: 바이브코딩이란?

> "The hottest new programming language is English." - Andrej Karpathy

**바이브코딩(Vibe Coding)**은 코드를 한 줄 한 줄 직접 타이핑하는 것이 아니라, AI에게 자연어로 의도(Intent)와 흐름(Vibe)을 전달하며 코드를 작성하는 새로운 개발 패러다임을 의미합니다.

개발자는 이제 '작성자(Writer)'가 아닌 **'관리자(Manager)'**이자 **'리뷰어(Reviewer)'**가 되어, AI가 작성한 결과물의 논리를 검증하고 전체적인 아키텍처를 조율하는 데 집중합니다.

---

## 바이브코딩을 위한 3대 요소

바이브코딩을 시작하기 위해서는 다음 3가지 도구의 조합이 필요합니다.

1.  **Brain (LLM)**: 코드를 실제로 생각하고 작성하는 인공지능 모델 (예: Claude 3.5 Sonnet, Gemini 1.5 Pro)
2.  **Body (IDE)**: AI와 개발자가 소통하며 코드를 편집하는 환경 (예: Cursor, VS Code, Windsurf)
3.  **Memory (Context)**: AI에게 프로젝트의 정보를 제공하는 문맥 관리 도구 (예: MCP, RAG, Project Indexing)

---

## 주요 기술 스택 비교: The "Meta" vs The "Challenger"

현재 바이브코딩 커뮤니티에서 가장 주목받는 두 가지 조합의 특징과 장단점을 비교합니다.

### 1️. The "Meta" Stack: Claude + Cursor + MCP
현재 가장 널리 쓰이며, 정교한 코딩 능력에 최적화된 조합입니다.

* **Claude (3.5 Sonnet)**: 현존하는 LLM 중 코딩 로직과 추론 능력이 가장 뛰어납니다. "개떡같이 말해도 찰떡같이 알아듣는" 센스를 가졌습니다.
* **Cursor**: AI Native IDE의 선두주자입니다. `Tab` 키 하나로 코드를 완성하고 수정하는 경험(DX)이 압도적입니다.
* **MCP (Model Context Protocol)**: Anthropic이 만든 표준 프로토콜로, 로컬 파일뿐만 아니라 Google Drive, Slack, Github 등 외부 데이터 소스를 AI에게 안전하게 연결해 줍니다.

| 장점 (Pros)                           | 단점 (Cons)                                           |
| :---------------------------------- | :-------------------------------------------------- |
| **압도적인 코딩 품질**: 복잡한 알고리즘 구현에 유리함    | **Context 제한**: 토큰 제한으로 인해 초대형 프로젝트 전체를 한 번에 넣기 어려움 |
| **강력한 생태계**: 다양한 MCP 서버를 통해 확장성 무한대 | **비용**: API 사용량이 많아질수록 비용 부담이 발생할 수 있음              |
| **사용자 경험(UX)**: Cursor의 직관적인 인터페이스  |                                                     |

### 2️. The "Challenger" Stack: Gemini + Anti-Gravity
구글의 생태계와 방대한 문맥 처리 능력을 앞세운 강력한 조합입니다.

* **Gemini (1.5 Pro/Flash)**: **압도적인 컨텍스트 윈도우(1M~2M 토큰)**를 자랑합니다. 수만 개의 파일이 있는 프로젝트 전체를 "통째로" 읽고 이해할 수 있습니다.
* **Anti-Gravity**: *(참고: Gemini 기반의 고성능 환경을 지칭)* 중력(Gravity)과 같은 기존 개발의 무게(설정, 레거시 파악 등)를 무시하고, 거대한 컨텍스트를 가볍게 띄워 처리한다는 의미의 차세대 환경을 뜻합니다.
* **Multimodal**: 텍스트뿐만 아니라 기획서 이미지, 시연 영상 등을 보고 바로 코드로 변환하는 능력이 탁월합니다.

| 장점 (Pros) | 단점 (Cons) |
| :--- | :--- |
| **Massive Context**: 프로젝트 전체 문서, 로그, 코드를 한 번에 입력 가능 | **디테일 부족**: 로직 생성 시 Claude보다 약간의 "게으름"이 발생할 수 있음 |
| **속도와 비용**: Gemini Flash 모델 사용 시 매우 빠르고 저렴함 | **도구 성숙도**: Cursor에 비해 IDE 통합 경험이 아직 발전 중인 단계 |
| **멀티모달**: UI 스케치만 던져줘도 프론트엔드 코드 생성 가능 | |

---

## 결론: 무엇을 선택해야 할까?

* **정교한 로직과 복잡한 리팩토링**이 필요하다면? 👉 **Claude + Cursor**
* **거대한 레거시 코드 분석**이나 **문서 기반의 빠른 프로토타이핑**이 필요하다면? 👉 **Gemini 기반 스택**

바이브코딩은 도구보다 **"어떻게 질문하느냐(Prompting)"**가 더 중요합니다. 지금 바로 시작해 보세요!

---
*Last Updated: 2026-02-02*