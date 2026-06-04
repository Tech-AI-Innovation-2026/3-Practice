# 개발 명세서 — RAG 코드베이스에 MCP 서버 레이어 얹기

> **대상**: 이미 `main.py`로 RAG pipeline이 동작하는 코드베이스만 가진 상태에서, Claude Code/MCP 클라이언트가 바로 사용할 수 있는 MCP 서버 레이어를 *한 번에* 추가하기 위한 명세서.
> **결과**: 본 문서만 보고 구현하면, `~/.claude.json`에 단 한 줄 등록으로 `rag_query` / `list_documents` 툴이 동작한다.

---

## 0. 시작 전 가정 (Pre-conditions)

본 명세서는 다음이 이미 갖춰진 상태를 가정한다.

- 리포 루트에 다음이 존재
  - `main.py` — llama-index 기반 RAG pipeline 학습/실험 스크립트
  - `data/` — 인덱싱할 문서 폴더 (`*.txt`, `*.pdf`, `*.docx`, `*.xlsx`, `*.hwp`, `*.png`)
  - `requirements.txt` — llama-index 코어 + readers + embeddings 의존성
  - `Dockerfile` — `python:3.10-slim` 기반, `requirements.txt` 설치, `tesseract-ocr` 등 시스템 패키지 포함
- 임베딩은 **HuggingFace `BAAI/bge-m3`** (API key 불필요)
- LLM 호출은 *클라이언트가 책임*. 본 서버는 **retrieval만** 노출
- 컨테이너는 `docker compose`로 운용 (단일 서비스 `rag`)

---

## 1. 목표 아키텍처

### 1.1 한 그림

```
                ┌────────────────────────────────────────────┐
                │      Docker container: rag                 │
                │  (image: 본 리포 Dockerfile, restart:      │
                │   unless-stopped, volume: .:/app)          │
                │                                            │
                │  [main process]                            │
                │  python mcp_http_server.py                 │
                │   └─ FastMCP("RAG Practice")               │
                │      └─ transport="http", port=8000        │
                │      └─ in-memory VectorStoreIndex         │
                │                                            │
                │  [on-demand process, docker exec로 spawn]  │
                │  python mcp_server.py                      │
                │   └─ 같은 FastMCP, stdio transport         │
                │   └─ 자체 in-memory index (별도 캐시)      │
                │                                            │
                │  [on-demand process, 옵션]                 │
                │  python mcp_stdio_proxy.py                 │
                │   └─ http://127.0.0.1:8000/mcp 로 위임     │
                │      → HTTP 인덱스 공유, stdio로 노출       │
                └────────────────────────────────────────────┘

호스트의 ~/.claude.json은 위 세 진입점 중 하나를 등록.
```

### 1.2 transport 3종 분리 이유 (왜 한 파일 아닌가)

| 파일 | transport | 사용 시나리오 |
|---|---|---|
| `mcp_server.py` | stdio (직접) | Claude Code MCP 등록의 *가장 단순한 경로*. compose 컨테이너만 떠 있으면 됨. 단점: stdio 세션마다 인덱스 재빌드. |
| `mcp_http_server.py` | HTTP | 컨테이너 main process. 24/7 listen, 인덱스 1회 빌드 후 공유. 웹/원격 클라이언트 접속용. |
| `mcp_stdio_proxy.py` | stdio → HTTP 위임 | stdio만 받는 클라이언트인데 HTTP 인덱스를 공유하고 싶을 때. *옵션*. |

핵심: **동일한 `mcp` 객체**를 세 진입점이 공유한다 (tool 정의는 `mcp_server.py`에만 둔다).

---

## 2. 추가/수정해야 할 파일 목록

기존 리포에서 *새로 만들거나 수정*해야 할 항목은 단 6개.

| # | 파일 | 액션 | 라인 수 (참고) |
|---|---|---|---|
| 1 | `mcp_server.py` | **NEW** — tool 정의 + 인덱스 빌더 + stdio 진입 | ~70 |
| 2 | `mcp_http_server.py` | **NEW** — HTTP 진입 thin wrapper | ~7 |
| 3 | `mcp_stdio_proxy.py` | **NEW** — HTTP→stdio proxy thin wrapper (옵션) | ~12 |
| 4 | `requirements.txt` | **EDIT** — `fastmcp>=2.0.0` 추가 | +1 |
| 5 | `docker-compose.yaml` | **NEW/EDIT** — 단일 `rag` 서비스, `command: python mcp_http_server.py`, port 8000 | ~12 |
| 6 | `Dockerfile` | 기존 유지 — `CMD ["tail","-f","/dev/null"]`이면 compose의 `command:`가 덮어쓴다 | — |

`main.py`, `data/`, `.env`, `.env.example`는 *건드리지 않는다*.

---

## 3. 파일별 구현 명세

### 3.1 `mcp_server.py` (NEW, 핵심)

**책임**
- `FastMCP("RAG Practice")` 인스턴스 1개 생성 (모듈 최상위에 `mcp` 변수)
- llama-index `VectorStoreIndex`를 *lazy singleton* 패턴으로 빌드 (모듈 전역 `_index`)
- 두 개의 툴 `@mcp.tool()`로 등록
- `if __name__ == "__main__": mcp.run()` → fastmcp 기본 stdio 진입

**구조 (의사코드)**

```python
"""FastMCP server wrapping the RAG pipeline from main.py."""
import logging, os
from fastmcp import FastMCP

logging.basicConfig(level=logging.INFO)

# llama-index imports (Settings는 모듈 로드 시 한 번만 set)
from llama_index.core import SimpleDirectoryReader, VectorStoreIndex, Settings
from llama_index.core.node_parser import SimpleNodeParser
from llama_index.embeddings.huggingface import HuggingFaceEmbedding
from llama_index.readers.file import (
    DocxReader, HWPReader, ImageReader, PDFReader, PandasExcelReader, FlatReader,
)

mcp = FastMCP("RAG Practice")
_index = None

def get_index():
    global _index
    if _index is not None:
        return _index

    Settings.embed_model = HuggingFaceEmbedding(model_name="BAAI/bge-m3")

    # 확장자 → reader 매핑
    file_extractor = {
        ".txt":  FlatReader(),
        ".pdf":  PDFReader(),
        ".docx": DocxReader(),
        ".xlsx": PandasExcelReader(pandas_config={"header": None}),
        ".hwp":  HWPReader(),
        ".png":  ImageReader(parse_text=True, text_type="plain_text"),
    }

    data_dir = os.path.join(os.path.dirname(__file__), "data")
    documents = SimpleDirectoryReader(data_dir, file_extractor=file_extractor).load_data()

    parser = SimpleNodeParser.from_defaults(chunk_size=512, chunk_overlap=50)
    nodes  = parser.get_nodes_from_documents(documents)

    _index = VectorStoreIndex(nodes)
    return _index


@mcp.tool()
def rag_query(question: str, top_k: int = 3) -> str:
    """Ask a question against the indexed documents. Returns relevant passages."""
    index = get_index()
    retriever = index.as_retriever(similarity_top_k=top_k)
    results = retriever.retrieve(question)
    parts = []
    for i, r in enumerate(results, 1):
        src   = r.metadata.get("file_name", "?")
        score = f"{r.score:.4f}" if r.score is not None else "N/A"
        parts.append(f"[{i}] ({score}) {src}\n{r.text[:300]}")
    return "\n\n".join(parts) if parts else "No results found."


@mcp.tool()
def list_documents() -> str:
    """List all documents in the data/ folder."""
    data_dir = os.path.join(os.path.dirname(__file__), "data")
    if not os.path.exists(data_dir):
        return "data/ folder not found."
    files = sorted(os.listdir(data_dir))
    return "\n".join(f"- {f}" for f in files) if files else "No files in data/."


if __name__ == "__main__":
    mcp.run()
```

**구현 체크리스트**

- [ ] `FastMCP` 생성자 인자 = 사용자에게 노출될 서버 이름 (`"RAG Practice"`).
- [ ] `_index`는 *모듈 전역 변수*. 같은 프로세스 내 두 번째 호출부터는 캐시 사용.
- [ ] `Settings.embed_model = HuggingFaceEmbedding(...)`는 `get_index()` 안에서 — 모듈 import 시점에 무거운 모델 로드를 피한다.
- [ ] `SimpleDirectoryReader`는 `data_dir`를 *절대경로*로 받게 한다 (`os.path.dirname(__file__)`). 컨테이너 작업 디렉터리 차이로 깨지는 것을 방지.
- [ ] `PandasExcelReader`는 `pandas_config={"header": None}`으로 *헤더 가정 없이* 모든 행을 데이터로 처리.
- [ ] `ImageReader`는 `parse_text=True, text_type="plain_text"`로 OCR 결과를 플레인 텍스트화. Dockerfile에 `tesseract-ocr` 필수.
- [ ] 결과 포맷은 *문자열 1개*로 합쳐 반환. fastmcp는 dict/list도 받지만 LLM 가독성 위해 텍스트 포맷 권장.
- [ ] `r.score`가 `None`일 수 있다 (retriever 종류에 따라). 분기로 안전 처리.
- [ ] `r.text[:300]` — passage가 매우 길면 LLM 컨텍스트 낭비. 300자 절단을 *기본값*으로.

### 3.2 `mcp_http_server.py` (NEW, thin wrapper)

**책임**: HTTP transport 모드로 같은 `mcp` 인스턴스를 띄움. `0.0.0.0:8000`에 listen.

```python
"""HTTP entrypoint for the RAG Practice FastMCP server."""
from mcp_server import mcp

if __name__ == "__main__":
    mcp.run(transport="http", host="0.0.0.0", port=8000)
```

**체크리스트**
- [ ] `from mcp_server import mcp` 한 줄로 같은 서버 객체 재사용. *tool을 두 번 정의하지 말 것*.
- [ ] `host="0.0.0.0"` 필수 — 컨테이너 밖에서 접근하려면 `127.0.0.1`이면 안 된다.
- [ ] 포트는 `8000`. compose 포트 매핑과 일치시킨다.

### 3.3 `mcp_stdio_proxy.py` (NEW, 옵션)

**책임**: 이미 떠 있는 HTTP MCP 엔드포인트를 *stdio로 다시 노출*. 클라이언트가 stdio만 받지만 HTTP 인덱스를 공유하고 싶을 때 사용.

```python
"""STDIO proxy for clients that do not talk to HTTP MCP directly."""
from fastmcp import Client
from fastmcp.server import create_proxy

http_client = Client("http://127.0.0.1:8000/mcp")
proxy = create_proxy(http_client, name="RAG Practice STDIO Proxy")

if __name__ == "__main__":
    proxy.run()
```

**체크리스트**
- [ ] `127.0.0.1:8000`은 *컨테이너 내부* 주소. proxy 프로세스도 같은 컨테이너 안에서 spawn될 때만 동작.
- [ ] `create_proxy`의 `name` 인자는 디스플레이용. 등록 시 식별을 도와줌.
- [ ] HTTP 서버가 죽어 있으면 proxy는 즉시 connection error로 죽는다. *HTTP가 떠 있다는 전제* 하에서만 사용.

### 3.4 `requirements.txt` (EDIT)

기존 의존성 끝에 한 줄 추가.

```
fastmcp>=2.0.0
```

**주의**
- 기존에 `mcp` 패키지(공식 SDK)를 쓰고 있다면 *충돌하지 않는지* 확인. fastmcp 2.x는 내부에 mcp SDK를 포함하지만 별도 namespace를 사용한다.
- HWP/PNG 로더가 빠져 있으면 함께 추가: `olefile`, `Pillow`, `pytesseract`, `pypdf`, `docx2txt`, `openpyxl`, `pandas`, `sentence-transformers>=2.2.0`.

### 3.5 `docker-compose.yaml` (NEW/EDIT)

**책임**: 컨테이너 1개를 24/7 띄우고 HTTP MCP 서버를 기본 명령으로 실행. stdin/tty 열어둬서 *나중에 `docker exec`로 stdio 진입* 가능.

```yaml
services:
  rag:
    build: .
    container_name: rag_practice
    command: python mcp_http_server.py
    tty: true
    stdin_open: true
    volumes:
      - .:/app
    env_file:
      - .env
    ports:
      - "8000:8000"
    restart: unless-stopped
```

**필수 옵션 설명**

| 옵션 | 이유 |
|---|---|
| `command: python mcp_http_server.py` | 컨테이너 main process가 HTTP MCP. Dockerfile의 `CMD`를 덮어씀. |
| `tty: true` + `stdin_open: true` | 추후 `docker compose exec -T rag python mcp_server.py`로 *stdio 세션 spawn*하기 위함. 없으면 stdio attach 실패. |
| `volumes: .:/app` | `mcp_server.py` 등을 호스트에서 수정하면 컨테이너에 즉시 반영 (개발 편의). 운영용은 빼도 됨. |
| `ports: 8000:8000` | HTTP MCP를 호스트에서 접근. stdio direct 등록만 쓸 거면 *필수 아님*. |
| `restart: unless-stopped` | 호스트 재부팅 시 자동 복구. |
| `env_file: .env` | `OPENAI_API_KEY` 등 — 본 retrieval 서버는 LLM을 호출하지 않으므로 *필수는 아님*. 단 `main.py`/실습 코드가 쓰면 필요. |

**서비스 이름** = `rag`. 이 이름은 `docker compose exec rag ...`에서 그대로 쓰인다.
**컨테이너 이름** = `rag_practice`. `docker ps`/`docker logs`에서 식별용.

### 3.6 `Dockerfile` (기존 유지 — 점검만)

확인 사항:
- [ ] `python:3.10-slim` (또는 호환 가능한 3.10/3.11) 베이스.
- [ ] 시스템 패키지: `build-essential libgl1 libglib2.0-0 tesseract-ocr` — 마지막 항목은 `.png` OCR 필수.
- [ ] `pip install -r requirements.txt`.
- [ ] `WORKDIR /app`.
- [ ] `CMD ["tail","-f","/dev/null"]` — compose `command:`가 덮어쓰므로 무의미해 보이나, *compose 없이 `docker run`* 했을 때 컨테이너가 즉시 죽는 것을 막아준다.

---

## 4. 빌드 & 기동 절차

```bash
# 0) 위 6개 파일 적용 후
cd <repo-root>

# 1) 이미지 빌드 + 컨테이너 기동
docker compose build
docker compose up -d

# 2) HTTP MCP 헬스 (선택)
docker compose logs --tail=20 rag
# "Uvicorn running on http://0.0.0.0:8000" 등 startup 메시지 확인

# 3) stdio MCP 진입 동작 점검 (선택)
docker compose exec -T rag python mcp_server.py < /dev/null
# fastmcp가 stdin handshake 대기 후 즉시 exit. 에러만 안 나면 OK.
```

`docker compose ps`로 `rag_practice` 컨테이너가 `Up`이고 port `8000`이 매핑됐는지 확인.

---

## 5. Claude Code MCP 클라이언트 등록

### 5.1 기본 (direct stdio) — 가장 단순

`~/.claude.json`의 `mcpServers`에 다음을 추가.

```json
{
  "mcpServers": {
    "class-test-rag": {
      "command": "docker",
      "args": [
        "compose",
        "-f", "<리포 절대경로>/docker-compose.yaml",
        "exec", "-T",
        "rag",
        "python", "mcp_server.py"
      ]
    }
  }
}
```

**동작**:
1. Claude Code가 시작될 때 위 명령을 spawn.
2. `docker compose exec`가 `rag_practice` 컨테이너 *안에* `python mcp_server.py` 프로세스를 띄움.
3. 그 프로세스가 stdio로 MCP handshake. Claude Code는 그 stdio를 그대로 사용.
4. `rag_query`, `list_documents` 툴 즉시 가용.

**전제**: `rag_practice` 컨테이너가 *이미 떠 있어야* 한다 (`docker compose up -d` 완료 상태).

### 5.2 옵션 (HTTP 공유 인덱스 + stdio proxy)

인덱스 빌드를 *컨테이너당 1회*로 줄이고 싶다면 등록 명령만 바꾸면 된다.

```json
"args": [
  "compose", "-f", "<리포 절대경로>/docker-compose.yaml",
  "exec", "-T",
  "rag",
  "python", "mcp_stdio_proxy.py"
]
```

이 경우 proxy 프로세스는 `127.0.0.1:8000`(컨테이너 내부)에 떠 있는 HTTP 서버에 위임 → 모든 클라이언트가 같은 `_index` 공유.

---

## 6. 검증 시나리오 (구현 완료 판정 기준)

다음 6개가 모두 통과하면 구현 완료로 본다.

1. **컨테이너 health**
   `docker compose ps`에서 `rag_practice` STATE=`running`, PORTS=`0.0.0.0:8000->8000/tcp`.
2. **HTTP MCP 응답**
   `docker compose logs rag`에 fastmcp/uvicorn startup 메시지가 보이고, 종료 없이 유지.
3. **stdio MCP 진입**
   `docker compose exec -T rag python mcp_server.py < /dev/null` 실행 시 (a) llama-index가 모델을 로드하며 빌드 시작, (b) stdin EOF로 정상 종료. 에러 stack 없음.
4. **Claude Code 등록**
   Claude Code 재기동 후 MCP 서버 목록에 `class-test-rag`(또는 원하는 이름)가 보임.
5. **`list_documents` 호출**
   클라이언트에서 호출 시 `data/` 하위 파일 목록이 텍스트로 반환.
6. **`rag_query` 호출**
   `data/`에 있는 문서 내용 관련 질문을 했을 때 적절한 score와 함께 chunk가 반환됨.

---

## 7. 구조적 주의사항 (구현 후 운영 메모)

### 7.1 인덱스 캐시 분리 문제

- direct stdio 방식(5.1)으로 등록하면 *세션마다 새 프로세스 → 새 `_index`*. 첫 query에서 bge-m3 임베딩 비용이 발생한다 (`data/`가 크면 수 초~수십 초).
- HTTP 프로세스의 `_index`와 stdio 프로세스의 `_index`는 *공유되지 않는다*. 같은 컨테이너 안 두 프로세스가 같은 데이터를 따로 임베딩한다.
- **인덱스 1회 빌드만 원하면** 5.2(stdio proxy) 사용.

### 7.2 `data/` 변경 시 재인덱싱

- `_index`는 모듈 전역 캐시이므로 *프로세스 살아있는 동안* 변경을 반영하지 못한다.
- 운영 중 `data/`를 갱신하면 `docker compose restart rag`로 HTTP 프로세스를 새로 띄우거나, 별도 reset tool을 추가해야 한다 (선택 확장).

### 7.3 `mcp` 명칭 충돌

- `fastmcp` 2.x는 내부적으로 공식 `mcp` SDK를 사용한다.
- 리포 안에 `mcp.py` 같은 파일을 만들지 말 것 — import 충돌.
- 본 명세는 `mcp_server.py`, `mcp_http_server.py`, `mcp_stdio_proxy.py`로 prefix를 분리해 충돌을 피한다.

### 7.4 transport 선택 가이드

| 상황 | 권장 transport |
|---|---|
| Claude Code 단일 사용자, 단순 시작 | direct stdio (5.1) |
| 여러 클라이언트 동시 접속, 인덱스 공유 필요 | HTTP + stdio proxy (5.2) |
| 원격 서버 호스팅, 다중 호스트 클라이언트 | HTTP only + 클라이언트가 HTTP MCP 직접 지원 |
| 학습/실습 환경, 재현성 우선 | direct stdio + `docker compose down/up`으로 매번 깨끗하게 |

---

## 8. 최종 산출물 매핑

본 명세를 모두 적용했을 때 리포 루트는 다음과 같다 (RAG-only 시작 상태와의 *차이*만 강조).

```
<repo-root>/
├── data/                       # 변경 없음
├── main.py                     # 변경 없음
├── Dockerfile                  # 변경 없음 (또는 패키지 점검)
├── requirements.txt            # + fastmcp>=2.0.0
├── docker-compose.yaml         # NEW
├── mcp_server.py               # NEW — 핵심 (FastMCP + tools + lazy index + stdio entry)
├── mcp_http_server.py          # NEW — HTTP entry (1줄 import + run)
└── mcp_stdio_proxy.py          # NEW — 옵션 (HTTP→stdio proxy)
```

`~/.claude.json`의 `mcpServers["<이름>"]` 한 블록 추가로 끝.

---

## 9. 흔한 실패 패턴과 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| Claude Code에서 서버 connection 실패 | `rag_practice` 컨테이너가 꺼져 있음 | `docker compose up -d` |
| `docker compose exec` hangs forever | `tty/stdin_open` 미설정 또는 `-T` 누락 | compose에 `tty: true, stdin_open: true` + 등록 명령에 `-T` |
| 첫 query에서 OOM/타임아웃 | bge-m3 임베딩 + 대용량 PDF | chunk_size↓, top_k↓, 또는 stdio proxy로 1회 빌드 공유 |
| `.png` query에서 빈 결과 | tesseract 미설치 또는 `parse_text=False` | Dockerfile에 `tesseract-ocr`, `ImageReader(parse_text=True)` |
| HWP 로드 실패 | `olefile`/`HWPReader` 미설치 | requirements에 `olefile`, `llama-index-readers-file` 확인 |
| 같은 데이터가 두 번 임베딩 (HTTP + stdio 각각) | direct stdio 방식의 본질적 동작 | 5.2 proxy 등록으로 통일 |

---

본 명세서대로 6개 파일을 만들고 `~/.claude.json`에 1개 블록만 추가하면, *RAG-only 상태에서 MCP-enabled 상태로의 전환*이 완결된다.
