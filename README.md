# Knowledge Publishing Office

에이전트 파이프라인의 진행 상태를 픽셀 사무실로 보여주는 정적 프론트엔드다. 의존성이 없는 단일 HTML 파일이며, 빌드 단계가 없다.

요청을 입력하면 에이전트가 자리에서 일하고, 산출물이 책상 사이를 건너다니다가, 파티션 너머 Head Director 결재함에 도착한다. 그 의자는 비어 있다 — 최종 승인은 사람이 한다.

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
RUNTIME_ALLOWED_ORIGINS=https://<owner>.github.io node runtime/backend/server.mjs
```

그다음 상단 바에 런타임 주소(`http://127.0.0.1:8787`)를 넣고 **런타임 연결**을 누른다. 주소는 브라우저 localStorage에 저장된다.

HTTPS로 서빙되는 이 페이지에서 `http://127.0.0.1`로 요청하는 것은 mixed content 예외에 해당해 Chrome과 Firefox에서 차단되지 않는다. Chrome의 Private Network Access 프리플라이트는 런타임이 `Access-Control-Allow-Private-Network` 헤더로 응답해 처리한다. **Safari는 이 예외를 더 엄격하게 다루므로 동작을 보장하지 않는다.**

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
