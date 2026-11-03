# API 명세

FE / BE / AI 간 계약의 단일 기준. API 변경 전 이 문서 먼저 수정 필요.

상태: **초안 (2026-11-03)**. BE·AI 담당 확인 전. 확정되면 이 줄을 "확정"으로 바꾸고 날짜를 적음.

## 통신 구조

```
FE(웹·안드로이드) ──[FE↔BE API]──> BE ──[BE↔AI API]──> AI ──> LLM(Ollama / OpenAI)
```

- FE는 BE만 호출함. AI를 직접 호출하지 않음
- LLM 키는 BE(와 AI)에만 둠. 앱에는 넣지 않음
- AI는 상태 없는 서비스. 저장은 전부 BE가 함
- 문제는 업로드 시점에 미리 생성해 기기에 내려받음. 카드가 뜨는 순간에는 서버를 호출하지 않음

## 공통 규칙

| 항목 | 내용 |
|---|---|
| 형식 | JSON (UTF-8). 파일 업로드만 `multipart/form-data` |
| 기준 주소 | `/api/v1` (BE), `/internal/v1` (AI). 실제 호스트는 TBD |
| 사용자 식별 | 로그인 없음. 앱이 처음 실행할 때 만든 무작위 ID를 `X-Device-Id` 헤더로 보냄 |
| 시각 | ISO 8601, KST 오프셋 포함 (예: `2026-11-03T21:30:00+09:00`) |
| 오류 응답 | HTTP 상태 코드와 함께 `{ "error": { "code": "...", "message": "..." } }` |

오류 코드

| HTTP | code | 의미 |
|---|---|---|
| 400 | `invalid_request` | 형식 오류, 필수값 누락 |
| 404 | `not_found` | 없는 자료·문제 |
| 413 | `file_too_large` | 파일 용량·쪽수 상한 초과 (상한 값 TBD) |
| 415 | `unsupported_file` | 텍스트 PDF가 아님 (스캔본·사진 제외) |
| 429 | `too_many_materials` | 동시 처리 자료 5개 초과 |
| 502 | `llm_unavailable` | 로컬·유료 모델 모두 실패 |

## 데이터 모양

### Question (문제)

AI 레포(`app/prompts.py`)가 만드는 형식을 그대로 따름. 여기에 BE가 `id`, `materialId`, `page`를 붙임.

```json
{
  "id": "q_001",
  "materialId": "m_001",
  "type": "ox",
  "question": "차집합 연산자는 첫 번째 관계에만 있는 튜플을 만든다.",
  "answer": "O",
  "accepted": [],
  "evidence": "차집합 연산자(-)는 두 관계에 대해 작용하며, ...",
  "page": 3
}
```

| 필드 | 타입 | 설명 |
|---|---|---|
| `id` | string | BE가 부여 |
| `materialId` | string | 출처 자료 |
| `type` | `"ox"` \| `"short"` | OX 또는 단답. 객관식 없음 |
| `question` | string | 문제 문장. OX는 서술문이고 물음표를 쓰지 않음 |
| `answer` | string | OX는 `"O"` 또는 `"X"`만 허용. 단답은 정답 용어 |
| `accepted` | string[] | 단답 허용 답안. OX는 빈 배열 |
| `evidence` | string | 원문에서 글자 그대로 가져온 근거 문장. 해설·오답노트에서만 표시 |
| `page` | number \| null | 원문 쪽수. 해설과 오답노트에 표시. **AI 출력에는 아직 없음 (아래 확인 필요 1번)** |

### Material (자료)

```json
{
  "id": "m_001",
  "name": "운영체제.pdf",
  "status": "processing",
  "chunksTotal": 12,
  "chunksDone": 4,
  "questionCount": 8
}
```

`status`: `processing` (생성 중) / `ready` (완료) / `failed` (실패). 만들어지는 대로 쓸 수 있으므로 `processing`이어도 `questionCount`가 0보다 크면 출제에 사용 가능.

## FE ↔ BE

| 엔드포인트 | 메서드 | 요청 | 응답 | 담당 |
|---|---|---|---|---|
| `/api/v1/materials` | POST | `multipart/form-data`, `files`: PDF 1~5개 | `202` `{ "materials": [Material] }` | BE |
| `/api/v1/materials` | GET | - | `{ "materials": [Material] }` | BE |
| `/api/v1/materials/{id}` | DELETE | - | `204` (자료와 파생 문제 삭제) | BE |
| `/api/v1/questions` | GET | 쿼리 `since`(선택, ISO 시각) | `{ "questions": [Question], "serverTime": "..." }` 기기에 내려받을 문제. `since` 이후 새로 생긴 것만 | BE |
| `/api/v1/questions/{id}/grade` | POST | `{ "userAnswer": "페이징" }` (단답만) | `{ "correct": true, "by": "rule" }` | BE → AI |
| `/api/v1/answers` | POST | `{ "answers": [Answer] }` 배치 | `{ "recorded": 3 }` | BE |
| `/api/v1/review` | GET | - | `{ "items": [ReviewItem] }` 오답노트 | BE |
| `/api/v1/questions/{id}/report` | POST | `{ "reason": "wrong_answer" }` | `204` | BE |
| `/api/v1/usage/daily` | POST | `{ "days": [UsageDay] }` | `{ "recorded": 1 }` | BE |
| `/api/v1/advice/today` | GET | - | `{ "message": "...", "suggestion": { "label": "...", "action": "..." } \| null }` | BE → AI |

### Answer (풀이 기록)

```json
{
  "questionId": "q_001",
  "result": "wrong",
  "answeredAt": "2026-11-03T21:30:00+09:00"
}
```

`result`: `correct` / `wrong` / `skipped`. 틀린 문제의 재출제 일정은 BE가 계산하고, 다음 `GET /questions`에 다시 포함해서 내려줌.

### ReviewItem (오답노트 항목)

```json
{
  "question": { "...": "Question" },
  "wrongCount": 2,
  "lastWrongAt": "2026-11-03T21:30:00+09:00"
}
```

### UsageDay (시청 기록 일별 집계)

기기에 저장한 원본 기록에서 **일별 합계만** 보냄. 영상 제목·채널·댓글·계정은 보내지 않고 수집하지도 않음.

```json
{
  "date": "2026-11-03",
  "sessionCount": 4,
  "totalMinutes": 38,
  "swipes": 210,
  "cardsShown": 21,
  "cardsAnswered": 15,
  "cardsSkipped": 6,
  "correct": 11,
  "lateNightShare": 0.42,
  "sessionLengthBuckets": { "lt5": 1, "5to10": 2, "10to20": 1, "gte20": 0 }
}
```

- `lateNightShare`: 밤 12시 이후 시청 비중 (0~1)
- 조언을 **할지, 무슨 내용인지**는 BE가 이 수치의 규칙으로 정하고, AI는 그 사실을 동료 말투 문장으로만 바꿈

## BE ↔ AI (내부)

AI는 상태 없는 서비스. 요청에 필요한 정보를 모두 담아 보내고, 결과를 돌려받아 BE가 저장함.

| 엔드포인트 | 메서드 | 요청 | 응답 | 담당 |
|---|---|---|---|---|
| `/internal/v1/generate` | POST | `{ "text": "...", "page": 3 }` 약 2000자 조각 1개 | `{ "questions": [GeneratedQuestion], "provider": "ollama", "discarded": 1 }` | AI |
| `/internal/v1/verify` | POST | `{ "question": GeneratedQuestion, "source": "..." }` | `{ "supported": true, "reason": "..." }` 근거만 보고 다시 풀어 정답 비교 | AI |
| `/internal/v1/grade` | POST | `{ "question": "...", "answer": "...", "accepted": [..], "userAnswer": "..." }` | `{ "correct": true, "by": "rule" }` 허용 답안에 있으면 LLM을 부르지 않고 `rule` | AI |
| `/internal/v1/advice` | POST | `{ "facts": { ... }, "suggestion": { ... } }` 규칙이 정한 사실과 수치 | `{ "message": "...", "fallback": false }` | AI |
| `/internal/v1/health` | GET | - | `{ "ok": true, "provider": "ollama" }` 폴백 전환 판단용 | AI |

`GeneratedQuestion`은 위 Question에서 `id`, `materialId`를 뺀 것 (`page`는 요청으로 받은 값을 그대로 돌려줌).

규칙

- `generate`: 근거 문장이 없거나 원문에 없는 문제는 AI가 버리고, 버린 개수를 `discarded`로 알림
- `advice`: 문장 속 숫자가 입력과 다르면 AI가 폐기하고 수치만 나열한 고정 문장을 `fallback: true`로 돌려줌
- 모델 우선순위: 로컬 Ollama → 실패 시 OpenAI 유료 API (조건부, 의사결정 로그 참고)

## 처리 흐름

```
1. FE  POST /materials (PDF)          → BE: 접수, 텍스트 추출, 약 2000자 조각으로 나눠 작업 큐에 적재
2. BE  POST /internal/generate (조각) → AI: 문제 생성·검사 → BE: 저장 (자료별로 번갈아 처리)
3. FE  GET /materials                 → 진행 상태 표시
4. FE  GET /questions                 → 기기에 문제 저장 (이후 카드는 오프라인으로 표시)
5. FE  카드 표시, 풀이                → POST /answers 로 결과 전송 (모아서 배치)
6. FE  하루 1회 POST /usage/daily, GET /advice/today
```

## 확인 필요 (확정 전에 팀이 정할 것)

1. **`page` 필드**: 해설과 오답노트가 원문 쪽수를 보여주기로 했는데, 현재 AI 출력에는 쪽수가 없음. BE가 조각마다 쪽수를 붙여 `generate` 요청에 넘기는 안으로 작성함. AI·BE 확인 필요
2. **단답 채점 위치**: 카드가 뜬 순간 서버를 부르지 않는 원칙과 충돌함. 단답은 (a) 기기에서 `accepted`와 먼저 비교하고, 일치하지 않을 때만 `/grade` 호출 (b) 오프라인이면 건너뛰거나 나중에 채점. FE에서는 (a)를 제안함
3. **`/questions` 증분 방식**: `since` 시각 기반으로 작성함. 삭제·신고된 문제를 기기에서 지우는 방법이 필요하면 `deletedIds`를 응답에 추가해야 함
4. **파일 상한**: 자료당 최대 쪽수·용량 (값 미정)
5. **인증**: 기기 ID만으로 충분한지. 공개 배포 URL이 심사 기간 동안 열려 있으므로 남용 방지(요청 횟수 제한) 필요 여부
6. **웹 체험판**: 샘플 영상과 실제 문제 생성을 같은 API로 쓰는 것으로 가정함. 체험판용 별도 엔드포인트가 필요하면 추가
7. **호스트와 포트**: BE·AI 배포 위치 (현재 AI는 작성자 Mac의 Ollama + 터널 구상, AI #6)
