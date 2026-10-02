## 회랑 GraphRAG 파이프라인

학습 자료(한국사 교과서, 통계 교안 등)를 1인칭 3D 기억의 궁전으로 만드는 백엔드.
자료의 목차를 LLM이 만들고 그 섹션을 "방"으로 써서 개념을 배정·선별한 뒤,
GraphRAG 인덱스 위에서 질의(RAG)와 쇼케이스 팰리스를 서빙한다.

## 레포 구조

| 폴더 | 역할 |
|---|---|
| `backend/` | 서빙(통합 FastAPI). `app`(진입점) + `serve`(RAG 질의) + `showcase`(팰리스·이미지) + `query`(엔진) |
| `orchestrator/` | 업로드 → 인덱싱 → palace → 등록 자동 파이프라인(라이브 인제스트) |
| `palace/` | 방 빌드 정본. TOC → 방 배정/선별 → `palace.json` (`run.py`, `build_rooms.py`, `configs/`, `tests/`) |
| `indexing/` | 도메인별 GraphRAG 인덱싱 config (`<domain>/settings.yaml` + 튜닝 프롬프트, `_template.settings.yaml`) |
| `preprocessing/` | 원본 PDF → 정제 코퍼스 (`pipeline_v2.py`, `normalize.py`, `source/`, `result/`) |
| `snapshots/` | 빌드된 GraphRAG 인덱스 (`korean_history`=국사 골든, `statistics`) |
| `deliverables/` | 프론트가 읽는 도메인별 산출물 (`<domain>/palace.json`, `palace_with_images.json`, `images/`) |
| `docs/` | 분석 문서 (`RUNBOOK.md`, `EXPERIMENTS.md`, `CONVENTIONS.md`, `audit/`) |
| `input/` | 도메인별 정제 코퍼스 (저작물, gitignored) |
| `archive/` | 동결된 실험·분석 (`ARCHIVED.md` 표시, 새 작업 금지) |
| `prompts/` | 공유 프롬프트(검색 + 기본 인덱싱). 루트 `settings.yaml`은 쿼리 config |

## 환경 준비

1. Python 3.13 + venv:
   ```
   python -m venv .venv
   .venv\Scripts\activate
   pip install -r requirements.txt
   ```
2. `.env.example`을 `.env`로 복사하고 키를 채운다(.env는 gitignored):
   - `GRAPHRAG_API_KEY`, `GRAPHRAG_API_BASE`: 쿼리·인덱싱(Azure OpenAI)
   - `CONTENT_UNDERSTANDING_*`, `OPEN_AI_*`: 전처리(`preprocessing/pipeline_v2.py`)에서만 필요
3. 기본 스냅샷 `snapshots/korean_history/`(국사, 357 엔티티)가 있어야 골든 검증 재현 가능.

## 백엔드 실행 + API

```
gunicorn backend.app:app --worker-class uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
# 개발:  uvicorn backend.app:app --reload
# App Service: "Startup Command" = bash startup.sh
# var 위치는 ORCH_VAR_DIR 로 바꿀 수 있으나 /home(Azure Files/SMB)은 쓰지 말 것:
#   스냅샷 lancedb 가 SMB 에서 0바이트로 깨진다(palace rooms·RAG load 실패). 기본값
#   REPO/var(로컬)로 두고, 재시작 생존이 필요하면 Blob 영속을 따로 붙인다(예정).
```

서빙 도메인: `korean_history`, `statistics`.

| 메서드 · 경로 | 용도 |
|---|---|
| `GET /palace/{name}` | 쇼케이스 팰리스 JSON(이미지 포함) |
| `GET /images/{name}/{file}` | 팰리스가 참조하는 그림 |
| `POST /query` | RAG 질의. body `{"question": "...", "snapshot": "korean_history"}` → `{"answer", "snapshot"}` |
| `GET /health` · `/ready` | 헬스 / 스냅샷 준비 여부 |
| `POST /orchestrator/upload?filename=&domain=` | 파일(raw body) 업로드 → 자동 체인 → `job_id` |
| `GET /orchestrator/jobs/{id}/status` · `/palace` | 잡 진행 / 산출 팰리스(이미지 매칭 시 노드에 `images[]` 포함) |
| `GET /orchestrator/jobs/{id}/toc` | 조기 LLM 목차(방 생성 전, `toc_ready` 시). 둘러보기 페이지용 |
| `GET /orchestrator/jobs/{id}/images/{file}` | 라이브 잡이 매칭한 그림(노드 `images[].path` = `images/<file>`) |
| `POST /jobs/{id}/query` | 라이브 잡 RAG 질의 |

```bash
# 쇼케이스 팰리스 (프론트)
curl http://127.0.0.1:8000/palace/korean_history

# RAG 질의
curl -X POST http://127.0.0.1:8000/query \
  -H 'Content-Type: application/json' \
  -d '{"question":"조선 전기의 통치 제도를 설명해줘.","snapshot":"korean_history"}'

# 라이브 업로드 → 자동 인덱싱·palace (이후 /orchestrator/jobs/<id>/status 폴링)
curl -X POST "http://127.0.0.1:8000/orchestrator/upload?filename=corpus.txt&domain=statistics" \
  --data-binary @input/statistics/corpus.txt
```

## 프론트 연동 노트

라이브 잡과 쇼케이스는 경로 규칙이 다르다. 아래를 그대로 따른다.

**경로 prefix 주의**
- 잡 상태/팰리스/이미지는 `/orchestrator/jobs/{id}/...`.
- 라이브 RAG 질의만 prefix 없는 루트 `/jobs/{id}/query` (serve가 `/`에 마운트). `/orchestrator/jobs/{id}/query` 가 아니다.

**노드 이미지 → 실제 URL.** 노드의 `images[].path` 는 라이브·쇼케이스 모두 `images/<file>` 형식(끝의 파일명만 의미). 서빙 URL은 출처별로 조립한다:
- 쇼케이스: `/images/{name}/<file>` (예: `images/fig_10_2.png` → `/images/korean_history/fig_10_2.png`)
- 라이브 잡: `/orchestrator/jobs/{id}/images/<file>` (예: `/orchestrator/jobs/<job_id>/images/fig_6_3_cv_1.png`)
- 즉 둘 다 `<file>`(basename)만 떼어 각자 base에 붙인다.

**잡 상태 폴링** `GET /orchestrator/jobs/{id}/status` 응답: `{job_id, state, toc_ready, palace_ready, rag_ready, domain, showcase_key, run_id, error, created_at, updated_at, progress}`.
- `state`: `QUEUED → PREPROCESSING → TOC_READY → INDEXING → BUILDING_PALACE → PALACE_READY → DONE` (실패 시 `FAILED`, `error`에 사유). `RAG_READY`는 DONE 직전의 순간 상태.
- 게이팅: `toc_ready=true` 면 `/orchestrator/jobs/{id}/toc`(방 생성 전 목차 미리보기), `palace_ready=true` 면 `/orchestrator/jobs/{id}/palace`, `rag_ready=true`(=DONE) 면 `/jobs/{id}/query`.

**로딩 바** 응답의 `progress` 를 그대로 쓴다(`state`+플래그에서 파생):
```json
"progress": {
  "percent": 25,
  "current_step": "indexing",
  "steps": [
    {"key":"preprocess","label":"전처리","weight":25,"est_seconds":90,"status":"done"},
    {"key":"indexing","label":"인덱싱","weight":55,"est_seconds":280,"status":"active"},
    {"key":"rooms","label":"방 생성","weight":20,"est_seconds":60,"status":"pending"}
  ]
}
```
- `status`: `pending | active | done | failed`. 스텝퍼는 이걸 그대로 그리면 된다.
- `percent`: 완료 step 가중치 합(서버는 단계 **내부** 진행률을 모름). 긴 인덱싱 구간은 `active` step 의 `est_seconds` + `updated_at`(그 state 진입 시각)으로 프론트가 보간해 바를 움직인다.
- 텍스트 추출·정제는 한 백엔드 단계라 `preprocess`(전처리)로 합쳐 있다. 목차 완료는 별도 `toc_ready` 플래그로 표시(원하면 그 사이 "목차 준비됨" 표기).

구현 레시피:
1. 업로드 응답의 `job_id` 로 `GET /orchestrator/jobs/{id}/status` 를 2~3초 간격 폴링(`state` 가 `DONE`/`FAILED` 면 중단).
2. 스텝퍼는 `progress.steps[].status` 를 그대로 렌더. 바 기본값은 `progress.percent`.
3. 긴 `active` 단계(인덱싱)에서 바가 멈춰 보이지 않게, 그 단계만 경과시간으로 보간한다:
```js
// status = 방금 받은 progress, enteredAt = 그 state 진입 시각(updated_at)
const active = status.steps.find(s => s.status === "active");
let pct = status.percent;
if (active) {
  const elapsed = (Date.now() - new Date(enteredAt)) / 1000;
  const frac = Math.min(elapsed / active.est_seconds, 0.95); // 끝까진 안 채움
  pct += active.weight * frac;
}
// pct 로 바를 그리되, 다음 폴링에서 단계가 넘어가면 그 단계 weight 까지 차오른다.
```
4. `FAILED` 면 `failed` 인 step + `error` 를 노출.

**조기 목차** `GET /orchestrator/jobs/{id}/toc` → LLM 목차 JSON `{..., "sections":[{"name", "start_marker", ...}]}`. 인덱싱(방 생성)과 분리돼 먼저 생성되므로 PDF 업로드 직후 둘러보기 페이지에서 목차를 보여줄 수 있다. `toc_ready=true` 전엔 409.

**질의 본문/응답**
- `POST /query` body: `{"question": str, "snapshot": str, "method"?: "auto"|"local"|"global"}`. `snapshot` 누락 시 400.
- `POST /jobs/{id}/query` body: `{"question": str, "method"?: ...}` (snapshot 불필요, job_id가 path).
- 응답(둘 공통): `{"answer", "snapshot", "mode", "sources"}`. `sources` = `{reports:[{id,title}], entities:[{id,title,type,degree,description,provenance}], entities_total}`. 라우터가 고른 모드의 인용 종류에 맞춰 엔티티(노드)로 변환하며, 각 엔티티에 **`provenance`**가 붙는다: `cited`(local — 답변이 직접 인용한 엔티티/관계, **정확**) · `chunk`(basic — 인용 청크 `Sources`의 엔티티) · `related`(global — 인용 커뮤니티 `Reports` 구성 개념을 degree 상위로 펼친 **근사**). 상위 N은 `RAG_SOURCE_MAX_ENTITIES`(기본 12), `cited` 우선 정렬. `reports`는 실제 인용된 커뮤니티 리포트(global). 인용 없으면 `sources`는 `null`. 프론트는 `entities[].title`로 방과 잇는다.

**업로드 제한.** `POST /orchestrator/upload` 본문 30MB 초과 시 `413`, 빈 본문 `422`.

## 도메인 추가 (큐레이션 쇼케이스)

```
preprocessing/source/<domain>.pdf                              # 원본(gitignored)
python -m preprocessing.pipeline_v2 --pdf preprocessing/source/<domain>.pdf   # 정제 -> result/<domain>_vN
python preprocessing/normalize.py --result preprocessing/result/<domain>_vN --domain <domain>
                                                               # -> input/<domain>/{corpus.txt, captions.md, pagesplit.txt, images/}
graphrag index --root indexing/<domain>                        # -> output/<domain>, snapshots/<domain> 로 복사(캡처)
python -m palace.run --config palace/configs/<domain>.json --phase toc   # LLM TOC 검토용
python -m palace.run --config palace/configs/<domain>.json --phase rooms # -> palace.json
python -m palace.match_images --palace deliverables/<domain>/palace.json \
  --snapshot snapshots/<domain> \
  --figures-json preprocessing/result/<domain>_vN/meta/figures.json \
  --pagesplit preprocessing/result/<domain>_vN/txt/content_paged.txt \
  --out-dir deliverables/<domain>
# figures.json 단일 소스로 매칭(STEP4 분리 자식 fig_p_i_cv_k 도 캡션 달고 흐름).
# -> deliverables/<domain>/{palace_with_images.json, unplaced_figures.json, images/}
# 마지막: backend/showcase.py SHOWCASE_PALACES 와 backend/serve.py SNAPSHOTS 에 도메인 등록
```

신규 인덱싱 config는 `indexing/_template.settings.yaml`을 복사해 `<domain>` 경로만 수정.
라이브 사용자 업로드는 이 수동 경로가 아니라 orchestrator(`/upload`, `var/jobs/`)가 자동 처리.

## 정본 검증 (국사 골든)

```
python -m palace.run --config palace/configs/korean_history.json --phase rooms
python palace/tests/compare_golden.py --run-id korean_history   # 캐시 히트 시 byte-identical
```
국사는 `assignment: chunk_overlap`으로 핀(골든 동작 보존), 신규 도메인은 `fine_pos` 기본.

## 더 보기

- 전체 API 명세(프론트 연동 정본): [docs/API.md](./docs/API.md)
- 정본 방 빌드 상세: [palace/README.md](./palace/README.md)
- 재현 절차: [docs/RUNBOOK.md](./docs/RUNBOOK.md)
- 실험 누적 narrative: [docs/EXPERIMENTS.md](./docs/EXPERIMENTS.md)
- 규약: [docs/CONVENTIONS.md](./docs/CONVENTIONS.md)

## 단발 개발 문의

이 저장소는 공개 프로젝트 예시이며 유료 고객 납품 사례가 아닙니다. 문서 전처리, 검색 결과의 근거 추적, 재현 가능한 RAG 오류 수정, FastAPI 연동처럼 범위가 분명한 **유료 단발 개발**을 문의하려면 [LinkedIn 프로필](https://www.linkedin.com/in/hyoseok-oh-791530263/)로 연락해 주세요.

현재 입력 자료의 형태, 기대하는 결과, 재현 가능한 오류 또는 최소 예시, 원하는 일정과 예산을 알려 주시면 먼저 작업 범위를 검토하겠습니다. 고객 데이터와 비밀키는 보내지 말고 비식별 샘플을 사용해 주세요. 범위와 비용은 자료를 확인한 뒤 개별 협의합니다.
