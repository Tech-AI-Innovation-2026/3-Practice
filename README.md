> **⏰ 평가 안내**
> - **대상 리파지토리**: 본인의 MCP 전용 리파지토리 (지난주 mcp-* repo 그대로 사용)
> - **대상 브랜치**: `main`
> - **마감**: **2026-04-30(목) 12:20 이전 commit** 된 코드만 채점 (만점 기준)
>
> ⚠️ `main` 브랜치가 아닌 다른 브랜치에 push하면 채점 시 반영되지 않습니다.

# 이번 주 실습: Streamable HTTP MCP 서버 + RAG 한계 발견

지난주 실습에서 STDIO 트랜스포트로 동작하는 MCP 서버를 만들고 Gemini CLI에 연결했습니다. 이번 주는 두 가지 미션을 수행합니다.

- **Mission 1 (필수)**: 같은 RAG 도구를 **Streamable HTTP** 트랜스포트로 노출하고 동작 검증
- **Mission 2 (필수)**: VectorStoreIndex 기반 RAG 의 **한계를 스스로 발견** — Top-10 검색에 정답 노드가 들어오지 않는 데이터-질문 페어를 찾아 보고

두 미션 모두 본인 mcp-* 리파지토리 `main` 브랜치에 push 해야 합니다.

---

## 🎯 Mission 1 — Streamable HTTP MCP 서버 (필수)

### 학습 목표

지난주의 STDIO 서버는 *클라이언트가 자식 프로세스로 띄워서* 통신하는 1:1 구조였습니다. 그러나 LlamaIndex 인덱스처럼 메모리에 무거운 자원을 들고 있어야 하는 도구는 **상시 가동 서버**로 운영해야 효율적입니다. 이번 미션에서 그 차이를 직접 체험합니다.

### 핵심 개념 (5주차 강의 §4 참조)

- `mcp.run()` — STDIO (지난주 사용)
- `mcp.run(transport="http", host="127.0.0.1", port=8000)` — **Streamable HTTP** (이번 주)
- 인자명은 `"http"` 지만 실체는 *POST 요청 + SSE 스트림 응답* 결합
- 서버는 클라이언트와 무관하게 *독립 프로세스*로 동작

### 체크리스트

#### [1] 환경 준비

- [ ] 지난주 mcp-* 리파지토리에서 작업 (새 디렉토리 만들지 말 것)
- [ ] `requirements.txt` 또는 Dockerfile 에 `fastmcp` 가 포함되어 있는지 확인
- [ ] 지난주 작성한 RAG QueryEngine 코드가 정상 동작하는지 먼저 확인

#### [2] STDIO → Streamable HTTP 전환

- [ ] 지난주의 `simple_server.py` 같은 파일을 복제하여 `http_server.py` (또는 `streamable_server.py`) 생성
- [ ] 서버 진입점의 `mcp.run()` 호출만 수정:
  ```python
  if __name__ == "__main__":
      mcp.run(transport="http", host="0.0.0.0", port=8000)
  ```
  > 💡 **Docker 환경에서 외부 접속이 필요하면 `host="0.0.0.0"` 으로 바인딩**. `127.0.0.1` 은 컨테이너 내부에서만 접근 가능합니다 (5주차 강의 §3.2 Docker 네트워킹 참조).
- [ ] 기존 RAG QueryEngine 도구 정의(`@mcp.tool()`)는 수정 불필요 — 트랜스포트만 바뀌고 비즈니스 로직은 100% 동일

#### [3] 서버 기동 및 동작 검증

- [ ] 터미널 A 에서 서버 실행: `python http_server.py`
- [ ] 터미널 B 에서 별도 클라이언트 스크립트(`test_http_server.py`)로 호출 검증:
  ```python
  from fastmcp import Client
  import asyncio

  async def test():
      client = Client("http://127.0.0.1:8000/mcp")
      async with client:
          await client.ping()
          tools = await client.list_tools()
          print("도구 목록:", [t.name for t in tools])
          result = await client.call_tool("rag_query", {"question": "테스트 질문"})
          print("응답:", result)

  asyncio.run(test())
  ```
- [ ] STDIO 서버와 달리 **서버를 미리 띄우지 않으면 클라이언트가 실패**하는 점을 확인
- [ ] 클라이언트를 두 번 연속 실행해도 *서버는 죽지 않고 계속 실행 중*인지 확인 (HTTP 모델의 핵심 이점)

#### [4] 동작 증거 수집

- [ ] 서버 기동 시 터미널 출력 캡처
- [ ] 클라이언트 호출 결과 캡처 (도구 목록 + 실제 RAG 응답)
- [ ] 브라우저로 `http://127.0.0.1:8000/mcp` 접속 시 `406 Not Acceptable` 응답이 오는 스크린샷 *(SSE 헤더 필요 — Streamable HTTP의 증거)*

### 📝 Mission 1 보고서 양식

본인 리파지토리 `README.md` 하단에 추가:

```markdown
# Mission 1 — Streamable HTTP MCP 서버 보고서

## 1. 코드 변경 요약

| 항목 | STDIO (지난주) | Streamable HTTP (이번 주) |
|---|---|---|
| 진입점 코드 | `mcp.run()` | `mcp.run(transport="http", host="...", port=...)` |
| 서버 실행 방식 | 클라이언트가 subprocess 로 띄움 | 별도 터미널에서 상시 실행 |
| 클라이언트 연결 식별자 | `command/args` | URL (`http://host:port/mcp`) |
| 서버 수명 | 클라이언트 종료 시 함께 종료 | 클라이언트와 무관하게 유지 |

## 2. 동작 증거

(서버 기동 로그 / 클라이언트 호출 결과 / 브라우저 406 응답 스크린샷 첨부)

## 3. STDIO 대비 체감한 차이점

(자유 서술 — 인덱스 재로드 비용, 동시 접속 가능성, Docker 배포 가능성 등 본인이 느낀 차이를 2~3 문장)
```

---

## 🚩 Mission 2 — RAG 의 한계 발견하기 (필수, 챌린징)

### 학습 목표

VectorStoreIndex 의 Top-K 검색은 **임베딩 유사도 상위 K개 노드만 LLM에 전달**합니다. 이 K 안에 정답 근거가 들어오지 않으면 RAG 는 **반드시 틀린 답을 생성**합니다. 이번 미션의 목표는 **본인이 준비한 자료로, RAG가 시스템적으로 실패하는 상황을 스스로 찾아내는 것**입니다.

이는 단순히 "잘 동작하는 RAG" 를 만드는 것보다 한 단계 깊은 이해를 요구합니다 — *어떤 종류의 데이터·질문 조합이 임베딩 공간에서 정답 노드를 멀리 보내는가* 를 직접 관찰합니다.

### 데이터 제약

- **본인이 직접 준비한 자료** 사용 (위키피디아, 본인 문서, 공개 데이터셋 등 무엇이든 가능)
- **총 용량 10MB 이하**
- 단일 파일이 아니어도 됨 (여러 파일을 `data/` 폴더에 묶어 사용)
- 저작권 분쟁 소지가 없는 자료만 사용

### 체크리스트

#### [1] VectorStoreIndex 기반 RAG 구축

- [ ] `data/challenge/` 폴더 생성 후 본인 자료 배치 (10MB 이하)
- [ ] `SimpleDirectoryReader` 또는 적절한 Reader 로 Document 로드
- [ ] `SimpleNodeParser` 또는 다른 NodeParser 로 Node 분할 (chunk_size 기본값 또는 본인 선택)
- [ ] **`VectorStoreIndex`** 로 인덱싱 (`SummaryIndex` 등 다른 인덱스는 인정되지 않음)
- [ ] `index.as_retriever(similarity_top_k=10)` 으로 Retriever 생성

#### [2] 챌린징한 질문 설계

다음 중 *최소 한 가지* 패턴을 의식하여 RAG 가 실패할 만한 질문을 설계합니다.

| 실패 패턴 | 설명 | 예시 |
|---|---|---|
| 🅰 어휘 불일치 | 정답 노드가 *다른 단어*로 같은 의미를 표현 | 본문은 "RTT" 만 사용, 질문은 "왕복 지연" 사용 |
| 🅱 다단계 추론 | 정답이 여러 노드의 정보를 *결합*해야 도출됨 | "회사 X 의 2024년 매출이 Y 의 2023년 매출 대비 몇 % 인가?" |
| 🅲 표·수치 단절 | 정답이 표 안에 있는데 임베딩이 표 구조를 잃음 | xlsx의 셀 값, hwp의 다중 컬럼 표 |
| 🅳 부정·예외 표현 | 정답이 *"X 가 아닌 경우"* 같은 부정문에 있음 | 본문에 "단, 비상시에는 적용되지 않는다" 같은 단서 |
| 🅴 질의 추상도 차이 | 질문이 *추상적·개괄적*인데 정답 노드는 *구체 사례* | 질문 "이 정책의 목적은?" → 정답 "표 3에 사례 5건 나열" |

#### [3] 실패 검증 — Top-10에 정답 노드가 없음을 증명

- [ ] 설계한 질문을 Retriever에 전달:
  ```python
  retrieved_nodes = retriever.retrieve("본인 질문")
  for i, n in enumerate(retrieved_nodes):
      print(f"[{i+1}] score={n.score:.3f}")
      print(n.node.text[:200])
      print(n.node.metadata)
      print("---")
  ```
- [ ] **Top-10 결과 전체를 출력**하고, 그 중 정답 근거가 *하나도 없음* 을 명시
- [ ] 정답이 실제로 어느 노드에 들어있는지 직접 확인 (예: `nodes[42].text` 에 정답 → 그러나 Top-10에는 노드 42 가 없음)
- [ ] QueryEngine 으로도 응답을 받아 *실제로 틀린 답을 생성*함을 확인

#### [4] 실패 원인 분석

- [ ] 위 5가지 실패 패턴 중 어느 것에 해당하는지 분류
- [ ] 임베딩 공간 관점에서 왜 정답 노드의 score 가 낮게 나왔는지 추론 (어휘 불일치 / chunk 경계 분리 / 추상도 차이 등)
- [ ] (선택) 같은 질문을 다르게 표현해서 실패가 회피되는지 시도

### 📝 Mission 2 보고서 양식

본인 리파지토리 `README.md` 하단에 추가:

```markdown
# Mission 2 — RAG Top-10 실패 케이스 보고서

## 1. 데이터 구성

| 항목 | 값 |
|---|---|
| 데이터 출처 | (예: 본인이 작성한 회의록, 위키피디아 X 항목, 공개 정책 문서 등) |
| 파일 수 | (개수) |
| 총 용량 | (MB, 10 이하) |
| 파일 포맷 | (txt / pdf / docx / xlsx / hwp / png 중 어떤 것) |
| Document 수 | `len(documents)` |
| Node 수 | `len(nodes)` |
| 사용한 NodeParser | (예: SimpleNodeParser, chunk_size=512) |
| 사용한 Embedding 모델 | (예: text-embedding-3-small, BAAI/bge-m3 등) |

## 2. 챌린징한 질문

**질문**: (본인이 설계한 질문 한 줄)

**기대 답변**: (자료 원본을 직접 보고 작성한 모범 답안)

**해당 실패 패턴**: 🅰 / 🅱 / 🅲 / 🅳 / 🅴 중 하나 (또는 복합)

## 3. Top-10 검색 결과

| 순위 | score | 노드 ID 또는 metadata | 텍스트 앞 100자 | 정답 근거 포함? |
|---|---|---|---|---|
| 1 | | | | ❌ |
| 2 | | | | ❌ |
| ... | | | | ❌ |
| 10 | | | | ❌ |

> 위 표에서 정답 근거를 포함하는 노드가 *하나도 없음* 을 확인.

## 4. 정답 노드의 실제 위치

- 정답이 들어있는 노드: 노드 인덱스 (예: `nodes[42]`)
- 해당 노드의 텍스트 앞 200자: ...
- (선택) Retriever 가 이 노드에 부여한 score: ... (Top-10 에 없으므로 별도 계산 필요)

## 5. QueryEngine 의 실제 응답

(LLM 이 Top-10 만 가지고 생성한 응답 — 기대 답변과 비교)

| 기대 답변 | 실제 응답 | 오답 여부 |
|---|---|---|
| | | ✅ 오답 |

## 6. 실패 원인 분석

(2~3 문단으로 자유 서술. 임베딩 공간 관점에서 왜 정답 노드의 유사도가 낮았는지, 본인이 추정하는 메커니즘을 적습니다.)

## 7. (선택) 실패 회피 시도

같은 의도의 질문을 다르게 표현했을 때 Top-10 에 정답이 들어오는지 시도한 결과.
```

---

## 📋 평가 기준

| 항목 | 비중 | 설명 |
|---|---|---|
| Mission 1 동작 검증 | 40% | Streamable HTTP 서버가 실제로 별도 프로세스로 떠 있고, 클라이언트가 호출에 성공함을 증거로 보일 것 |
| Mission 2 실패 케이스 | 40% | Top-10 에 정답이 없고 QueryEngine 이 오답을 생성한 케이스를 *실제로* 찾아내야 함. 가짜 시나리오 X |
| Discussion 참여도 | 20% | 질문/답변/트러블슈팅 공유 — 실패 케이스의 원인을 두고 학생끼리 토론한 흔적이 있으면 가산 |

---

## 마감 — 2026-04-30(목) 12:20

- 본인 mcp-* 리파지토리 `main` 브랜치에 두 미션의 코드 + 보고서 모두 push
- **수업 시간 종료 시점(12:20) 이전 commit 만 채점 대상**
- `git push origin main` 으로 푸시 완료 후, GitHub 웹에서 commit 시각이 12:20 이전인지 직접 확인할 것

> ⚠️ **마감 시각 이후의 commit 은 채점되지 않습니다.** 시간 여유를 두고 완성도보다 *제출 자체* 를 먼저 확보하세요.

---

## 참고 자료

- 5주차 MCP 강의자료: `5주차_MCP_이론.md` §3 (FastMCP), §4 (트랜스포트), §4.6 (코드 비교)
- 지난주 실습 백업: `README_W4_MCP_Gemini_RAG.md`
- 3주차 RAG 기본기: `README_llamaindex_basic.md`
- LlamaIndex 공식 문서: <https://docs.llamaindex.ai/>
- FastMCP 공식 문서: <https://gofastmcp.com/>
