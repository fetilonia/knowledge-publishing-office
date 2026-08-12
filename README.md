# Knowledge Publishing Office

**https://fetilonia.github.io/knowledge-publishing-office/**

에이전트 파이프라인의 진행 상태를 픽셀 사무실로 보여주는 정적 프론트엔드다. 의존성이 없는 단일 HTML 파일이며, 빌드 단계가 없다.

요청을 입력하면 에이전트가 자리에서 일하고, 각자 지금 하는 일이 말풍선으로 뜨며, 산출물이 책상 사이를 건너다니다가 Proofreader 맞은편 Head Director 책상으로 보고가 올라간다. 그 의자는 비어 있다 — 최종 승인은 사람이 한다.

오른쪽 게시판에는 에이전트별 액션 히스토리가 쌓인다. 화면이 좁아지면 게시판은 사무실 아래 카드로 떨어진다.

## 두 가지 모드

| 모드 | 동작 | 필요 조건 |
| --- | --- | --- |
| **Demo** | 실제 모델 호출 없이 5단계 파이프라인 전체를 재생한다. | 없음. 페이지를 열면 바로 동작한다. |
| **Live** | 런타임에 붙어 실제 Job을 만들고 SSE 이벤트로 사무실을 움직인다. | 로컬에서 실행 중인 Knowledge Publishing Runtime. |

## Live 모드 연결

런타임을 띄운다.

```bash
node runtime/backend/server.mjs
```

이 페이지의 origin을 런타임 허용 목록에 추가해야 한다.

```bash
RUNTIME_ALLOWED_ORIGINS=https://fetilonia.github.io node runtime/backend/server.mjs
```

그다음 상단 바에 런타임 주소(`http://127.0.0.1:8787`)를 넣고 **런타임 연결**을 누른다. 주소는 브라우저 localStorage에 저장된다.

HTTPS로 서빙되는 이 페이지에서 `http://127.0.0.1`로 요청하는 것은 mixed content 예외에 해당해 Chrome과 Firefox에서 차단되지 않는다. Chrome의 Private Network Access 프리플라이트는 런타임이 `Access-Control-Allow-Private-Network` 헤더로 응답해 처리한다. **Safari는 이 예외를 더 엄격하게 다루므로 동작을 보장하지 않는다.**

## 결재

QA를 통과한 문서는 카운터가 아니라 **항목 목록**으로 결재함에 쌓인다. 상단 `결재 대기` 타일이나 승인 바의 **결재함 열기**로 목록을 열고, 항목을 선택하면 문서 본문이 열리며, 거기서 건별로 승인하거나 반려한다. 내용을 보지 않은 채 결재하지 않도록 승인 버튼이 문서를 먼저 연다.

문서 본문은 마크다운으로 렌더링된다. 본문은 신뢰하지 않는다 — 전부 이스케이프한 뒤 허용된 문법만 되살리고, 이미지와 링크는 `https:` 와 `data:image/` 스킴만 통과시킨다. 이미지는 로드 성공·실패·차단을 건수로 표시하므로 Proofreader 단계의 이미지 삽입 검토가 실제 확인으로 이어진다.

Live 모드의 아티팩트 조회 경로는 런타임 계약이 확정되지 않아 `/api/jobs/{id}/artifacts/{artifactId}` 와 `/api/jobs/{id}/artifacts` 를 순서대로 시도한다. 둘 다 실패하면 시도한 경로를 화면에 그대로 보여 준다. 런타임 경로가 다르면 `loadLiveDoc()` 의 `urls` 배열을 맞추면 된다.

## 백엔드 매핑

런타임 워커는 셋이다.

| 화면의 자리 | 런타임 에이전트 |
| --- | --- |
| Researcher | `research-worker` |
| Documentor | `document-writer` |
| Technical QA | `qa-critic` |
| Poster | *(없음 — Live에서 미연결로 표시)* |
| Proofreader | *(없음 — Live에서 미연결로 표시)* |
| Head Director | Human Approval (`POST /api/jobs/{id}/approval`) |

Poster와 Proofreader는 아직 런타임에 구현되지 않았다. Live 모드에서는 흐리게 표시되며 Demo 모드에서만 동작한다.

소비하는 SSE 이벤트: `job.created`, `job.transition`, `agent.status.changed`, `artifact.created`, `approval.required`, `approval.decided`, `job.failed`.

## 보안 주의

**런타임에는 인증이 없다.** 방어 수단은 CORS origin 검사뿐이고, CORS는 브라우저에만 적용되므로 `curl` 한 줄이면 우회된다. 런타임을 공개 주소로 노출하지 말 것 — URL을 아는 누구나 임의 프롬프트를 실행해 구독 할당량을 소진하고 생성된 아티팩트를 읽을 수 있다.

외부에서 접근해야 한다면 런타임에 먼저 인증을 붙이고, 그다음 Cloudflare Tunnel + Access처럼 신원 기반 게이트 뒤에 두어야 한다.

승인은 자동으로 이루어지지 않는다. 검토자 ID를 직접 입력해야 결재 요청이 전송되며, 비워두면 클라이언트에서 막는다.

## 그림에 대하여

캐릭터와 사무실은 전부 코드로 그렸다. 외부 아트 에셋을 쓰지 않는다.

구현 기법은 [Star Office UI](https://github.com/ringhyacinth/Star-Office-UI)의 MIT 라이선스 코드를 참고했다 — 타일 그리드, A* 길찾기, Y축 깊이 정렬, 4프레임 걷기 사이클, 상태별 목적지 이동, 그리고 책상을 캐릭터보다 앞에 그려 하반신을 가리는 착석 처리. 해당 저장소의 아트 에셋은 비상업 용도로 제한되어 있어 **어떤 에셋도 복사하지 않았다.**
