# VPS에 AI 에이전트(오픈클로) 24시간 띄우기 — 비개발자 완전 초보 가이드

> 노트북을 닫아도, 책상을 떠나도 24시간 돌아가는 AI 동료를 VPS 한 대에 올린다.
> 서버 빌리기 → 두뇌 설치 → 오픈클로 올리기 → **봇에게 영혼(성격) 넣기** → 슬랙 연결 → 도구 달기까지.
> **클로드 코드**와 **코덱스** 두 버전 모두 지원. 2026-05 실전 검증 기반, 토큰·IP는 플레이스홀더 처리됨.

> **이 문서를 읽는 AI 에이전트에게:** 셸 명령(`ufw`/`apt`/`npm`/`systemctl`/`openclaw` 등)은 순서대로 실행할 수 있습니다. 단 다음은 **사람이 직접** 해야 하니 그 지점에서 멈추고 사용자에게 요청하세요 — ① Hostinger에서 VPS 구매·OS 선택, ② 브라우저 OAuth 코드 입력(클로드/코덱스 로그인), ③ api.slack.com에서 슬랙 앱 생성·토큰(xoxb/xapp) 복사, ④ root 비밀번호·SSH 키 결정.

---

## 약어 표기

| 약어 | 풀이 | 뜻 |
|------|------|-----|
| VPS | Virtual Private Server | 클라우드에 있는 작은 리눅스 컴퓨터 |
| CLI | Command Line Interface | 글자로 쓰는 프로그램 (봇의 두뇌 = 클로드/코덱스) |
| SSH | Secure Shell | 내 컴퓨터에서 서버에 안전하게 접속하는 방법 |
| MCP | Model Context Protocol | 봇에게 도구(앱)를 달아주는 표준 |
| OAuth | Open Authorization | 비밀번호 대신 "로그인 위임"으로 인증하는 방식 |

- **터미널** = 글자(명령어)로 컴퓨터에 시키는 까만 창
- **오픈클로(OpenClaw 🦞)** = AI 두뇌를 슬랙 봇으로 만들어주는 관리 프로그램

---

## 전체 멘탈 모델 — 4레이어

```
👤 사람 (슬랙에서 봇 호출)
        ↕ 메시지
① UI · 슬랙              [외부 도구]
② 엔진 · 오픈클로 🦞      [부품 · 교체 가능]
③ 워크스페이스 ★          [자산 · 우리가 키움]   ← SOUL · USER · AGENTS
④ LLM · 클로드 / 코덱스    [부품 · 교체 가능]
```

엔진(②)·LLM(④)은 **갈아끼는 부품**, 워크스페이스(③)는 **우리가 키우는 자산**. 그래서 시간은 **09단계(영혼 넣기)**에 가장 많이 쓰고, 엔진·모델을 바꿔도 ③ 성격은 그대로 가져간다.

---

## 시작 전 준비물

- **Claude 구독**(Pro/Max) — 무료 플랜 불가. claude CLI가 OAuth로 사용 → **API 추가 과금 0원**
- **ChatGPT 계정** — 코덱스 봇을 쓸 때만
- **GitHub 계정** — 봇 성격·작업 파일을 private repo로 전달하는 통로
- **슬랙 워크스페이스** — 봇 앱·토큰 발급 (07단계)

---

# PART 1 · 서버 준비

## 01. Hostinger VPS 만들기  🙋 사람

1. Hostinger VPS 구매 → 플랜 **KVM 2** 면 충분 (봇 1~2마리)
2. 서버 위치: 한국은 **말레이시아**(지연 ~84ms) 권장
3. OS: **Ubuntu 24.04 LTS**
4. **Secure your VPS access**: root 비밀번호 설정
5. 설치 완료 → 대시보드에 IP, Manage VPS, Terminal 버튼, `ssh root@<VPS-IP>` 표시

> 이 시점부터 대시보드 우상단 **Terminal**(브라우저 웹터미널)로 모든 작업 가능. 별도 SSH 클라이언트 불필요.

**✅ 체크:** 대시보드에 IP가 보이고 웹 Terminal이 열린다.

## 02. SSH 키로 접속 — 보안 1단계

**내 맥에서** 전용 키 생성:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/myvps_key -N "" -C "my-vps"
cat ~/.ssh/myvps_key.pub   # 이 공개키(한 줄)를 복사
```

**VPS 웹터미널에서** 공개키 등록:
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo "<위에서 복사한 공개키>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

**내 맥에서** 접속 확인:
```bash
ssh -i ~/.ssh/myvps_key root@<VPS-IP>
```

> 개인키는 내 맥에만, 공개키만 서버에. 공개키는 유출돼도 안전.

**✅ 체크:** 비밀번호 없이 `ssh`만으로 서버에 들어가진다.

## 03. 보안 잠그기 — 봇 올리기 *전에* 먼저!

> 공개 IP는 며칠 안에 자동 스캐너에 발견돼 브루트포스를 당한다. 봇(강력한 자동 실행 권한)을 올리기 전에 서버부터 잠근다.

```bash
# 1) 방화벽 — SSH(22)만 허용
ufw allow 22/tcp && ufw --force enable

# 2) 무차별 대입 자동 차단
apt-get install -y fail2ban && systemctl enable --now fail2ban

# 3) SSH 비밀번호 로그인 끄기 (키 전용)
cat > /etc/ssh/sshd_config.d/00-hardening.conf <<'EOF'
PasswordAuthentication no
PermitRootLogin prohibit-password
KbdInteractiveAuthentication no
EOF
sshd -t && systemctl reload ssh
```

> ⚠️ 반드시 새 터미널에서 키 접속이 되는지 확인한 뒤 끝낸다(잠김 방지). 막히면 Hostinger 웹터미널이 복구 경로.

**✅ 체크:** 방화벽·fail2ban 켜짐, 비밀번호 접속 차단, 키 접속 정상.

---

# PART 2 · 두뇌와 오픈클로 설치

## 04. AI 엔진 설치 — 클로드 / 코덱스

> 화면 없는 서버에서도 **구독 OAuth 로그인**이 된다(디바이스 코드 방식). 종량 과금 없이 구독으로 운영.

### 클로드 코드 버전
```bash
curl -fsSL https://claude.ai/install.sh | bash          # sudo 금지!
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc
claude --version
claude   # 위저드: 로그인(Pro/Max) → 브라우저 URL → 코드 붙여넣기 → /exit
```
🙋 브라우저 OAuth 코드 입력은 사람이.

### 코덱스 버전
```bash
npm install -g @openai/codex
codex login        # ChatGPT로 로그인 → 화면에 뜬 URL 열고 코드 붙여넣기
codex login status # "Logged in using ChatGPT" 확인
```

**엔진 차이:** 클로드 코드는 파일 읽기/쓰기·웹 검색·복잡한 판단에 강하고 실행 전 확인 루프가 기본. 코덱스는 코드 생성·수정·반복 작업을 빠르게 처리.

**✅ 체크:** `claude --version` 또는 `codex login status`가 정상 출력.

## 05. 오픈클로 설치 + 24시간 켜두기

오픈클로를 처음 깔면 기본 봇 한 마리가 들어있다(성격은 디폴트 — 09단계에서 바꾼다).

```bash
# 오픈클로 설치
npm install -g openclaw
openclaw plugins install @openclaw/slack    # 슬랙 채널 플러그인

# 게이트웨이를 백그라운드 서비스(systemd)로
openclaw config set gateway.mode local
openclaw gateway install
```

24시간 유지 (핵심 2가지):
```bash
export XDG_RUNTIME_DIR=/run/user/0
loginctl enable-linger root                          # SSH 끊겨도·재부팅돼도 유지
systemctl --user enable --now openclaw-gateway.service   # 시작 + 부팅 자동
```

**✅ 체크:** 게이트웨이 서비스가 `active (running)`.

---

# PART 3 · 봇 깨우고 슬랙에 연결

## 06. 에이전트 설정 — 어떤 두뇌·모델을 쓸지

`~/.openclaw/openclaw.json`에 봇·엔진·모델·채널을 적는다. `openclaw config patch --stdin`으로 JSON 병합.

핵심 블록:
- **auth** — 클로드 `anthropic:claude-cli`(oauth), 코덱스 `openai-codex:<email>`(oauth)
- **agentRuntime** — 클로드 봇 = `claude-cli`, 코덱스 봇 = `codex-cli`
- **agents.list** — 봇별 id·workspace(09단계 폴더)·model·호칭(mentionPatterns)
- **bindings** — 슬랙 계정 → 어느 봇에게 보낼지 라우팅 (07단계 연결)

> ⚠️ **잘 막히는 2가지**
> 1. 기본 모델이 openai로 잡혀 있음 → "Missing API key for provider openai" 에러. `agents.defaults.model.primary`를 anthropic로 지정.
> 2. 슬랙 토큰을 환경변수로 두면 서비스가 못 읽어 안 켜짐 → config에 **토큰 글자를 그대로(리터럴)** 적기.

```bash
openclaw config validate
systemctl --user restart openclaw-gateway.service
```

**✅ 체크:** `config validate` 통과 + 봇·엔진·모델 연결.

## 07. 슬랙 봇 페어링

슬랙에서 앱을 만들어 토큰 2개를 받고 → 오픈클로에 등록. 이 단계가 끝나면 봇이 슬랙에서 대화에 답하기 시작한다(성격은 디폴트, 영혼은 09단계).

### ① 슬랙 앱 만들기 (매니페스트) 🙋 사람

[api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From a manifest** → 아래 JSON 붙여넣기. **권한·이벤트·Socket Mode가 한 번에** 설정된다. 봇 이름(`name` / `bot_user.display_name`)만 바꾸면 재사용 가능.

```json
{
  "display_information": {
    "name": "내봇이름",
    "description": "OpenClaw AI 에이전트",
    "background_color": "#611F69"
  },
  "features": {
    "app_home": {
      "home_tab_enabled": false,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "bot_user": {
      "display_name": "mybot",
      "always_online": true
    },
    "assistant_view": {
      "assistant_description": "OpenClaw AI 에이전트",
      "suggested_prompts": []
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "chat:write.customize",
        "chat:write.public",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "assistant_thread_context_changed",
        "assistant_thread_started",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": false
    },
    "org_deploy_enabled": false,
    "socket_mode_enabled": true,
    "token_rotation_enabled": false
  }
}
```

> 매니페스트 붙여넣기 주의: ① 바꿀 곳은 `"name"`과 `"display_name"` 2군데만 (나머지는 그대로). ② 주석(`//`) 금지 — 순수 JSON만 받음.

### ② 토큰 2개 발급 🙋 사람

토큰은 직접 발급해야 하고, 둘 다 발급 즉시 메모장에 복사·저장한다(특히 App Token은 한 번 닫으면 다시 못 봄).

| 토큰 | 발급 경로 (클릭 순서) | 형식 |
|------|---------------------|------|
| App Token (실시간 수신) | 왼쪽 **Basic Information** → 스크롤 **App-Level Tokens** → **Generate Token and Scopes** → 이름 아무거나 → scope `connections:write` **하나만** → Generate | `xapp-...` |
| Bot Token (메시지 권한) | 왼쪽 **Install App** → **Install to Workspace** → 권한 승인 허용 | `xoxb-...` |

> ⚠️ scope는 반드시 `connections:write`(빠지면 Socket Mode 안 됨). `xapp-` 토큰은 창 닫으면 다시 못 보니 즉시 저장.

### ③ 오픈클로에 연결
```bash
openclaw channels add --channel slack \
  --name "내 워크스페이스" \
  --bot-token xoxb-여기에-봇토큰 \
  --app-token xapp-여기에-앱토큰

openclaw channels list   # slack ✓ configured 확인
```

> ⚠️ **토큰 끝 줄바꿈 = "조용한 실패"**: 복사 버튼이 토큰 뒤 줄바꿈(`\n`)까지 복사하면 에러 없이 봇이 무응답한다(가장 흔한 함정). 변수로 공백 제거 후 연결하면 안전:
> ```bash
> BOT="$(echo -n 'xoxb-...붙여넣기...' | tr -d '[:space:]')"
> APP="$(echo -n 'xapp-...붙여넣기...' | tr -d '[:space:]')"
> openclaw channels add --channel slack --name "내 워크스페이스" \
>   --bot-token "$BOT" --app-token "$APP"
> ```

### ④ 슬랙에서 봇 찾아 페어링 — 첫 대화 승인 (이걸 빼먹으면 봇이 답을 안 한다!)

②에서 슬랙에 봇을 설치하고 ③에서 오픈클로에 연결했으니, 이제 슬랙에서 봇을 찾아 첫 대화를 튼다. 오픈클로는 아무나 봇에게 DM 못 하게 처음 한 번 "이 사람이 진짜 주인인지" 확인한다(페어링). 한 번만 하면 이후엔 바로 응답.

1. 슬랙 왼쪽 사이드바 맨 아래 **앱 영역**에서 ②에서 설치한 봇을 찾아 클릭 → **DM 보내기**("안녕!"이면 충분). 봇이 안 보이면: 앱 영역 점 3개(⋯) → "표시 및 정렬" → 대화 필터를 **"모두"**로.
2. 봇이 **페어링 안내 메시지**를 보냄 (당황 금지 — "주인한테 확인받고 와"라는 뜻):
   ```
   OpenClaw: access not configured.

   Your Slack user id: U0XXXXXXXXX
   Pairing code:

    ABCD1234

   Ask the bot owner to approve with:
   openclaw pairing approve slack ABCD1234
   ```
3. 봇이 알려준 명령을 **VPS 터미널에 그대로** 붙여넣어 승인 (`--notify` 붙이면 승인 후 봇이 슬랙으로 알림):
   ```bash
   openclaw pairing approve slack ABCD1234 --notify
   ```
4. 다시 슬랙에서 말 걸기 → **이번엔 답한다!** 🎉

> ⚠️ 페어링 코드는 1시간 후 만료. 보이는 즉시 승인. 만료되면 DM을 다시 보내 새 코드를 받는다.

**안 될 때:** 채널에서 쓰려면 `/invite @봇이름`(DM은 초대 불필요). 어디서 막히는지 보려면 `openclaw logs --follow` 켜고 DM 다시 보내기.

**✅ 체크:** 페어링 승인 후 봇이 디폴트 성격으로 답한다 — 봇이 살아남! 🎉

## 08. 봇 접근 제한 & 세부설정

> 봇은 강한 권한으로 명령을 실행한다. 아무나 슬랙으로 위험한 명령을 시키면(프롬프트 인젝션) 서버가 털릴 수 있다. 연결 직후 바로 제한.

- `channels.slack.allowFrom` 에 **내 슬랙 user ID만** 넣어 봇 호출을 본인으로 제한
- `commands.ownerAllowFrom` 도 설정

### 주요 설정 옵션 (`~/.openclaw/openclaw.json`)

| 옵션 | 무엇 | 값 |
|------|------|-----|
| `dmPolicy` | DM 허용 범위 | **pairing**(주인만, 기본·권장) / **allowlist**(특정 ID들) / **open**(전원, `allowFrom:["*"]` 필요) |
| `groupPolicy` | 채널 활성화 | **open**(초대된 모든 채널) / **allowlist**(지정 채널만) |
| `replyToMode` | 답변 위치 | **all**(스레드로 답) 등 |
| `mentionPatterns` | 이름 호출 | 등록하면 @멘션 없이 이름만 불러도 반응 (예: `["뽀야","뽀야야"]`) |

> ⚠️ 특정 채널에 `requireMention: false`(모든 메시지 반응)를 두면 봇이 채널의 모든 메시지를 읽어 **토큰 소모가 급증**한다. 꼭 필요한 채널에만.

### 💬 JSON이 어렵다면 — 봇에게 시키기

슬랙에서 봇에게 이 프롬프트를 보내면 알아서 설정한다:
```
~/.openclaw/openclaw.json 의 내 슬랙 계정 설정을 이렇게 맞춰줘:
- DM은 나만 받기 (dmPolicy: pairing, 또는 allowlist에 내 슬랙 ID)
- 답변은 스레드에서 (replyToMode: all)
- 내 이름 "(___)"으로 불러도 반응하게 (mentionPatterns에 추가)
그리고 AGENTS.md에 "공개 채널에서 민감한 명령 실행 금지" 규칙도 추가해줘.

끝나면 게이트웨이를 재시작하고,
"재시작 끝났으니 슬랙 DM에서 저 다시 한번 불러주세요"라고 보고해줘.
(재시작하면 네 슬랙 세션도 끊겨서 내가 먼저 말 걸어야 깨어나니까.)
```

**✅ 체크:** 나 외의 사람이 봇을 불러도 반응하지 않고, DM·채널·이름호출 규칙이 내 취향대로 설정됨.

---

# PART 4 · 봇에게 영혼(성격) 넣기

## 09. 봇 워크스페이스 만들기 — SOUL · USER · AGENTS

봇 하나는 자기 워크스페이스 폴더 안에 성격 파일을 둔다. 봇이 켜질 때마다 자동 주입됨. **오픈클로가 자동으로 읽는 건 3개(SOUL·AGENTS·USER)** — 신규 봇은 이 3개로 시작.

### 폴더 구조
```
~/.openclaw/
├── workspace-<내봇이름>/         # 봇 하나 = 폴더 하나
│   ├── SOUL.md                  # 영혼: 성격·미션·말투
│   ├── USER.md                  # 집사(나)는 누구인가
│   └── AGENTS.md                # 행동 규칙 (처음엔 빈 골격)
├── skills/<스킬이름>/            # 운영·자동화 스킬
└── agents/<id>/agent/auth-profiles.json  # 봇별 로그인 정보(코덱스)
```
이 폴더 이름을 06단계 `agents.list`의 workspace에 적으면 연결된다.

### ① SOUL.md
```markdown
# SOUL.md — <내 봇 이름>

## 미션
(이 봇이 내 일에서 해결하려는 것 — 한 줄)

## 관계
나는 <봇 이름>. <나와의 관계 한 줄>

## 말투
- <나에게는 어떻게>
- <외부에는 어떻게>
- <금지어 / 금지 행동>

## 가치관
- <봇이 우선하는 것 3가지>
```
> 30줄 안에서. 미션·관계·말투 3가지만 채우고 운영하며 늘린다. "친절하게" 같은 추상어 대신 구체적 행동으로.

### ② USER.md
```markdown
# USER.md — 집사

- **이름**: <내 이름>
- **호칭**: <봇이 나를 부를 호칭>
- **타임존**: Asia/Seoul (KST, UTC+9)
- **하는 일**: <한 줄 직무>
- **선호 스타일**: <답변 스타일 1~2개>
- **자주 쓰는 도구**: <도구 이름들>
```
> ⚠️ 비밀번호·API 키·주민번호 금지. 워크스페이스 폴더가 통째로 공유될 수 있음.

### ③ AGENTS.md (빈 골격으로 시작)
```markdown
# AGENTS.md — <내 봇 이름>

## Red Lines
(절대 넘지 않는 선 — 사고 나면 추가)

## 행동 규칙
(반복적으로 잘못하는 것 — 사고 나면 추가)

## 보안
(공개 채널에서 하면 안 되는 것)
```
> AGENTS.md는 처음엔 비어있어야 정상. 봇이 실수할 때마다 한 줄씩 누적 → 봇 길들이기의 핵심.

### 💬 더 쉬운 방법 — 봇에게 직접 시키기

파일 작성이 부담되면, 07단계에서 살아난 봇에게 슬랙에서 이 프롬프트를 보내 세팅을 맡긴다(봇이 자기 워크스페이스 파일 쓰기 권한이 있을 때):

```
이제 너를 내 전용 봇으로 세팅할게. 너의 워크스페이스 폴더에
SOUL.md · USER.md · AGENTS.md 3개 파일을 아래 내용으로 만들어줘.

[SOUL.md]
- 미션: (이 봇이 내 일에서 해결할 것 — 한 줄)
- 관계: 나는 (봇 이름), (나와의 관계 한 줄)
- 말투: 나에게는 (___), 외부에는 (___), 금지어/금지행동: (___)
- 가치관: (우선하는 것 3가지)

[USER.md]
- 이름: (내 이름) / 호칭: (나를 이렇게 불러줘)
- 타임존: Asia/Seoul (KST)
- 하는 일: (한 줄 직무)
- 선호 스타일: (답변 스타일 1~2개)
- 자주 쓰는 도구: (___)
  ※ 비밀번호·API키 같은 민감정보는 절대 넣지 마

[AGENTS.md]
- 지금은 'Red Lines / 행동 규칙 / 보안' 섹션 제목만 있는
  빈 골격으로 만들어줘 (내용은 나중에 사고 날 때마다 추가할게)

다 만들면 저장하고, 새 성격이 적용되게 세션을 재시작해줘.
```

> ⚠️ 파일을 직접 고쳤다면 같은 대화창에선 새 성격이 적용 안 됨. 반드시 세션을 새로 시작(봇 재시작)해야 적용. (봇에게 시킨 경우 프롬프트의 "재시작"이 처리)

**✅ 체크:** 봇 재시작 후 디폴트가 아니라 내가 정한 성격으로 답한다. 진짜 동료 완성! 🦞

---

# PART 5 · 도구 달기 & 파일 연결 (선택)

## 10. MCP로 도구 달기 — 인기 MCP 카탈로그

MCP는 봇에게 앱(도구)을 달아주는 표준. 필요한 것만 골라 `~/.openclaw/openclaw.json`의 `mcp.servers`에 추가하거나 `openclaw mcp set ...`으로 등록.

**연결 2방식:** ① 원격(HTTP) = URL만 적음 / ② 로컬 실행(npx) = 봇이 작은 프로그램 실행.

| 도구 | 방식 | 연결 정보 | 인증 |
|------|------|-----------|------|
| 🎨 Figma | 원격 | `https://mcp.figma.com/mcp` | OAuth (유료 플랜) |
| 🐙 GitHub | 원격 | `https://api.githubcopilot.com/mcp/` | OAuth 또는 PAT |
| 📝 Notion | 원격 | `https://mcp.notion.com/mcp` | OAuth |
| 📚 Context7 (라이브러리 문서) | 원격/로컬 | `https://mcp.context7.com/mcp` 또는 `npx -y @upstash/context7-mcp` | API 키(권장) |
| 🎭 Playwright (브라우저 자동화) | 로컬 | `npx -y @playwright/mcp@latest` | 불필요 |
| 📁 구글 워크스페이스 (드라이브·캘린더·문서) | 로컬 | `@piotr-agier/google-drive-mcp` | Google OAuth |
| 🌐 Fetch (웹페이지 읽기) | 로컬 | `uvx mcp-server-fetch` (npx 아님, Python) | 불필요 |

> ⚠️ 구글 드라이브 공식 `@modelcontextprotocol/server-gdrive`와 슬랙 공식 패키지는 **아카이브(유지보수 중단)**됨. 구글은 위 `@piotr-agier/google-drive-mcp` 사용.
> ⚠️ 구글 MCP는 OAuth 키 경로뿐 아니라 **사용자 토큰 경로**(`GOOGLE_DRIVE_MCP_TOKEN_PATH`)도 명시해야 연결됨.

## 11. 봇 성격·코드 git 배포 체계

봇 성격 파일과 작업 코드는 GitHub에 두고 VPS가 받아온다(백업·버전관리 겸함).

```
GitHub(private, 소스) ──pull──▶ VPS 클론 ──배포 스크립트──▶ 라이브 워크스페이스
        ▲push (내 맥에서 수정)
```

- GitHub **배포키**(deploy key)를 VPS에 등록 (읽기전용/쓰기 구분)
- 배포 스크립트 = `git pull` + 정의 파일만 복사 (런타임 상태 MEMORY/HEARTBEAT는 안 건드림 → 충돌 방지)
- 워크플로우: 내 맥에서 수정 → commit → push → VPS에서 배포 스크립트 실행

> Obsidian Sync·iCloud 같은 폐쇄 동기화는 봇이 접근 못 함. git이 표준.

## 12. 로컬 파일·옵시디안 연동

- **권장:** 필요한 폴더/프로젝트만 git repo로 봇에 연결 (11과 동일 패턴). 경로가 repo로 자연히 제한되고 24시간 작동.
- **옵시디안 위키:** vault를 git+GitHub로 → VPS가 클론 → 봇은 읽기 + `_inbox/`에만 새 노트 발행. 내 맥은 Obsidian Git 플러그인으로 자동 동기화.

> ⚠️ 노드 페어링 비권장: 오픈클로 노드는 "이 폴더만" 경로 제한이 안 돼 맥 사용자 권한 전체가 노출됨. git 핸드오프가 더 안전.

---

## 막혔을 때 — 함정 총정리

| # | 증상 | 해결 |
|---|------|------|
| 1 | 봇이 새 성격대로 안 움직임 | 파일 저장 후 봇(세션) 재시작 |
| 2 | "Missing API key for provider openai" | 기본 모델을 anthropic로 지정 (06단계) |
| 3 | 서비스가 안 켜짐 (토큰 문제) | 슬랙 토큰을 config에 글자 그대로 |
| 4 | 슬랙에서 봇이 연결 안 됨 | App Token scope `connections:write` + Socket Mode ON |
| 5 | 재부팅·SSH끊김에 봇 죽음 | `loginctl enable-linger` + 서비스 enable |
| 6 | 구글(MCP) 연동 안 됨 | `GOOGLE_DRIVE_MCP_TOKEN_PATH` 경로 명시 |
| 7 | 공개 IP 공격당함 | 키전용 SSH + ufw + fail2ban (봇 전에!) |

## 운영 치트시트

```bash
systemctl --user status openclaw-gateway       # 상태
systemctl --user restart openclaw-gateway       # 재시작
journalctl --user -u openclaw-gateway -f         # 실시간 로그
```
> 모든 `systemctl --user` 명령은 먼저 `export XDG_RUNTIME_DIR=/run/user/0` 가 필요할 수 있음.

---

## 🔄 이미 로컬 맥에서 봇을 쓰던 경우 (이전 전용)

처음부터 새로 만드는 게 아니라 맥에서 돌리던 봇을 VPS로 옮기는 거라면, 아래 3가지만 추가로:

### ① 코덱스 로그인 정보 옮기기 (코덱스 봇만)
클로드는 VPS에서 다시 로그인하면 되지만, 코덱스는 로그인 정보가 파일로 저장돼서 옮겨야 한다.
```bash
# 내 맥에서
scp -i ~/.ssh/myvps_key ~/.codex/auth.json root@<VPS-IP>:~/.codex/auth.json
scp -i ~/.ssh/myvps_key \
  ~/.openclaw/agents/<봇id>/agent/auth-profiles.json \
  root@<VPS-IP>:~/.openclaw/agents/<봇id>/agent/auth-profiles.json
```
> 코덱스는 토큰을 `~/.codex/auth.json`에도, 오픈클로 봇별 폴더(`auth-profiles.json`)에도 보관. 둘 다 있어야 작동. (모르면 "코덱스 봇 인증 실패")

### ② 맥에서 돌던 봇은 반드시 끄기
같은 슬랙 토큰을 맥·VPS가 동시 연결하면 메시지가 둘로 나뉘어 간헐 무응답·이중 응답. VPS로 옮겼으면 맥 봇 종료.

### ③ 봇 성격·작업 파일 가져오기
맥의 SOUL·AGENTS 등과 작업 폴더는 GitHub를 거쳐 가져온다(11·12단계와 동일). 맥에서 commit·push → VPS에서 pull.

---

*출처: 「VPS에서 AI 에이전트 24시간 운영하기」 · OpenClaw 에이전트 최소 파일 구성 · 봇 온보딩 가이드 (2026-05, 실전 기반). 모든 토큰·IP·계정은 플레이스홀더 처리됨. by 올라운드플레이.*
