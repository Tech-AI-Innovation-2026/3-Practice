[신입 사원 온보딩 미션] "연구원님, 이제 AI에게 질문할 수 있는 시스템을 만들어 주세요!"
연구원님! 지난주에 Docker 환경을 성공적으로 구축한 덕분에 팀 전체가 동일한 환경에서 개발할 수 있게
되었습니다. 이번 주 미션은 그 환경 위에서 "문서를 넣으면 질문에 답하는 AI 시스템", 즉 첫 번째 RAG 파
이프라인을 조립하는 것입니다.
지난 이론 수업에서 배운 Document → Node → Index → QueryEngine 의 네 가지 빌딩 블록을 직접
코드로 연결하고, 내부에서 무슨 일이 벌어지는지 로그를 열어 확인하는 것이 핵심입니다.
📍 1단계: [협력 구간] 첫 RAG 파이프라인 조립 (Pass/Fail)
목표: 1단계의 목적은 "전원 성공"입니다. 4개의 빌딩 블록을 하나씩 연결하여 질의응답이 동작하는 상
태까지, 막히는 팀원이 있다면 Slido를 통해 적극적으로 도와주세요. 모두가 아래 체크리스트를 달성해
야 다음 단계로 넘어갈 수 있습니다.
[1] 환경 확인 (2주차 Docker 재활용)
2주차에 만든 Docker 컨테이너가 정상 실행되는가?
requirements.txt 에 llama-index-readers-wikipedia , wikipedia 패키지를 추가하고
재빌드했는가?
[2] Block 1 — Document 로드 (데이터 수집)
WikipediaReader를 사용하여 지정된 위키피디아 문서를 Document 객체로 로드했는가?
len(documents) 와 documents[0].metadata 를 출력하여 정상 로드를 확인했는가?
[3] Block 2 — Node 분할 (데이터 구조화)
SimpleNodeParser를 사용하여 Document를 Node 리스트로 변환했는가?
len(nodes) 를 출력하여 몇 개의 Node가 생성되었는지 확인했는가?
nodes[0].text[:200] 과 nodes[0].metadata 를 출력하여 Node 내용과 메타데이터 상속을
확인했는가?
[4] Block 3 — Index 구축 (검색 구조화)
SummaryIndex를 사용하여 Node 리스트로 Index를 구축했는가?
[5] Block 4 — QueryEngine 실행 (질의응답)
index.as_query_engine() 으로 QueryEngine을 생성하고, 질문을 던져 자연어 응답을 받았는
가?
[6] 블랙박스 열기 — Logging 활성화
logging.basicConfig(level=logging.DEBUG) 를 코드 최상단에 추가했는가?
