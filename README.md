<p align="center">
  <img src="assets/memo.png" alt="Memo" width="180" />
</p>

# Hermes Agent OpenAI Codex Provider `NoneType` 오류 분석 및 해결 기록

이 문서는 Hermes Agent에서 provider/model을 **OpenAI Codex**로 선택했을 때 발생한 다음 오류를 분석하고, 원인과 해결 방법을 정리한 공개 기록입니다.

```text
TypeError: 'NoneType' object is not iterable
```

## 1. 상황 요약

Hermes Agent에서 기존에는 OpenAI Codex provider를 사용하고 있었습니다. 이후 Google AI Studio/Gemini provider로 변경했다가, 다시 OpenAI Codex provider/model을 사용하려고 했을 때 다음과 같은 문제가 발생했습니다.

- OpenAI Codex provider 선택 시 응답 생성이 실패함
- Hermes Agent가 정상적으로 답변하지 못함
- 내부 로그에서 `TypeError: 'NoneType' object is not iterable` 오류가 확인됨
- Gemini provider에서는 동일 오류가 발생하지 않음

문제는 Hermes Agent의 일반 설정 문제가 아니라, **OpenAI Codex API 응답 형식과 OpenAI Python SDK의 파싱 로직 사이의 불일치** 때문에 발생했습니다.

---

## 2. 발생한 오류

대표적인 오류는 다음과 같습니다.

```text
TypeError: 'NoneType' object is not iterable
```

오류가 발생한 위치는 OpenAI Python SDK 내부 파일이었습니다.

```text
openai/lib/_parsing/_responses.py
```

문제가 된 코드는 개념적으로 다음과 같은 형태였습니다.

```python
for output in response.output:
    ...
```

여기서 `response.output`이 리스트라고 가정하고 반복문을 실행합니다. 하지만 실제 Codex API 응답에서 `response.output` 값이 `null`, 즉 Python 기준으로는 `None`으로 들어오면서 문제가 발생했습니다.

Python에서는 `None`을 반복할 수 없기 때문에 다음 오류가 발생합니다.

```text
TypeError: 'NoneType' object is not iterable
```

---

## 3. 왜 발생했는가?

핵심 원인은 다음과 같습니다.

> OpenAI Codex API가 특정 응답에서 `output` 필드를 빈 배열 `[]`이 아니라 `null`로 반환했고, OpenAI Python SDK의 파싱 로직은 이 값을 반복 가능한 리스트로 가정하고 처리했다.

즉, SDK 내부 로직은 다음과 같은 응답을 기대했습니다.

```json
{
  "output": []
}
```

하지만 실제로는 다음과 같은 응답이 들어왔습니다.

```json
{
  "output": null
}
```

이 차이 때문에 SDK 내부에서 다음 코드가 실패했습니다.

```python
for output in response.output:
    ...
```

`response.output`이 `None`이면 위 코드는 다음과 동일합니다.

```python
for output in None:
    ...
```

따라서 Python 런타임은 `NoneType`은 반복할 수 없다고 판단해 오류를 발생시킵니다.

---

## 4. 어떻게 발생했는가? 흐름 정리

문제 발생 흐름은 다음과 같습니다.

1. Hermes Agent에서 provider/model을 OpenAI Codex로 설정함
2. Hermes Agent가 Codex endpoint로 요청을 보냄

   ```text
   https://chatgpt.com/backend-api/codex
   ```

3. Codex API가 응답을 반환함
4. 응답 객체의 `output` 필드가 빈 리스트가 아니라 `null`로 들어옴
5. OpenAI Python SDK가 응답을 파싱하면서 `response.output`을 반복하려고 함
6. `response.output`이 `None`이므로 반복문에서 `TypeError` 발생
7. Hermes Agent의 응답 생성이 중단됨

정리하면, **Hermes Agent → OpenAI Codex API → OpenAI SDK 파서**로 이어지는 처리 과정 중 SDK 파서 단계에서 오류가 발생한 것입니다.

---

## 5. Gemini에서는 왜 문제가 없었는가?

Google Gemini provider에서는 동일한 오류가 발생하지 않았습니다.

이유는 Gemini provider가 OpenAI Codex provider와 다른 API 경로 및 다른 응답 처리 방식을 사용하기 때문입니다.

즉, Gemini는 문제가 된 OpenAI SDK의 Codex 응답 파싱 로직을 타지 않았습니다.

따라서 Gemini provider 자체가 문제를 해결한 것은 아니고, 단지 **문제가 있는 코드 경로를 지나지 않았기 때문에 오류가 발생하지 않은 것**입니다.

---

## 6. 해결 방법

해결은 OpenAI Python SDK 내부 파싱 로직이 `response.output`이 `None`인 경우에도 안전하게 동작하도록 수정하는 방식으로 진행했습니다.

기존 코드는 다음과 같은 형태였습니다.

```python
for output in response.output:
    ...
```

이를 다음과 같이 수정했습니다.

```python
for output in (response.output or []):
    ...
```

이 변경의 의미는 다음과 같습니다.

- `response.output`이 정상적인 리스트이면 그대로 반복
- `response.output`이 `None`이면 빈 리스트 `[]`로 대체
- 결과적으로 `NoneType` 반복 오류를 방지

즉, `null` 응답이 들어와도 파서가 실패하지 않고 안전하게 넘어가도록 방어 코드를 추가한 것입니다.

---

## 7. 실제 패치 위치

이번 문제에서는 Hermes Agent가 사용하는 Python 가상환경 안의 OpenAI SDK 파일을 직접 수정했습니다.

예시 경로:

```text
/opt/hermes/.venv/lib/python3.13/site-packages/openai/lib/_parsing/_responses.py
```

수정 전:

```python
for output in response.output:
```

수정 후:

```python
for output in (response.output or []):
```

---

## 8. 패치 후 검증 결과

패치 후 실제 Hermes Agent의 Codex provider 환경에서 테스트를 진행했습니다.

테스트 내용:

```text
안녕?
```

정상 응답 예시:

```text
안녕하세요! 무엇을 도와드릴까요?
```

이를 통해 다음을 확인했습니다.

- OpenAI Codex provider 호출이 다시 정상 작동함
- `TypeError: 'NoneType' object is not iterable` 오류가 재발하지 않음
- Gemini provider로 우회하지 않고도 Codex provider를 다시 사용할 수 있음

---

## 9. Hermes Agent 설정 복구

패치 후 Hermes Agent의 기본 provider/model을 다시 OpenAI Codex로 되돌렸습니다.

사용한 설정 값은 다음과 같습니다.

```yaml
model:
  provider: openai-codex
  default: gpt-5.5
  base_url: https://chatgpt.com/backend-api/codex
```

CLI로 설정할 경우 예시는 다음과 같습니다.

```bash
hermes config set model.provider openai-codex
hermes config set model.default gpt-5.5
hermes config set model.base_url https://chatgpt.com/backend-api/codex
```

설정 변경 후에는 기존 세션이 이전 provider/model 정보를 들고 있을 수 있으므로, Slack/게이트웨이 환경에서는 새 세션을 시작하는 것이 좋습니다.

```text
/reset
```

또는 현재 세션에서 모델만 바꾸고 싶다면 다음 명령을 사용할 수 있습니다.

```text
/model gpt-5.5
```

---

## 10. 근본적인 해결 방향

이번 조치는 운영 중인 환경에서 빠르게 문제를 복구하기 위한 로컬 패치입니다.

장기적으로는 다음 중 하나가 필요합니다.

1. OpenAI Python SDK에서 `response.output is None`인 경우를 안전하게 처리하도록 upstream 수정
2. Hermes Agent의 Codex runtime 쪽에서 응답 파싱 전 `output` 값을 정규화
3. Codex API 응답이 항상 `output: []` 형태를 반환하도록 API 측 수정

가장 안전한 방어 코드는 다음 형태입니다.

```python
for output in (response.output or []):
    ...
```

이 방식은 `response.output`이 다음 중 어떤 값이어도 안전하게 동작합니다.

- 정상 리스트
- 빈 리스트
- `None`

---

## 11. 결론

이번 오류는 사용자의 provider 설정 실수나 인증 문제라기보다는, OpenAI Codex API 응답의 `output: null` 케이스를 OpenAI Python SDK가 안전하게 처리하지 못해 발생한 문제였습니다.

핵심 정리는 다음과 같습니다.

| 항목 | 내용 |
|---|---|
| 발생 위치 | OpenAI Python SDK 응답 파싱 로직 |
| 직접 원인 | `response.output`이 `None`인데 반복문으로 처리함 |
| 오류 메시지 | `TypeError: 'NoneType' object is not iterable` |
| Gemini에서 미발생한 이유 | 다른 provider 경로를 사용하여 해당 SDK 파서를 타지 않음 |
| 해결 방법 | `response.output or []` 형태로 방어 코드 추가 |
| 결과 | OpenAI Codex provider 정상 복구 |

최종적으로, OpenAI Codex provider를 계속 사용하려면 SDK 또는 Hermes Agent 내부에서 `output: null` 응답을 안전하게 처리하도록 방어 로직을 추가하는 것이 필요합니다.

---

## 12. 참고: 최소 패치 예시

```diff
- for output in response.output:
+ for output in (response.output or []):
```

이 한 줄 변경으로 `response.output`이 `None`인 경우에도 빈 리스트처럼 처리되어 오류를 방지할 수 있습니다.
