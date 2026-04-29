# Weekly AI Trends — Scheduled Task 운영 노트

마지막 업데이트: 2026-04-29

## 2026-04-29 실행에서 발견된 두 가지 문제

이번 자동 실행에서 발생한 실수와 그 원인을 기록한다. 다음 실행에서 같은 문제를 또 일으키지 않도록 해야 한다.

### 1. `date` 필드 오해석 (재발 방지 필요)

**증상**: 4/29(수) 실행에서 `date=2026-04-20`을 사용해 URL이 `/2026-04-20.html`로 나갔다. 사용자가 기대한 값은 `date=2026-04-27` (이번 주 월요일).

**원인**: 스케줄 SKILL.md에 "date는 해당 주의 월요일 날짜"라고만 적혀 있어, "리서치한 주" vs "발행하는 주" 둘 다로 해석 가능했다. 예시 JSON(`example_brief.json`)도 date=일요일을 쓰고 있어서 일관되지 않았다.

**불변 규칙(이걸로 굳힌다)**:

- `date = THIS_MON` = "오늘이 속한 주의 월요일" (오늘이 화·수·목·금·토·일이면 그 주의 월요일로 끌어올림. 월요일이면 오늘 그대로).
- `subtitle 날짜 범위 = (THIS_MON − 7) ~ (THIS_MON − 1)` = 직전 완료된 Mon~Sun 한 주.
- 리서치 시간 범위도 `(THIS_MON − 7) ~ (THIS_MON − 1)`.
- 파일명 = `ai-trends-week-${THIS_MON}.html`, URL = `/${THIS_MON}.html`.
- 예: 오늘 2026-04-29(수) → THIS_MON=2026-04-27, 범위=2026-04-20~2026-04-26.

**적용 방법**: 아래 "스케줄 태스크 SKILL.md 패치" 섹션의 새 프롬프트를 스케줄 태스크에 붙여넣기. (이번 실행 컨텍스트에서는 자기 자신을 수정할 수 없게 락이 걸려 있어 자동 적용 못 함.)

### 2. BOT_TOKEN 경로가 샌드박스에서 안 잡힘 (해결됨)

**증상**: `~/Tools/telegram-mcp/.env`가 Cowork 샌드박스 마운트 밖이라 `cat` 불가. curl 기반 봇 전송 경로가 첫 실행부터 막혔었음.

**해결 (2026-04-29)**: 사용자가 archive 폴더의 `.env`에 `BOT_TOKEN`을 추가. 이제 `cd /sessions/*/mnt/weekly-ai-trends-archive && source .env`로 GITHUB_TOKEN/VERCEL_TOKEN과 같은 셸에서 로드되어 curl 기반 봇 송신이 정상 작동함. 같은 4/29 실행에서 봇 발신으로 재전송 성공 확인 (`"ok":true`).

**남은 메모**: archive `.env`는 `.gitignore`에 이미 박혀 있어 GitHub로 새지 않음. 토큰 회전 시 이 파일만 갱신하면 됨.

## 스케줄 태스크 SKILL.md 패치 (다음 주 실행 전 반영 필요)

`/Users/matthew/Documents/Claude/Scheduled/weekly-ai-trends-briefing/SKILL.md`의 본문(YAML 헤더 아래)을 아래 내용으로 교체.

> 이번 자동 실행 세션에서는 "Cannot update scheduled tasks from within a scheduled task session" 제한 때문에 직접 적용 못 함. 사용자가 별도 세션에서 다음 메시지를 보내면 적용된다: "scheduled task `weekly-ai-trends-briefing`의 prompt를 RUN-NOTES.md의 패치 본문으로 갱신해 줘."

```markdown
weekly-ai-trends 스킬을 사용해서 지난 한 주(최근 7일) AI 트렌드 주간 브리핑 슬라이드를 생성하고, 아카이브 누적·GitHub push·Vercel 자동 배포·텔레그램 URL 전송까지 완료해라.

# 0. 날짜 계산 단계 (다른 모든 단계 시작 전 반드시 먼저 실행)

`date`/`subtitle`/파일명/URL이 한 변수로 묶여 있어서, 한 군데만 잘못 잡으면 그 주 브리프 전체가 어긋난다. 매 실행의 첫 동작으로 아래 bash 한 줄을 돌려서 변수를 확정하고, 이후 단계는 모두 그 변수에서 파생된 값만 쓴다. 절대 직접 어림으로 계산하지 말 것.

\`\`\`bash
TODAY=$(date +%Y-%m-%d)
THIS_MON=$(python3 -c "
import datetime
t = datetime.date.today()
mon = t - datetime.timedelta(days=t.weekday())
print(mon.isoformat())
")
RANGE_START=$(python3 -c "import datetime; d=datetime.date.fromisoformat('${THIS_MON}'); print((d-datetime.timedelta(days=7)).isoformat())")
RANGE_END=$(python3 -c "import datetime; d=datetime.date.fromisoformat('${THIS_MON}'); print((d-datetime.timedelta(days=1)).isoformat())")
echo "TODAY=$TODAY THIS_MON=$THIS_MON RANGE=$RANGE_START~$RANGE_END"
\`\`\`

규칙(불변):

- 브리프 JSON의 `date` = `THIS_MON`. 발행이 화·수·목·금·토·일이어도 그 주 월요일로 끌어올린다.
- `subtitle`의 날짜 범위 = `RANGE_START ~ RANGE_END` = THIS_MON 직전 7일(Mon~Sun). 형식: `"<RANGE_START> ~ <RANGE_END> · Claude Code를 중심으로"`.
- 리서치 시간 범위도 `RANGE_START ~ RANGE_END`.
- 파일명은 `ai-trends-week-${THIS_MON}.html`, URL은 `/${THIS_MON}.html`.
- "리서치한 주의 월요일"(=RANGE_START)을 `date`에 넣지 말 것. 2026-04-29 실행에서 발생했던 실수.

예시(2026-04-29 수): THIS_MON=2026-04-27, RANGE=2026-04-20~2026-04-26 → date=2026-04-27, subtitle="2026-04-20 ~ 2026-04-26 · Claude Code를 중심으로", 파일명 ai-trends-week-2026-04-27.html.

# 리서치 · 생성 단계
(이하 기존 내용과 동일하되 `<Monday>` 자리에 `${THIS_MON}`, 날짜 범위 자리에 `${RANGE_START} ~ ${RANGE_END}`를 넣는다.)

# 아카이브 누적 단계
(기존 그대로 + FUSE deadlock 폴백 추가)

4. **FUSE deadlock 우회**: `update_archive.py`가 `Resource deadlock avoided`로 죽으면, `git clone --depth 5` → 클론에 새 HTML/meta 복사 → 인덱스 인-메모리 재생성 → 클론에서 push. (실제 발생 사례 있음.)

# 검증 단계 (push 후 30~60초 대기, 텔레그램 전 필수)

\`\`\`bash
sleep 45
curl -s -o /dev/null -w "slide:%{http_code}\n" "https://weekly-ai-trends-archive.vercel.app/${THIS_MON}.html"
curl -s -o /dev/null -w "home:%{http_code}\n"  "https://weekly-ai-trends-archive.vercel.app/"
curl -s "https://weekly-ai-trends-archive.vercel.app/${THIS_MON}.html" | grep -F "${RANGE_START} ~ ${RANGE_END}" | head -1
\`\`\`

200 + 본문에 `RANGE_START ~ RANGE_END` 매칭이 떠야 정상. 둘 중 하나라도 실패하면 텔레그램 단계로 넘어가지 말고 보고.

# 텔레그램 전송 단계
(기존 그대로 + 폴백 명시)

**샌드박스 폴백**: `~/Tools/telegram-mcp/.env`가 Cowork 샌드박스 마운트 밖이면 curl 경로가 막힌다. 이때 `mcp__telegram__send_message`(personal 계정, `chat_id=7876494067`, `parse_mode="html"`)로 동일 본문 전송. 첫 줄에 "[자동전송 폴백 — 봇 대신 본인 계정에서 전송됨]" 표시.
```

## 적용 체크리스트 (사람이 직접)

- [ ] 위 패치를 스케줄 태스크 SKILL.md에 반영 (또는 새 세션에서 Claude에게 갱신 요청)
- [x] `BOT_TOKEN`을 `weekly-ai-trends-archive/.env`에 추가 (2026-04-29 완료)
- [ ] 다음 월요일 9시 자동 실행 직후 URL이 `/${이번 주 월요일}.html` 형태로 나오는지 확인
