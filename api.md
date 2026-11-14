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
| 429 | `rate_limited` | 횟수 제한 초과 (기기 ID별 업로드 하루 10개, IP별 업로드 하루 30개, 서버 전체 하루 생성 조각 2000개, 전체 생성 대기열 상한). 한도가 풀릴 때까지 다시 시도해도 소용없음 |
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
| `evidence` | string | 원문에서 글자 그대로 가져온 근거 문장. 앱에는 표시하지 않고, AI 자동 검사(원문 대조)와 BE의 쪽수 찾기에 씀 |
| `page` | number \| null | 원문 쪽수. PPTX는 슬라이드 번호. AI는 쪽수를 다루지 않고, BE가 근거 문장을 쪽(슬라이드)별 원문에서 찾아 붙임. 찾지 못하면 `null` |

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
| `/api/v1/answers` | POST | `{ "answers": [Answer] }` 배치. 한 번에 200개까지. 같은 `questionId`와 `answeredAt`의 기록은 한 번만 받으므로 전체를 다시 보내도 됨 | `{ "recorded": 3 }` (이미 받은 기록은 세지 않음) | BE |
| `/api/v1/questions/{id}/report` | POST | `{ "reason": "wrong_answer" }` | `204` | BE |

### Answer (풀이 기록)

```json
{
  "questionId": "q_001",
  "result": "wrong",
  "answeredAt": "2026-11-03T21:30:00+09:00"
}
```

`result`: `correct` / `wrong` / `skipped`. 풀이 기록은 새 문제 채우기(아래 규칙)에 씀. 틀린 문제를 언제 다시 낼지는 기기가 풀이 기록으로 계산하고(FE #14), BE는 틀린 문제를 `GET /questions`에 다시 내려주지 않음.

## BE ↔ AI (내부)

AI는 상태 없는 서비스. 요청에 필요한 정보를 모두 담아 보내고, 결과를 돌려받아 BE가 저장함.

| 엔드포인트 | 메서드 | 요청 | 응답 | 담당 |
|---|---|---|---|---|
| `/internal/v1/generate` | POST | `{ "text": "...", "avoid": ["이미 낸 문제 글자", ...] }` 약 2000자 조각 1개. `avoid`는 선택이며 같은 문제는 응답에서 뺌. 쪽수는 받지 않음 | `{ "questions": [GeneratedQuestion], "provider": "openai", "discarded": 1 }` | AI |
| `/internal/v1/verify` | POST | `{ "question": GeneratedQuestion, "source": "..." }` | `{ "supported": true, "reason": "..." }` 근거만 보고 다시 풀어 정답 비교 | AI |
| `/internal/v1/health` | GET | - | `{ "ok": true, "provider": "openai" }` | AI |

`GeneratedQuestion`은 위 Question에서 `id`, `materialId`, `page`를 뺀 것. BE가 저장할 때 `evidence`를 쪽(슬라이드)별 원문에서 찾아 `page`를 붙임.

규칙

- `generate`: 조각당 문제는 OX 3개와 단답 2개(총 5개)임. 요청의 `avoid`와 같은 문제는 응답에서 뺌
- 새 문제 채우기: `POST /api/v1/answers`로 풀이 기록을 받으면 BE가 그 문제의 자료에서 한 번도 안 푼 문제가 5개 미만일 때 새 문제를 만들어 채움. 자료 하나에 최대 10번까지 채우고, 이미 낸 문제와 같은 내용은 피함. 새 문제는 `GET /questions?since=`로 내려가며 앱이 받는 시점은 다음 동기화(15분 주기 또는 앱을 다시 열 때)임. 기록을 보내지 않으면 아무 일도 일어나지 않음
- `generate`: 근거 문장이 없거나 원문에 없는 문제는 AI가 버리고, 버린 개수를 `discarded`로 알림
- `generate`: 단답 정답은 명사 하나로 출제하고(프롬프트 지시) 같은 대상의 다른 이름을 `accepted`에 넣음. 공백 제외 10자 초과나 문장 종결(~다)인 단답은 자동 검사에서 버림. 명사인지는 검사하지 않음 (단답 채점에 LLM을 쓰지 않으므로 출제 단계에서 정함)
- `generate`: OX는 모델에게 자료와 맞는 문장과 사실 하나만 바꾼 틀린 문장을 함께 받고, AI 서비스가 둘 중 하나를 무작위로 골라 `answer`를 O 또는 X로 정함. 모델이 정답을 O로 몰아 내는 것을 막기 위함(AI PR #17, 2026-11-14). 응답 형식은 그대로임
- 모델: OpenAI API(gpt-5.4-mini, 서버 환경 변수 `OPENAI_MODEL`)만 씀. 로컬 모델(Ollama)과 로컬에서 OpenAI로 넘어가는 전환 경로는 2026-11-11에 폐기함(AI #13). 키나 모델이 비어 있으면 `/internal/v1/health`는 `ok: false`, 나머지는 `502 llm_unavailable`

## 처리 흐름

```
1. FE  POST /materials (PDF·PPTX)     → BE: 접수, 텍스트 추출(쪽·슬라이드별), 약 2000자 조각으로 나눠 작업 큐에 적재
2. BE  POST /internal/generate (조각) → AI: 문제 생성·검사 → BE: 근거 문장으로 쪽수를 찾아 붙여 저장 (자료별로 번갈아 처리)
3. FE  GET /materials                 → 진행 상태 표시
4. FE  GET /questions                 → 기기에 문제 저장 (이후 카드는 오프라인으로 표시)
5. FE  카드 표시, 풀이                → 단답은 기기에서 글자 비교로 채점(서버 호출 없음), 재출제는 기기가 계산. 결과는 POST /answers 로 전송 (모아서 배치)
```

## 확정한 것 (2026-11-04, BE·AI 담당 결정)

BE #4, AI #10, DOCS #4 리뷰에서 정한 내용이며, 위 본문에 반영함. 2번은 2026-11-08에 바뀌었고, 5번에서 LLM 채점 횟수 제한이 빠짐.

| 항목 | 결정 |
|---|---|
| 1. `page` 필드 | AI는 바꾸지 않음. BE가 근거 문장을 쪽별 텍스트에서 찾아 붙임. `generate` 요청에서 `page`를 뺌 |
| 2. 단답 채점 위치 | 기기에서 `answer`와 `accepted`를 글자로 비교(공백·대소문자 무시)한 결과가 최종 채점임. 단답 채점에 LLM을 쓰지 않기로 함 (2026-11-08, BE·AI 결정). 서버 `/grade`는 호환용으로 남기고 같은 비교만 함. 이전 결정은 "다를 때만 `/grade` 호출, 오프라인이면 저장했다가 채점"이었음 |
| 3. 삭제·재출제 | `deletedIds`는 지금 넣지 않음. 자료 삭제와 신고는 기기에서 시작하므로 기기가 직접 지움. BE가 문제를 지우는 기능이 생기면 추가. 재출제는 기기가 계산 (FE #14) |
| 4. 파일 상한 | 파일당 20MB, 100쪽(슬라이드), 동시 처리 자료 5개. 30장 자료 약 1분, 100쪽 자료 3~4분이면 생성이 끝남 (조각당 약 4초 기준) |
| 5. 인증·남용 방지 | 로그인 없이 기기 ID별 횟수 제한(구글 로그인은 이번 제출에 넣지 않음, 2026-11-11). 기기별 업로드 하루 10개, IP별 업로드 하루 30개, 서버 전체 하루 생성 조각 2000개, 전체 생성 대기열 상한. 초과 시 `429 rate_limited`. 서버의 `/docs`(API 문서 화면)는 꺼 둠 (LLM 채점 하루 100회 제한은 채점에서 LLM을 빼면서 삭제) |
| 6. 웹 체험판 | 만들지 않기로 함 (2026-11-06, FE 담당 결정). 같은 API와 같은 횟수 제한을 쓰고 `[샘플]`로 표기한다는 앞선 결정은 적용하지 않음 |
| 7. 배포 | BE와 AI를 Fly.io 앱 1개(도쿄, 512MB 1대)의 한 컨테이너에 올림. 주소 `https://scrollmate-be.fly.dev`. 외부에는 BE(8080)만 열고 AI 서비스(8001)는 컨테이너 안에서만 접근하며, SQLite는 볼륨에 저장함. 처음 결정(2026-11-04)은 같은 Mac에 두고 터널로 공개하는 것이었으나 Mac을 심사 종료까지 켜 둘 수 없어 2026-11-11에 바꿈(AI #13, BE #12). 터널과 로컬 모델은 폐기함 |

업로드 형식은 PDF와 PPTX로 정함 (수업 자료에 PPTX가 많음, 노션 의사결정 로그 11/4). PPTX는 python-pptx로 슬라이드별 글자 추출을 확인했고, 슬라이드 번호를 `page`에 넣음.

## FE에서 정한 것

- 과목(학습 주제)과 시험일은 기기에만 저장하고 서버로 보내지 않음. 그래서 API에 과목 필드가 없음
- 한 번에 올린 파일은 모두 같은 학습 주제에 연결하므로 `POST /materials` 응답의 자료 순서는 중요하지 않음
- 모르는 `type`의 문제는 FE가 버리고, 응답의 모르는 JSON 필드는 무시함. `accepted`와 `page`는 생략할 수 있음
- 429 중 `rate_limited`는 다시 시도하지 않고 실패로 처리함. `too_many_materials`와 5xx, 네트워크 오류는 최대 3번까지 다시 시도함

## 만들지 않기로 한 것 (2026-11-13)

- 오답노트(`GET /review`), 시청 기록 집계(`POST /usage/daily`), 오늘의 한마디 조언(`GET /advice/today`, `/internal/v1/advice`). 앱이 쓰지 않아 서버에서도 지움(BE PR #18, AI PR #17)
- 감지 대상은 안드로이드 YouTube 쇼츠만. 인스타그램 릴스는 다루지 않음

## 아직 정해지지 않은 것

- 없음. 이전에 정해지지 않았던 구글 로그인의 토큰 전달 방식은 로그인을 이번 제출에 넣지 않아 정하지 않음(BE #5), 서버 주소는 위 확정 사항 7번에 적음
