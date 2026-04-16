# ollama-custom

ollama/ollama:latest 기반 커스텀 이미지.
gcube 워크로드 환경변수로 주입한 모델명을 컨테이너 기동 시 자동으로 설치한다.

---

## 버전

| 항목 | 값 | 확인 기준 |
|------|-----|---------|
| Ollama | **v0.20.7** | 2026-04-16 기준 최신 |
| 베이스 이미지 | `ollama/ollama:latest` | 빌드 캐시 없음 — workflow_dispatch 시 항상 최신 pull |

> 이미지 빌드 시점에 따라 버전이 달라질 수 있습니다. 정확한 버전은 GitHub Actions 빌드 로그에서 확인하세요.

---

## 베이스 이미지

```
ollama/ollama:latest
```

---

## 추가된 내용

| 구분 | 내용 |
|------|------|
| 추가 패키지 | `curl` — 헬스체크 대기 루프에서 서버 Ready 확인용 |
| ENTRYPOINT | `/entrypoint.sh` — 모델 자동 pull 처리 |

---

## entrypoint.sh 동작 순서

```
1. ollama serve 백그라운드 기동
2. curl로 :11434 헬스체크 — Ready 될 때까지 2초 간격 대기 (최대 60초)
3. OLLAMA_MODEL 환경변수 값을 공백 기준으로 분리해서 순서대로 ollama pull
4. 모든 모델 준비 완료 → 서버 foreground 유지
```

> OLLAMA_MODEL이 비어있으면 모델 pull 없이 서버만 기동됨

---

## 환경변수

| KEY | 기본값 | 설명 |
|-----|--------|------|
| `OLLAMA_HOST` | `0.0.0.0` | 이미지 내 기본값. 모든 인터페이스 수신 (gcube 내부 통신 허용) |
| `OLLAMA_MODEL` | `""` | gcube 환경변수에서 주입. 설치할 모델명 |

**OLLAMA_MODEL 입력 예시**

```
# 단일 모델
gemma4:12b

# 복수 모델 (공백 구분)
gemma4:12b nomic-embed-text
```

---

## 포트

| 포트 | 용도 |
|------|------|
| `11434` | Ollama API (gcube 내부 통신 전용, 외부 노출 불필요) |

---

## GPU 호환

| GPU 세대 | Compute Capability | 최소 CUDA | 최소 드라이버 |
|----------|--------------------|-----------|---------------|
| RTX 40 시리즈 (Ada Lovelace) | 8.9 | 12.0 | 525+ |
| RTX 50 시리즈 (Blackwell) | 12.0 | 12.8 | 570+ |

ollama:latest 이미지가 런타임에 GPU를 감지하여 cuda_v12 / cuda_v13 백엔드를 자동 선택하므로 단일 이미지로 RTX 40/50 모두 동작한다.

---

## 예시 모델

> 아래는 참고용 예시입니다. 정확한 태그와 VRAM 요구량은 [ollama.com/library](https://ollama.com/library) 에서 확인하세요.
> VRAM 수치는 기본 quantization(Q4) 기준 대략값입니다.

| 모델 | 태그 예시 | VRAM | 특징 |
|------|---------|------|------|
| Gemma 4 | `gemma4:4b` | ~3 GB | Google — 멀티모달(텍스트+이미지), 128k 컨텍스트 |
| Qwen 3 | `qwen3:8b` | ~6 GB | Alibaba — 코딩·수학 강세, 하이브리드 추론 모드 |
| Gemma 4 | `gemma4:12b` | ~8 GB | 멀티모달 밸런스, 오픈소스 리더보드 상위권 |
| Qwen 3 | `qwen3:14b` | ~10 GB | 코딩·다국어·에이전틱 벤치마크 최상위권 |
| Phi 4 | `phi4:14b` | ~10 GB | Microsoft — 소형 고성능, 수학·추론 특화 |
| DeepSeek R1 | `deepseek-r1:14b` | ~12 GB | `<think>` 블록으로 추론 과정 실시간 노출 |
| Gemma 4 | `gemma4:27b` | ~20 GB | Gemma 4 고성능 버전 |
| Qwen 3 | `qwen3:30b-a3b` | ~20 GB | MoE 구조 — 30B 성능을 20GB VRAM으로 |
