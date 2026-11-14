# 외부 라이브러리·API·데이터 출처

| 이름 | 용도 | 사용 레포 | 링크 | 라이선스 |
|---|---|---|---|---|
| Android Gradle Plugin | 안드로이드 앱 빌드, Kotlin 내장 지원 | FE | https://developer.android.com/build | Apache-2.0 |
| Gradle | 빌드 도구, 저장소에 Wrapper 포함 | FE | https://gradle.org | Apache-2.0 |
| Android SDK | 앱 컴파일, 접근성 서비스 API | FE | https://developer.android.com/studio | Android SDK License |
| Kotlin Compose·serialization 컴파일러 플러그인 2.2.10 | Compose 화면 컴파일, JSON 직렬화 | FE | https://kotlinlang.org | Apache-2.0 |
| Jetpack Compose (BOM 2026.09.00, Material3, UI) | 앱 화면 | FE | https://developer.android.com/jetpack/compose | Apache-2.0 |
| AndroidX Activity Compose 1.13.0 | Compose 액티비티, 파일 선택 | FE | https://developer.android.com/jetpack/androidx/releases/activity | Apache-2.0 |
| AndroidX Room 2.8.5 | 기기 안 데이터베이스 (학습 주제, 자료, 문제, 풀이 기록) | FE | https://developer.android.com/jetpack/androidx/releases/room | Apache-2.0 |
| KSP 2.3.13 | Room 코드 생성 | FE | https://github.com/google/ksp | Apache-2.0 |
| AndroidX WorkManager 2.10.5 | 화면이 닫혀도 이어지는 백그라운드 업로드 | FE | https://developer.android.com/jetpack/androidx/releases/work | Apache-2.0 |
| OkHttp 5.5.1 | BE 호출용 HTTP 클라이언트 | FE | https://square.github.io/okhttp/ | Apache-2.0 |
| kotlinx.serialization 1.9.0 | 서버 JSON 변환 | FE | https://github.com/Kotlin/kotlinx.serialization | Apache-2.0 |
| kotlinx.coroutines 1.10.2 | 비동기 처리 | FE | https://github.com/Kotlin/kotlinx.coroutines | Apache-2.0 |
| OkHttp MockWebServer 5.5.1 | 서버 응답 단위 테스트 (테스트 전용) | FE | https://square.github.io/okhttp/ | Apache-2.0 |
| JUnit 4.13.2 | 단위 테스트 (테스트 전용) | FE | https://junit.org/junit4/ | EPL-1.0 |
| uv | Python 패키지·가상환경 관리 | AI | https://github.com/astral-sh/uv | Apache-2.0 (GitHub 표기) |
| httpx | OpenAI API 호출(AI), AI 서비스 호출(BE)용 HTTP 클라이언트 | BE, AI | https://www.python-httpx.org | BSD-3-Clause |
| pypdf | PDF 텍스트 추출. BE가 업로드 파일에서 추출하고, AI는 평가 도구에서만 씀 | BE, AI | https://pypdf.readthedocs.io | BSD-3-Clause |
| FastAPI 0.162.2 | API 서버 | BE, AI | https://fastapi.tiangolo.com | MIT |
| Uvicorn 0.54.0 | API 서버 실행 | BE, AI | https://www.uvicorn.org | BSD-3-Clause |
| python-multipart 0.0.33 | 파일 업로드(multipart) 받기 | BE | https://github.com/Kludex/python-multipart | Apache-2.0 |
| python-pptx 1.0.2 | PPTX 슬라이드 글자 추출 | BE | https://github.com/scanny/python-pptx | MIT |
| pytest 9.2.0 | 단위 테스트 (테스트 전용) | BE, AI | https://docs.pytest.org | MIT |
| Fly.io | BE와 AI 서비스를 올린 호스팅(컨테이너 1대, 볼륨에 SQLite 저장) | BE (배포 환경) | https://fly.io | 서비스 약관 확인 필요 |
| OpenAI API (gpt-5.4-mini) | 문제 생성 모델. 업로드한 자료의 글자가 전송됨 | AI | https://platform.openai.com | OpenAI 이용 약관 확인 필요 |
| Ollama | 로컬 모델 실행. 초기 모델 평가용이며 서비스에는 쓰지 않음 | AI | https://ollama.com | MIT |
| Qwen3.5 (4b, 9b) | 문제 생성 모델 후보. 평가용이며 서비스에는 쓰지 않음(2026-11-11 폐기) | AI | https://ollama.com/library/qwen3.5 | 확인 필요 |
| Gemma 3 (4b, 12b) | 문제 생성 모델 후보. 평가용이며 서비스에는 쓰지 않음 | AI | https://ollama.com/library/gemma3 | Gemma Terms of Use (https://ai.google.dev/gemma/terms) |
| EXAONE 3.5 (7.8b) | 문제 생성 모델 후보. 평가용이며 서비스에는 쓰지 않음 | AI | https://ollama.com/library/exaone3.5 | 확인 필요 |
| 한국어 위키백과 「운영체제」 | 평가용 샘플 입력 | AI | https://ko.wikipedia.org/w/index.php?title=운영체제&oldid=42489093 | CC BY-SA 4.0 |
| 한국어 위키백과 「관계형 데이터베이스」 | 평가용 샘플 입력 | AI | https://ko.wikipedia.org/w/index.php?title=관계형_데이터베이스&oldid=42532633 | CC BY-SA 4.0 |
| 한국어 위키백과 「미분」 | 평가용 샘플 입력 | AI | https://ko.wikipedia.org/w/index.php?title=미분&oldid=42583243 | CC BY-SA 4.0 |
| 한국어 위키백과 「음주 운전」 | 학습에 쓰지 않은 평가용 샘플 입력 | AI | https://ko.wikipedia.org/w/index.php?title=음주_운전&oldid=41116109 | CC BY-SA 4.0 |
| 한국어 위키백과 「광합성」 | 시연·테스트용 샘플 학습 자료. 앞부분 약 7,000자를 PDF로 옮김(레포에 포함하지 않음) | 시연 | https://ko.wikipedia.org/w/index.php?title=%EA%B4%91%ED%95%A9%EC%84%B1&oldid=42725227 | CC BY-SA 4.0 |
| 한국어 위키백과 「임진왜란」 | 시연·테스트용 샘플 학습 자료. 앞부분 약 7,000자를 PDF로 옮김(레포에 포함하지 않음) | 시연 | https://ko.wikipedia.org/w/index.php?title=%EC%9E%84%EC%A7%84%EC%99%9C%EB%9E%80&oldid=42391845 | CC BY-SA 4.0 |
| 한국어 위키백과 「수요와 공급」 | 시연·테스트용 샘플 학습 자료. 앞부분 약 7,000자를 PDF로 옮김(레포에 포함하지 않음) | 시연 | https://ko.wikipedia.org/w/index.php?title=%EC%88%98%EC%9A%94%EC%99%80%20%EA%B3%B5%EA%B8%89&oldid=42656812 | CC BY-SA 4.0 |
| 한국어 위키백과 「자동차 운전면허」 | 시연·테스트용 샘플 학습 자료. 앞부분 약 7,000자를 PDF로 옮김(레포에 포함하지 않음) | 시연 | https://ko.wikipedia.org/w/index.php?title=%EC%9E%90%EB%8F%99%EC%B0%A8%20%EC%9A%B4%EC%A0%84%EB%A9%B4%ED%97%88&oldid=42747680 | CC BY-SA 4.0 |
| Qwen3 1.7B·4B | 소형 모델 학습 실험 대상. 실험을 종료했고 채택하지 않음 | AI (실험, 레포에 포함하지 않음) | https://huggingface.co/Qwen | Apache-2.0 (모델 카드 확인 필요) |

학습 실험에 쓴 한국어 위키백과 문서 55개의 제목과 버전(revid) 목록은 AI 레포 로컬에만 있음. 학습 코드와 학습한 모델은 레포에 넣지 않음. 필요하면 목록을 별도 파일로 정리해 전달함 (AI #8).
