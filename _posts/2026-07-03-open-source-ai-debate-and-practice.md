---
layout: post
title: "오픈소스 AI 논란과 로컬 오픈웨이트 LLM 실행법"
date: 2026-07-03 09:00:00 +0900
categories: ai dev
tags: [opensource, qwen, deepseek, ollama]
---

Anthropic CEO 다리오 아모데이가 "오픈소스 AI가 위험한 길로 들어선다"고 경고한 뒤, 기존 빅테크의 방패라는 비판도 나왔습니다. 이 논란 속에서 Qwen·DeepSeek 같은 오픈웨이트 모델이 빠르게 성능을 높이고 있습니다. 이 글에선 **로컬에서 오픈웨이트 LLM을 직접 실행하는 실전 예제**를 정리합니다.

## 왜 오픈소스 AI인가?

**투명성**과 **비용**이 가장 큰 장점입니다. 구조 검증이 가능하고 외부 API에 프라이버시를 노출하지 않습니다.

## Ollama로 Qwen2.5 실행하기

```bash
# 설치
brew install ollama

# 모델 다운로드 및 실행
ollama run qwen2.5:3b
```

```python
import requests

res = requests.post("http://localhost:11434/api/chat", json={
    "model": "qwen2.5:3b",
    "messages": [{"role": "user", "content": "한국어로 주제 3가지를 제안해줘."}]
})
print(res.json()["message"]["content"])
```

## DeepSeek-R1과 추론 비교

추론 특화 모델인 DeepSeek-R1을 같은 환경에서 테스트합니다.

```bash
ollama run deepseek-r1:1.5b
```

```python
import requests
import time

models = ["qwen2.5:3b", "deepseek-r1:1.5b"]
prompt = "369 * 24를 계산해줘."

for m in models:
    start = time.time()
    res = requests.post("http://localhost:11434/api/chat", json={
        "model": m,
        "messages": [{"role": "user", "content": prompt}]
    })
    print(f"[{m}] {round(time.time()-start,2)}s")
```

## 마치며

오픈소스 AI가 모델의 모든 문제에서 자유롭진 않지만, 로컬 실행이 필요한 경우나 교육·프로토타이핑에는 **실용적인 선택지**입니다. 라이선스와 한계를 이해한 후 활용하는 게 중요합니다.

- Ollama: https://ollama.com/library
- Qwen2.5: https://huggingface.co/Qwen/Qwen2.5-3B
- DeepSeek-R1: https://arxiv.org/abs/2501.12948
