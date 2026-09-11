# R&D 지원기관 링크 대시보드

KEIT · TIPA · KIAT · KOITA 네 기관의 공고·서식 페이지를 한 화면에 모으고,
각 기관 공고를 자동으로 수집해 접수 마감일을 D-day로 보여주는 단일 HTML 대시보드.

- 서비스: <https://todayplus.github.io/KEIT_TIPA_KIAT/>
- 수집 API(워커): `https://soft-glade-215e.wjk0219.workers.dev`

---

## 1. 구성

```
index.html        대시보드 본체 (단일 파일, 의존성 없음)
worker.js         Cloudflare Workers — 공고 수집 API + CORS 중계
README.md         이 문서
```

정적 호스팅(GitHub Pages) + 서버리스 함수 하나. 빌드 과정도 데이터베이스도 없다.
사용자 데이터(편집 내용·메모·클릭 기록)는 전부 브라우저 localStorage에 남는다.

```
[브라우저 index.html]
      │  ① 링크 목록 렌더 / 편집       → localStorage
      │  ② fetch /feed?src=...         → [Cloudflare Worker]
      │                                      │ 기관 사이트 HTML 요청
      │                                      ▼
      │                                 KEIT · SMTECH · KIAT · KOITA
      │                                 (KOITA는 NTIS/IRIS까지 1단계 추적)
      │  ③ JSON 수신 → 마감일을 D-day에 반영
```

---

## 2. 개발 경과

### v1 — 초기 링크 대시보드
KEIT·TIPA·KIAT 3개 기관의 주요 페이지를 카드로 묶은 정적 링크 모음.
검색, 기관·분류 필터, URL 복사, 다크/라이트 자동 전환.

### v2 — KOITA 추가 + 직접 편집
- KOITA(한국산업기술진흥협회) 카드와 국가R&D 사업공고 링크 추가
- 편집 모드: 링크·그룹·기관을 화면에서 직접 추가/수정/삭제
- localStorage 저장, JSON 내보내기/가져오기, 초기화
- ★ 즐겨찾기 토글

### v3 — 기능 확장 7종
| 기능 | 내용 |
|---|---|
| 링크 점검 | 응답 없는 링크에 "확인필요" 표시 |
| 공고 수집 | RSS/Atom/JSON 피드를 읽어 최신 공고 패널에 표시 |
| 마감 D-day | 링크별 마감일 → D-3 이내 빨강, 14일 이내 주황 |
| 메모·태그 | 링크별 자유 메모와 사용자 태그(필터 칩 자동 생성) |
| 공유 링크 | 목록 전체를 gzip 압축해 URL 해시에 담아 전달 |
| 정렬·드래그 | 직접 지정 / 클릭 수 / 최근 클릭 / 마감 임박 / 이름순 + 드래그 재배치 |
| 테마 | 자동 / 다크 / 라이트 수동 전환 |

### v3 배포 후 — 왜 공고가 안 들어왔나
GitHub Pages는 정적 호스팅이라 페이지가 다른 도메인의 응답을 직접 읽을 수 없다(CORS).
게다가 **네 기관 모두 공식 RSS를 제공하지 않았다.**

```
https://www.koita.or.kr/rss.do   → 404
https://www.keit.re.kr/rss/rss.jsp → 302
https://www.smtech.go.kr/rss.do  → 404
https://www.kiat.or.kr/rss.do    → 404
```

→ 중계 서버가 필요하고, 그 서버가 공고 목록 HTML을 파싱해 JSON으로 내려주는 구조로 결정.

### v4 — 워커 + 마감일 자동 연동
- Cloudflare Workers로 기관별 파서 작성 (아래 3장)
- 피드가 준 마감일을 대시보드 링크의 D-day에 자동 반영
- 피드 항목 "+ 담기" 버튼, 자동 반영값에 "자동" 배지

### v5 — 이중 프록시 버그 수정
피드 주소를 항상 프록시로 감싸다 보니 **워커가 자기 자신을 호출**했고,
워커의 도메인 화이트리스트에 자기 도메인이 없어 403으로 거절 → 수집 0건.
피드 host가 프록시 host와 같으면 감싸지 않고 직접 호출하도록 수정.
아울러 실패 사유(상태코드·시간초과·빈 응답)를 기관별로 화면에 노출.

---

## 3. 기관별 수집 방식

조사해 보니 네 곳이 전부 구조가 달랐다.

| 기관 | 목록 경로 | 구조 | 상세 링크 | 마감일 출처 |
|---|---|---|---|---|
| KEIT | `srome.keit.re.kr/.../retrieveTaskAnncmListView.do?prgmId=XPG201040000` | `div.table_box` 카드 | `f_detail('ancmId','bsnsYy')` → `retrieveTaskAnncmInfoView.do` | 목록의 접수기간 |
| TIPA | `smtech.go.kr/front/ifg/no/notice02_list.do` | `table.tbl_base` | `notice02_detail.do` (jsessionid 제거 필요) | 목록의 접수기간 |
| KIAT | `kiat.or.kr/front/board/boardContentsListAjax.do` (**POST**) | Ajax 응답 HTML | `contentsView('id')` → `boardContentsView.do` | 목록의 접수기간 |
| KOITA | `koita.or.kr/board/commBoardGovRnDList.do` | 일반 테이블 | `page_move(...{no:N})` → `commBoardGovRnDView.do?no=N` | **상세 본문 파싱** |

주의할 점 몇 가지.

- **KIAT**: 목록이 Ajax로 그려져 페이지 HTML에는 공고가 없다. Ajax 엔드포인트를 직접 POST하면 세션 없이도 응답한다.
- **TIPA**: 링크에 `;jsessionid=...`가 박혀 나온다. 제거하지 않으면 나중에 만료된다.
  `javascript:goMove()`인 행은 IRIS로 이관된 공고라 `iris.go.kr`로 연결한다.
- **KOITA**: 목록에 접수기간 칸이 아예 없다. 상세 페이지를 열어야 하고,
  본문이 NTIS·IRIS `<iframe>`인 공고가 많아 그 페이지까지 한 단계 더 따라간다.

### 마감일 추출 규칙
1. `마감일 : 2026.03.06` 같은 라벨 값 (NTIS/IRIS 요약표)
2. `접수기간 / 신청기간 / 공모기간 / 제출기한` 뒤 250자 안의 날짜 중 마지막 값
3. `'26. 2. 4(수) ~ 3. 6(금)`처럼 종료일 연도가 생략된 경우 시작 연도로 보정 (역전 시 +1년)
4. 못 찾으면 **빈 값**. 추측하지 않는다.

현재 추출률: KEIT 6/6 · KIAT 6/6 · TIPA 6/6 · KOITA 5/8
(KOITA 나머지 3건은 본문에 접수기간 문구 자체가 없는 공고)

---

## 4. 워커 API

```
GET /feed?src=keit|smtech|kiat|koita|all&limit=15&detail=1
GET /?url=<encoded-url>          링크 점검용 CORS 중계
GET /                            사용법 안내
```

응답:

```json
{
  "updatedAt": "2026-09-11T04:24:15Z",
  "count": 12,
  "items": [
    { "agency": "KIAT",
      "title": "2026년 기술거래기관 및 사업화전문회사 지정 신청 공고",
      "link": "https://www.kiat.or.kr/front/board/boardContentsView.do?...",
      "date": "2026-09-09",
      "period": "2026-09-09~2026-10-12",
      "due": "2026-10-12" }
  ]
}
```

- `due`가 대시보드 D-day에 그대로 쓰인다.
- `detail=0`을 붙이면 KOITA 상세 조회를 건너뛴다. 0.7초로 끝나지만 마감일은 비게 된다(기본 6~7초).
- 응답은 10분 캐시. 일부 기관만 실패하면 `errors` 필드에 사유가 담긴다.

보안 설정은 파일 상단 두 줄로 조절한다.

```js
const ORIGIN = 'https://todayplus.github.io';  // 호출 허용 출처 ('' = 전체)
const ALLOW  = ['koita.or.kr', 'keit.re.kr', ...];  // 중계 허용 도메인
```

허용 목록 밖 도메인은 403. 오픈 프록시로 악용되지 않게 하는 장치다.

---

## 5. 배포

### 대시보드
`index.html`을 리포지토리에 커밋하면 GitHub Pages가 자동 반영한다.
**교체 후에는 브라우저에서 Ctrl+Shift+R로 강제 새로고침**할 것. (캐시 때문에 "예전 버전이 그대로"인 상황이 실제로 있었다.)

### 워커
1. dash.cloudflare.com → Workers & Pages → Create → Workers → Hello World → Get started
2. 이름 입력 후 Deploy (샘플 코드가 먼저 올라간다)
3. Edit code → 기존 내용 전체 삭제 → `worker.js` 붙여넣기 → Deploy
4. 주소 형식: `https://<이름>.<계정>.workers.dev`

### 대시보드에 연결
⚙️ 설정 → 프록시 URL 템플릿 (`{url}`은 치환 자리이므로 그대로 둔다)

```
https://soft-glade-215e.wjk0219.workers.dev/?url={url}
```

✏️ 편집 모드 → 기관별 수정 → 공고 피드 URL

| 기관 | 값 |
|---|---|
| KEIT | `https://soft-glade-215e.wjk0219.workers.dev/feed?src=keit` |
| TIPA | `.../feed?src=smtech` |
| KIAT | `.../feed?src=kiat` |
| KOITA | `.../feed?src=koita` |

---

## 6. 데이터 저장

| 키 | 내용 |
|---|---|
| `rnd-dash-v3-data` | 기관·그룹·링크 목록 (편집 결과) |
| `rnd-dash-v3-cfg` | 프록시 주소, 테마, 정렬, 자동 연동 옵션 |
| `rnd-dash-v3-hits` | 링크별 클릭 수·최근 클릭 시각 |
| `rnd-dash-v3-feed` | 마지막으로 수집한 공고 목록 |

서버에 저장되는 것은 없다. 브라우저를 바꾸거나 동료에게 넘길 때는
**내보내기 JSON** 또는 **공유 링크**를 쓴다.

마감일에는 `autoDue` 표식이 있다. 자동 반영된 값에만 붙으며,
사람이 직접 입력하거나 수정한 마감일은 이후 자동 갱신 대상에서 빠진다.

---

## 7. 문제가 생기면

| 증상 | 확인 |
|---|---|
| 공고가 0건 | 워커 주소 + `/feed?src=all`을 브라우저로 직접 열어 `errors` 확인 |
| 특정 기관만 0건 | 그 기관이 페이지 구조를 바꿨을 가능성. 파서 수정 필요 |
| CORS 오류 | 워커의 `ORIGIN` 값이 현재 사이트 주소와 다름 |
| 수정했는데 그대로 | Ctrl+Shift+R |
| 워커 되돌리기 | Cloudflare → 해당 워커 → Deployments → Rollback |

**가장 주의할 점**: 기관 사이트가 개편되면 파서가 오류 없이 *빈 목록*을 반환한다.
조용히 실패하므로, 특정 기관만 계속 0건이면 구조 변경을 의심해야 한다.

---

## 8. 남은 과제

- KOITA 미추출 3건 — 본문 문구가 제각각이라 규칙화 여지가 남아 있음
- 파서 상태 자동 점검 (일정 기간 0건이면 알림)
- 마감 임박 공고 이메일/메신저 알림
- 다중 사용자 공유 (현재는 브라우저 단위 저장)
