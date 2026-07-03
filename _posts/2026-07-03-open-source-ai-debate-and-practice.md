---
layout: post
title: "오픈소스 AI 논란 속에서 실제로 살펴본 오픈웨이트 모델 사용법"
date: 2026-07-03 09:00:00 +0900
categories: ai dev
tags: [opensource, llm, qwen, deepseek, ollama]
---

최근 몇 주간 AI 업계의 가장 뜨거운 화두는 단연 **오픈소스 AI**입니다. Anthropic CEO 다리오 아모데이가 "오픈소스 AI는 위험한 길로 들어서고 있다"고 경고한 데 이어, 그의 발언이 오히려 기존 빅테크의 경쟁 방패라는 비판까지 나왔습니다. 동시에 중국에서 공개된 Qwen, DeepSeek 같은 오픈웨이트 모델들이 급속도로 성능을 높이면서, 실제 현장에서도 이들을 도입하는 사례가 늘고 있습니다.

이 글에서는 이런 논란을 배경으로 하여, **오픈소스 LLM을 로컬에서 직접 실행해보는 방법**을 정리했습니다. 설치부터 API 호출까지 예제를 통해 보여드립니다.

## 왜 오픈소스 AI인가?

오픈소스 AI의 가장 큰 장점은 **투명성**과 **비용**입니다. 모델 구조를 확인할 수 있고, 프라이빗 데이터를 외부 API로 보내지 않아도 됩니다. 또한 Hugging Face나 Ollama를 통해 단 몇 줄 만에 모델을 내려받아 실행할 수 있습니다.

## 실제 코드 예제: Ollama로 Qwen2.5 실행하기

Ollama는 로컬에서 LLM을 쉽게 실행할 수 있는 도구입니다. 아래 명령 하나면 Qwen2.5 3B를 다운로드하고 실행할 수 있습니다.

```bash
# 1. Ollama 설치 (macOS / Linux)
brew install ollama
# Linux의 경우 공식 스크립트 사용:
# curl -fsSL https://ollama.com/install.sh | sh

# 2. 모델 다운로드 및 실행
ollama run qwen2.5:3b

# 3. Python에서 API 호출
import requests

response = requests.post(
    "http://localhost:11434/api/chat",
    json={
        "model": "qwen2.5:3b",
        "messages": [
            {"role": "user", "content": "한국어로 기술 블로그 주제 3가지를 제안해줘."}
        ]
    }
)

print(response.json()["message"]["content"])
```

## DeepSeek-R1과 추론 성능 비교

DeepSeek-R1은 추론 특화 모델로 주목받고 있습니다. 같은 Ollama 환경에서 비교 테스트를 해보면 차이를 느낄 수 있습니다.

```bash
# DeepSeek-R1 1.5B 실행
ollama run deepseek-r1:1.5b

# 간단한 추론 문제 테스트 (Python)
python3 - <<'PY'
import requests
import time

models = ["qwen2.5:3b", "deepseek-r1:1.5b"]
prompt = "369 * 24의 값을 단계별로 계산해줘."

for model in models:
    start = time.time()
    res = requests.post("http://localhost:11434/api/chat", json={
        "model": model,
        "messages": [{"role": "user", "content": prompt}]
    })
    elapsed = round((time.time() - start), 2)
    content = res.json().get("message", {}).get("content", "")
    print(f"[{model}] {elapsed}s")
    print(f"결과 미리보기: {content[:120]}...")
    print("-" * 40)
PY
```

## 마치며

아모데이의 경고가 기업의 안전 마케팅이라는 비판도 있지만, 오픈소스 AI가 **보안, 법적 책임, 편향성** 문제에서 자유롭지 않은 것은 사실입니다. 따라서 "오픈소스면 무조건 안전하다"는 태도보다는, 모델의 라이선스와 한계를 이해한 후 적절히 활용하는 접근이 중요합니다.

실제로 로컬에서 간단한 모델을 돌려보니, 프라이버시가 중요한 내부 업무 도구나 빠른 프로토타이핑에는 오픈웨이트 LLM이 충분한 선택지가 될 수 있겠다는 생각이 들었습니다.

### 참고 링크
- Ollama Docs: https://ollama.com/library
- Qwen2.5 모델카드: https://huggingface.co/Qwen/Qwen2.5-3B
- DeepSeek-R1 논문: https://arxiv.org/abs/2501.12948
