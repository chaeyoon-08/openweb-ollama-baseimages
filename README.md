# gcube OpenWebUI + Ollama 멀티컨테이너

Ollama(LLM 서빙)와 OpenWebUI(채팅 UI)를 gcube 플랫폼에 멀티컨테이너로 배포하기 위한 커스텀 Docker 이미지 레포지토리.

---

## 레포지토리 구조

```
.
├── .github/workflows/
│   ├── build-ollama.yml       ← Ollama 이미지 빌드 + GHCR push / 캐시 없음 (항상 최신 ollama:latest pull)
│   └── build-openwebui.yml    ← OpenWebUI 이미지 빌드 + GHCR push / GHA 캐시 활용 (PyTorch 빌드 최적화)
├── ollama/
│   ├── Dockerfile
│   ├── entrypoint.sh          ← ollama serve → OLLAMA_MODEL 공백 구분 파싱 → 순서대로 모델 pull
│   └── README.md              ← Ollama 이미지 상세 (환경변수, 추천 모델, GPU 호환)
├── openwebui/
│   ├── Dockerfile
│   └── README.md              ← OpenWebUI 이미지 상세 (환경변수, 마운트 경로)
└── CLAUDE.md
```

---

## 이미지

| 이미지 | 태그 | GHCR 주소 |
|--------|------|-----------|
| Ollama 커스텀 | `latest` | `ghcr.io/data-alliance/ollama-custom:latest` |
| OpenWebUI 커스텀 | `latest` | `ghcr.io/data-alliance/openwebui-custom:latest` |

> 두 패키지 모두 **Public** 설정 필수 — gcube에서 인증 없이 pull 가능해야 함.

---

## 이미지 구성

### Ollama (`ollama/`)

| 항목 | 내용 |
|------|------|
| 베이스 이미지 | `ollama/ollama:latest` |
| 추가 패키지 | `curl` (헬스체크 대기 루프용) |
| ENTRYPOINT | `/entrypoint.sh` |

**entrypoint.sh 동작 순서**

```
1. ollama serve 백그라운드 기동
2. curl로 :11434 헬스체크 — Ready 될 때까지 2초 간격 대기 (최대 60초)
3. OLLAMA_MODEL 환경변수를 공백 기준으로 파싱 → 순서대로 ollama pull
4. 모든 모델 준비 완료 → 서버 foreground 유지
```

> `OLLAMA_MODEL`이 비어있으면 모델 pull 없이 서버만 기동.

**주요 환경변수**

| KEY | 설명 |
|-----|------|
| `OLLAMA_HOST` | `0.0.0.0` 고정 (gcube 내부 통신 허용) |
| `OLLAMA_MODEL` | 공백 구분으로 복수 모델 지정 가능 |

---

### OpenWebUI (`openwebui/`)

| 항목 | 내용 |
|------|------|
| 베이스 이미지 | `ghcr.io/open-webui/open-webui:main` |
| 한국어 로케일 | `ko_KR.UTF-8` 생성, `fonts-nanum` |
| OCR | `tesseract-ocr` + `tesseract-ocr-kor` + `tesseract-ocr-eng` |
| PyTorch | CUDA 12.8 빌드 — RTX 50 Blackwell (CC 12.0) 호환 |
| 데이터 디렉토리 | `mkdir -p /workplace/service_v1/data /workplace/service_v1/uploads` (Dockerfile에서 사전 생성) |

> 별도 시작 스크립트 없음 — 베이스 이미지 기본 CMD 그대로 사용.

**주요 환경변수**

| KEY | 기본값 | 설명 |
|-----|--------|------|
| `DATA_DIR` | `/workplace/service_v1/data` | DB 및 설정 저장 경로 |
| `UPLOAD_DIR` | `/workplace/service_v1/uploads` | 업로드 파일 저장 경로 |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama 연동 주소 |
| `DEFAULT_MODELS` | — | UI 기본 선택 모델 |
| `WEBUI_SECRET_KEY` | — | JWT 서명 키, 재시작 후 세션 유지 필수 |

---

## 빌드 파이프라인

### 자동 빌드

`main` 브랜치에 push 시 경로 필터에 해당하는 워크플로우가 자동 실행됩니다.

| 워크플로우 | 트리거 경로 | 캐시 전략 | 빌드 소요 시간 |
|-----------|-----------|----------|-------------|
| `build-ollama.yml` | `ollama/**`, `.github/workflows/build-ollama.yml` | 캐시 없음 (항상 최신 ollama:latest pull) | 약 10분 |
| `build-openwebui.yml` | `openwebui/**`, `.github/workflows/build-openwebui.yml` | GHA 캐시 활용 (PyTorch 빌드 최적화) | 약 20~30분 |

### 수동 빌드 (workflow_dispatch)

코드 변경 없이 최신 베이스 이미지만 다시 가져오고 싶을 때 사용합니다.
특히 **Ollama가 신규 모델을 지원하지 않을 경우** 수동 빌드로 최신 Ollama를 반영할 수 있습니다.

| 단계 | 내용 |
|------|------|
| 1 | GitHub 레포 → **Actions** 탭 |
| 2 | 대상 워크플로우 선택 |
| 3 | **Run workflow** 클릭 |

### 이미지 공개 설정

gcube에서 인증 없이 이미지를 pull하려면 패키지를 **Public**으로 설정해야 합니다.

| 단계 | 내용 |
|------|------|
| 1 | GitHub 레포 → 우측 사이드바 **Packages** 클릭 |
| 2 | `ollama-custom` 또는 `openwebui-custom` 선택 |
| 3 | **Package settings → Change visibility → Public** |
| 4 | 두 패키지 모두 반복 |

---

## Git Remote

| Remote | 주소 | 용도 |
|--------|------|------|
| `origin` | `https://github.com/chaeyoon-08/openweb-ollama-baseimages.git` | 개발용 |
| `gcube` | `https://github.com/Data-Alliance/ollama-webui-multi.git` | 배포용 |
