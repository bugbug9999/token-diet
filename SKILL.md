---
name: token-diet
description: Claude Code 토큰 사용량 진단 + CLAUDE.md 다이어트 + 비용 최적화 제안
user-invocable: true
argument-hint: "[diagnose | apply | report]"
---

# Token Diet — Claude Code 토큰 절약 진단 스킬

Claude Code 세션의 토큰 소비를 분석하고, 실질적인 절약 방안을 제시한다.

## 사용법

- `/token-diet` 또는 `/token-diet diagnose` — 현재 상태 진단
- `/token-diet apply` — 진단 후 자동 최적화 적용
- `/token-diet report` — 진단 결과를 markdown 리포트로 저장

---

## Step 1: 매턴 토큰 소비 진단

매 API 턴마다 반복 전송되는 항목을 측정한다. **이것들이 비용의 핵심이다.**

### 1-1. CLAUDE.md 측정

```bash
# 글로벌 CLAUDE.md
GLOBAL_MD="$HOME/.claude/CLAUDE.md"
if [ -f "$GLOBAL_MD" ]; then
  SIZE=$(wc -c < "$GLOBAL_MD")
  LINES=$(wc -l < "$GLOBAL_MD")
  TOKENS=$((SIZE / 4))
  echo "글로벌 CLAUDE.md: ${SIZE} bytes / ${LINES}줄 / ~${TOKENS} tokens"
else
  echo "글로벌 CLAUDE.md: 없음"
fi

# 프로젝트 CLAUDE.md (cwd 기준)
for f in "./CLAUDE.md" "./.claude/CLAUDE.md" "./CLAUDE.local.md"; do
  if [ -f "$f" ]; then
    SIZE=$(wc -c < "$f")
    TOKENS=$((SIZE / 4))
    echo "프로젝트 $(basename $f): ${SIZE} bytes / ~${TOKENS} tokens"
  fi
done

# .claude/rules/ 디렉토리
if [ -d "./.claude/rules" ]; then
  TOTAL=$(find ./.claude/rules -name "*.md" -exec cat {} + | wc -c)
  echo ".claude/rules/: ${TOTAL} bytes / ~$((TOTAL / 4)) tokens"
fi
```

**판정 기준:**
| 등급 | CLAUDE.md 크기 | 매턴 토큰 | 조치 |
|------|---------------|----------|------|
| 🟢 양호 | < 3KB | < 750 | 유지 |
| 🟡 주의 | 3-8KB | 750-2000 | 분리 검토 |
| 🔴 과다 | > 8KB | > 2000 | 즉시 다이어트 |

### 1-2. MEMORY.md 측정

```bash
# 프로젝트별 메모리 (자동 메모리)
MEMORY_DIR="$HOME/.claude/projects"
if [ -d "$MEMORY_DIR" ]; then
  find "$MEMORY_DIR" -name "MEMORY.md" -exec sh -c 'echo "$(wc -c < "$1") bytes — $1"' _ {} \;
fi
```

**MEMORY.md 제한:** 최대 200줄 / 25,000 bytes. 초과 시 자동 잘림.

### 1-3. MCP 도구 스키마 측정

연결된 MCP 서버 수를 확인한다. **MCP 1개당 ~1-5K 토큰이 매턴 추가된다.**

```bash
# settings.json에서 MCP 설정 확인
SETTINGS="$HOME/.claude/settings.json"
if [ -f "$SETTINGS" ]; then
  python3 -c "
import json
with open('$SETTINGS') as f:
    s = json.load(f)
mcps = s.get('mcpServers', {})
print(f'MCP 서버: {len(mcps)}개')
for name, conf in mcps.items():
    disabled = conf.get('disabled', False)
    status = '❌ 비활성' if disabled else '✅ 활성'
    print(f'  {status} {name}')
" 2>/dev/null
fi
```

**판정:** 안 쓰는 MCP가 활성 상태면 매턴 토큰 낭비.

---

## Step 2: 비용 구조 분석

### 2-1. 모델별 비용표

| 모델 | 입력 ($/1M) | 출력 ($/1M) | 캐시읽기 | 캐시쓰기 |
|------|-----------|-----------|---------|---------|
| Haiku 4.5 | $1 | $5 | $0.10 | $1.25 |
| Sonnet 4.6 | $3 | $15 | $0.30 | $3.75 |
| Opus 4.6 | $15 | $75 | $1.50 | $18.75 |
| Opus 4.6 Fast | $30 | $150 | $3.00 | $37.50 |

### 2-2. 매턴 기본 비용 추정

```
기본 시스템 프롬프트:     ~20,000 tokens (고정, 줄일 수 없음)
도구 스키마:              ~10,000 tokens (활성 도구 수에 비례)
CLAUDE.md (글로벌+프로젝트): 측정값
MEMORY.md:                측정값
MCP 스키마:               MCP 수 × ~3,000 tokens
스킬 메타데이터:          활성 스킬 수 × ~500 tokens
────────────────────────────────
합계 = 매턴 기본 입력 토큰
```

**10턴 대화 비용 추정** (캐시 히트 90% 가정):
```
첫턴: 합계 × 입력단가 (풀 비용)
2~10턴: 합계 × 캐시읽기단가 × 9
+ 출력: 턴당 ~2,000 tokens × 출력단가 × 10
```

---

## Step 3: CLAUDE.md 다이어트 가이드

### 3-1. 분류 기준

CLAUDE.md의 각 섹션을 아래 기준으로 분류:

| 분류 | 기준 | 조치 |
|------|------|------|
| **매턴 필수** | 매 대화에서 참조하는 규칙/설정 | CLAUDE.md에 유지 |
| **가끔 필요** | 특정 작업 시에만 필요한 가이드 | `refs/`로 분리 |
| **거의 안 씀** | 과거 기록, 사고 이력, 상세 테이블 | `refs/`로 분리 |

### 3-2. 분리 구조

```
~/.claude/
├── CLAUDE.md              ← 매턴 필수만 (목표: 3KB 이하)
├── refs/                  ← 필요 시 읽기
│   ├── topic-a.md
│   └── topic-b.md
```

CLAUDE.md 하단에 포인터 추가:
```markdown
# 상세 레퍼런스 (필요 시 읽기)
- 주제A 가이드: `~/.claude/refs/topic-a.md`
- 주제B 가이드: `~/.claude/refs/topic-b.md`
```

### 3-3. 다이어트 체크리스트

아래 항목이 CLAUDE.md에 있으면 분리 대상:

- [ ] 코드 예시 (3줄 이상의 코드 블록)
- [ ] ID/UUID 테이블 (에이전트, 프로젝트 등)
- [ ] 배포 커맨드 (aws, docker, kubectl 등)
- [ ] API 엔드포인트 전체 목록 (핵심 3개만 남기고 분리)
- [ ] 사고 기록 / 과거 이력
- [ ] 환경별 설정값 (dev/staging/prod)
- [ ] 팀원 목록 / 조직도 (MEMORY.md나 refs로)

---

## Step 4: 세션 운용 최적화

### 4-1. 캐시 유지 습관

캐시가 히트하면 시스템 프롬프트가 **90% 할인**. 깨지면 풀 비용.

**캐시가 깨지는 행동:**
- `/model` 명령으로 모델 전환 → 캐시 전체 리셋
- MCP 서버 연결/해제 → 도구 스키마 변경으로 캐시 깨짐
- 5분 이상 입력 없음 (일반) / 1시간 (구독자) → TTL 만료

**캐시 유지 팁:**
- 한 세션에서 모델 바꾸지 않기
- MCP는 세션 시작 전에 설정 완료
- 생각하는 동안에도 가벼운 메시지 보내서 TTL 유지

### 4-2. 컨텍스트 관리

- **작업 바뀌면 `/clear`** — 이전 대화 끌고가면 매턴 누적
- **`/compact` 적극 사용** — 대화가 길어졌다 싶으면 수동 압축
- **파일 읽기 시 범위 지정** — offset/limit으로 필요한 부분만

### 4-3. 모델 라우팅

| 작업 | 추천 모델 | 이유 |
|------|----------|------|
| 파일 읽기, grep, 간단한 수정 | Sonnet | 5배 저렴, 충분한 품질 |
| 아키텍처 설계, 복잡한 디버깅 | Opus | 깊은 추론 필요 |
| 구조화된 판단 (템플릿 기반) | Sonnet | 프레임워크 따르면 Sonnet OK |
| 대량 코드 생성 | Sonnet | 출력 토큰이 비용 주도 |

### 4-4. 환경 변수 (고급)

```bash
# 서브에이전트를 Sonnet으로 강제
export CLAUDE_CODE_SUBAGENT_MODEL="claude-sonnet-4-6"

# 1M 컨텍스트 비활성화 (200K 강제 → 자동 압축 빈번 → 누적 절약)
export CLAUDE_CODE_DISABLE_1M_CONTEXT=1

# 파일 읽기 토큰 제한 (기본 25K → 10K)
export CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS=10000
```

---

## Step 5: Paperclip 에이전트 최적화 (해당 시)

Paperclip claude_local 어댑터 사용 시 추가 진단:

### 5-1. 에이전트 설정 점검

```bash
COMPANY_ID="여기에_회사ID"
curl -s "http://127.0.0.1:3100/api/companies/$COMPANY_ID/agents" | python3 -c "
import json, sys
for a in json.load(sys.stdin):
    if a.get('adapterType') == 'claude_local':
        ac = a.get('adapterConfig', {}) or {}
        model = ac.get('model', '미설정')
        turns = ac.get('maxTurnsPerRun', '무제한')
        print(f\"[{a['status']:8s}] {a['name']:20s} | model={model} | maxTurns={turns}\")
"
```

**점검 항목:**
- [ ] 모델이 작업 난이도에 맞는가? (판단 업무 → Sonnet, 복잡한 코딩 → Opus)
- [ ] maxTurnsPerRun이 적절한가? (분석/판단: 30-50, 코딩: 100-200)
- [ ] 에이전트 워크스페이스에 경량 CLAUDE.md가 있는가?

### 5-2. 에이전트 워크스페이스 CLAUDE.md

Paperclip 에이전트는 워크스페이스 cwd에서 CLAUDE.md를 읽음.
없으면 글로벌 CLAUDE.md만 로드 — **불필요한 정보가 매턴 들어감**.

```bash
# 워크스페이스 경로
WS="$HOME/.paperclip/instances/default/workspaces/{agentId}"

# 에이전트 전용 최소 CLAUDE.md 생성
cat > "$WS/CLAUDE.md" << 'EOF'
# Agent Name
- 한국어로 응답
- 역할: [핵심 역할 한줄]
- API: http://127.0.0.1:3100/api
EOF
```

### 5-3. Paperclip 자동 주입 스킬

Paperclip이 매 실행마다 4개 스킬을 자동 주입 (~25K 토큰):
- paperclip (19KB) — API 레퍼런스
- paperclip-create-agent — 에이전트 생성
- paperclip-create-plugin — 플러그인 생성
- para-memory-files — PARA 메모리

→ 에이전트가 안 쓰는 스킬이면 Paperclip 설정에서 제거 검토.

---

## 출력 형식

진단 결과를 아래 형식으로 요약:

```
═══════════════════════════════════════
  Token Diet 진단 결과
═══════════════════════════════════════

📊 매턴 입력 토큰 추정
  시스템 프롬프트:  ~20,000
  CLAUDE.md:        ~{n}    {🟢|🟡|🔴}
  MEMORY.md:        ~{n}
  MCP 스키마:       ~{n}    ({m}개 서버)
  ─────────────────────────
  합계:             ~{total}

💰 10턴 대화 예상 비용 ({모델명})
  캐시 히트 시: ${cost_cached}
  캐시 미스 시: ${cost_uncached}

🔧 개선 제안
  1. {제안1}
  2. {제안2}
  3. {제안3}

📁 파일 구조
  CLAUDE.md:  {size} bytes ({판정})
  MEMORY.md:  {size} bytes
  MCP 서버:   {n}개 활성
  refs/:      {n}개 파일
═══════════════════════════════════════
```

`/token-diet report` 시 위 내용을 `~/.claude/token-diet-report-{date}.md`로 저장.
