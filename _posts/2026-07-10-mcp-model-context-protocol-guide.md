---
layout: post
title: "MCP(Model Context Protocol) 완전 정복: AI 에이전트에 툴을 연결하는 표준 방식"
date: 2026-07-10 09:00:00 +0900
categories: ai dev
tags: [mcp, ai, protocol, fastmcp, automation]
---

2024년 말 Anthropic가 발표한 **Model Context Protocol(MCP)** 는 불과 1년 반 만에 AI 생태계의 사실상 표준이 됐습니다. 2026년 현재 등록된 MCP 서버는 **12,000개 이상**, Python·TypeScript SDK 월 다운로드는 **9,700만 건**을 돌파했고, Cursor·Claude Code·Windsurf·Cline 등 주요 AI 코딩 툴이 모두 MCP 클라이언트를 내장하고 있습니다.

이 글에서는 MCP가 무엇인지, 왜 필요한지, 그리고 **FastMCP로 직접 MCP 서버를 만드는 실전 예제**까지 다룹니다.

## MCP란 무엇인가

MCP는 LLM 애플리케이션이 외부 데이터 소스와 툴에 연결하는 **개방형 표준 프로토콜**입니다. 이전에는 각 AI 툴이 Notion API, GitHub API, 데이터베이스 등을 각기 다른 방식으로 통합해야 했습니다. N개의 AI 클라이언트 × M개의 툴 = N×M개의 통합 코드가 필요했죠.

MCP는 이를 **M+N** 구조로 바꿉니다. 툴 제공자는 MCP 서버를 한 번 만들면, 모든 MCP 호환 AI 클라이언트에서 즉시 사용할 수 있습니다.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  AI Client  │ ←→ │  MCP Client │ ←→ │  MCP Server │ ←→ DB, API, File…
│ (Cursor 등) │     │  (Host 내장) │     │  (툴 제공자) │
└─────────────┘     └─────────────┘     └─────────────┘
```

## 2025-11-25 스펙 업데이트 핵심

2025년 11월, MCP 창시 1주년에 맞춰 새 스펙이 발표됐습니다. 주요 변경점은 다음과 같습니다.

| 기능 | 설명 |
|------|------|
| **Tasks (실험적)** | 오래 걸리는 비동기 작업을 퍼스트클래스로 지원 |
| **OAuth 2.1 강화** | OpenID Connect Discovery, 점진적 동의(scope consent) 지원 |
| **Elicitation** | 서버가 클라이언트에게 추가 입력을 요청하는 양방향 대화 |
| **Structured Tool Output** | 툴 결과를 구조화된 JSON으로 반환 |
| **Tool Icons** | 툴/리소스에 아이콘 메타데이터 부여 |

이중 **Elicitation**이 특히 주목받습니다. 서버가 실행 중 클라이언트에게 "이 작업을 계속할까요?"라고 묻는 상호작용이 표준화된 것입니다.

## 실전: FastMCP로 서버 만들기

Python에서 MCP 서버를 만드는 가장 쉬운 방법은 **FastMCP** 프레임워크를 사용하는 것입니다. 프로토콜의 복잡한 디테일을 숨기고, 비즈니스 로직에만 집중할 수 있습니다.

### 설치

```bash
pip install fastmcp
```

### 최소 서버 예제

간단한 파일 검색 툴을 만들어 보겠습니다.

```python
# search_server.py
from fastmcp import FastMCP
from pathlib import Path

mcp = FastMCP("FileSearchServer")

@mcp.tool
def search_files(directory: str, pattern: str = "*.py") -> list[str]:
    """디렉토리 하위에서 패턴과 일치하는 파일을 검색합니다.

    Args:
        directory: 검색할 디렉토리 경로
        pattern: glob 패턴 (기본값: *.py)

    Returns:
        일치하는 파일 경로 리스트
    """
    base = Path(directory).expanduser()
    if not base.is_dir():
        raise ValueError(f"'{directory}' is not a valid directory")
    return sorted(str(p) for p in base.rglob(pattern))

@mcp.tool
def read_snippet(file_path: str, max_lines: int = 20) -> str:
    """파일의 처음 N줄을 반환합니다."""
    lines = Path(file_path).read_text(encoding="utf-8").splitlines()
    return "\n".join(lines[:max_lines])

@mcp.resource("config://app-info")
def app_info() -> str:
    """서버 메타데이터를 클라이언트에 노출합니다."""
    return "FileSearchServer v1.0 — 로컬 파일 시스템 검색 도구"

if __name__ == "__main__":
    mcp.run(transport="stdio")  # stdio 전송으로 AI 클라이언트와 통신
```

### Claude Code에서 연결하기

서버를 만든 뒤, Claude Code의 MCP 설정 파일에 등록하면 즉시 사용할 수 있습니다.

```json
// ~/.claude/claude_desktop_config.json (또는 .mcp.json)
{
  "mcpServers": {
    "file-search": {
      "command": "python3",
      "args": ["/절대경로/search_server.py"]
    }
  }
}
```

이제 AI 에이전트에게 "내 프로젝트에서 모든 `test_*.py` 파일 찾아줘"라고 말하면, MCP 서버의 `search_files` 툴이 자동으로 호출됩니다.

## MCP 도입 시 주의사항

**1. 보안 — 최소 권한 원칙**  
MCP 서버는 사실상 LLM에게 코드 실행 권한을 주는 것과 같습니다. 프로덕션에서는 OAuth 2.1 스코프를 세분화하고, 파일 시스템 접근을 화이트리스트로 제한하세요.

**2. 구조화된 출력 활용**  
2025-11-25 스펙의 Structured Output을 사용하면 클라이언트가 툴 결과를 안정적으로 파싱할 수 있습니다. 단순 문자열보다 타입이 명확한 딕셔너리를 반환하는 것이 권장됩니다.

```python
@mcp.tool
def get_repo_info(owner: str, repo: str) -> dict:
    """레포지토리 정보를 구조화된 형태로 반환합니다."""
    return {
        "name": repo,
        "owner": owner,
        "stars": 142,
        "language": "Python"
    }
```

**3. 커뮤니티 서버 검증**  
12,000+ 서버 중 신뢰할 수 있는 것은 일부에 불과합니다. 공식 레지스트리(modelcontextprotocol.io)와 Linux Foundation 거버넌스 하에 검증된 서버를 우선 사용하세요.

## 마무리

MCP는 AI 에이전트가 "대화만 하는 봇"에서 "실제 툴을 조작하는 자율 에이전트"로 진화하는 핵심 인프라입니다. 단 몇 줄의 장식자(`@mcp.tool`)만으로 기존 API를 AI 친화적 툴로 변환할 수 있다는 점이 매력적입니다.

오늘 살펴본 패턴은 단순하지만, 이 구조 위에 인증·로깅·레이트리밋을 얹으면 프로덕션급 MCP 서버가 됩니다. 처음이라면 공식 SDK 문서(modelcontextprotocol.io)의 퀵스타트부터, 그리고 직접 `FastMCP`로 작은 서버 하나를 만들어 보는 것을 권장합니다.