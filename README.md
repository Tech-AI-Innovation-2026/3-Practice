> **⏰ 평가 안내**
> - **대상 리파지토리**: 본인의 mcp-* 리파지토리 (지난주 그대로 사용)
> - **대상 브랜치**: `main`
> - **마감**: **2026-04-30(목) 12:20 이전 commit** 만 채점
>
> ⚠️ `main` 외의 브랜치 push 는 반영되지 않습니다.

# 이번 주 실습: Streamable HTTP MCP + RAG 한계 발견

두 가지 미션. 둘 다 본인 mcp-* 리파지토리 `main` 브랜치에 코드 + 보고서 push.

## 🎯 Mission 1 — Streamable HTTP MCP 서버

지난주의 STDIO 서버를 **Streamable HTTP** 트랜스포트로 바꿔 동작시키세요.

- [ ] FastMCP 서버를 Streamable HTTP 모드로 기동
- [ ] 별도 클라이언트 스크립트로 도구 호출 성공 확인
- [ ] 동작 증거(로그 또는 스크린샷)를 `README.md` 하단에 첨부

> 힌트: 5주차 강의자료 §4 (트랜스포트) 참조

## 🚩 Mission 2 — RAG Top-10 실패 케이스 찾기

**VectorStoreIndex** 기반 RAG를 만들고, **Top-10 검색에 정답 근거 노드가 들어오지 않는 데이터-질문 페어**를 직접 발굴하세요.

- [ ] 본인이 준비한 자료로 VectorStoreIndex 구축 (**총 10MB 이하**)
- [ ] 질문 → Retriever Top-10 출력 → 정답 노드가 *없음* 을 증명
- [ ] 실패 원인을 본인 언어로 분석해 `README.md` 에 정리
- [ ] 사용한 데이터를 리파지토리에 함께 push

## 평가 기준

- ✅ Mission 1 정상 동작 + 증거
- ✅ Mission 2 실패 케이스가 실제로 재현 가능 + 원인 분석
- ✅ git Discussion 참여도

## 참고

- 지난주 가이드: [`README_W4_MCP_Gemini_RAG.md`](./README_W4_MCP_Gemini_RAG.md)
- 3주차 RAG 기본기: [`README_llamaindex_basic.md`](./README_llamaindex_basic.md)
