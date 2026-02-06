# 🦞 Windows 환경 OpenClaw + Discord 연동 완전 가이드

> **Windows WSL2 환경에서 OpenClaw을 설치하고, Discord 봇과 연결한 뒤, Cursor IDE + Claude Code로 실시간 개발하는 전체 과정을 담은 실전 가이드입니다.**
>
> 이 가이드는 2026년 2월 기준, 실제 설치 과정에서 발생한 모든 문제와 해결 방법을 포함합니다.

---

## 📋 목차

- [전체 아키텍처](#전체-아키텍처)
- [사전 요구사항](#사전-요구사항)
- [PHASE 1: WSL2 환경 구축](#phase-1-wsl2-환경-구축)
- [PHASE 2: Node.js 22+ 설치](#phase-2-nodejs-22-설치)
- [PHASE 3: OpenClaw 설치 및 온보딩](#phase-3-openclaw-설치-및-온보딩)
- [PHASE 4: Discord 봇 생성 및 연결](#phase-4-discord-봇-생성-및-연결)
- [PHASE 5: Cursor IDE + Claude Code 연동](#phase-5-cursor-ide--claude-code-연동)
- [PHASE 6: 실시간 개발 워크플로우](#phase-6-실시간-개발-워크플로우)
- [트러블슈팅 가이드](#트러블슈팅-가이드)
- [비용 및 보안](#비용-및-보안)
- [명령어 치트시트](#명령어-치트시트)

---

## 전체 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│  Windows 11/10                                          │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Cursor IDE (Windows 네이티브)                    │    │
│  │  ├─ WSL2 Remote 연결                             │    │
│  │  ├─ Claude Code Extension (터미널 에이전트)       │    │
│  │  └─ 통합 터미널 → Ubuntu (WSL2)                  │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                               │
│  ┌─────────────────────────────────────────────────┐    │
│  │  WSL2 (Ubuntu 24.04)                             │    │
│  │  ├─ Node.js 22+                                  │    │
│  │  ├─ OpenClaw Gateway (포트 18789)                │    │
│  │  └─ Discord 채널 플러그인                         │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                               │
└─────────────────────────┼───────────────────────────────┘
                          │ (인터넷)
              ┌───────────┴───────────┐
              │   Discord Bot API     │
              │   LLM API (Anthropic  │
              │   / OpenAI 등)        │
              └───────────────────────┘
```

---

## 사전 요구사항

| 항목 | 요구사항 |
|------|----------|
| OS | Windows 10 (Version 2004+) 또는 Windows 11 |
| RAM | 최소 4GB, **8GB 이상 권장** |
| 권한 | Windows 관리자 계정 |
| 인터넷 | 안정적인 인터넷 연결 |
| LLM 인증 | Anthropic API 키 **또는** Claude Max/Pro 구독 |
| Discord | Discord 계정 + 서버 관리자 권한 |

---

## PHASE 1: WSL2 환경 구축

### 1-1. WSL2 + Ubuntu 설치

PowerShell을 **관리자 권한**으로 열고:

```powershell
wsl --install -d Ubuntu-24.04
```

설치 후 **PC를 재부팅**합니다.

재부팅 후 Ubuntu 터미널이 자동으로 열리면 **UNIX 사용자명과 비밀번호**를 설정합니다.

> ⚠️ 이 사용자명과 비밀번호를 반드시 기억하세요. `sudo` 명령에 계속 사용됩니다.

### 1-2. Ubuntu 시스템 업데이트

```bash
sudo apt update && sudo apt upgrade -y
```

### 1-3. systemd 활성화 확인

OpenClaw 데몬이 systemd 서비스로 등록되므로 반드시 필요합니다:

```bash
cat /etc/wsl.conf
```

`[boot]` 섹션에 `systemd=true`가 없다면:

```bash
sudo tee /etc/wsl.conf << 'EOF'
[boot]
systemd=true

[network]
generateResolvConf=true
EOF
```

PowerShell에서 WSL 재시작:

```powershell
wsl --shutdown
wsl
```

확인:

```bash
systemctl --version   # 버전 출력되면 정상
```

---

## PHASE 2: Node.js 22+ 설치

OpenClaw는 **Node.js 22 이상**을 필수로 요구합니다:

```bash
# nvm 설치
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# 쉘 재로드
source ~/.bashrc

# Node.js 22 설치 및 기본값 설정
nvm install 22
nvm use 22
nvm alias default 22

# 확인
node --version    # v22.x.x
npm --version
```

> ⚠️ **주의:** nvm이 새 터미널에서 로드되지 않을 수 있습니다. 아래를 `~/.bashrc`에 추가:
> ```bash
> echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.bashrc
> echo '[ -s "$NVM_DIR/nvm.sh" ] && source "$NVM_DIR/nvm.sh"' >> ~/.bashrc
> source ~/.bashrc
> ```

---

## PHASE 3: OpenClaw 설치 및 온보딩

### 3-1. OpenClaw CLI 설치

```bash
npm install -g openclaw@latest
openclaw --version    # openclaw/2026.x.x
```

### 3-2. 온보딩 위저드 실행

```bash
openclaw onboard --install-daemon
```

### 3-3. 온보딩 선택지 가이드

| 단계 | 질문 | 권장 선택 | 이유 |
|------|------|-----------|------|
| 1 | Onboarding mode | Manual | 세부 설정 제어 가능 |
| 2 | What to set up? | Local gateway (this machine) | 로컬 실행 |
| 3 | Model/auth provider | Anthropic | Claude 모델 사용 |
| 4 | **Auth method** | ⭐ 아래 참조 | 구독 여부에 따라 다름 |
| 5 | Default model | 테스트: `claude-haiku-4-5` | 비용 절약 |
| 6 | Gateway port | 18789 (기본값) | — |
| 7 | **Gateway bind** | **LAN (0.0.0.0)** | WSL2에서 Windows 브라우저 접근 필요 |
| 8 | Gateway auth | Password | 보안 필수 |
| 9 | Tailscale exposure | **Off** | 로컬 테스트에는 불필요 |
| 10 | Configure skills? | Yes → **Skip for now** | 나중에 개별 설치 |
| 11 | API key 질문들 | 전부 **No** | Discord 연결과 무관 |
| 12 | Hatch your bot? | **Hatch in TUI** | 첫 대화로 부트스트랩 완료 |

### 3-4. ⭐ 인증 방식 선택 (매우 중요)

#### Claude Max/Pro 구독자 → setup-token 방식

API 크레딧이 없으므로 **반드시 setup-token 방식**을 사용해야 합니다.

```
✅ Anthropic token (paste setup-token)  ← 이것 선택
○ Anthropic API key
○ Back
```

**토큰 생성 방법:**

1. **별도 터미널**을 열고 (온보딩 터미널은 그대로 두고):
   ```bash
   # Claude Code CLI가 없으면 먼저 설치
   npm install -g @anthropic-ai/claude-code
   
   # setup-token 생성
   claude setup-token
   ```
2. 출력된 토큰 문자열을 복사
3. 온보딩 화면에 붙여넣기

> ⚠️ `claude` 명령이 안 될 경우: Cursor의 Claude Code Extension이 인증된 상태라면, Cursor 내 WSL 터미널에서 실행하면 됩니다.

#### API 키 사용자 → API key 방식

[console.anthropic.com](https://console.anthropic.com) → API Keys → Create Key → `sk-ant-...` 복사 → 붙여넣기

> 🔴 **Max/Pro 구독자가 API key를 선택하면 크레딧 잔액 0으로 인해 모든 응답이 실패합니다!**
> ```
> HTTP 400 invalid_request_error: Your credit balance is too low to access the Anthropic API.
> ```
> 이 에러가 보이면 인증 방식을 setup-token으로 변경하세요:
> ```bash
> openclaw models auth paste-token --provider anthropic
> # → claude setup-token으로 생성한 토큰 붙여넣기
> openclaw gateway restart
> ```

### 3-5. TUI에서 부트스트랩 완료

온보딩 마지막에 **Hatch in TUI** 선택 후 봇과 첫 대화를 진행합니다.

봇이 이름/정체성 설정을 물어보면:

```
기본값으로 채우고 BOOTSTRAP.md 지워줘
```

이렇게 입력하면 부트스트랩이 자동 완료됩니다.

> 🔴 **부트스트랩이 PENDING 상태면 Discord에서 자동 응답이 안 됩니다!**
> ```
> Agent: main
> Bootstrap: PENDING   ← 이 상태에서는 Discord 응답 불가
> ```
> TUI에서 대화를 진행하여 반드시 DONE으로 변경해야 합니다.

### 3-6. 상태 확인

```bash
openclaw status --all
openclaw health
openclaw doctor
```

정상 상태:
```
Agents: 1 total · 0 bootstrapping · 1 active
Bootstrap: (없음 = DONE)
```

---

## PHASE 4: Discord 봇 생성 및 연결

### 4-1. Discord Developer Portal에서 봇 생성

1. [Discord Developer Portal](https://discord.com/developers/applications) 접속
2. **New Application** → 이름 입력 (예: `OpenClaw-Bot`)
3. 좌측 **Bot** 메뉴 클릭
4. **Reset Token** → 토큰을 **안전한 곳에 저장**

> 🔴 토큰은 비밀번호와 같습니다. 채팅, 커밋, 스크린샷에 노출하지 마세요.

### 4-2. Privileged Gateway Intents 활성화

Bot 설정 페이지 → **Privileged Gateway Intents**:

- ✅ **Message Content Intent** — 필수 (메시지 텍스트 읽기)
- ✅ **Server Members Intent** — 권장 (allowlist, 멤버 조회)
- ⬜ Presence Intent — 불필요

**Save Changes** 클릭

> 🔴 Message Content Intent가 꺼져있으면 봇이 서버에 접속은 되지만 **메시지를 읽지 못합니다.**

### 4-3. 봇을 Discord 서버에 초대

1. 좌측 **OAuth2** → **URL Generator**
2. SCOPES: `bot` 체크
3. BOT PERMISSIONS 체크:
   - ✅ View Channels
   - ✅ Send Messages
   - ✅ Read Message History
   - ✅ Embed Links
   - ✅ Attach Files
   - ✅ Add Reactions
4. 생성된 URL 복사 → 브라우저에서 열기 → 서버 선택 → 승인

> 💡 이미 초대된 봇이 있어도 같은 URL로 다시 초대하면 **권한만 업데이트**됩니다. 봇이 2개가 되지 않습니다.

> 🔴 **BOT PERMISSIONS를 체크하지 않으면 봇이 메시지를 읽거나 보내지 못합니다!**

### 4-4. Discord Developer Mode 활성화

Discord 앱:
1. 사용자 설정 → **고급** → **개발자 모드** ON
2. 서버 이름 우클릭 → **서버 ID 복사** (나중에 필요)
3. **텍스트** 채널 우클릭 → **채널 ID 복사**

> ⚠️ 반드시 **텍스트(채팅) 채널**을 선택하세요. OpenClaw는 음성 채널을 지원하지 않습니다.

### 4-5. 채널 권한 설정

Discord에서 해당 텍스트 채널 → ⚙️ 설정 → **권한** → **OpenClaw-Bot** 추가:

- ✅ 채널 보기
- ✅ 메시지 보내기
- ✅ 메시지 기록 보기

**"채팅 채널 카테고리와 동기화되지 않은 권한"** 경고가 나오면 → **지금 동기화하기** 클릭

### 4-6. OpenClaw에 Discord 연결

```bash
# Discord 플러그인 활성화
openclaw plugins enable discord

# 게이트웨이 재시작
openclaw gateway restart

# Discord 채널 추가 (한 줄로 입력)
openclaw channels add --channel discord --name "discord" --token "봇_토큰_여기_붙여넣기"
```

> ⚠️ `\`로 줄바꿈하면 에러가 발생할 수 있습니다. **반드시 한 줄로** 입력하세요.
> ```
> error: too many arguments for 'add'. Expected 0 arguments but got 6.
> ```
> 이 에러가 나오면 줄바꿈 없이 한 줄로 재입력하세요.

### 4-7. Guild(서버) 설정 추가

> 🔴 **이 단계를 빠뜨리면 DM은 되지만 서버 채널에서 봇이 응답하지 않습니다!**

`~/.openclaw/openclaw.json`을 직접 편집합니다:

```bash
nano ~/.openclaw/openclaw.json
```

`channels` > `discord` 섹션에 `guilds`를 추가:

```json
"channels": {
    "discord": {
      "name": "discord",
      "enabled": true,
      "token": "봇토큰...",
      "guilds": {
        "여기에_서버ID_입력": {
          "requireMention": true
        }
      }
    }
}
```

> 🔴 guild 설정 시 `"enabled": true`를 넣지 마세요! Unrecognized key 에러가 발생합니다:
> ```
> Config validation failed: channels.discord.guilds.<ID>: Unrecognized key: "enabled"
> ```
> `requireMention`만 넣으면 됩니다.

저장: `Ctrl + O` → `Enter` → `Ctrl + X`

```bash
openclaw gateway restart
```

### 4-8. 연결 확인

```bash
# 채널 상태
openclaw channels status

# 전체 상태
openclaw status --all
```

정상 출력:
```
Discord │ ON │ OK │ token config · accounts 1/1
```

### 4-9. Discord 테스트

**서버 채널:**
```
@OpenClaw-Bot 안녕하세요
```

**DM (최초 접근 시 페어링 필요):**
1. 봇에게 DM 전송
2. 봇이 페어링 코드를 알려줌
3. 터미널에서 승인:
   ```bash
   openclaw pairing approve discord <페어링코드>
   ```
4. DM 재전송 → 응답 확인

**CLI 테스트:**
```bash
openclaw message send --channel discord --target <채널ID> --message "테스트 메시지"
```

---

## PHASE 5: Cursor IDE + Claude Code 연동

### 5-1. Cursor 설치

[cursor.com](https://www.cursor.com/)에서 Windows 버전 다운로드 및 설치

### 5-2. WSL 확장 설치 및 연결

1. Cursor에서 `Ctrl + Shift + X` → **"WSL"** 검색 → 설치
2. `Ctrl + Shift + P` → **"WSL: Connect to WSL"** 실행
3. 좌측 하단에 `WSL: Ubuntu-24.04` 표시 확인

### 5-3. 프로젝트 폴더 열기

```bash
mkdir -p ~/dev/openclaw-project
```

Cursor에서 `Ctrl + O` → `/home/<사용자명>/dev/openclaw-project` 열기

> 🔴 **반드시 WSL 내부 경로**(`/home/...`)에서 작업하세요.
> `/mnt/c/...` 경로에서 작업하면 I/O 성능이 크게 저하됩니다.

### 5-4. Claude Code 설치

**방법 A: Extension (권장)**
1. `Ctrl + Shift + X` → **"Claude Code"** 검색 → 설치
2. 사이드바 Claude Code 아이콘 → Anthropic 계정 로그인

**방법 B: CLI**
```bash
npm install -g @anthropic-ai/claude-code
claude
```

> ⚠️ Claude Code CLI는 **Windows 네이티브(PowerShell/CMD)에서 실행 불가**합니다:
> ```
> Error: No suitable shell found. Claude CLI requires a POSIX shell environment.
> ```
> 반드시 WSL 터미널에서 실행하세요.

### 5-5. IDE 연결 확인

Claude Code 실행 후:
```
/ide
```

정상: `IDE: Cursor (connected)`

> `/ide`에서 "No IDE detected"가 나오면:
> - Cursor가 WSL Remote 모드로 연결되어 있는지 확인
> - WSL 터미널에서 `cursor .` 으로 재실행
> - Claude Code Extension 재설치

### 5-6. 프로젝트 메모리 초기화

```
/init
```

`.claude/claude.md` 파일이 생성되어 프로젝트 컨텍스트가 Claude Code에 전달됩니다.

---

## PHASE 6: 실시간 개발 워크플로우

### 6-1. Claude Code로 OpenClaw 설정 수정

```
@claude openclaw.json에서 Discord 채널의 requireMention을 false로 변경해줘
```

### 6-2. 커스텀 스킬 개발

```
@claude OpenClaw용 커스텀 스킬 만들어줘.
Discord에서 "!요약"이면 최근 50개 메시지를 요약하는 기능
```

### 6-3. 실시간 디버깅

```bash
# 터미널 1: 실시간 로그 모니터링
journalctl --user -u openclaw-gateway -f

# 터미널 2: Claude Code로 로그 분석
@claude 이 에러 로그 분석하고 해결 방법 알려줘

# 터미널 3: 수정 후 재시작
openclaw gateway restart
```

### 6-4. 자동화 (Cron)

```bash
openclaw cron add "0 9 * * *" "오늘 일정 요약해줘"
```

---

## 트러블슈팅 가이드

### 🔴 (no output) — 봇이 응답하지 않음 (TUI)

**증상:**
```
Wake up, my friend!
(no output)
tokens 0/200k (0%)   ← 토큰 사용량 0%
```

**원인 1: API 크레딧 부족 (Max/Pro 구독자가 API key 사용)**
```
HTTP 400 invalid_request_error: Your credit balance is too low
```

**해결:**
```bash
# setup-token 방식으로 인증 변경
claude setup-token           # 다른 터미널에서 토큰 생성
openclaw models auth paste-token --provider anthropic  # 토큰 붙여넣기
openclaw gateway restart
```

**원인 2: API 키 오류**

[console.anthropic.com](https://console.anthropic.com) → Billing에서 잔액 확인

---

### 🔴 Bootstrap PENDING — Discord 자동 응답 안 됨

**증상:**
```
Agents: 1 total · 1 bootstrapping
Bootstrap: PENDING
```

**해결:**
```bash
openclaw tui
```

TUI에서 대화 후 봇이 이름/정체성 설정을 물어보면:
```
기본값으로 채우고 BOOTSTRAP.md 지워줘
```

완료 후:
```bash
# Ctrl+C로 TUI 종료
openclaw gateway restart
```

확인:
```bash
openclaw status --all
# Agents: 1 total · 0 bootstrapping ← DONE
```

---

### 🔴 Discord DM은 되지만 서버 채널에서 응답 없음

**원인: guild 설정 누락**

**해결:** `~/.openclaw/openclaw.json` 편집:

```json
"channels": {
    "discord": {
      "name": "discord",
      "enabled": true,
      "token": "봇토큰...",
      "guilds": {
        "서버ID": {
          "requireMention": true
        }
      }
    }
}
```

> ⚠️ `"enabled": true`를 guild 안에 넣지 마세요:
> ```
> Unrecognized key: "enabled"
> ```

```bash
openclaw gateway restart
```

---

### 🔴 Discord 봇이 메시지를 감지하지 못함

**체크리스트:**

1. **Message Content Intent** 활성화 여부
   - Developer Portal → Bot → Privileged Gateway Intents → ✅ 확인

2. **Bot Permissions** 설정 여부
   - OAuth2 → URL Generator에서 권한 체크 → URL로 재초대

3. **채널 권한**
   - 채널 설정 → 권한 → 봇에게 메시지 보기/보내기/기록 보기 허용

4. **연결 상태**
   ```bash
   openclaw channels status
   # Discord │ ON │ OK  ← 정상
   ```

---

### 🔴 CLI 명령어 줄바꿈 에러

**증상:**
```
error: too many arguments for 'add'. Expected 0 arguments but got 6.
```

**원인:** `\`로 줄바꿈이 제대로 처리되지 않음

**해결:** 한 줄로 입력:
```bash
openclaw channels add --channel discord --name "discord" --token "토큰"
```

---

### 🔴 Config validation 에러

**증상:**
```
Config validation failed: channels.discord.guilds: Invalid input
```

**원인:** CLI의 `openclaw config set` 명령이 구조를 잘못 생성

**해결:** `nano ~/.openclaw/openclaw.json`으로 직접 편집

---

### 🔴 Claude Code CLI — Windows에서 실행 불가

**증상:**
```
Error: No suitable shell found. Claude CLI requires a POSIX shell environment.
```

또는 "내부 또는 외부 명령, 실행할 수 있는 프로그램, 또는 배치 파일이 아닙니다"

**해결:** 터미널을 WSL bash로 전환:
- Cursor 터미널 드롭다운 → **Ubuntu (WSL)** 선택
- 또는 터미널에 `wsl` 입력

---

### 🟡 Windows 브라우저에서 대시보드 접근 불가

**원인:** WSL2 가상 네트워크

**해결 1:** WSL IP로 직접 접근:
```bash
hostname -I   # IP 확인
# http://<IP>:18789 로 접근
```

**해결 2:** PowerShell에서 포트 포워딩:
```powershell
$WslIp = (wsl -- hostname -I).Trim().Split(" ")[0]
netsh interface portproxy add v4tov4 listenport=18789 listenaddress=0.0.0.0 connectport=18789 connectaddress=$WslIp
```
이후 `http://localhost:18789` 접근 가능

> ⚠️ WSL IP는 재부팅 시 변경되므로 포트 포워딩을 다시 설정해야 할 수 있습니다.

---

### 🟡 WSL2 네트워크 전체 문제

```powershell
# PowerShell (관리자)
wsl --shutdown
wsl
```

Windows 방화벽 또는 VPN이 WSL2 네트워크를 차단할 수 있습니다.

---

### 🟡 Gateway가 시작되지 않음

```bash
# 수동 시작
openclaw gateway --port 18789 --verbose

# 로그 확인
journalctl --user -u openclaw-gateway --no-pager -n 50

# systemd 서비스 재등록
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway
```

---

### 🟡 nvm이 새 터미널에서 로드 안 됨

```bash
echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.bashrc
echo '[ -s "$NVM_DIR/nvm.sh" ] && source "$NVM_DIR/nvm.sh"' >> ~/.bashrc
source ~/.bashrc
```

---

## 비용 및 보안

### 💰 비용 구조

| 항목 | 비용 | 비고 |
|------|------|------|
| OpenClaw | **무료** | 오픈소스 |
| WSL2 / Ubuntu | **무료** | Windows 내장 |
| Cursor IDE | 무료/유료 | Free 플랜으로 시작 가능 |
| Claude Code | 구독 필요 | Anthropic Pro($20)/Max($100,$200) |
| LLM API 호출 | **종량제** | ← 주요 비용 발생 지점 |

> ⚠️ **비용 주의:** OpenClaw는 LLM API를 지속적으로 호출합니다.
> - 테스트 단계: `claude-haiku-4-5` 사용 권장
> - 일부 사용자가 하루 $130~$150 API 비용 발생 보고
> - Max 구독자는 setup-token으로 구독 내 포함 사용 가능 (rate limit 있음)

**모델 변경:**
```bash
# 저렴한 모델로 변경
openclaw models set anthropic/claude-haiku-4-5

# 본격 사용 시
openclaw models set anthropic/claude-sonnet-4-5
```

### 🔒 보안 체크리스트

- [ ] Gateway 비밀번호 설정 완료
- [ ] Discord 봇 토큰을 코드/채팅/스크린샷에 노출하지 않음
- [ ] DM 정책: `pairing` 또는 `allowlist`
- [ ] 포트 18789을 공용 인터넷에 노출하지 않음
- [ ] 토큰 유출 시 즉시 재발급

```bash
# 보안 감사
openclaw security audit --deep
openclaw doctor
```

> 🔴 **토큰이 유출되면 즉시:**
> - Discord Developer Portal → Bot → **Reset Token**
> - `~/.openclaw/openclaw.json`에서 token 및 password 변경
> - `openclaw gateway restart`

---

## 명령어 치트시트

```bash
# ━━━ 설치 ━━━
wsl --install -d Ubuntu-24.04              # WSL2 설치
nvm install 22 && nvm use 22               # Node.js 22
npm install -g openclaw@latest             # OpenClaw 설치
openclaw onboard --install-daemon          # 온보딩 + 데몬

# ━━━ 인증 ━━━
claude setup-token                         # setup-token 생성 (Max/Pro)
openclaw models auth paste-token --provider anthropic  # 토큰 등록
openclaw models set anthropic/claude-haiku-4-5         # 모델 변경

# ━━━ Discord 연결 ━━━
openclaw plugins enable discord
openclaw gateway restart
openclaw channels add --channel discord --name "discord" --token "토큰"
openclaw pairing approve discord <CODE>    # DM 페어링 승인

# ━━━ 상태 확인 ━━━
openclaw status --all
openclaw health
openclaw doctor
openclaw channels status
systemctl --user status openclaw-gateway

# ━━━ 게이트웨이 관리 ━━━
systemctl --user start openclaw-gateway
systemctl --user stop openclaw-gateway
systemctl --user restart openclaw-gateway
journalctl --user -u openclaw-gateway -f   # 실시간 로그

# ━━━ 설정 파일 편집 ━━━
nano ~/.openclaw/openclaw.json             # 직접 편집 (권장)

# ━━━ TUI ━━━
openclaw tui                               # 터미널 UI 대화
# Ctrl+C로 종료

# ━━━ 메시지 테스트 ━━━
openclaw message send --channel discord --target <채널ID> --message "테스트"

# ━━━ Claude Code (Cursor 터미널) ━━━
claude                                     # Claude Code 시작
/ide                                       # IDE 연결 확인
/init                                      # 프로젝트 메모리 초기화
/model                                     # 현재 모델 확인
/status                                    # 전체 상태 확인
/compact                                   # 긴 대화 요약
```

---

## 📚 참고 자료

- [OpenClaw 공식 문서](https://docs.openclaw.ai/)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw Discord 채널 문서](https://docs.openclaw.ai/channels/discord)
- [OpenClaw Windows(WSL2) 가이드](https://docs.openclaw.ai/platforms/windows)
- [Discord Developer Portal](https://discord.com/developers/applications)
- [Claude Code 공식 문서](https://code.claude.com/docs/en/vs-code)

---

## 음성 지원 참고

현재 OpenClaw ↔ Discord 연결은 **텍스트 전용**입니다.

```
사용자 (음성) → [STT] → 텍스트 → OpenClaw → 텍스트 → [TTS] → 음성
```

- macOS/iOS: Talk Mode 지원 (음성 입출력)
- **Windows WSL: 음성 미지원**
- 음성이 필요하면 별도 STT/TTS 파이프라인 구축 필요 (Whisper API, ElevenLabs 등)
- 간편 대안: Discord 모바일 앱의 음성 입력(키보드 마이크) → 텍스트 변환 → 봇에 전송

---

*작성일: 2026-02-06 | OpenClaw 2026.2.3-1 | Windows 11 + WSL2 Ubuntu 24.04 환경 기준*
