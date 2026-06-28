# 민원 답변 멀티에이전트 × 픽셀 오피스 시각화

> **민원 답변 멀티에이전트 서비스**가 동작하는 과정을, **픽셀 오피스(pixel-agents)** 안의 캐릭터로 실시간 시각화한 결합 프로젝트입니다.

민원을 입력하고 `답변 초안 생성`을 누르면, 6개의 AI 에이전트가 각자 한 명의 직원 캐릭터가 되어 사무실에서 일합니다. 평소에는 라운지에서 대기하다가, 자기 차례가 되면 자기 책상으로 걸어가 읽고·쓰고·검수하는 모습이 그대로 화면에 나타납니다.

---


https://github.com/user-attachments/assets/7fb98290-6572-4a93-94c0-f773629e85d9




## 시연 영상

<!-- 영상은 GitHub에서 클릭 시 페이지 내 재생됩니다 -->



> (영상 파일: `docs/demo-minwon-pixel-agent.mp4`)

---

## 무엇을 만들었나

OpenRouter 기반 민원 답변 멀티에이전트 서비스(`minwon/minwon_multiagent.html`)는, 각 에이전트의 동작 시점마다 픽셀 오피스로 hook 이벤트를 보내는 코드를 내장하고 있습니다. 이 프로젝트는 **픽셀 오피스(pixel-agents)** 쪽을 수정하여 그 신호를 받아 캐릭터로 시각화되도록 두 시스템을 결합했습니다.

### 6개 에이전트와 처리 흐름

| 역할 (캐릭터 머리 위 라벨) | 처리 단계 | 동작 |
|------|------|------|
| 조정자/분류 | STEP1 (병렬) | 읽기 |
| 법령·근거 조사관 | STEP1 (병렬) | 읽기 |
| 소관·선례 조사관 | STEP1 (병렬) | 읽기 |
| 톤·리스크 검토관 | STEP1 (병렬) | 읽기 |
| 답변 작성가 | STEP2 | 타이핑 |
| 품질 검수관 | STEP3 (검수 루프) | 타이핑 |

1. **STEP1 — 병렬 분석**: 조정자·법령·선례·톤 4명이 동시에 책상으로 가서 민원을 읽고 분석합니다.
2. **STEP2 — 답변 작성**: 답변 작성가가 4종 분석을 통합해 공문체 초안을 작성합니다.
3. **STEP3 — 품질 검수**: 품질 검수관이 초안을 검수하고, 미달이면 작성가로 되돌려 재작성합니다(최대 2회).

---

## 동작 원리 (결합 구조)

```
민원 HTML  ──POST /api/hooks/minwon──▶  pixel-agents 서버 (127.0.0.1:3100)
 emitPixel()                              hookEventHandler
                                              │
                                              ▼
                                        캐릭터 상태 머신
                                  (라운지 대기 → 책상으로 이동 → 읽기/타이핑)
                                              │
                                              ▼
                                       브라우저 픽셀 오피스
```

민원 HTML은 Claude Code hook 규격의 이벤트(`SessionStart`/`PreToolUse`/`PostToolUse`/`Stop`)를 보냅니다. `SessionStart`의 `cwd`에 역할명이 담겨 캐릭터 머리 위 라벨이 되고, `PreToolUse`의 `tool_name`(Read/Write/Edit)이 캐릭터 동작을 결정합니다.

### 결합을 위해 수정한 부분

| 대상 | 수정 내용 |
|------|------|
| `server/src/httpServer.ts` | hook 라우트를 루프백(localhost) 요청에 한해 무인증 허용 (HTML의 토큰 없는 POST 수신) |
| `server/src/agentRuntime.ts` | `transcriptPath` 없는 hooks-only 외부 세션을 정식 채택하도록 게이트 완화 |
| `webview-ui` (`ToolOverlay.tsx` 등) | 캐릭터 머리 위 역할명 라벨 상시 표시 |
| 레이아웃·캐릭터 동작 | 초기 라운지 대기 → 자기 차례에 책상 착석·동작 → 종료 후 대기 복귀, 6석 확보 |

자세한 설계 배경·결정 사항은 [`DESIGN.md`](./DESIGN.md)를 참고하세요.

---

## 실행 방법

### 사전 준비
- Node.js 20 이상, Git, VS Code

### 1) 픽셀 오피스 빌드·기동

```bash
npm install
cd webview-ui
npm install
cd ..
npm run build
npx pixel-agents
```

기동되면 브라우저에서 `http://127.0.0.1:3100` 으로 접속해 픽셀 오피스를 엽니다.

### 2) 민원 서비스 실행

`minwon/minwon_multiagent.html` 을 브라우저로 엽니다.

1. `개발자 설정` 화면에서 **OpenRouter API Key** 입력
2. **픽셀 오피스 연동 ON**, 서버 주소 `http://127.0.0.1:3100` 확인
3. `민원 담당자` 화면에서 민원 입력 후 **답변 초안 생성** 클릭
4. 픽셀 오피스에서 6명이 차례대로 책상으로 가 동작하는지 확인

---

## 프로젝트 구조

```
.
├── README.md                  # 본 문서
├── DESIGN.md                  # 결합 설계 문서
├── minwon/
│   └── minwon_multiagent.html # 민원 답변 멀티에이전트 서비스 (클라이언트)
├── docs/
│   └── demo-minwon-pixel-agent.mp4  # 시연 영상
├── server/                    # pixel-agents 서버 (hook 수신·상태 처리) — 일부 수정
├── webview-ui/                # pixel-agents 브라우저 시각화 — 일부 수정
└── ...                        # 그 외 pixel-agents 원본 파일
```

---

## 출처 및 라이선스

이 프로젝트는 오픈소스 [pixel-agents](https://github.com/pixel-agents-hq/pixel-agents)(MIT License)를 기반으로 결합·수정하여 제작했습니다. 원본 라이선스는 [`LICENSE`](./LICENSE) 파일에 그대로 포함되어 있습니다.
