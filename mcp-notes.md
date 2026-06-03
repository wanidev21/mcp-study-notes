# MCP(Model Context Protocol) 학습 노트

> 작성: wanidev21 · 2026-06-03
> Ubuntu 환경에서 Python으로 MCP 서버를 구현하고 GitHub 파일 업로드를 자동화한 과정을 정리한 문서

---

## 1. MCP란?

Anthropic이 2024년 11월 공개한 오픈 프로토콜.
**AI 모델이 외부 도구·데이터·서비스에 연결되는 방식을 표준화한 규격**이다.

이전에는 AI 앱 M개 × 도구 N개 = **M×N개**의 맞춤 연동 코드가 필요했다.
MCP는 연결 규격을 하나로 통일해 **M+N**으로 줄인다.
도구를 MCP 서버로 한 번만 만들면 어떤 MCP 호환 AI든 바로 활용할 수 있다.

---

## 2. 핵심 구조

```mermaid
graph LR
    subgraph Host["Host (Claude Code)"]
        AI["AI 모델"]
        Client["Client"]
    end
    Client -- "① MCP 규격\nJSON-RPC 2.0" --> Server["my-first-server\n(MCP 서버)"]
    Server -- "② GitHub REST API" --> GitHub[("GitHub")]
```

| 역할 | 위치 | 하는 일 |
|------|------|---------|
| **Host** | 사용자가 사용하는 앱 | Claude Code, Cursor 등. AI 모델과 Client를 포함 |
| **Client** | Host 안 | 하나의 MCP 서버와 1:1 연결 관리. Host에 내장되어 있어 별도 구현 불필요 |
| **Server** | 외부 프로그램 | 기존 서비스/API를 MCP 형식으로 감싸는 변환기. 개발자가 구현하는 영역 |

**통신은 두 다리로 나뉜다**
- `①` Client ↔ MCP 서버: **MCP 규격(JSON-RPC 2.0)** — 표준화된 구간
- `②` MCP 서버 ↔ 외부 서비스: **각 서비스의 원래 API** — 서버가 통역하는 구간

### Server가 제공하는 것 (Primitives)

| 종류 | 통제 주체 | 설명 |
|------|----------|------|
| **Tools** | AI 모델 | AI가 직접 호출하는 실행 함수 |
| **Resources** | Host 앱 | 읽기 전용 데이터 (파일, DB 조회 등) |
| **Prompts** | 사용자 | 재사용 가능한 프롬프트 템플릿 |

---

## 3. MCP 서버 구현

### 전체 요청 흐름

파일 업로드를 요청하면 실제로 다음과 같이 처리된다.

```mermaid
sequenceDiagram
    actor User as 사용자
    participant CC as Claude Code
    participant CL as Client
    participant SV as my-first-server
    participant GH as GitHub API

    User->>CC: "mcp-notes.md 파일 올려줘"
    CC->>CL: upload_to_github 도구 호출 결정
    CL->>SV: JSON-RPC tools/call
    Note over CL,SV: ① 왼쪽 다리 (MCP 규격)
    SV->>GH: PUT /repos/wanidev21/mcp-study-notes/contents/mcp-notes.md
    Note over SV,GH: ② 오른쪽 다리 (GitHub REST API)
    GH-->>SV: 201 Created + 파일 URL
    SV-->>CL: "성공: https://github.com/..."
    CL-->>CC: 결과 반환
    CC-->>User: "성공적으로 업로드했어요"
```

AI가 도구를 호출할 때 Claude Code 화면에 `mcp__서버이름__도구이름` 형태로 표시된다.
예: `mcp__my-first-server__upload_to_github`

---

### 3-1. 환경 설정

```bash
mkdir ~/mcp-notes && cd ~/mcp-notes

python3 -m venv .venv       # 가상환경: 패키지가 시스템에 영향 주지 않도록 격리
source .venv/bin/activate   # 가상환경 활성화

pip install mcp             # MCP Python SDK (FastMCP 포함)
pip install httpx           # GitHub API 호출용 HTTP 클라이언트
```

---

### 3-2. 서버 코드 (server.py)

```python
import os, base64, httpx
from mcp.server.fastmcp import FastMCP

# ─────────────────────────────────────────────
# 서버 선언
# FastMCP: JSON-RPC 처리를 모두 담당하는 공식 SDK
# 괄호 안 이름이 Claude Code에서 서버를 식별하는 이름이 된다
# ─────────────────────────────────────────────
mcp = FastMCP("my-first-server")


# ─────────────────────────────────────────────
# 도구 선언: @mcp.tool() 데코레이터
# 이 데코레이터 하나로 일반 Python 함수가 AI가 호출할 수 있는 MCP Tool이 된다
# docstring → AI가 이 도구를 언제, 어떻게 써야 하는지 판단하는 근거
# ─────────────────────────────────────────────
@mcp.tool()
def add(a: int, b: int) -> int:
    """두 숫자를 더한다"""
    return a + b


@mcp.tool()
def reverse_text(text: str) -> str:
    """글자를 거꾸로 뒤집는다"""
    return text[::-1]


@mcp.tool()
def upload_to_github(path: str, content: str, message: str = "Update via MCP") -> str:
    """GitHub 레포에 파일을 올리거나 수정한다. path=레포 안 파일경로, content=파일내용"""

    # 환경변수에서 인증 정보를 읽는다 (코드에 직접 작성 시 노출 위험)
    token = os.environ.get("GITHUB_TOKEN")
    repo  = os.environ.get("GITHUB_REPO")   # 형식: "사용자명/레포명"
    if not token or not repo:
        return "GITHUB_TOKEN / GITHUB_REPO 환경변수가 설정되지 않았습니다."

    url = f"https://api.github.com/repos/{repo}/contents/{path}"
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/vnd.github+json"
    }

    # 파일이 이미 존재하면 수정(Update), 없으면 생성(Create)
    # GitHub API는 파일 수정 시 현재 파일의 sha 값을 반드시 요구한다
    sha = None
    r = httpx.get(url, headers=headers)
    if r.status_code == 200:
        sha = r.json()["sha"]

    # GitHub API는 파일 내용을 base64로 인코딩하여 전송한다
    body = {
        "message": message,
        "content": base64.b64encode(content.encode()).decode()
    }
    if sha:
        body["sha"] = sha

    resp = httpx.put(url, headers=headers, json=body)
    if resp.status_code in (200, 201):
        return f"성공: {resp.json()['content']['html_url']}"
    return f"실패({resp.status_code}): {resp.text[:200]}"


# ─────────────────────────────────────────────
# 서버 실행
# mcp.run() 기본값: stdio transport
# → Claude Code가 이 프로세스를 직접 실행하고 stdin/stdout으로 통신
# 주의: 도구 함수 안에서 print() 사용 금지 (stdout이 MCP 통신 채널이라 충돌)
#       로그 출력이 필요하면 반드시 sys.stderr 사용
# ─────────────────────────────────────────────
if __name__ == "__main__":
    mcp.run()
```

---

### 3-3. Claude Code에 서버 등록

```bash
claude mcp add-json my-first-server '{
  "type":    "stdio",
  "command": "/root/mcp-notes/.venv/bin/python",
  "args":    ["/root/mcp-notes/server.py"],
  "env": {
    "GITHUB_TOKEN": "github_pat_...",
    "GITHUB_REPO":  "wanidev21/mcp-study-notes"
  }
}'
```

각 필드의 역할:

| 필드 | 값 | 의미 |
|------|-----|------|
| `type` | `"stdio"` | 같은 PC에서 프로세스로 실행하는 방식 |
| `command` | 가상환경의 python 경로 | 가상환경의 python을 사용해야 mcp 패키지를 인식한다 |
| `args` | server.py 절대 경로 | 실행할 서버 파일 |
| `env` | 토큰, 레포명 | 서버 프로세스에 환경변수로 주입. `~/.claude.json`에 저장 |

설정은 `~/.claude.json`에 저장된다. `claude mcp list`로 연결 상태를 확인한다.

```bash
claude mcp list
# → my-first-server: /root/... - ✓ Connected
```

등록 직후 Claude Code가 서버에 `tools/list`를 호출해 도구 목록을 자동으로 가져온다.
이것이 Discovery(발견) 과정이며, 이 시점에 `add`, `reverse_text`, `upload_to_github` 세 도구가 등록된다.

---

### 3-4. GitHub 토큰(PAT) 발급

경로: GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens

필요한 권한:
- Repository access: **Only select repositories** → 해당 레포만 선택
- Permissions → Contents: **Read and write**

보안 주의:
- 토큰을 `server.py`에 직접 작성하지 않는다 (레포에 올리면 GitHub이 자동으로 무효화)
- `claude mcp add-json`의 `env` 필드로 전달하면 `~/.claude.json`에만 저장된다
- 토큰이 노출되면 즉시 Revoke 후 재발급

---

## 4. 직접 구현한 서버를 통해 본 MCP의 본질

### 공식 GitHub MCP 서버와의 비교

MCP 서버는 누구나 직접 만들 수 있으며, 공식으로 배포된 서버와 구조가 동일하다.

| | 직접 구현한 서버 | 공식 GitHub MCP 서버 |
|---|---|---|
| 도구 수 | 3개 | 수십 개 |
| 주요 기능 | 파일 올리기, 숫자 계산, 글자 뒤집기 | PR, 이슈, 브랜치, 코드 검색 등 |
| 구현 방식 | Python | Go (바이너리) |
| 유지보수 | 직접 관리 | GitHub 공식 관리 |
| **MCP 구조** | **동일** | **동일** |

두 서버의 차이는 제공하는 도구의 수와 범위뿐이다.
왼쪽 다리(AI ↔ MCP 서버)와 오른쪽 다리(MCP 서버 ↔ GitHub API)의 구조는 완전히 같다.

---

### 서버 구성 요소별 역할

**① 도구 등록 — `@mcp.tool()`**

함수 위에 붙이는 이 표시가 AI에게 "이 기능을 사용할 수 있다"고 알리는 것이다.
이 표시가 없으면 AI는 해당 함수의 존재를 전혀 알 수 없다.
MCP에서는 이렇게 등록된 기능 하나하나를 **도구(Tool)** 라고 한다.

**② AI의 도구 선택 기준 — 설명 한 줄**

> "GitHub 레포에 파일을 올리거나 수정한다"

AI는 함수 내부 코드를 볼 수 없다. 오직 이 설명만 보고 지금 사용자 요청에 어떤 도구가 필요한지 판단한다.
설명이 구체적일수록 AI가 도구를 더 정확하게 활용한다.

**③ 도구에 필요한 정보 — 입력값 목록**

도구를 사용할 때 AI가 직접 채워야 하는 정보들이다.

- `path` : 저장할 파일 경로 (예: `mcp-notes.md`)
- `content` : 파일에 담길 내용
- `message` : 커밋 메시지 (예: `Add mcp-notes.md via MCP`)

사용자가 "파일 올려줘"라고 하면 AI가 이 말을 분석해 세 항목을 스스로 채운 뒤 도구를 호출한다.

**④ 실제 작업 실행 — 외부 서비스 요청**

AI가 도구를 호출하면 서버가 GitHub에 실제로 요청을 보낸다.
- 파일이 이미 존재하면 → 수정 요청
- 존재하지 않으면 → 새로 생성 요청

이 구간이 오른쪽 다리에 해당한다. AI는 이 과정을 알지 못하며 결과만 전달받는다.

**⑤ 결과 반환**

작업이 완료되면 결과를 AI에게 돌려준다.

> "성공: https://github.com/..."

AI는 이 결과를 받아 사용자에게 최종 답변을 전달한다.
Claude Code 화면에 표시되는 메시지가 여기서 만들어지는 것이다.

**⑥ 서버 실행**

`mcp.run()`이 서버를 실제로 시작하는 부분이다.
Claude Code에 등록해두면 연결 시 자동으로 서버가 켜지고, 같은 컴퓨터 안에서 AI와 통신을 시작한다.

**⑦ 인증 정보 분리 보관**

GitHub 토큰 같은 민감한 정보는 코드에 직접 작성하지 않고, 서버 실행 시점에 외부에서 주입받는다.
코드를 공개 저장소에 올려도 인증 정보가 노출되지 않는 이유가 여기에 있다.

**⑧ 도구 목록 자동 제공 — Discovery**

서버가 연결되면 AI 측에서 자동으로 "이 서버가 어떤 도구를 제공하는지" 목록을 요청한다.
서버가 목록을 전달하면 그때부터 AI가 해당 도구들을 활용할 수 있다.
새 도구를 추가한 뒤 서버를 재시작하면 별도 설정 없이 AI가 즉시 인식한다.

---

### 정리

| 구성 요소 | 역할 |
|---|---|
| 도구 등록 표시 | AI에게 "이 기능 사용 가능"을 알림 |
| 설명 한 줄 | AI가 도구를 선택하는 기준 |
| 입력값 목록 | AI가 스스로 채워야 하는 정보 |
| 외부 서비스 요청 | 오른쪽 다리 — 실제 작업 수행 |
| 결과 반환 | AI가 사용자에게 전달하는 내용 |
| 인증 정보 분리 | 민감 정보를 코드에서 분리하는 방법 |
| 도구 목록 자동 제공 | 연결 시 도구가 AI에 자동 등록됨 |

MCP 서버의 규모(도구 수)가 크든 작든 이 구성은 동일하다.
서버를 직접 구현해보면 어떤 MCP 서버를 접하더라도 구조를 파악할 수 있다.

---

## 5. 핵심 요약

```
MCP 서버 = 기존 API를 MCP 형식으로 감싼 변환 프로그램

@mcp.tool() 데코레이터 → 일반 Python 함수가 AI 호출 가능 도구로 변환

왼쪽 다리 (Client ↔ 서버): MCP 규격으로 표준화 → 어떤 AI든 연결 가능
오른쪽 다리 (서버 ↔ 외부): 각 서비스의 원래 API → 서버가 통역

개발자가 구현하는 것: 서버(도구)
Host에 내장된 것: Client (Claude Code 등이 기본 제공)
```

---

*구현 레포: [wanidev21/mcp-study-notes](https://github.com/wanidev21/mcp-study-notes)*
