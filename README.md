# remote-control-agent

# 🚀 Universal Coding Agent Web UI & Local Bridge Architecture

본 문서는 사용자의 로컬 컴퓨터에서 실행되는 Google Antigravity, Open Interpreter 등 다양한 코딩 에이전트 프로세스를 안전하게 웹 브라우저 챗 인터페이스와 연결하고 제어하기 위한 범용적인 테크 스택 및 아키텍처 명세서입니다.

---

## 1. 시스템 아키텍처 개요 (System Architecture)

시스템은 UI 파트와 실행 파트를 철저하게 분리하는 **3티어 어댑터 아키텍처(Three-Tier Adapter Architecture)**를 채택하여, 향후 어떤 새로운 코딩 에이전트(CLI/API)가 추가되더라도 프론트엔드 수정 없이 확장할 수 있도록 설계합니다.


[ Frontend Web UI ] (Vite / Next.js 기반 챗 화면)││ (Secure WebSockets / JSON-RPC 2.0 / MCP 통신)▼[ Remote/Cloud Orchestrator ] (사용자 인증 및 에이전트 라우팅 레이어)││ (E2EE 암호화 터널: Cloudflare Tunnels / Ngrok)▼[ Local Bridge Daemon ] (유저 PC 내 백그라운드 서버 - FastAPI / Node.js)││ (OS Process Spawning: stdin / stdout 스트림 캡처)▼[ Coding Agents Runtime ] (Antigravity, Open Interpreter, Aider 등)



---

## 2. 레이어별 상세 테크 스택 (Tech Stack Breakdown)

### A. Frontend Layer (통합 챗 인터페이스)
사용자가 에이전트와 대화하고, 실행 중인 터미널 로그를 실시간으로 확인하며, 생성된 코드나 산출물을 시각적으로 모니터링하는 레이어입니다.

* **핵심 프레임워크:** `Next.js (App Router)` 또는 `Vite + React (TypeScript)`
* **상태 관리 (State Management):** `Zustand` 
  * 에이전트의 실시간 스트리밍 로그 및 챗 메시지 상태를 가볍고 빠르게 관리하기 위함.
* **UI/UX 컴포넌트:** `Shadcn UI` + `TailwindCSS`
  * 코드 블록 하이라이팅, 마크다운 렌더링, 인라인 가상 터미널 구현에 용이.
* **터미널 에뮬레이터 (선택):** `xterm.js`
  * 유저 데스크톱에서 실제로 돌아가는 에이전트의 CLI 출력을 브라우저 상에 리얼타임 개발자 콘솔 형태로 뿌려줄 때 사용.
* **실시간 통신:** `Socket.io-client` 또는 브라우저 네이티브 `WebSocket` API.

### B. Transport & Integration Layer (통신 표준 프로토콜)
웹 UI와 로컬 실행부 간의 통신 규격을 일반화(Generalize)하고 보안을 확보하는 레이어입니다.

* **통신 프로토콜 표준:** **MCP (Model Context Protocol)** 및 **JSON-RPC 2.0**
  * Anthropic이 주도하고 AI 생태계가 채택한 MCP 표준 규격(`read_resource`, `call_tool` 등)을 준수하여 메시지 포맷을 단일화합니다.
* **보안 터널링 (Secure Tunneling):** `Cloudflare Tunnels (cloudflared)` 또는 `Ngrok API`
  * 유저의 로컬 PC 포트를 외부 인터넷에 직접 개방하지 않고, 웹 브라우저 앱과 로컬 브릿지를 E2EE(종단간 암호화) 상태로 안전하게 중계합니다.
* **데이터 포맷 예시 (JSON-RPC):**
  ```json
  {
    "jsonrpc": "2.0",
    "method": "execute_agent",
    "params": { 
      "agent_id": "google-antigravity", 
      "prompt": "src/main.py 파일의 메모리 누수 구간을 리팩토링해줘" 
    },
    "id": "msg_01J2X4B"
  }
  ```

### C. Local Bridge Layer (로컬 데스크톱 가상 어댑터)
사용자 PC 내부 백그라운드에서 가동되며, 웹의 명령을 받아 실제 OS 레벨의 CLI/프로세스를 조율하는 핵심 러너(Runner)입니다.

* **런타임 환경:** `Python (FastAPI / Uvicorn)` 또는 `Node.js (Express / NestJS)`
  * 에이전트 생태계가 Python 중심인 경우 `FastAPI`를, 웹 생태계 친화적 지향 시 `Node.js`를 권장합니다.
* **환경 및 패키지 관리:** `uv` (Python 환경인 경우)
  * 에이전트별로 독립된 가상환경을 매우 빠른 속도로 빌드하고 관리하기 위해 필수적입니다.
* **프로세스 매니저:** Python의 `subprocess` / `asyncio.create_subprocess_exec` 또는 Node.js의 `child_process`
  * Antigravity나 여타 에이전트의 CLI 명령어(예: `antigravity run`)를 백그라운드 자식 프로세스로 실행하고, `stdin`(입력)과 `stdout/stderr`(출력)을 실시간으로 가로채(Hooking) 웹으로 스트리밍하는 핵심 역할을 수행합니다.

---

## 3. 범용성 확보를 위한 핵심 설계 원칙 (Generalization Strategy)

1. **에이전트의 'CLI 가상화' 격리**
   * 웹 인터페이스나 로컬 브릿지 코드가 특정 에이전트의 전용 SDK에 종속되면 안 됩니다.
   * 로컬 브릿지는 모든 코딩 에이전트를 하나의 **"입출력이 가능한 블랙박스 CLI 프로그램"**으로 취급해야 합니다.
2. **출력 어댑터 패턴 (Output Adapter Pattern) 적용**
   * 에이전트들마다 로그를 출력하는 형태(텍스트, JSON, 마크다운 믹스 등)가 다릅니다.
   * 로컬 브릿지 단에서 각 에이전트별 정규식(Regex) 파서를 플러그인 형태로 구성하여, 프론트엔드로 쏠 때는 항상 일관된 스키마(`{"status": "executing_code", "data": "..."}`)로 정제해 송신합니다.
3. **보안 가드레일 (Security Guardrails)**
   * 유저 로컬 권한을 웹에 넘겨주는 구조이므로 웹 브라우저에서 보낸 모든 요청 핸들샤이닝 단계에서 암호화 토큰 키 검증이 의무화되어야 합니다.

---

## 4. 벤치마킹 및 참고 오픈소스 (Prior Arts & References)

프로젝트 빌드 시 아래의 검증된 오픈소스 구조와 소스코드를 참고하여 구현 속도를 높일 수 있습니다.

* **Model Context Protocol (MCP):** AI와 로컬 도구 간 연결 표준 [GitHub](https://github.com)
* **OpenChamber (OpenCode UI):** 로컬 인프라와 외부 웹/VS Code를 릴레이 터널로 엮어 원격 제어하는 에이전트 인터페이스 프로젝트 [GitHub](https://github.com)
* **Open Multi-Agent Canvas:** Next.js와 LangGraph 기반으로 다중 에이전트를 대시보드 형태로 통제하는 오픈소스 [GitHub](https://github.com)
* **Open WebUI:** 로컬에 구현된 다양한 MCP 서버 및 에이전트 런타임을 동적으로 주입받아 채팅창으로 제어하는 인터페이스 [GitHub](https://github.com)
