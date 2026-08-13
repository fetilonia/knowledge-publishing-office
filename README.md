# Knowledge Publishing Office

**https://fetilonia.github.io/knowledge-publishing-office/**

에이전트 파이프라인의 진행 상태를 픽셀 사무실로 보여주는 정적 프론트엔드다. 의존성이 없는 단일 HTML 파일이며, 빌드 단계가 없다.

요청을 입력하면 에이전트가 자리에서 일하고, 각자 지금 하는 일이 말풍선으로 뜨며, 산출물이 책상 사이를 건너다니다가 Proofreader 맞은편 Head Director 책상으로 보고가 올라간다. 그 자리엔 은발에 금테 단안경을 낀 **Final Reviewer**가 앉아 팔짱을 끼고 마지막으로 트집을 잡은 뒤 결재함으로 넘긴다 — 다만 그 검수는 연출일 뿐, 실제 최종 승인은 여전히 결재함에서 사람이 직접 한다.

오른쪽 게시판에는 에이전트별 액션 히스토리가 쌓인다. 화면이 좁아지면 게시판은 사무실 아래 카드로 떨어진다.

## 두 가지 모드

| 모드 | 동작 | 필요 조건 |
| --- | --- | --- |
| **Demo** | 실제 모델 호출 없이 5단계 파이프라인 전체를 재생한다. | 없음. 페이지를 열면 바로 동작한다. |
| **Live** | 이 저장소의 `status.json`을 폴링해 실제 파이프라인 진행을 사무실에 반영한다. | 없음(읽기 전용). 요청 제출과 실행 취소에만 개인 PAT가 필요하다. |

## Live 모드 연결

로컬 런타임은 더 이상 없다. `knowledge-publishing-agents`의 GitHub Actions가 각 단계를 마칠 때마다 이 저장소에 `status.json`을 커밋하고, 콘솔은 그 정적 파일을 같은 origin에서 폴링한다. 상단 바의 경로(`status.json`)를 두고 **상태 폴링 시작**을 누르면 된다. 설계 근거는 `knowledge-publishing-agents/docs/architecture/console-status-polling.md`.

`status.json`은 job **목록**을 담는다(`schema_version` 0.2.0). 서로 다른 이슈는 병렬로 실행되므로 파일 하나에 여러 job이 동시에 들어 있을 수 있다. 콘솔은 그중 자기가 접수한 job만 따라가고 나머지는 목록으로만 알린다.

### GitHub 토큰

요청 제출(= Issue 생성)과 「초기화」의 실행 취소는 **방문자 본인의** fine-grained PAT로 이루어진다. 이 토큰은 브라우저 `localStorage`에만 저장되고 `api.github.com` 외에는 전송되지 않으며, 저장소 소스코드에는 들어가지 않는다.

`knowledge-publishing-agents` 저장소 하나로 범위를 좁히고 다음 두 권한만 준다.

| 권한 | 쓰이는 곳 | 없으면 |
| --- | --- | --- |
| **Issues: Read and write** | 요청 제출 — Issue를 만들고 `kpo:ready` 라벨을 붙인다 | 요청 제출 불가 |
| **Actions: Read and write** | 「초기화」 — 진행 중인 실행 목록 조회와 취소 | 취소만 실패하고 나머지는 정상 |

파이프라인이 `status.json`을 쓸 때 쓰는 `OFFICE_STATUS_TOKEN`은 이것과 전혀 다른 토큰이다 — CI 시크릿이고, 이 저장소에만 쓰기 권한이 있으며, 브라우저로 내려가지 않는다.

## 결재

QA를 통과한 문서는 카운터가 아니라 **항목 목록**으로 결재함에 쌓인다. 상단 `결재 대기` 타일이나 승인 바의 **결재함 열기**로 목록을 열고, 항목을 선택하면 문서 본문이 열리며, 거기서 건별로 승인하거나 반려한다. 내용을 보지 않은 채 결재하지 않도록 승인 버튼이 문서를 먼저 연다.

문서 본문은 마크다운으로 렌더링된다. 본문은 신뢰하지 않는다 — 전부 이스케이프한 뒤 허용된 문법만 되살리고, 이미지와 링크는 `https:` 와 `data:image/` 스킴만 통과시킨다. 이미지는 로드 성공·실패·차단을 건수로 표시하므로 Proofreader 단계의 이미지 삽입 검토가 실제 확인으로 이어진다.

Live 모드의 아티팩트 조회 경로는 런타임 계약이 확정되지 않아 `/api/jobs/{id}/artifacts/{artifactId}` 와 `/api/jobs/{id}/artifacts` 를 순서대로 시도한다. 둘 다 실패하면 시도한 경로를 화면에 그대로 보여 준다. 런타임 경로가 다르면 `loadLiveDoc()` 의 `urls` 배열을 맞추면 된다.

## 백엔드 매핑

`status.json`의 `agents` 맵에 상태를 보고하는 워커는 넷이다.

| 화면의 자리 | 보고하는 키 |
| --- | --- |
| Researcher | `research-worker` |
| Documentor | `document-writer` |
| Proofreader | `proofreader` |
| Technical QA | `qa-critic` |
| Poster | *(없음 — Live에서 미연결로 표시)* |
| Final Reviewer | *(없음 — 데모 전용 연출, Live에서 미연결로 표시)* |
| Head Director 책상(결재함) | Human Approval — 해당 PR을 GitHub에서 merge |

`status.json`에 키가 없는 자리는 "실패"가 아니라 **이 상태 소스가 아직 보고하지 않음**으로 흐리게 표시된다. Poster의 게시 단계와 사람의 최종 승인은 아직 이 파일에 반영되지 않으므로, QA 통과 이후는 GitHub의 해당 PR에서 확인해야 한다. Final Reviewer는 애초에 실제 게이트가 아니다 — 아무리 거만하게 굴어도 최종 승인 권한은 없고, 결재함으로 올려 보내는 연출만 한다.

## 보안 주의

이 페이지는 정적 파일이고 자체 백엔드가 없다. 상태 표시는 공개된 `status.json`을 읽기만 하며, 그 파일에는 오케스트레이션 메타데이터(상태, 어떤 워커가 도는지, 타임스탬프)만 담긴다 — claim, 문서 본문, evidence, source는 들어가지 않는다.

쓰기 동작(요청 제출, 실행 취소)은 전부 방문자 본인의 PAT로 이루어진다. 그러므로 **토큰을 저장한 브라우저를 남과 공유하지 말 것** — 저장된 토큰으로 누구나 요청을 제출해 구독 할당량을 소진하거나 진행 중인 실행을 취소할 수 있다. 「지우기」로 언제든 제거할 수 있다.

최종 승인은 이 페이지에서 이루어지지 않는다. QA를 통과한 문서는 `knowledge-publishing-agents`에 PR로 올라오고, 그 PR을 사람이 merge하는 것이 승인이다.

## 그림에 대하여

캐릭터와 사무실은 전부 코드로 그렸다. 외부 아트 에셋을 쓰지 않는다.

구현 기법은 [Star Office UI](https://github.com/ringhyacinth/Star-Office-UI)의 MIT 라이선스 코드를 참고했다 — 타일 그리드, A* 길찾기, Y축 깊이 정렬, 4프레임 걷기 사이클, 상태별 목적지 이동, 그리고 책상을 캐릭터보다 앞에 그려 하반신을 가리는 착석 처리. 해당 저장소의 아트 에셋은 비상업 용도로 제한되어 있어 **어떤 에셋도 복사하지 않았다.**
