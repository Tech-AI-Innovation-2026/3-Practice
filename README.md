# RAG Retrieval Live Evaluation

이번 실습은 `web-` repo에 제출된 RAG endpoint를 교수자 평가 서버가 주기적으로 호출하고, GitHub Issue 리더보드를 갱신하는 방식으로 진행합니다.

목표는 LLM 답변 생성이 아닙니다. **검색 대상 문서를 잘 ingest하고, 질문과 관련된 원본 `doc_id`를 상위 rank에 올리는 retrieval 성능**을 평가합니다.

## 중요: 데이터가 추가되었습니다

기존 SciFact corpus에 challenge 문서가 추가되었습니다. 기존 index를 그대로 쓰면 새 평가 문서를 검색할 수 없습니다.

반드시 최신 `3-Practice`를 pull한 뒤 **`data/scifact/corpus.jsonl` 전체를 다시 ingest**하세요.

추가된 공개 데이터:

- `data/scifact/학칙.hwp`
  - `corpus.jsonl`에는 `text`가 비어 있고 `filename`으로 연결됩니다.
  - HWP 파일을 직접 읽어 ingest해야 합니다.
- `data/scifact/python-sdk-src/src/mcp/**/*.py`
  - `modelcontextprotocol/python-sdk` 일부 source snapshot입니다.
  - 각 Python source file이 하나의 평가 문서입니다.
  - `corpus.jsonl`에는 `text`가 비어 있고 `filename`으로 연결됩니다.

평가 질문과 정답은 공개하지 않습니다. 공개되는 것은 검색 대상 문서뿐입니다.

## 평가 대상

- 대상 repo: 본인의 `web-` prefix GitHub Classroom repo
- 대상 branch: `main`
- 제출 파일: repo root의 `rag_endpoint.json`
- 제출 코드: RAG 서버, ingest, index 생성, retrieval 구현 코드
- 평가 endpoint: `GET /health`, `POST /retrieve`
- 평가 metric: `nDCG@10`
- 평가 주기: 약 10분마다 교수자 Windows 평가 서버가 실행
- 공개 결과: 이 repo의 GitHub Issue 리더보드

## 데이터

ingest 대상 데이터는 이 `3-Practice` repo에 포함되어 있습니다.

```text
data/scifact/corpus.jsonl
```

학생 RAG 시스템에는 **`data/scifact/corpus.jsonl` 전체**를 넣으세요.

`corpus.jsonl`은 JSON Lines 형식이며, 한 줄이 문서 하나입니다. 기본 SciFact 문서는 `title`과 `text`가 바로 들어 있습니다.

```json
{
  "_id": "31715818",
  "title": "...",
  "text": "..."
}
```

일부 challenge 문서는 `text`가 비어 있고 `filename`이 들어 있습니다. 이 경우 `data/scifact/` 아래의 해당 파일을 직접 읽어서 ingest해야 합니다.

```json
{
  "_id": "mcp_python_sdk_src_3eb5799:src/mcp/client/session.py",
  "title": "modelcontextprotocol/python-sdk src/mcp/client/session.py",
  "text": "",
  "filename": "python-sdk-src/src/mcp/client/session.py"
}
```

여기서 `_id`가 평가에 쓰는 원본 `doc_id`입니다. chunking은 자유지만, 모든 chunk metadata에 원본 `doc_id`를 반드시 보존해야 합니다.

평가 서버는 비공개 평가 질문과 정답을 사용합니다. 학생은 어떤 질문이 들어올지 모른다고 가정해야 합니다. 특정 `query_id`에 대한 정답 `doc_id`를 하드코딩해서 반환하는 방식은 실습 취지에 맞지 않습니다.

## 필수 구현

### `GET /health`

RAG 서버와 index가 준비되었는지 확인합니다.

응답 예:

```json
{
  "status": "ok",
  "ready": true
}
```

### `POST /retrieve`

요청 예:

```json
{
  "query_id": "eval_001",
  "question": "question text from the evaluator",
  "top_k": 10
}
```

응답 예:

```json
{
  "query_id": "eval_001",
  "contexts": [
    {
      "doc_id": "31715818",
      "chunk_id": "31715818::chunk_003",
      "score": 0.91,
      "text": "retrieved chunk text"
    }
  ]
}
```

필수 조건:

- `contexts`는 JSON list여야 합니다.
- 각 context에는 원본 문서 ID인 `doc_id`가 있어야 합니다.
- `doc_id`는 `corpus.jsonl`의 `_id`와 정확히 같아야 합니다.
- 각 context에는 numeric `score`가 있어야 합니다.
- `score`는 높을수록 더 관련 있는 문서라는 의미여야 합니다.
- `text`는 디버깅을 위해 포함하세요.
- `chunk_id`는 권장 필드입니다.

평가 서버는 `score` 기준 내림차순으로 context rank를 만든 뒤, 같은 `doc_id`는 첫 등장만 유지하고, 문서 단위 ranked list로 바꿔 `nDCG@10`을 계산합니다.

## `rag_endpoint.json` 제출

본인 `web-` repo root에 아래 파일을 만들고 commit/push하세요.

```json
{
  "api_base_url": "https://YOUR-TUNNEL.trycloudflare.com"
}
```

주의:

- 파일명은 정확히 `rag_endpoint.json`이어야 합니다.
- `main` branch에 push해야 합니다.
- `api_base_url` 끝에 `/health`나 `/retrieve`를 붙이지 마세요.
- Cloudflare URL이 바뀌면 이 파일을 수정해서 다시 push하세요.

## Cloudflare Quick Tunnel

RAG API 서버는 Docker 또는 로컬에서 실행해도 됩니다. 서버는 외부에서 접근 가능해야 하며, Docker 안에서는 `0.0.0.0`에 bind해야 합니다.

예:

```bash
uvicorn app:app --host 0.0.0.0 --port 8000
```

host PC에서 Cloudflare tunnel을 실행합니다.

```bash
cloudflared tunnel --url http://127.0.0.1:8000
```

Windows PowerShell:

```powershell
.\cloudflared.exe tunnel --url http://127.0.0.1:8000
```

Cloudflare URL은 `cloudflared` 프로세스가 살아 있는 동안 유지됩니다. `cloudflared`를 끄거나 PC를 재부팅하면 URL이 바뀝니다.

## 직접 확인

`localhost`가 아니라 Cloudflare public URL로 확인하세요.

```bash
curl https://YOUR-TUNNEL.trycloudflare.com/health
```

```bash
curl -X POST https://YOUR-TUNNEL.trycloudflare.com/retrieve \
  -H "Content-Type: application/json" \
  -d '{"query_id":"selfcheck_001","question":"scientific claim text for testing","top_k":10}'
```

Windows PowerShell에서는 `curl.exe`를 쓰세요.

## 리더보드

교수자 Windows 평가 서버가 약 10분마다 다음 작업을 수행합니다.

1. `web-` repo 목록 조회
2. 각 repo의 `rag_endpoint.json` 읽기
3. `/health` 확인
4. 비공개 평가 질문으로 `/retrieve` 호출
5. `contexts[*].score` 기준으로 검색 결과 정렬
6. `contexts[*].doc_id`를 문서 rank로 변환
7. `nDCG@10` 계산
8. GitHub Issue 리더보드 갱신

서버가 꺼져 있거나 URL이 바뀌면 `latest_status`가 실패 상태로 표시됩니다. 현재 실습에서 이전에 성공한 최고 점수는 `best_score_so_far`로 보존됩니다.

`best_score_so_far`는 실습 단위로 관리됩니다. 새 실습이 시작되면 리더보드와 함께 초기화되며, 이전 실습의 최고 점수는 다음 실습으로 이어지지 않습니다.

## 주차별 최종 점수

각 주차의 리더보드는 별도 Issue로 백업합니다. 최종 rank는 주차별 `best_score_so_far`를 사용해 계산합니다.

discount factor는 `0.5`입니다. 최근 주차일수록 더 크게 반영합니다.

예를 들어 3개 주차를 모두 진행한 경우:

```text
final_score = 0.25 * week1_best + 0.5 * week2_best + 1.0 * week3_best
```

정규화 여부와 관계없이 rank 순서는 같습니다. 이 방식은 앞 주차 성과를 반영하되, 뒤 주차에서 크게 개선하면 역전할 수 있도록 설계한 것입니다.

## Discussion 평가

Discussion 활동은 실습 평가의 **15%**에 반영됩니다.

높은 상대점수를 받을 수 있는 활동 예시는 다음과 같습니다.

- 이번 실습의 검색 실패 원인에 대한 토의
- retrieval 성능을 높이는 ingest, chunking, metadata, reranking 방식에 대한 토론
- 현재 SciFact보다 실습 난이도를 높일 수 있는 데이터셋 또는 평가 문제 추천
  - 데이터셋 추천은 파싱 관점 또는 retrieval 관점에서 작성하세요. 파싱 관점이라면 기존 `corpus.jsonl`처럼 `title`과 `text`가 바로 주어지는 경우와 달리, `title`과 `text`가 비어 있고 `filename`만 주어졌을 때 해당 파일을 직접 파싱해서 ingest해야 하는 구조를 제안할 수 있습니다.
  - retrieval 관점이라면 query와 문서의 lexical gap, 긴 문서에서 근거 구간을 찾아야 하는 정도, 유사하지만 정답이 아닌 distractor 문서의 존재, metadata가 검색 성능에 주는 영향, nDCG@10 같은 평가 지표로 난이도를 측정할 수 있는지 등을 설명하세요.
- 답이 명확히 주어진 문제가 아니라, 직접 문제를 정의하고 평가 방법을 제안하는 글
- 다른 학생의 구현 방식에 대한 구체적인 질문과 개선 제안

AI 도구가 쉬운 답을 빠르게 만들어 주는 시대에는, 답이 없는 문제를 정의하고 풀어가는 능력이 중요합니다. 단순히 정답을 맞히는 것보다 어떤 문제가 중요한지, 어떤 데이터와 평가 방식이 그 문제를 잘 드러내는지 설명하는 참여를 높게 평가합니다.

## 코드 제출

본인 `web-` repo에는 평가 endpoint만 올리는 것이 아니라, 재현 가능한 구현 코드도 함께 올려야 합니다.

- RAG API 서버 코드
- corpus ingest 코드 또는 ingest 절차
- index 생성 코드 또는 index 로딩 코드
- retrieval 코드
- 실행에 필요한 dependency 파일
- 실행 방법을 적은 `README.md`

API key, token, 개인 경로 같은 secret은 repo에 올리지 마세요.

## 보고서

본인 `web-` repo의 `README.md` 하단에 아래 내용을 간단히 작성하세요.

```markdown
# Retrieval Report

## 실행 방식
- 제출 코드 위치:
- RAG 서버 실행 방식:
- 서버 실행 명령:
- Cloudflare Quick Tunnel URL:
- 사용한 retriever/index:

## 데이터 처리
- corpus ingest 방식:
- filename 기반 문서 처리 방식:
- HWP 처리 방식:
- Python source 파일 처리 방식:
- chunk size / overlap:
- doc_id 보존 방식:

## 인덱스와 검색
- embedding 모델:
- vector store/index:
- score 계산 방식:
- top_k 설정:

## 성능 개선 방법
- baseline에서 바꾼 점:
- 성능을 높이기 위해 시도한 방법:
- 효과가 있었던 방법:

## Self-check
- /health 결과:
- 자체 테스트 질문:
- 자체 테스트 검색 결과 top doc_id:
- 자체 테스트 검색 결과 top score:
- 실패한 점 / 개선할 점:
```

## 참고 구현 방향

최근 좋은 성능을 보인 구현들은 공통적으로 "답변 생성"보다 "정답 문서 ID를 상위에 올리는 retrieval 문제"에 집중했습니다. 아래 내용은 참고 방향이며, 그대로 복사할 정답 구현은 아닙니다.

- `corpus.jsonl`의 `_id`를 모든 검색 결과의 `doc_id`로 끝까지 보존합니다.
- `text`가 비어 있고 `filename`만 있는 문서는 실제 파일을 읽어서 ingest합니다.
- HWP, Python source, 일반 텍스트처럼 파일 형식이 다른 문서는 같은 방식으로 처리하지 말고 형식에 맞는 reader를 둡니다.
- 제목, 파일명, 경로, 코드 symbol, 문서 metadata처럼 검색에 도움이 되는 정보를 chunk metadata에 남깁니다.
- dense vector 검색만 쓰기보다 keyword 기반 검색을 함께 사용하면 짧은 질의, 고유명사, 함수명, 조항 번호 같은 질의에서 안정성이 좋아질 수 있습니다.
- 여러 검색 결과를 합친 뒤 재순위화하는 구조를 쓰면 유사하지만 정답이 아닌 문서가 앞에 오는 문제를 줄일 수 있습니다.
- chunk 단위로 검색하더라도 최종 응답에서는 같은 문서가 반복되지 않도록 문서 단위 ranking을 점검하세요.
- `/retrieve` 응답에는 `doc_id`, `chunk_id`, `score`, `text`를 명확히 포함하고, `top_k` 요청값을 지키세요.

성능 개선을 할 때는 먼저 실패한 query를 모아 어떤 유형에서 틀리는지 확인하세요. 예를 들어 keyword가 중요한 경우, 의미가 비슷한 문서가 많은 경우, 파일 파싱이 안 된 경우, metadata가 사라진 경우는 개선 방법이 서로 다릅니다.
