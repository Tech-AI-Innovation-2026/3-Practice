> **⏰ 평가 안내**
> - **대상 리파지토리**: 새로 배정되는 **MCP 전용 리파지토리** (GitHub Classroom 초대 링크로 생성)
> - **대상 브랜치**: `main`
> - **마감**: `main` 브랜치에 **12:20 이전 commit**된 코드만 평가
>
> ⚠️ 기존 리파지토리가 아닌 **새 MCP 리파지토리에 commit한 것만** 평가 대상입니다.

# 이번 주 실습: FastMCP + Gemini CLI로 RAG 연결하기

지난주에 만든 RAG 파이프라인을 **FastMCP로 MCP 서버로 감싸고**, **Gemini CLI**에서 호출할 수 있게 연결하세요.

## 목표

**Gemini CLI → MCP 서버 → RAG 엔진** 경로로 자연어 질의응답이 동작하면 됩니다.

- [ ] FastMCP로 RAG QueryEngine을 MCP 도구(tool)로 노출
- [ ] Gemini CLI에 MCP 서버를 연결
- [ ] Gemini CLI에서 질문을 던지면 MCP 도구가 호출되어 응답이 돌아오는지 확인
- [ ] 동작 증거(스크린샷 또는 터미널 로그)를 본인 레포지토리 `README.md` 하단에 첨부
- [ ] 관련 코드와 함께 **`main` 브랜치에** push

> **⚠️ `main` 브랜치가 아닌 다른 브랜치에 push하면 채점 시 반영되지 않습니다.**

## 📋 평가 기준

- ✅ 정상 작동 여부 (Gemini CLI → MCP 도구 호출 → 응답 반환)
- ✅ 코드 + 동작 증거가 `main` 브랜치에 push되어 있음
- ✅ **git Discussion 참여도** — 질문, 답변, 트러블슈팅 공유 등 Discussion 활동이 점수에 반영됩니다
