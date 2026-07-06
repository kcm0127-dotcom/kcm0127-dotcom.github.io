---
layout: post
title: "AI 에이전트로 워크플로 통합하기: 오케스트레이션 없는 완전 자동화 파이프라인 구축"
date: 2026-07-06 09:00:00 +0900
categories: [ai, dev]
tags: [ai, agent, automation, workflow]
---

AI가 단순한 채팅을 넘어 실제 업무 흐름을 끝까지 처리하는 형태로 바뀌고 있다. 하지만 아직도 AI를 프로덕션 워크플로에 녹일 때는 복잡한 오케스트레이션 코드가 붙는 경우가 많다.

이번 글에서는 오케스트레이터를 따로 두지 않고, 개별 단계가 스스로 실행·검증·성공 시 다음 단계로 넘어가는 구조를 구현해본다.

## 무엇을 자동화할 것인가

대표적인 시나리오를 하나 골랐다.

1. 특정 GitHub 리포지토리의 새 이슈 목록 조회
2. 각 이슈 텍스트를 AI가 요약·분류
3. 새 파일로 태그 생성
4. 결과를 GitHub Discussion에 보고

이 흐름은 요청 → 조회 → 처리 → 보고까지 단일 파일로 묶을 수 있을 만큼 명확하다.

## 전체 코드

이 예제는 `requests`와 외부 LLM API만 사용한다.

```python
import os
import json
import requests
from datetime import datetime, timezone, timedelta

GITHUB_TOKEN = os.environ["GITHUB_TOKEN"]
MODEL = os.environ.get("AI_MODEL", "gpt-4o")
REPO = os.environ.get("GITHUB_REPO", "kcm0127-dotcom/kcm0127-dotcom.github.io")
HEADERS = {
    "Authorization": f"token {GITHUB_TOKEN}",
    "Accept": "application/vnd.github+json"
}

KST = timezone(timedelta(hours=9))

def github_request(method, path, **kwargs):
    url = f"https://api.github.com/repos/{REPO}{path}"
    r = requests.request(method, url, headers=HEADERS, **kwargs)
    if r.status_code >= 400:
        raise RuntimeError(f"GitHub request failed: {r.status_code} {r.text}")
    return r.json()

def get_open_issues():
    return github_request("GET", "/issues?state=open&per_page=10")

def summarize_issue(issue):
    prompt = (
        "다음 이슈를 읽고 핵심 주제와 해결 방법을 한국어로 3줄 요약하세요.\n"
        f"제목: {issue['title']}\n본문: {issue.get('body','')}"
    )
    payload = {
        "model": MODEL,
        "messages": [
            {"role": "system", "content": "개발 블로그에 올릴 기술 요약 리뷰어"},
            {"role": "user", "content": prompt}
        ]
    }
    r = requests.post("https://api.openai.com/v1/chat/completions",
                      headers={"Authorization": f"Bearer {os.environ['OPENAI_API_KEY']}"},
                      json=payload)
    r.raise_for_status()
    return r.json()["choices"][0]["message"]["content"]

def post_discussion(summary, issue_number):
    body = "### AI 이슈 요약 리포트\n\n" + summary
    discussion = github_request(
        "POST",
        "/discussions",
        json={"title": f"Issue #{issue_number} 자동 요약", "body": body}
    )
    return discussion["html_url"]

def main():
    issues = get_open_issues()
    report = []

    for issue in issues:
        number = issue["number"]
        summary = summarize_issue(issue)
        url = post_discussion(summary, number)
        report.append({"issue": number, "summary": summary, "discussion": url})

    print(json.dumps(report, ensure_ascii=False, indent=2))

if __name__ == "__main__":
    main()
```

관련 코드는 심플하게 plain HTTP로 작성했다. `requests` 정도만 있으면 어디서나 실행 가능하다.

## 실행 방법

```bash
export GITHUB_TOKEN="ghp_***"
export OPENAI_API_KEY="sk-***"
export GITHUB_REPO="user/repo"
python workflow/auto_issue_report.py
```

이 스크립트는 오케스트레이터 클래스를 따로 두지 않았다. 대신 각 단계가 필요할 때 다음 단계를 호출하는 방식으로 자연스럽게 연결된다.

## 실제 적용할 때 주의점

- **토큰 권한**: GitHub 토큰에는 `repo`, `read:discussion` 스코프가 필요하다.
- **대량 실행 제한**: `for` 루프에서 외부 API를 계속 호출하면 rate limit에 걸린다. 토큰 단위로 배치하거나, `backoff`를 두는 것을 권장한다.
- **보안**: 토큰을 코드에 직접 적으면 안 된다. 환경 변수나 Vault를 사용하자.
- **결과 저장**: 매 실행 결과를 JSON으로 저장해두면, 나중에 재실행 없이 이력 추적이 가능하다.

## 마무리

AI 워크플로 자동화는 결국 단순한 파이프라인 모델이 아니다. 단계별로 상태를 확인하고, 실패한 부분만 재시도하는 방식이 가장 안정적이다. 위 코드는 단순하지만 이런 구조의 시작점이 될 수 있다.