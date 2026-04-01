# 🍃 Token Diet — Claude Code 토큰 절약 스킬

Claude Code의 토큰 사용량을 진단하고, CLAUDE.md 다이어트 + 비용 최적화를 제안하는 슬래시 커맨드 스킬.

## 뭘 해주나?

- **매턴 토큰 소비 측정** — CLAUDE.md, MEMORY.md, MCP 서버가 매 API 호출마다 얼마나 먹는지 계산
- **비용 추정** — 모델별 (Haiku/Sonnet/Opus) 10턴 대화 예상 비용
- **CLAUDE.md 다이어트 가이드** — 매턴 필수 / 가끔 필요 / 분리 대상 자동 분류
- **캐시 최적화 팁** — 캐시 깨지는 행동 감지 + 유지 방법
- **Paperclip 에이전트 진단** — claude_local 어댑터 사용 시 모델/턴수/워크스페이스 점검

## 설치

```bash
# 방법 1: git clone
git clone https://github.com/bugbug9999/token-diet.git
mkdir -p ~/.claude/skills/token-diet
cp token-diet/SKILL.md ~/.claude/skills/token-diet/

# 방법 2: 파일 하나만 복사
mkdir -p ~/.claude/skills/token-diet
curl -sL https://raw.githubusercontent.com/bugbug9999/token-diet/main/SKILL.md \
  -o ~/.claude/skills/token-diet/SKILL.md
```

## 사용법

Claude Code에서:

```
/token-diet              # 현재 상태 진단
/token-diet apply        # 진단 후 자동 최적화 적용
/token-diet report       # markdown 리포트로 저장
```

## 진단 예시

```
═══════════════════════════════════════
  Token Diet 진단 결과
═══════════════════════════════════════

📊 매턴 입력 토큰 추정
  시스템 프롬프트:  ~20,000
  CLAUDE.md:        ~4,700   🔴 과다
  MEMORY.md:        ~1,300
  MCP 스키마:       ~6,000   (2개 서버)
  ─────────────────────────
  합계:             ~32,000

💰 10턴 대화 예상 비용 (Opus 4.6)
  캐시 히트 시: $0.98
  캐시 미스 시: $4.80

🔧 개선 제안
  1. CLAUDE.md 19KB → refs/ 분리로 3KB 이하 (매턴 ~3,400 토큰 절약)
  2. MCP "notion" 비활성화 (사용 안 함, 매턴 ~3,000 토큰)
  3. /model sonnet 으로 단순 작업 시 5배 절약
═══════════════════════════════════════
```

## 왜 토큰을 아껴야 하나?

Claude Code는 매 턴마다 시스템 프롬프트 + CLAUDE.md + 도구 스키마를 **반복 전송**한다. CLAUDE.md가 크면:

| CLAUDE.md 크기 | 매턴 토큰 | 10턴 비용 (Opus, 캐시 미스) |
|---------------|----------|---------------------------|
| 3KB | ~750 | $1.50 |
| 10KB | ~2,500 | $3.75 |
| **20KB** | **~5,000** | **$7.50** |

**CLAUDE.md만 줄여도 세션당 $3-6 절약 가능.**

캐시가 히트하면 90% 할인이지만, 모델 전환/MCP 변경/5분 무입력 시 캐시가 깨진다.

## 핵심 절약 원칙

1. **CLAUDE.md는 3KB 이하** — 상세 가이드는 `refs/`로 분리, 필요할 때만 읽기
2. **안 쓰는 MCP 끄기** — MCP 1개당 ~1-5K 토큰이 매턴 추가
3. **단순 작업은 Sonnet** — Opus 대비 입출력 모두 5배 저렴
4. **작업 바뀌면 `/clear`** — 이전 대화 끌고가면 매턴 누적
5. **캐시 유지** — 한 세션에서 모델/MCP 바꾸지 않기

## 라이선스

MIT
