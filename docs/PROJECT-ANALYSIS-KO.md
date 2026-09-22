# MoneyPrinterTurbo 전수조사 분석 정리 (한국어)

> 이 문서는 MoneyPrinterTurbo 저장소를 폴더/코드 단위로 전수조사한 결과와,
> 설치·사용법, 성격(스킬/MCP 여부), API 키 정책, 수익화 아이디어를 정리한 문서입니다.

## 🔗 관련 GitHub 주소

| 구분 | 주소 |
| --- | --- |
| 이 저장소 (포크) | https://github.com/bmshin94/MoneyPrinterTurbo |
| 원본(업스트림) 저장소 | https://github.com/harry0703/MoneyPrinterTurbo |
| 릴리스 (윈도우 원클릭 패키지) | https://github.com/harry0703/MoneyPrinterTurbo/releases/latest |
| 이슈 트래커 | https://github.com/harry0703/MoneyPrinterTurbo/issues |
| AI 에이전트용 Skill 문서 | https://raw.githubusercontent.com/harry0703/MoneyPrinterTurbo/main/docs/skill/SKILL.md |
| 에이전트 헬퍼 스크립트 | https://raw.githubusercontent.com/harry0703/MoneyPrinterTurbo/main/docs/skill/mpt_agent.py |
| Colab 노트북 | https://colab.research.google.com/github/harry0703/MoneyPrinterTurbo/blob/main/docs/MoneyPrinterTurbo.ipynb |
| 제작자의 다른 프로젝트 (MangoDisk) | https://github.com/harry0703/MangoDisk |

---

## 1. 이 프로젝트가 뭐야?

**주제 한 줄만 입력하면 대본 → 성우 내레이션 → 영상 소재 → 자막 → 배경음악까지 자동으로 만들어
완성된 숏폼 MP4를 출력하는 "AI 숏폼 영상 자동 생성 앱"** 입니다.

- 버전: `v1.3.7` / 라이선스: **MIT** (상업적 사용 가능)
- 언어/런타임: **Python 3.11+**, FastAPI + Streamlit + moviepy(ffmpeg)
- 출력 비율: 세로 `9:16 (1080x1920)`, 가로 `16:9 (1920x1080)`, 정사각 `1:1 (1080x1080)`

### 한 줄 실행 예시

```bash
uv run python cli.py --video-subject "커피가 몸에 좋은 이유"
```

---

## 2. 폴더 구조 전수조사

```
MoneyPrinterTurbo/
├── main.py                  # API 서버 진입점 (uvicorn -> app.asgi:app, 포트 8080)
├── cli.py                   # CLI 진입점 (단건/배치 생성, --stop-at 단계 제어)
├── webui/
│   ├── Main.py              # Streamlit WebUI (8,227줄)
│   ├── styles.css
│   └── i18n/                # 13개 언어 (ko / en / zh / ja 없음, ko.json 포함)
├── app/
│   ├── asgi.py, router.py   # FastAPI 앱 + 라우터 등록
│   ├── controllers/
│   │   ├── base.py          # verify_token() : x-api-key 인증
│   │   ├── v1/video.py      # 영상/자막/오디오 생성, 태스크 조회, 스트리밍/다운로드 (508줄)
│   │   ├── v1/llm.py        # 대본 생성 API
│   │   └── manager/         # memory_manager / redis_manager (태스크 큐)
│   ├── services/            # 핵심 로직
│   │   ├── task.py   (1,750줄)  # 전체 파이프라인 오케스트레이션
│   │   ├── voice.py  (3,198줄)  # TTS 12종 어댑터 (최대 파일)
│   │   ├── material.py (2,511줄)# 영상 소재 검색/생성/다운로드
│   │   ├── video.py  (1,644줄)  # moviepy 컷 편집/합성/트랜지션
│   │   ├── llm.py    (1,214줄)  # LLM 25종 어댑터
│   │   ├── subtitle.py          # edge 타임스탬프 / faster-whisper 전사
│   │   ├── bgm.py               # 배경음악 선택/믹싱
│   │   ├── upload_post.py       # TikTok / Instagram / YouTube Shorts 자동 업로드
│   │   ├── state.py             # MemoryState / RedisState (태스크 상태 저장)
│   │   └── ofox.py, muapi.py, metaso_minimax.py, volcengine_seedance.py,
│   │       loomloom.py, elevenlabs_music.py, twelvelabs.py, sonilo.py ...
│   ├── models/              # schema.py(Pydantic), llm_provider.py, const.py
│   └── utils/               # file_security.py(경로 탈출 방어), logging_utils.py
├── docs/
│   ├── skill/SKILL.md       # AI 에이전트용 Skill 문서 (158줄)
│   ├── skill/mpt_agent.py   # 에이전트가 실행하는 자동 설치+생성 헬퍼 (797줄)
│   ├── MoneyPrinterTurbo.ipynb  # Google Colab 실행용
│   ├── voice-list.txt       # Edge TTS 목소리 목록
│   └── loomloom/            # LoomLoom 템플릿 JSON
├── resource/
│   ├── songs/               # 기본 배경음악 30곡 (output000~029.mp3)
│   ├── fonts/               # 중문/베트남/영문 폰트 (한글 폰트 없음)
│   └── public/index.html
├── test/                    # 테스트 60개+ (커버리지 하한 70% 강제)
├── config.example.toml      # 설정 템플릿 (첫 실행 시 config.toml 자동 생성)
├── Dockerfile / Dockerfile.gpu / Dockerfile.claude
├── docker-compose{,.gpu,.release,.claude}.yml
├── pyproject.toml + uv.lock # 의존성 단일 소스 (uv 기반)
├── requirements.txt         # 레거시 pip 호환용
└── .github/workflows/       # CI + GHCR 도커 이미지 자동 빌드
```

---

## 3. 실행 파이프라인 (app/services/task.py 기준)

| 단계 | 함수 | 진행률 | 하는 일 |
| --- | --- | --- | --- |
| 1 | `generate_script()` | 5 → 10% | LLM이 주제로 영상 대본 작성 |
| 2 | `generate_terms()` | → 20% | 대본에서 영상 검색 키워드 추출 |
| 3 | `generate_audio()` | → 30% | TTS로 내레이션 음성 생성 |
| 4 | `generate_subtitle()` | → 40% | 자막(SRT) 생성 (edge 또는 whisper) |
| 5 | `get_video_materials()` | → 50% | 스톡/AI 영상 소재 수집 |
| 6 | `generate_final_videos()` | → 100% | 컷 편집 + 자막 + BGM 합성 → `final-1.mp4` |
| 7 | `_run_cross_post()` | (옵션) | TikTok / Instagram / YouTube Shorts 자동 업로드 |

- `--stop-at` 으로 중간 단계에서 종료 가능 (대본만, 오디오만 등)
- 상태 저장은 `MemoryState`(단일 프로세스) 또는 `RedisState`(멀티 워커) 선택

---

## 4. 사용 방법 4가지

### ① WebUI (초보자용)
```bash
sh webui.sh          # macOS / Linux
.\webui.bat          # Windows
# → http://localhost:8501
```
LAN 개방: `MPT_WEBUI_HOST=0.0.0.0 sh webui.sh`

### ② REST API (자동화/연동용)
```bash
uv run python main.py
# → http://localhost:8080/docs (Swagger)
```
주요 엔드포인트: `POST /v1/videos`, `POST /v1/subtitle`, `POST /v1/audio`,
`GET /v1/tasks`, `GET /v1/tasks/{task_id}`, `GET /v1/stream/{path}`, `GET /v1/download/{path}`

### ③ CLI (서버/배치용)
```bash
uv run python cli.py --video-subject "AI가 바꾸는 일상"
uv run python cli.py --help
uv run python cli.py --batch-file ./tasks.json --stop-at video
```
배치 매니페스트는 최대 100개 태스크 / 1MiB 제한, 종료 시 JSON 요약 출력.

### ④ AI 에이전트 Skill (설치까지 자동)
```text
Use this Skill: https://raw.githubusercontent.com/harry0703/MoneyPrinterTurbo/main/docs/skill/SKILL.md
Create a video with the topic "AI가 바꾸는 일상"
```

### 설치 방법 요약

| 방법 | 명령 |
| --- | --- |
| Windows 원클릭 | Releases의 `.7z` 다운로드 → `update.bat` → `start.bat` |
| uv (권장) | `uv python install 3.11` → `uv sync --frozen` → `sh webui.sh` |
| pip (레거시) | `python3.11 -m venv .venv` → `pip install -r requirements.txt` |
| Docker | `cp config.example.toml config.toml` → `docker compose -f docker-compose.release.yml up` |
| Colab | `docs/MoneyPrinterTurbo.ipynb` |

### 최소 사양

| 항목 | 최소 | 권장 |
| --- | --- | --- |
| CPU | 4코어 | 6~8코어 |
| RAM | 4GB | 8~16GB |
| GPU | 불필요 | 4GB+ VRAM (whisper/배치 시) |

---

## 5. 플러그인인가, 스킬인가, MCP인가?

| 형태 | 해당 여부 | 설명 |
| --- | --- | --- |
| 독립 실행형 앱 | **O (본질)** | Python + FastAPI + Streamlit 기반 완성형 애플리케이션 |
| Skill | **O (부가 지원)** | `docs/skill/SKILL.md` + `mpt_agent.py`, Claude Skill 포맷(YAML frontmatter) 준수 |
| 플러그인 | X | 특정 호스트 앱의 확장이 아님 |
| MCP 서버 | X | 저장소 내 MCP 프로토콜 구현 없음 |

> 확장 아이디어: 이미 REST API가 완성돼 있으므로, 이를 감싸는 **MCP 서버**를 만들면
> `create_video` / `get_task` / `list_tasks` / `download_video` 4개 툴만으로 MCP화 가능.

---

## 6. API 토큰(키) 정책

### (A) 외부 서비스 키 — 기능을 쓰기 위해 필요한 것

| 용도 | 필수 여부 | 비용 |
| --- | --- | --- |
| LLM (대본 생성) | 사실상 필수 | 유료 (단, **Ollama 로컬 모델이면 무료**) |
| Pexels / Pixabay (영상 소재) | 필요 | **무료 가입** |
| Edge TTS (성우) | **불필요** | **완전 무료** |
| faster-whisper (자막) | 불필요 | 로컬 실행 무료 |
| Seedance / OFox / MuAPI 등 AI 영상 생성 | 선택 | 유료 (클립당 과금) |
| ElevenLabs / Azure / Fish Audio TTS | 선택 | 유료 |
| upload-post (자동 업로드) | 선택 | 유료 |

→ **최소 조합**: LLM 키 1개 + Pexels 키 1개. (Ollama 사용 시 Pexels만으로 사실상 0원 운영)

### (B) 이 앱 자체의 보호 키

```toml
[app]
api_key = ""   # 비워두면 인증 없음(로컬용). 값을 넣으면 x-api-key 헤더 필수
```

```bash
curl -H "x-api-key: <설정한 키>" http://localhost:8080/v1/tasks
```

- 구현 위치: `app/controllers/base.py` 의 `verify_token()`
- **서버에 공개 배포한다면 반드시 설정할 것** (미설정 시 외부인이 내 API 키로 비용 발생 가능)
- `config.toml`은 `.gitignore` 대상 — 절대 커밋 금지
- API 키는 배열로 여러 개 등록 가능 (자동 로테이션): `pexels_api_keys = ["key1", "key2"]`
- 외부 프론트엔드에서 호출하려면 `CORS_ALLOWED_ORIGINS` 환경변수 설정

---

## 7. 지원 외부 서비스 목록

- **LLM (25종+)**: Kimi/Moonshot(기본값), OpenAI, Anthropic Claude, Google Gemini, DeepSeek,
  Qwen, Azure OpenAI, ByteDance VolcEngine Ark, xAI Grok, MiniMax, Xiaomi MiMo,
  OpenRouter, Cloudflare AI Gateway, ModelScope, AIHubMix, AIML API, EvoLink,
  Groq, Pollinations AI, OneAPI, LiteLLM, **Ollama(로컬)** 등
- **영상 소재**: Pexels, Pixabay, Coverr (무료) / WaveSpeed, Seedance, OFox, MuAPI,
  Metaso MiniMax H3, OpenAI 이미지→영상 (유료) / 로컬 업로드
- **TTS (12종)**: **Edge TTS(무료, 키 불필요)**, Azure Speech, SiliconFlow, Gemini TTS,
  Xiaomi MiMo, MiniMax, ElevenLabs, Chatterbox, Kokoro, Fish Audio, ModelBest VoxCPM(음성 클로닝)
- **자막**: edge(TTS 타임스탬프, 기본) / whisper(faster-whisper, large-v3 약 3GB)
- **업로드**: upload-post.com 경유 TikTok / Instagram / YouTube Shorts

---

## 8. 왜 GitHub에서 유명한가

1. 이름(“MoneyPrinter”)이 만들어내는 강력한 클릭 유인
2. “AI로 숏폼 자동 생성 = 수익화” 라는 대중적 니즈 정조준
3. Edge TTS 무료 등으로 **지갑 없이 끝까지 체험 가능**
4. WebUI + API + CLI + Docker + 테스트 60개 + 13개 국어의 **제품급 완성도**
5. 신규 모델(Kimi K3, Seedance, MiniMax H3 등) 반영 속도가 매우 빠름
6. 스폰서 10곳 이상을 확보한 드문 오픈소스 수익 구조
7. Trendshift 등재 + Windows 원클릭 패키지로 비개발자까지 유입
8. 중문 README 기본 제공 — 중국어권 커뮤니티의 대규모 확산

---

## 9. 로컬 에이전트 구축에 주는 도움

1. **에이전트가 호출할 도구로서 이상적** — REST + task_id 폴링 구조라 장시간 작업에 적합
2. **Skill 설계 교과서** — `SKILL.md`의 배울 점
   - description에 트리거 조건을 촘촘히 나열
   - “Required Behavior”로 에이전트의 나쁜 습관을 명시적으로 금지
     (sleep 폴링 금지, 성공 후 전체 로그 읽기 금지, API 키 출력 금지 등)
   - **Exit code 분기**: `0`=성공, `10`=자격증명 필요 → 사용자에게 딱 한 번만 질의
   - `MPT_RESULT` / `VIDEO_FILE=` 같은 **기계 파싱용 마커** 출력
3. **완전 로컬 스택 예제** — Ollama + Edge TTS + Whisper + ffmpeg 조합

### 추천 로드맵
1주차 MCP 서버 래퍼 → 2주차 n8n 스케줄 자동화 → 3주차 텔레그램/디스코드 봇 → 4주차 품질 평가 에이전트

---

## 10. React / PHP로 만들 수 있는가

**전체 재작성은 비권장**입니다. moviepy/ffmpeg 합성, faster-whisper, edge-tts 모두
Python 생태계 의존도가 높고, 25,000줄 규모를 재작성하면 업스트림 업데이트를 따라갈 수 없습니다.

**권장: 래핑(Wrapping) 아키텍처**

```
React (Next.js)  →  PHP / Laravel (회원·결제·크레딧)  →  MoneyPrinterTurbo REST API (Python)
```

### React 예시
```js
const res = await fetch('http://localhost:8080/v1/videos', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'x-api-key': KEY },
  body: JSON.stringify({ video_subject: subject, video_aspect: '9:16' }),
});
const { data: { task_id } } = await res.json();

const timer = setInterval(async () => {
  const r = await fetch(`http://localhost:8080/v1/tasks/${task_id}`, { headers: { 'x-api-key': KEY } });
  const { data } = await r.json();
  setProgress(data.progress);
  if (data.state === 1) { setVideos(data.videos); clearInterval(timer); }
}, 3000);
```

### Laravel 예시
```php
$res = Http::withHeaders(['x-api-key' => config('mpt.key')])
    ->post(config('mpt.url').'/v1/videos', [
        'video_subject' => $request->subject,
        'video_aspect'  => '9:16',
    ]);
$taskId = $res->json('data.task_id');
```

> React 프론트를 붙일 때는 `CORS_ALLOWED_ORIGINS=http://localhost:3000` 설정이 필요합니다.

---

## 11. 수익화 아이디어

> 원칙: **영상 자체가 아니라 “영상 만드는 수고”를 판다.**

### 아이디어 1. SaaS (숏폼 자동 생성 웹서비스)

| 플랜 | 가격/월 | 제공 | 추정 원가 | 마진 |
| --- | --- | --- | --- | --- |
| Free | 0원 | 3개 (워터마크) | ~300원 | 유입용 |
| Basic | 9,900원 | 30개 | ~3,000원 | ~70% |
| Pro | 29,900원 | 150개 + 자동 업로드 | ~15,000원 | ~50% |
| Team | 99,000원 | 무제한급 + API 키 | ~40,000원 | ~60% |

원가 근거: 대본 LLM 10~50원 + 스톡 영상 0원 + Edge TTS 0원 + 인코딩 서버 50~200원
→ **영상 1개당 약 100원 내외**. (RecCloud가 이 프로젝트 기반 서비스를 운영 중 = 시장 검증됨)

### 아이디어 2. 영상 제작 대행 (가장 빠른 현금화)

| 상품 | 가격 | 실작업 시간 |
| --- | --- | --- |
| 숏폼 10개 패키지 | 15만원 | 약 30분 (배치 + 검수) |
| 월 구독 (주 3개) | 30만원/월 | 주 1시간 |
| 브랜딩 커스텀 | +10만원 | 초기 1회 |

타겟: 자영업, 부동산, 헬스장, 병원, 학원, 쇼핑몰. 시장 단가는 편집 1건당 3~10만원.

### 아이디어 3. 니치 채널 운영 (패시브 인컴)
영어 회화 / 역사 상식 / 재테크(CPM 최상위) / 명언 / 건강 정보 등 5~10개 채널 동시 운영.
수익원: 유튜브 쇼츠 수익 + 틱톡 펀드 + **제휴 마케팅(쿠팡파트너스 등)**.
`upload_post.py`로 생성→업로드 완전 자동화 가능.

### 아이디어 4. 스폰서 / 레퍼럴 모델
원본 README에 스폰서 10곳 이상, 링크마다 `?aff=` `?ref=` 레퍼럴 파라미터가 붙어 있음.
한국어 특화 가이드/강좌 트래픽에 API 플랫폼 레퍼럴을 붙이면 순수익화 가능.

### 아이디어 5. 정보 상품 (마진 100%)

| 상품 | 가격대 |
| --- | --- |
| 전자책 “AI 숏폼 자동화 완전정복” | 2~3만원 |
| 온라인 강의 (2시간) | 5~10만원 |
| 원클릭 한국어 세팅 패키지 | 3~5만원 |
| 유료 커뮤니티 | 월 1~3만원 |

### 아이디어 6. 한국어 특화 포크
한글 폰트 내장(현재 `resource/fonts`에 한글 폰트 없음), 한국어 TTS 프리셋,
저작권 프리 BGM 교체, 한국 스톡 소스 연동, 네이버 클립/카카오 지원.

### 추천 실행 순서

| 단계 | 기간 | 내용 | 기대 수익 |
| --- | --- | --- | --- |
| 1 | 1~2주 | 로컬 세팅 + 영상 30개 품질 검증 | 0원 (학습) |
| 2 | 3~4주 | 대행 서비스 오픈 | 월 50~200만원 |
| 3 | 2~3개월 | 니치 채널 3개 + 제휴 마케팅 | 월 30~100만원 |
| 4 | 3~6개월 | React + Laravel SaaS | 월 100만원~ |
| 5 | 병행 | 전자책/강의 + 레퍼럴 | 월 20~50만원 |

---

## 12. 수익화 전 필수 체크리스트

- [ ] **배경음악 전량 교체** — `resource/songs`는 YouTube 출처이며 README에 저작권 경고가 있음
- [ ] Pexels / Pixabay 라이선스 조건 확인 (상업적 사용 가능하나 조건 존재)
- [ ] 유튜브·틱톡의 **AI 생성 콘텐츠 표기** 정책 준수
- [ ] 유튜브 “반복성 콘텐츠(repetitious content)” 정책 대비 — 사람 검수 단계 필수
- [ ] 서버 배포 시 `[app] api_key` 설정으로 API 보호
- [ ] MIT 라이선스 저작권 고지 유지

---

## 13. 알려진 한계

1. 스톡 영상 조합이라 대본과 그림이 100% 일치하지는 않음 (`match_materials_to_script` 옵션 존재)
2. 영상 1개 생성에 5~20분 (CPU 인코딩 병목)
3. 한국어는 UI 번역(`ko.json`)은 있으나 TTS/폰트 최적화는 직접 튜닝 필요
4. 기본 배경음악의 저작권 리스크
5. Python 3.11+ 및 ffmpeg 필수, Windows 경로에 한글/공백/특수문자 사용 시 오류

---

## 14. 자주 겪는 오류

| 오류 | 해결 |
| --- | --- |
| `RuntimeError: No ffmpeg exe could be found` | ffmpeg 설치 후 `[app] ffmpeg_path` 지정 |
| `OSError: [Errno 24] Too many open files` | `ulimit -n 10240` |
| Whisper 모델 다운로드 실패 | Hugging Face에서 수동 다운로드 후 `./models/whisper-large-v3`에 배치 |

---

_작성: Claude Code 세션 대화 정리본_
