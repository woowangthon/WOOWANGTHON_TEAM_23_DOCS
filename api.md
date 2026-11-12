# API 명세

FE / BE / AI 간 계약의 단일 기준. API 변경 전 이 문서 먼저 수정 필요.

상태: **초안 (2026-11-03). 2026-11-06에 BE·AI 담당의 결정(BE #4, AI #10)과 학습 대상 확장(DOCS #6)을 반영함.** 최종 확인이 끝나면 이 줄을 "확정"으로 바꾸고 날짜를 적음.

## 통신 구조

```
FE(안드로이드) ──[FE↔BE API]──> BE ──[BE↔AI API]──> AI ──> OpenAI API
```

- FE는 BE만 호출함. AI를 직접 호출하지 않음
- OpenAI 키는 서버(Fly.io 시크릿)에만 둠. 앱과 저장소에는 넣지 않음
- AI는 상태 없는 서비스. 저장은 전부 BE가 함
- 문제는 업로드 시점에 미리 생성해 기기에 내려받음. 카드가 뜨는 순간에는 서버를 호출하지 않음
- 과목(학습 주제)과 시험일은 기기에만 저장함. 서버 API에는 이 개념이 없음

## 공통 규칙

| 항목 | 내용 |
|---|---|
| 형식 | JSON (UTF-8). 파일 업로드만 `multipart/form-data` |
| 기준 주소 | `/api/v1` (BE), `/internal/v1` (AI). 실제 호스트는 배포 때 정함 (아래 "확정한 것" 7번) |
| 사용자 식별 | 앱이 처음 실행할 때 만든 무작위 ID를 `X-Device-Id` 헤더로 보냄. 구글 로그인은 이번 제출에 넣지 않기로 함(BE #5, FE #12, 2026-11-11 결정). 로그인 없이 이 방식으로만 기기를 구분하므로, 공개 주소를 아는 누구나 호출할 수 있고 기기 ID를 바꾸면 기기별 한도를 피할 수 있음. 앱을 지우고 다시 설치하면 ID가 새로 생겨 이전에 올린 자료와 문제를 받지 못함 |
| 시각 | ISO 8601, KST 오프셋 포함 (예: `2026-11-03T21:30:00+09:00`) |
| 오류 응답 | HTTP 상태 코드와 함께 `{ "error": { "code": "...", "message": "..." } }` |

오류 코드

| HTTP | code | 의미 |
|---|---|---|
| 400 | `invalid_request` | 형식 오류, 필수값 누락 |
| 404 | `not_found` | 없는 자료·문제 |
| 413 | `file_too_large` | 파일당 20MB 또는 100쪽(슬라이드) 초과 |
| 415 | `unsupported_file` | 텍스트를 뽑을 수 없는 파일. 스캔본 PDF, 글자 없는 슬라이드만 있는 PPTX, PDF·PPTX 외 형식 |
| 429 | `too_many_materials` | 동시 처리 자료 5개 초과. 잠시 뒤 다시 시도하면 됨 |
| 429 | `rate_limited` | 기기 ID별 횟수 제한 초과 (업로드 하루 10개, 전체 생성 대기열 상한). 한도가 풀릴 때까지 다시 시도해도 소용없음 |
| 502 | `llm_unavailable` | 로컬·유료 모델 모두 실패 |

## 데이터 모양

사용자가 올리는 자료의 주제에는 제한이 없음 (토익, 운전면허, 자격증, 수업 복습 등). 아래 예시의 주제는 임의로 고른 것.

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
| `answer` | string | OX는 `"O"` 또는 `"X"`만 허용. 단답은 명사 하나로 출제함(프롬프트 지시). 공백 제외 10자 초과나 문장 종결(~다)은 AI의 자동 검사에서 버리고, 명사인지는 검사하지 않음 |
| `accepted` | string[] | 단답 허용 답안. 같은 대상의 다른 이름(정식 명칭, 줄임말, 영어 이름 등)을 넣음. 띄어쓰기·대소문자 차이는 비교할 때 무시하므로 넣지 않음. OX는 빈 배열 |
| `evidence` | string | 원문에서 글자 그대로 가져온 근거 문장. 해설·오답노트에서만 표시 |
| `page` | number \| null | 원문 쪽수. PPTX는 슬라이드 번호. AI는 쪽수를 다루지 않고, BE가 근거 문장을 쪽(슬라이드)별 원문에서 찾아 붙임. 찾지 못하면 `null`. 해설과 오답노트에 표시 |

### Material (자료)

```json
{
  "id": "m_001",
  "name": "자격증 요약본.pptx",
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
| `/api/v1/materials` | POST | `multipart/form-data`, `files`: PDF 또는 PPTX 1~5개. 파일당 20MB, 100쪽(슬라이드) 이하. HWP는 지원하지 않음 | `202` `{ "materials": [Material] }` | BE |
| `/api/v1/materials` | GET | - | `{ "materials": [Material] }` | BE |
| `/api/v1/materials/{id}` | DELETE | - | `204` (자료와 파생 문제 삭제) | BE |
| `/api/v1/questions` | GET | 쿼리 `since`(선택, ISO 시각) | `{ "questions": [Question], "serverTime": "..." }` 기기에 내려받을 문제. `since` 이후 새로 생긴 것만. 틀린 문제를 다시 내려주지 않음 | BE |
| `/api/v1/questions/{id}/grade` | POST | `{ "userAnswer": "페이징" }` (단답만). AI를 부르지 않고 `answer`와 `accepted`에 글자로만 비교함(공백·대소문자 무시). 기기 채점과 결과가 같으므로 FE는 호출하지 않아도 되고, 호환을 위해 남겨 둠 | `{ "correct": true, "by": "rule" }` `by`는 항상 `rule` | BE |
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

`result`: `correct` / `wrong` / `skipped`. 풀이 기록은 오답노트와 통계용으로만 받음. 틀린 문제를 언제 다시 낼지는 기기가 풀이 기록으로 계산하고(FE #14), BE는 틀린 문제를 `GET /questions`에 다시 내려주지 않음.

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
| `/internal/v1/generate` | POST | `{ "text": "..." }` 약 2000자 조각 1개. 쪽수는 받지 않음 | `{ "questions": [GeneratedQuestion], "provider": "ollama", "discarded": 1 }` | AI |
| `/internal/v1/verify` | POST | `{ "question": GeneratedQuestion, "source": "..." }` | `{ "supported": true, "reason": "..." }` 근거만 보고 다시 풀어 정답 비교 | AI |
| `/internal/v1/advice` | POST | `{ "facts": { ... }, "suggestion": { ... } }` 규칙이 정한 사실과 수치 | `{ "message": "...", "fallback": false }` | AI |
| `/internal/v1/health` | GET | - | `{ "ok": true, "provider": "ollama" }` 폴백 전환 판단용 | AI |

`GeneratedQuestion`은 위 Question에서 `id`, `materialId`, `page`를 뺀 것. BE가 저장할 때 `evidence`를 쪽(슬라이드)별 원문에서 찾아 `page`를 붙임.

규칙

- `generate`: 근거 문장이 없거나 원문에 없는 문제는 AI가 버리고, 버린 개수를 `discarded`로 알림
- `generate`: 단답 정답은 명사 하나로 출제하고(프롬프트 지시) 같은 대상의 다른 이름을 `accepted`에 넣음. 공백 제외 10자 초과나 문장 종결(~다)인 단답은 자동 검사에서 버림. 명사인지는 검사하지 않음 (단답 채점에 LLM을 쓰지 않으므로 출제 단계에서 정함)
- `advice`: 문장 속 숫자가 입력과 다르면 AI가 폐기하고 수치만 나열한 고정 문장을 `fallback: true`로 돌려줌
- 모델: OpenAI API(gpt-5.4-mini, 서버 환경 변수 `OPENAI_MODEL`)만 씀. 로컬 모델(Ollama)과 로컬에서 OpenAI로 넘어가는 전환 경로는 2026-11-11에 폐기함(AI #13). 키나 모델이 비어 있으면 `/internal/v1/health`는 `ok: false`, 나머지는 `502 llm_unavailable`

## 처리 흐름

```
1. FE  POST /materials (PDF·PPTX)     → BE: 접수, 텍스트 추출(쪽·슬라이드별), 약 2000자 조각으로 나눠 작업 큐에 적재
2. BE  POST /internal/generate (조각) → AI: 문제 생성·검사 → BE: 근거 문장으로 쪽수를 찾아 붙여 저장 (자료별로 번갈아 처리)
3. FE  GET /materials                 → 진행 상태 표시
4. FE  GET /questions                 → 기기에 문제 저장 (이후 카드는 오프라인으로 표시)
5. FE  카드 표시, 풀이                → 단답은 기기에서 글자 비교로 채점(서버 호출 없음), 재출제는 기기가 계산. 결과는 POST /answers 로 전송 (모아서 배치)
6. FE  하루 1회 POST /usage/daily, GET /advice/today
```

## 확정한 것 (2026-11-04, BE·AI 담당 결정)

BE #4, AI #10, DOCS #4 리뷰에서 정한 내용이며, 위 본문에 반영함. 2번은 2026-11-08에 바뀌었고, 5번에서 LLM 채점 횟수 제한이 빠짐.

| 항목 | 결정 |
|---|---|
| 1. `page` 필드 | AI는 바꾸지 않음. BE가 근거 문장을 쪽별 텍스트에서 찾아 붙임. `generate` 요청에서 `page`를 뺌 |
| 2. 단답 채점 위치 | 기기에서 `answer`와 `accepted`를 글자로 비교(공백·대소문자 무시)한 결과가 최종 채점임. 단답 채점에 LLM을 쓰지 않기로 함 (2026-11-08, BE·AI 결정). 서버 `/grade`는 호환용으로 남기고 같은 비교만 함. 이전 결정은 "다를 때만 `/grade` 호출, 오프라인이면 저장했다가 채점"이었음 |
| 3. 삭제·재출제 | `deletedIds`는 지금 넣지 않음. 자료 삭제와 신고는 기기에서 시작하므로 기기가 직접 지움. BE가 문제를 지우는 기능이 생기면 추가. 재출제는 기기가 계산 (FE #14) |
| 4. 파일 상한 | 파일당 20MB, 100쪽(슬라이드), 동시 처리 자료 5개. 30장 자료 약 1분, 100쪽 자료 3~4분이면 생성이 끝남 (조각당 약 4초 기준) |
| 5. 인증·남용 방지 | 로그인 없이 기기 ID별 횟수 제한(구글 로그인은 이번 제출에 넣지 않음, 2026-11-11). 업로드 하루 10개, 전체 생성 대기열 상한. 초과 시 `429 rate_limited` (LLM 채점 하루 100회 제한은 채점에서 LLM을 빼면서 삭제) |
| 6. 웹 체험판 | 만들지 않기로 함 (2026-11-06, FE 담당 결정). 같은 API와 같은 횟수 제한을 쓰고 `[샘플]`로 표기한다는 앞선 결정은 적용하지 않음 |
| 7. 배포 | BE와 AI를 Fly.io 앱 1개(도쿄, 512MB 1대)의 한 컨테이너에 올림. 주소 `https://scrollmate-be.fly.dev`. 외부에는 BE(8080)만 열고 AI 서비스(8001)는 컨테이너 안에서만 접근하며, SQLite는 볼륨에 저장함. 처음 결정(2026-11-04)은 같은 Mac에 두고 터널로 공개하는 것이었으나 Mac을 심사 종료까지 켜 둘 수 없어 2026-11-11에 바꿈(AI #13, BE #12). 터널과 로컬 모델은 폐기함 |

업로드 형식은 PDF와 PPTX로 정함 (수업 자료에 PPTX가 많음, 노션 의사결정 로그 11/4). PPTX는 python-pptx로 슬라이드별 글자 추출을 확인했고, 슬라이드 번호를 `page`에 넣음.

## FE에서 정한 것

- 과목(학습 주제)과 시험일은 기기에만 저장하고 서버로 보내지 않음. 그래서 API에 과목 필드가 없음
- 한 번에 올린 파일은 모두 같은 학습 주제에 연결하므로 `POST /materials` 응답의 자료 순서는 중요하지 않음
- 모르는 `type`의 문제는 FE가 버리고, 응답의 모르는 JSON 필드는 무시함. `accepted`와 `page`는 생략할 수 있음
- 429 중 `rate_limited`는 다시 시도하지 않고 실패로 처리함. `too_many_materials`와 5xx, 네트워크 오류는 최대 3번까지 다시 시도함

## 아직 정해지지 않은 것

- 없음. 이전에 정해지지 않았던 구글 로그인의 토큰 전달 방식은 로그인을 이번 제출에 넣지 않아 정하지 않음(BE #5), 서버 주소는 위 확정 사항 7번에 적음
