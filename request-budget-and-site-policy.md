# DVDPrime 요청 예산과 사이트 정책

## M0

- 자동 site-adapter 테스트는 vendored HTML fixture와 로컬 comment contract
  fixture만 사용하며 네트워크 요청을 만들지 않는다.
- 앱은 사용자가 연 WebView 탐색과 사용자가 직접 선택한 이미지의 저장·공유·전체화면
  표시 외 DVDPrime background 요청을 만들지 않는다. 이미지 요청은 선택한 한 장만
  처리하며 인접 이미지 선요청은 하지 않는다.
- 댓글 보관은 사용자가 현재 열어 둔 글에서 한 댓글의 비상호작용 영역을 길게 누르고 확인할 때
  이미 렌더링된 댓글 본문·표시 metadata와 그 댓글 안의 미디어 목록만 전달한다. 기존 카드와의
  변경 비교 및 이용자가 승인한 갱신도 그 시점에 이미 표시된 정확한 댓글만 사용한다. 댓글 카드를
  저장·비교하기 위한 별도 글·게시판 요청, 목록 순회, 주기 확인은 없다. 댓글 안 미디어의 실제
  저장 요청은 같은 명시적 동작 안에서 기존 보관함의 URL·redirect·크기 제한을 그대로 적용한다.
- 읽은 글 상세 저장을 사용자가 켜면 일반 글을 읽었다고 확인된 그 시점의 이미 렌더링된 본문
  또는 본문·표시 댓글 텍스트만 로컬 저장소로 전달한다. 별도 HTTP 요청, 댓글 페이지 이동,
  과거 읽음 기록의 소급 수집, 민감 게시판 상세 저장은 없으며 원본 HTML과 미디어 binary도
  저장하지 않는다.
- M0 확인 중 공개 홈페이지를 한 번 읽어 공식 로그인 폼이 홈 흐름에
  존재함만 확인했다. 로그인·추천·댓글·읽음 변경 요청은 하지 않았다.
- prefetch, prerender, crawler, polling, headless cookie 요청은 없다.

## M1/M2 원칙

- DOM 분석은 초기 스캔 뒤 `MutationObserver`의 변경 신호를
  80ms debounce/400ms max-wait로 합치고, 한 scan은 256 target으로
  제한한다. marker 자체 mutation은 재스캔하지 않는다.
- 확인된 연속 읽기 문맥의 페이지 경계에서는 바로 인접한 게시판 목록 1개만
  조회해 다음 페이지 첫 글 또는 이전 페이지 마지막 글의 존재를 확인한다. 응답에서는
  bounded 글 identity만 사용하며 제목·본문·작성자·댓글 내용은 네이티브로 전달하지 않는다.
- 다음/이전 게시글 본문 사전 준비는 한 번에 정확히 1개 URL로 제한한다. 기본값은
  요금이 부과되지 않는 연결에서만 허용하고, 사용자는 설정에서 모든 연결 허용 또는
  사용 안 함을 선택할 수 있다. 탐색·문맥 변경, 안전 모드와 기능 비활성화 시 진행 중인
  준비를 취소하며 실패를 자동 반복하지 않는다.
- 전체 페이지·게시판 순회, 복수 글 동시 준비, 주기 polling과 누락분 따라잡기는 하지 않는다.
- 사이트 기능 실패 시 WebView 원본 흐름을 유지한다. 단, first-party 정적 원본의 bounded native
  준비·디코드 실패는 메모리 상한을 무력화하는 자동 WebView 전환 없이 실패 UI로 끝낸다.
- 실제 네트워크 회귀는 로그인한 사용자가 정한 시간에 저빈도로만 한다.

### 사용자가 선택한 이미지

- 저장·공유는 bridge가 허용한 HTTPS URL에 대해서만 실행하고 redirect도 같은 안전성
  검사를 다시 거친다. 자동 재시도나 인접 이미지 prefetch는 없다.
- 원격 게시글의 전체화면 이미지는 공개 first-party 경로
  (`dvdprime.com/file/`, `cdn.mania.kr/dvdprime/file/`)의 JPEG·PNG·BMP와 Android 9 이상의
  GIF·WebP만 선택 시점에 한 장씩 네이티브 렌더러용으로 준비한다. cookie·Referer를 복제하지
  않는다. 다른 host, Android 7~8의 GIF·WebP, AVIF·HEIC는 현재 페이지 한 장만 WebView 호환
  경로를 사용하고, 인증이 필요한 first-party 이미지는 자동 우회하지 않고 실패로 안내한다.
- Android 9 이상의 GIF·animated WebP는 화면 면적에 맞춰 최대 1.5 MP·긴 변 4096 px로 제한한
  drawable로 표시하고 현재 선택 페이지에서만 재생한다. static WebP는 헤더로 구분해 정적
  JPEG·PNG·BMP와 같은 preview·region decode 경로를 사용한다.
- 네이티브 준비는 최대 1건 동시 실행, 응답 128 MiB, redirect 3회, call 30초로 제한한다.
  캐시는 `shared-post-images/viewer-cache` 아래 최대 8개·128 MiB·24시간이며 로컬 데이터
  초기화가 이 하위 트리를 함께 삭제한다. 실패는 자동 반복하거나 더 무거운 렌더러로
  우회하지 않고 사용자의 `다시 시도`를 기다린다.
- 미디어 보관함은 32 MiB 이하 원본을 그대로 저장한다. 이를 넘는 JPEG·PNG·BMP와 static WebP는
  사용자의 보관 동작 안에서만 최대 12 MP로 단계적 다운샘플링해 32 MiB 이하로 만든다.
  static WebP는 투명도 보존을 위해 PNG로 저장한다. 원본 다운로드는 128 MiB, 입력 bounds는
  250 MP로 제한한다. GIF·animated WebP는 정지 이미지로 몰래 변환하지 않고 한도를 넘으면
  저장하지 않는다.

## 이전 M3 조건부 활동 확인(현재 퇴역)

아래는 운영진 추가 회신 전에 존재했던 구현의 역사적 요청 경계다. 알림 제거 작업 소스는 활동
확인 UI와 예약을 노출하지 않고, 기존 worker가 호출되어도 네트워크 없이 성공 종료한다.

기능은 기본 꺼짐이며 명시적 opt-in 뒤에만 사용한다. 앱이 보이면 WebView 로그인
세션으로 foreground 확인을 하고, 별도의 `앱을 닫아도 주기 확인` opt-in을 켠 debug 빌드는 앱이 보이지 않을 때도 현재
CookieManager 세션을 일시 대여하는 periodic WorkManager를 등록한다. 두 경로는 같은
due 기준과 process-wide single-flight를 공유해 한 주기를 중복 요청하지 않는다.
Android 13+ 알림 권한은 알림이 처음 유효하게 켜질 때만 요청한다. release 빌드는
실기기 세션 안전성 확인 전까지 background scheduling을 닫아 둔다.

- 주기: 15/30/60분, 기본 30분
- 기존 foreground-only 설정은 background OFF로 이관하며 업데이트만으로 cookie를 빌리지 않음
- foreground 한 확인: summary 최대 1 GET + 필요한 detail 종류별 최대 1 GET, 총 최대 4 GET
- background 한 확인: summary probe 1회와 양수인 detail 종류별 최대 1회. summary는 서버가
  `PHPSESSID`를 교체하고 reload bootstrap을 반환한 경우에만 교체된 값을 메모리에서 사용해 정확히
  한 번 replay한다. 일반 로그인 확인은 최대 5 GET(요약 2 + 상세 3)이고, 직전 로그인 상태에서
  로그아웃 응답을 재확인하는 전환 경로의 절대 상한은 7 GET이다.
- 누락 주기를 따라잡는 연속 요청 없음
- network constraint, timeout, 응답 크기 제한. GET 전 로컬 준비 실패만 exponential backoff하며,
  한 번이라도 GET을 시작한 pass는 다시 요청하지 않고 다음 periodic slot을 기다림
- 사용자가 정한 야간 중지 구간에는 cookie를 읽거나 네트워크를 요청하지 않고 baseline도
  전진하지 않음. 허용 시각 뒤 첫 due check가 변화량을 한 번에 요약
- 읽음/전체 읽음/상태 변경 endpoint 0회
- raw HTML은 파싱 뒤 폐기
- 로그인→로그아웃 전환은 독립 두 번째 읽기 요청으로 확인
- cookie를 Room, 파일, 로그, 백업, Drive 또는 서버에 저장하지 않음

cookie는 요청 직전 exact URL용 header를 메모리로만 빌리고, OkHttp CookieJar와 redirect는
비활성화한다. reload bootstrap의 단일 summary replay 외에는 응답의 `Set-Cookie`를 적용하지 않으며
요청 뒤 CookieManager 값이 달라지거나
사이트/로컬 초기화·opt-out publication generation이 바뀌면 결과와 알림을 폐기한다.
별도 서버나 FCM은 사용하지 않는다. 이 경계를 실제 단말에서 입증하지 못하면 release의
WorkManager 자동 확인은 열지 않고 앱 사용 중·수동 확인만 남긴다.

## 이전 키워드·지켜보는 글 요청 상한(현재 0회)

키워드는 한 pass 최대 5개 rule을 exact GET으로 확인한다. 지켜보는 글은 15분 설정에서 최대 5개,
60/240분 설정에서 due 대상 전체를 제품 상한 30개까지 확인한다. 두 기능 모두 두 번째 요청부터
750 ms 간격을 두고 첫 HTTP 429에서 남은 pass를 중단하며 timeout·429를 즉시 재시도하지 않는다.

| 기능 | 15분 | 60분 | 240분 |
| --- | ---: | ---: | ---: |
| 키워드 최대 GET/day | 480 | 120 | 30 |
| 지켜보는 글 최대 GET/day | 480 | 720 | 180 |

이는 퇴역 전 WorkManager가 모든 slot을 실행하고 quiet-hours·429·timeout·오래된 고정 글 감속이
하나도 없다고 가정한 보수적 상한이다. 기본 키워드 15분 + 지켜보는 글 60분의 합계는 최대 1,200
GET/day였다. 알림 제거 작업 소스의 세 채널 요청 상한은 0회다. 자세한 과거 성능·wake-up 해석은
`docs/performance-resource-audit-2026-08-31.md`에 기록한다.

## 공개와 명칭

2026-09-02 DVDPrime 운영진은 회원 메모의 정책 위반, 실시간 알림의 무차별 crawling 가능성과
사이트 변경에 따른 지속적인 정책·호환성 위험을 이유로 운영자 명의의 홍보 허용이 어렵다는
의견을 전달했다. 이에 따라 signed `1.0.13 (27)`은 회원 메모와 Google Drive 동기화의 active
surface를 제거했다. 이전 로컬 메모와 원격 snapshot은 자동 삭제하지 않지만 앱은 이를 표시·편집·
백업·동기화하지 않는다.

2026-09-03 운영진은 이용자 증가에 따라 기기당 최대 1,200회 요청이 누적되는 점을 이유로 횟수
제한 여부와 관계없이 회원 메모와 알림 기능은 허용할 수 없다고 추가 회신했다. 두 기능을 제외한
상태에서는 차한잔 게시판의 한두 번 개인 소개를 묵시적으로 허용한다고 밝혔다. 이에 따라 알림
제거 작업은 다음을 active product contract로 삼는다.

- 활동·키워드·지켜보는 글 UI, 수동 확인, Android 알림 권한과 notification deep link 제거
- 세 periodic worker는 이전 예약에서 실행되어도 DVDPrime 요청 없이 성공 종료
- 앱 시작 시 세 고유 periodic work, 알림 만료 work, 이미 게시된 D-Rime 알림·알림 채널과 routing
  queue 정리
- 이전 알림 규칙·받은 글·지켜보는 글·기준값은 데이터 손실을 피하려 자동 삭제하지 않고 호환용으로만
  유지하며, `D-Rime 데이터 초기화` 또는 앱 데이터 삭제로 제거

다음 범위는 알림이 아닌 읽기 편의 기능으로 유지하되 사이트 정책 변경 시 D-Rime이 중단하거나
수정하고 DVDPrime에 호환성 유지를 요구하지 않는다.

- WebView 제공
- DOM marker 주입
- 최근 읽은 글·고정·연속 읽기·고급 검색·키워드 차단·보관함
- 앱 명칭·아이콘·상표 사용 범위
- Google Play 공개 배포

앱은 공식 DVDPrime 앱이나 과거 호환 대상 제품의 공식 앱처럼 보이는 명칭·로고를 사용하지 않고,
허용된 개인 소개 범위를 넘는 반복 홍보나 정책 우회를 하지 않는다.

회원 메모와 알림 기능을 다시 검토하는 경우에는 퇴역 커밋을 통째로 되돌리지 않는다. 각 퇴역
커밋의 parent를 복원 기준으로 삼아 현재 HEAD에 필요한 UI·runtime만 forward-port하고, 당시의
DVDPrime 정책·요청 예산·Android 권한·저장 schema를 새로 검증한다. 기존 로컬 회원 메모와 알림
데이터 decoder를 자동 삭제하지 않는 이유도 이 호환·복구 가능성을 보존하기 위해서다.

공개 판단 직전 최신 Google Play WebView, minimum functionality,
impersonation, IP, UGC, user-data, Data safety와 target API 정책을 날짜와
함께 다시 기록한다.
