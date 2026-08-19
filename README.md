# Knowledge Publishing Office

**https://fetilonia.github.io/knowledge-publishing-office/**

에이전트 파이프라인의 진행 상태를 픽셀 사무실로 보여주는 정적 프론트엔드다. 의존성이 없는 단일 HTML 파일이며, 빌드 단계가 없다.

Live 모드에서 오피스 화면은 **하나의 job이 아니라 전체 현황을 보여주는 대시보드**다. 서로 다른 이슈는 실제로 병렬 실행되므로(아래 "Live 모드 연결"), 각 에이전트 책상은 지금 그 단계에 있는 job번호(+PR번호가 있으면 그것도)를 말풍선으로 보여주고, 여러 건이 겹치면 "+N건"이 붙는다. 말풍선을 클릭하면 그 단계에서 지금 병렬로 도는 job 전체를 표로 볼 수 있다. Researcher·Documentor·Proofreader·Technical QA·Final Reviewer는 `status.json`의 `agents` 맵에서, Poster는 `status.json`의 job 상태(`PUBLISHING`)에서 실시간 상태를 읽는다 — 아래 "백엔드 매핑" 참고. 실제 최종 승인 여부는 결재함에서 사람이 PR을 merge하거나 final-reviewer가 자동 판정(AUTO_APPROVE/REJECT)한 결과로 정해진다.

오른쪽 벽에는 게시판 두 개가 세로로 나란히 걸려 있다. 위쪽(**진행 기록**)에는 에이전트별 액션 히스토리가 시간순으로 쌓인다. 아래쪽(**주제 추적**)은 시간순 로그가 아니라 job 단위 현황판이다 — 페이지를 새로 열어도, 내가 접수하지 않은 job이라도, 폴링 중인 `status.json`에 있는 모든 job을 상태와 5단계(연구·작성·교열·QA·최종검수) 완료 점으로 보여준다. 지금 이 콘솔이 따라가는 job은 이름이 붉게 강조된다.

진행 기록은 워커 단위 상태 전이(RESEARCHING→DRAFTING 같은)만 보여주지 않는다. `status.json` 0.3.0부터 각 job은 `activity[]`(감사 목적의 세부 작업 로그 — 게이트 검증 통과/실패, 재시도 횟수, `final-reviewer`의 conformance_score)를 함께 실어 보내고, 이 게시판이 그 항목을 실시간으로 그대로 얹는다. `documentor` 같은 Skill을 orchestrator 없이 사람이 직접 실행한 경우도 같은 경로로 보고되며, 그 항목은 `[Skill]` 표시로 자동 파이프라인 항목과 구분된다 — 자세한 설계는 `knowledge-publishing-agents/docs/architecture/console-status-polling.md`의 "세부 진행 기록 — activity[]" 참고. 상태 전이 로그와 달리 이 세부 로그는 "내가 접수한 job"으로 좁히지 않는다 — 감사 게시판이므로 지금 폴링 중인 모든 job의 활동을 함께 보여준다.

`state`(RESEARCHING 등 5개 값)와 `agents` 맵은 "어느 워커 차례인지"만
말해 준다 — 그 워커가 실제로 몇 분씩 도는 동안에도 이 두 필드는 안
바뀐다. `status.json`의 선택 필드 `activity`(2026-08-14 추가)는 그
공백을 메우는, orchestrator가 각 단계 전환마다 새로 싣는 한 줄 설명이다
(예: `"research_gate 통과 — document-writer가 문서 초안 작성 중"`).
있으면 말풍선에 job번호 뒤로 이어 붙고, "진행 기록" 게시판에는 값이
바뀔 때마다 별도 줄로 남으며, "주제 추적" 팝업 표에는 "지금 하는 일"
칸으로 뜬다. `title`/`pr_number`와 달리 sticky가 아니라서 — 한 번
받으면 다음 상태로 넘어가도 그 문구가 없어질 때까지는 남아 있는 게
아니라, 다음 게시가 새 값을 보내지 않으면 그냥 사라진다(자세한 병합
규칙은 `knowledge-publishing-agents/docs/architecture/console-status-polling.md`
참고).

각 게시판은 최근 몇 줄만 보여준다. 코르크 면 오른쪽 위의 **⤢** 버튼을 누르면 전체 목록을 **표**로 보는 팝업이 뜬다 — 코르크 미리보기는 "에이전트: 한 줄 요약"처럼 좁은 줄에 맞춘 형태지만, 팝업에서는 칸을 나눠 더 많은 정보를 보여준다. 진행 기록의 팝업은 에이전트 / 실행한 작업 / 결과 세 칸이고, 주제 추적의 팝업은 Job / 제목(원본 GitHub 이슈 제목) / 상태 / 진행 네 칸이다(코르크 미리보기는 `JOB-28`처럼 job_id만 보여 좁은 줄이 지저분해지지 않게 한다). 화면이 좁아지면 게시판은 사무실 아래 카드로 떨어지는데, 이때도 미리보기는 같은 상한만큼만 보이고 ⤢ 버튼은 그대로 남는다.

## 두 가지 모드

| 모드 | 동작 | 필요 조건 |
| --- | --- | --- |
| **Demo** | 실제 모델 호출 없이 5단계 파이프라인 전체를 재생한다. | 없음. 페이지를 열면 바로 동작한다. |
| **Live** | 이 저장소의 `status.json`을 폴링해 실제 파이프라인 진행을 사무실에 반영한다. | 없음(읽기 전용). 결재함·Job 관제에만 개인 PAT가 필요하다. |

## Live 모드 연결

로컬 런타임은 더 이상 없다. `knowledge-publishing-agents`의 GitHub Actions가 각 단계를 마칠 때마다 이 저장소에 `status.json`을 커밋하고, 콘솔은 그 정적 파일을 같은 origin에서 폴링한다. 상단 바의 경로(`status.json`)를 두고 **상태 폴링 시작**을 누르면 된다. 설계 근거는 `knowledge-publishing-agents/docs/architecture/console-status-polling.md`.

`status.json`은 job **목록**을 담는다(`schema_version` 0.2.0). 서로 다른 이슈는 병렬로 실행되므로 파일 하나에 여러 job이 동시에 들어 있을 수 있다. 오피스 화면(에이전트 책상)은 이 목록 전체를 항상 보여준다 — "내가 접수한 job"이라는 개념은 상단 타일과 티커, 주제 추적 게시판의 강조 표시에만 쓰인다(아래 "Job 관제"의 「내 Job으로 표시」).

## 요청 작성

하단 바의 **요청 작성**은 이 콘솔에서 제목·본문을 받지 않는다. GitHub API도 호출하지 않는다 — 누르는 즉시 `kpo:ready` 라벨이 이미 붙은 GitHub의 새 이슈 양식을 새 탭으로 열어 줄 뿐이고, 실제 요청 작성과 제출은 그 탭에서 GitHub 로그인 세션으로 이루어진다. 인증은 이미 있는 그 세션이 하고, 누가 요청을 넣을 수 있는지는 저장소 협업자 권한이 그대로 통제하므로 토큰이 필요 없다 — 요청 하나 넣으려고 PAT를 발급해 브라우저에 붙여넣게 하는 것은 과한 요구였다.

GitHub의 새 이슈 웹 폼으로 라벨을 미리 채운 채 만들어도 `issues.labeled` 웹훅은 정상적으로 온다(issue #29에서 실측). 한때는 "라벨이 생성과 동시에 붙으면 `opened`만 온다"고 보고 워크플로가 두 이벤트를 모두 받도록 했었는데, 실제로는 둘 다 와서 같은 이슈에 실행이 두 번 걸렸다 — `subagent-execution-claude-code.yml`은 지금 `labeled` 하나만 받는다.

제출한 이슈 번호는 그 탭에서 정해지고 이 탭은 알 수 없으므로, 콘솔은 버튼을 누른 시점에 있던 job 목록을 기억해 두었다가 다음 폴링에서 **처음 나타나는 새 job**을 "내 job"으로 자동 채택한다(`armNewJobAdoption`/`adoptNextNewJob`) — 상단 타일과 주제 추적 게시판의 강조 표시만 그 job으로 바뀐다. 같은 순간 다른 사람이 요청을 넣었다면 잘못 채택될 수 있는데, 그럴 때는 「Job 관제」의 「내 Job으로 표시」로 바로잡으면 된다.

## Job 관제

하단 바의 **Job 관제** 버튼이 지금 무엇이 돌고 있는지를 한 화면에 모은다. 목록은 두 출처를 `job_id`로 합쳐 만든다.

| 출처 | 아는 것 | 모르는 것 |
| --- | --- | --- |
| `status.json` | 파이프라인이 **게시한** 상태(단계, 워커별 진행, PR 번호) | 아직 첫 상태를 게시하지 않은 실행 |
| GitHub Actions API | 지금 **돌고 있는** 실행(실행 번호, queued/in_progress) | 이미 끝난 job의 결과 |

그래서 방금 접수해 아직 상태가 없는 job도 `실행 대기 중`으로 보이고, 끝난 job도 목록에 남는다. 둘을 잇는 키는 워크플로의 `run-name`에 박힌 `JOB-<이슈번호>`다.

각 항목에서 할 수 있는 것:

- **내 Job으로 표시** — 오피스 화면은 바꾸지 않는다(이미 모든 job을 보여주고 있다). 상단 타일·티커·주제 추적 게시판의 강조만 그 job으로 옮긴다. 요청 접수가 다른 job으로 잘못 채택됐을 때 바로잡는 용도다.
- **실행 #N 취소** — 그 job의 진행 중인 Actions 실행을 취소한다. 취소된 실행은 워크플로의 정리 스텝이 `FAILED`로 닫는다.
- **결재함** — QA를 통과해 PR이 열린 job이면 바로 결재함으로 넘어간다.

**초기화**는 이 콘솔의 화면 상태만 되돌린다. 실행을 멈추는 것은 별개의 일이라 Job 관제로 분리했다 — 화면만 정리하려던 사람이 남의 실행까지 취소하게 두지 않기 위해서다.

### GitHub 토큰 — 결재함과 Job 관제에서만 쓴다

문서를 읽거나, 승인·반려하거나, 실행을 취소하는 동작은 private 저장소를 브라우저에서 직접 읽고 써야 하므로 **방문자 본인의** fine-grained PAT가 필요하다. 이 토큰은 브라우저 `localStorage`에만 저장되고 `api.github.com` 외에는 전송되지 않으며, 저장소 소스코드에는 들어가지 않는다.

`knowledge-publishing-agents` 저장소 하나로 범위를 좁히고 다음 권한을 준다.

| 권한 | 쓰이는 곳 | 없으면 |
| --- | --- | --- |
| **Pull requests: Read and write** | 결재함 — PR의 문서 본문 열람, 반려 시 의견 등록과 PR 닫기 | 결재함 열람·반려 불가 |
| **Contents: Read and write** | 승인 — PR merge | 승인 불가 |
| **Issues: Read and write** | 반려 시 `kpo:rework` 라벨 재부착 | 반려 후 재작업 미시작 |
| **Actions: Read and write** | 「Job 관제」 — 진행 중인 실행 목록 조회와 취소 | Job 목록·전환은 되고 취소만 실패 |

권한이 없어도 나머지 동작은 그대로 되므로 필요한 만큼만 주면 된다. 다만 **Contents 쓰기는 저장소에 커밋할 수 있는 권한**이라는 점은 알고 주어야 한다 — merge 엔드포인트가 Pull requests가 아니라 Contents 권한으로 분류되기 때문이지, 이 콘솔이 커밋을 하기 때문은 아니다.

### 왜 서버 없이 이렇게 하는가

토큰을 한 곳(예: GitHub Actions secret으로 빌드에 주입)에 두고 방문자마다 입력하지 않게 할 수는 없을까? **없다.** 이 저장소(`knowledge-publishing-office`)와 배포된 Pages는 모두 **public**이다. 빌드 시 주입한 값은 배포된 `index.html`에 그대로 남아 페이지 소스로 누구나 읽을 수 있다 — `knowledge-publishing-agents`(private)에 대한 Issues·Contents·Pull requests·Actions 쓰기 권한을 가진 토큰을 인터넷에 공개하는 것과 같다. 이 문제를 서버 없이 피하는 유일한 방법이 각자의 토큰을 각자의 브라우저에만 두는 것이다.

파이프라인이 `status.json`을 쓸 때 쓰는 `OFFICE_STATUS_TOKEN`은 이것과 전혀 다른 토큰이다 — CI 시크릿이고, 이 저장소에만 쓰기 권한이 있으며, 브라우저로 내려가지 않는다.

## 결재

QA를 통과한 문서는 카운터가 아니라 **항목 목록**으로 결재함에 쌓인다. 상단 `결재 대기` 타일이나 승인 바의 **결재함 열기**로 목록을 열고, 항목을 선택하면 문서 본문이 열리며, 거기서 건별로 승인하거나 반려한다. 내용을 보지 않은 채 결재하지 않도록 승인 버튼이 문서를 먼저 연다.

Live 모드에서 이 결재는 **PR에 대한 실제 행위**다. 파이프라인은 QA 통과 후 PR을 열고 그 번호를 `status.json`에 실어 보내며, 결재함은 `state==="FINAL_REVIEW"`이고 `pr_number`가 있는 job을 **누가 접수했는지와 무관하게** 전부 올린다 — 같은 저장소를 함께 쓰는 협업자라면 내가 접수하지 않은 job도 결재할 수 있어야 결재함의 의미가 있기 때문이다. 문서 본문도 그 PR에서 직접 읽는다(`status.json`에는 본문이 담기지 않는다 — 아래 「보안 주의」).

트리거가 `FINAL_REVIEW`인 이유(2026-08-14, final-reviewer 도입 이후): `agents/orchestrator/contract.yaml`의 `final_review_gate`에 따르면 final-reviewer는 qa-critic 통과 직후 곧바로 실행되므로 job은 `QA_REVIEW`에 머물지 않고 `FINAL_REVIEW`로 넘어가며, 사람의 승인/반려 대기는 그 `FINAL_REVIEW` 상태에서 일어난다(REQUEST_APPROVAL 밴드). REJECT/AUTO_APPROVE 밴드는 사람 개입 없이 자동으로 REWORK/APPROVED로 넘어가므로 결재함에 필요 없다 — `syncApprovalQueue`가 매 폴링마다 결재함을 `FINAL_REVIEW`+`pr_number` job 집합으로 다시 계산하면서, 더 이상 `FINAL_REVIEW`가 아닌(자동 처리됐거나 다른 협업자가 이미 결재한) PR 항목은 걷어낸다.

오피스 화면(에이전트 책상)도 이미 모든 job을 보여주므로, 결재함이 "내 job"으로 좁히지 않는 것과 같은 방향이다 — "내가 접수한 job"이라는 구분은 상단 타일·티커·주제 추적의 강조 표시에만 남아 있다.

| 버튼 | 실제로 일어나는 일 |
| --- | --- |
| **최종 승인** | 그 PR을 merge한다. INV-14/15가 정한 대로 **merge가 곧 승인**이다. |
| **반려** | 반려 의견을 PR에 코멘트로 남기고 PR을 닫은 뒤, 이슈에 `kpo:rework` 라벨을 붙여 재작업을 시작한다. |

재작업은 조사부터 다시 하지 않고 **문서 작성(DRAFTING)부터** 재개한다 — 반려는 조사가 틀렸다는 판정이 아니라 문서가 반려됐다는 판정이기 때문이다(`agents/orchestrator/contract.yaml`의 `human_approval.on_rejection`). 오케스트레이터는 닫힌 PR의 브랜치에서 이전 초안(`jobs/<doc_id>.document-draft.json`)과 반려 코멘트를 읽어 문서만 고친 뒤 다시 교열·QA를 태우고 새 PR을 연다.

같은 라벨을 다시 붙이면 GitHub이 `labeled` 웹훅을 쏘지 않으므로 콘솔은 라벨을 뗐다가 다시 붙인다 — 그러지 않으면 두 번째 반려부터 재작업이 조용히 시작되지 않는다.

문서 본문은 마크다운으로 렌더링된다. 본문은 신뢰하지 않는다 — 전부 이스케이프한 뒤 허용된 문법만 되살리고, 이미지와 링크는 `https:` 와 `data:image/` 스킴만 통과시킨다. 이미지는 로드 성공·실패·차단을 건수로 표시하므로 Proofreader 단계의 이미지 삽입 검토가 실제 확인으로 이어진다.

Live 모드의 아티팩트 조회 경로는 런타임 계약이 확정되지 않아 `/api/jobs/{id}/artifacts/{artifactId}` 와 `/api/jobs/{id}/artifacts` 를 순서대로 시도한다. 둘 다 실패하면 시도한 경로를 화면에 그대로 보여 준다. 런타임 경로가 다르면 `loadLiveDoc()` 의 `urls` 배열을 맞추면 된다.

## 백엔드 매핑

`status.json`의 `agents` 맵에 상태를 보고하는 workflow1 워커는 다섯이다(2026-08-14부터 final-reviewer 포함). Poster(workflow2)는 `agents` 맵이 아니라 job 자체의 `state`로 보고한다.

| 화면의 자리 | 보고하는 키 |
| --- | --- |
| Researcher | `agents["research-worker"]` |
| Documentor | `agents["document-writer"]` |
| Proofreader | `agents["proofreader"]` |
| Technical QA | `agents["qa-critic"]` |
| Final Reviewer | `agents["final-reviewer"]` |
| Poster | `state==="PUBLISHING"`(agents 맵에는 키가 없다 — workflow2가 별도로 보고) |
| Head Director 책상(결재함) | Human Approval — 해당 PR을 GitHub에서 merge, 또는 final-reviewer의 AUTO_APPROVE/REJECT 자동 판정 |

`agents` 맵에 키가 아예 없는 backend(현재는 없음)만 "실패"가 아니라 **이 상태 소스가 아직 보고하지 않음**으로 흐리게(UNWIRED) 표시된다. Final Reviewer는 qa-critic·proofreader가 통과시킨 잔여 medium/low/info 이슈를 정합률 점수로 환산해 REJECT(≤70, 자동 반려)/REQUEST_APPROVAL(71-90, 결재함으로)/AUTO_APPROVE(≥91, 자동 승인) 세 갈래로 판정하는 실제 게이트다 — 사람이 직접 보는 QA_REVIEW 상태 없이 qa-critic 통과 직후 바로 실행된다(`agents/orchestrator/contract.yaml`의 `final_review_gate` 참고).

## 보안 주의

이 페이지는 정적 파일이고 자체 백엔드가 없다. 상태 표시는 공개된 `status.json`을 읽기만 하며, 그 파일에는 오케스트레이션 메타데이터(상태, 어떤 워커가 도는지, 타임스탬프)만 담긴다 — claim, 문서 본문, evidence, source는 들어가지 않는다. 진행 기록 게시판의 `activity[]` 세부 로그도 같은 제약을 받는다 — 담기는 것은 게이트 실패 코드·재시도 횟수·conformance_score 같은 메타데이터뿐이고, "감사 목적"이라는 이름이 문서 내용을 담아도 된다는 뜻은 아니다.

쓰기 동작(요청 제출, 실행 취소)은 전부 방문자 본인의 PAT로 이루어진다. 그러므로 **토큰을 저장한 브라우저를 남과 공유하지 말 것** — 저장된 토큰으로 누구나 요청을 제출해 구독 할당량을 소진하거나 진행 중인 실행을 취소할 수 있다. 「지우기」로 언제든 제거할 수 있다.

승인은 자동으로 이루어지지 않는다. 결재함에서 사람이 문서를 열어 보고 직접 눌러야 하며, 그 행위는 GitHub에 merge로 기록된다 — 누가 승인했는지는 토큰 소유자로 남는다.

## 그림에 대하여

캐릭터와 사무실은 전부 코드로 그렸다. 외부 아트 에셋을 쓰지 않는다.

구현 기법은 [Star Office UI](https://github.com/ringhyacinth/Star-Office-UI)의 MIT 라이선스 코드를 참고했다 — 타일 그리드, A* 길찾기, Y축 깊이 정렬, 4프레임 걷기 사이클, 상태별 목적지 이동, 그리고 책상을 캐릭터보다 앞에 그려 하반신을 가리는 착석 처리. 해당 저장소의 아트 에셋은 비상업 용도로 제한되어 있어 **어떤 에셋도 복사하지 않았다.**
