# fifaon-files

fifa-on 의 **공개 정적 파일 정본 저장소**(git 백본).
`files.fifa-on.com` 이 이 레포의 checkout 을 그대로 서빙한다. scouter-files 와 같은 방식이지만
레포·컨테이너·도메인을 따로 둬서 권한·캐시 규칙·사고 범위가 scouter 와 섞이지 않는다.

```
[이 레포 (GitHub)] ── push ←─ PC
        │ git pull (서버 2분 폴링)
        ▼
/home/gomao/fifaon-files ─ ro mount → nginx:alpine (127.0.0.1:8092)
        → cloudflared ingress → CF(Cache Everything) → 방문자
```

## 디렉토리

| 경로 | 용도 |
|---|---|
| `data/` | fifaon-next 가 SSR 에서 fetch 하는 데이터(패치노트 등) — **버저닝 예외(제자리 수정)**, 캐시 60초 |
| `img/` | 이미지 에셋 — **파일명 버저닝**, 캐시 1년 immutable |
| `deploy/` | 서버 배포 설정 (서빙 대상 아님 — 민감정보 두지 말 것) |

URL 매핑: `data/patchnote.json` → `https://files.fifa-on.com/data/patchnote.json`

### 패치노트 (`data/patchnote.json`)

```json
{ "board": [ { "date": "2026. 9. 17.", "note": "제목 한 줄", "description": ["줄1", "줄2"] } ] }
```
최신 항목을 **맨 앞**에 넣는다. 제목 한 줄 + 2~3줄, 개발 용어·수치 나열은 피한다.
반영 = pull 2분 + 캐시 1분(+ next `revalidate` 60초) = 최대 ~3분.

## 운영 규칙

1. **공개 서빙용 파일만.** 백업·사설 파일·시크릿 금지 (전부 공개 URL로 노출됨).
2. **같은 파일명 덮어쓰기 금지** — `img/` 는 캐시가 `immutable, max-age=1y` 라 갱신이 안 보인다.
   내용이 바뀌면 파일명을 바꿔라 (`banner-260917.png` 식 날짜 버저닝).
   예외: `data/` 아래는 제자리 수정하는 파일(캐시 60초).
   ⚠️ `deploy/fifaon-files.conf` 변경은 pull 로 자동 반영 안 됨 — 컨테이너에서 `nginx -t` 후 `nginx -s reload` 필요.
3. **배포 순서**: 코드가 새 파일을 참조하면, 이 레포 push 가 먼저 → 코드 배포는 나중.
4. 롤백 = `git revert` 후 push.
5. **서버 checkout(`/home/gomao/fifaon-files`)을 직접 만지지 말 것** — dirty 되면 cron pull 이 막혀 반영이 멈춘다.

## 서버 구성 요약

- checkout `/home/gomao/fifaon-files` + `deploy/docker-compose.yaml` 로 nginx 컨테이너(루프백 8092)
- cron 2분 폴링 `git pull --ff-only`
- cloudflared ingress `files.fifa-on.com` → `http://localhost:8092`, CF DNS(프록시) + Cache Everything 룰
- 확인: `curl -s http://127.0.0.1:8092/healthz` → `ok`, `https://files.fifa-on.com/data/patchnote.json` 200
