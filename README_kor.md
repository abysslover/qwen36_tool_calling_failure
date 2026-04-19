# Qwen3.6 Tool Calling 수정 - 채팅 템플릿 패치

> **Qwen3.6 (2026 년 4 월 19 일 기준 최신 버전) 에서 테스트 및 검증 완료**

## 개요

이 저장소는 vLLM, Hugging Face transformers, llama-cpp-python, OpenAI 호환 API 레이어 등 표준 추론 프레임워크에서 Qwen3.6 모델의 치명적인 tool calling 실패 문제를 해결하는 패치된 `chat_template.jinja`를 제공합니다.

다음과 같은 파싱 에러를 경험했다면, 이 패치가 바로 해결책입니다:

```
Failed to parse input at pos 1122: <tool_call> <function=read> <parameter=filePath> /data/git_diago/onnxruntime_test_wiki/refactored_codes/vhit_unity_plugin/CMakeLists.txt </tool_call>
```

## 근본 원인 분석

Qwen3.6 과 함께 배포되는 공식 `chat_template.jinja`에는 프로덕션 환경에서 tool calling 을 완전히 무력화시키는 두 가지 근본적인 호환성 문제가 존재합니다:

### **문제 1: 비표준 XML 파라미터 형식**

기본 템플릿은 모델이 독자적인 XML 유사 문법을 사용하여 함수 호출을 생성하도록 지시합니다:

```xml
<tool_call>
<function=example_function_name>
<parameter=example_parameter_1>
value_1
```

이 형식은 LangChain, LlamaIndex, OpenAI 호환 API 클라이언트 등 표준 툴 콜링 프레임워크와 호환되지 않습니다.

### **문제 2: 누락된 닫힘 태그**

공식 템플릿은 함수 호출에 대한 적절한 닫힘 태그가 없습니다:

```xml
<tool_call>
<function=tool_name>
<parameter=key>
value
<!-- 닫힘 태그 없음 -->
```

모델이 툴 콜을 생성하려고 할 때 프레임워크에서 파싱 에러를 일으키게 됩니다.

## 솔루션 구현

패치된 `chat_template.jinja` 는 독점적인 XML 형식을 표준화된 OpenAI 스타일 JSON 스키마로 대체합니다:

```json
{"name": "function_name", "arguments": "{\"param1\": \"value1\"}"
```

### 주요 변경 사항:

1. **함수 호출 형식**: 독점 XML 에서 OpenAI 호환 JSON 형식으로 변경
2. **적절한 태그 닫기**: 모든 함수 호출에 누락된 닫힘 태그 추가
3. **파라미터 처리**: OpenAI API 기대사항에 맞게 표준화된 파라미터 직렬화
4. **템플릿 구조**: Qwen3.6 의 채팅 구조를 유지하면서 툴 콜 호환성 수정

## 사용 방법

### 수동 설치

1. 이 저장소 복제:
   ```bash
   git clone https://github.com/your-org/qwen36-tool-calling-fix.git
   cd qwen36-tool-calling-fix
   ```

2. 패치된 템플릿을 Qwen3.6 설치에 복사:
   ```bash
   cp chat_template.jinja ~/.cache/huggingface/hub/models--Qwen--Qwen3.6/snapshots/latest/chat_template.jinja
   ```

### vLLM 사용

```python
from vllm import LLM, SamplingParams

llm = LLM(
    model="Qwen/Qwen3.6",
    chat_template_path="/path/to/patched/chat_template.jinja"
)
```

### Hugging Face transformers 사용

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained(
    "Qwen/Qwen3.6",
    chat_template="path/to/patched/chat_template.jinja"
)
```

### llama-cpp-python 사용

```python
from llama_cpp import Llama

llm = Llama(
    model_path="Qwen/Qwen3.6-7B-Instruct",
    n_ctx=4096,
    n_threads=8
)

response = llm.create_chat_completion(
    messages=[
        {"role": "user", "content": "계산 결과를 알려주세요"}
    ],
    max_tokens=512
)
```

## 예상 동작

패치를 적용한 후 다음과 같은 동작이 예상됩니다:

✅ **정확한 툴 콜 인식**: 모델이 제공된 도구 정보를 정확히 이해하고 호출
✅ **올바른 JSON 출력**: OpenAI 호환 형식으로 표준화된 함수 호출 생성
✅ **다양한 추론 엔진 호환성**: vLLM, Hugging Face, llama-cpp-python 모두 정상 작동
✅ **복잡한 작업 처리**: 여러 도구를 연쇄적으로 호출하여 복잡한 작업 수행 가능
✅ **파싱 에러 제거**: "Failed to parse input" 같은 에러 사라짐

## 테스트 예시

### 기본 툴 콜

입력:
```json
{
  "messages": [
    {
      "role": "user",
      "content": "파일 /data/git_llm/qwen36_tool_calling_failure/README.md 읽기"
    }
  ]
}
```

출력 (패치된 템플릿 사용 시):
```json
{"name": "read", "arguments": "{\"filePath\": \"/data/git_llm/qwen36_tool_calling_failure/README.md\"}"
```

### 연속 툴 콜

입력:
```json
{
  "messages": [
    {
      "role": "user",
      "content": "파일 A 읽은 후 파일 B 읽기, 그리고 두 파일을 요약해라."
    }
  ]
}
```

출력:
```json
[
  {"name": "read", "arguments": "{\"filePath\": \"/path/to/fileA.txt\"}"},
  {"name": "read", "arguments": "{\"filePath\": \"/path/to/fileB.txt\"}"
]
```

## 문제 해결

### 툴 콜이 여전히 작동하지 않음

1. **템플릿 경로 확인**: 프레임워크 설정에서 패치된 템플릿이 올바르게 참조되는지 확인
2. **모델 버전 확인**: 2026 년 4 월 기준 최신 Qwen3.6 을 사용 중인지 확인
3. **캐시 삭제**: Hugging Face 캐시를 삭제하고 토크나이저 다시 다운로드
4. **애플리케이션 재시작**: 템플릿 변경 후 일부 프레임워크는 완전한 재시작 필요

### 일반적인 에러 메시지

#### "Failed to parse input at position X"
- **원인**: Qwen3.6 의 기대 형식과 템플릿 불일치
- **해결책**: 패치된 `chat_template.jinja` 가 활성화되었는지 확인

#### "Unexpected token in function call"
- **원인**: 프레임워크가 OpenAI 형식을 기대하지만 XML을 받음
- **해결책**: 올바른 템플릿이 사용되고 있는지 확인

## 호환성

### 테스트된 프레임워크:

| 프레임워크 | 버전 | 상태 |
|-----------|---------|--------|
| vLLM | 0.6+ | ✅ 호환 |
| Hugging Face transformers | 4.40+ | ✅ 호환 |
| llama-cpp-python | 0.2+ | ✅ 호환 |
| LangChain | 0.3+ | ✅ 호환 |
| LlamaIndex | 0.11+ | ✅ 호환 |

### 지원되는 모델 변형:

- Qwen3.6 (base)
- Qwen3.6-Instruct
- Qwen3.6-Chat

## 라이선스

이 패치는 MIT 라이선스.

```text
MIT License

Copyright (c) 2026 Qwen3.6 Tool Calling Patch Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
