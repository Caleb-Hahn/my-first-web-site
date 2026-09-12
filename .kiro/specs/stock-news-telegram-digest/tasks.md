# Implementation Plan: stock-news-telegram-digest

## Overview

이 구현 계획은 `design.md`의 계층형 파이프라인 설계를 의존성 순서에 따라 점진적으로 구현하는 코딩 작업 목록이다. 구현 언어는 설계에서 확정된 **Python 3.11+**이며, 프로젝트 레이아웃은 설계의 "Suggested Project Structure"(`src/stock_digest/...`, `tests/properties`, `tests/unit`, `tests/integration`)를 따른다.

작업은 다음 순서로 진행한다.
1. 프로젝트 스캐폴딩과 데이터 모델
2. 순수 로직 컴포넌트(watchlist, settings, deduplicator, grouper/sorter, headline compiler, digest builder, message split) + property-based tests
3. I/O 어댑터(yfinance 소스, RSS 폴백, telegram client, scheduler) + mock 기반 통합 테스트
4. 파이프라인 오케스트레이터 wiring + end-to-end 스모크 테스트

`*`로 표시된 하위 작업은 선택적 테스트 작업이며, 코딩 에이전트는 이를 자동으로 구현하지 않는다. property-based test는 `hypothesis`(최소 100회 반복)로 작성하고, 각 테스트에는 `# Feature: stock-news-telegram-digest, Property {n}: {property_text}` 형식의 태그 주석을 단다.

## Tasks

- [ ] 1. 프로젝트 스캐폴딩과 데이터 모델
  - `src/stock_digest/` 패키지 디렉터리와 `__init__.py`, `sources/__init__.py` 생성
  - `tests/properties`, `tests/unit`, `tests/integration` 디렉터리와 `__init__.py`/`conftest.py` 골격 생성
  - `pyproject.toml`에 의존성 선언(yfinance, python-telegram-bot, apscheduler, rapidfuzz, tenacity, pytest, hypothesis)과 pytest 설정 추가
  - `.env.example`(TELEGRAM_BOT_TOKEN 등) 작성
  - _Requirements: 8.1_

  - [ ] 1.1 도메인 데이터 모델 정의 (`src/stock_digest/models.py`)
    - 설계의 Data Models에 따라 `@dataclass` 정의: `WatchlistEntry`, `NewsArticle`, `RawArticle`, `ExclusionRecord`, `SourceFailureRecord`, `HeadlineItem`, `HeadlineListStatus`(Enum), `HeadlineList`, `Digest`, `Settings`, `OperationResult`, `SendResult`, `SendFailureRecord`
    - 값 객체는 `frozen=True`, 시간 필드는 UTC tz-aware `datetime`으로 지정
    - `Digest.total_article_count` 프로퍼티 구현
    - _Requirements: 1.1, 2.2, 4.2, 5.2, 5.3, 6.1, 7.4, 7.6, 8.1, 8.2_

  - [ ] 1.2 상수·예외 정의 및 재시도 유틸 (`src/stock_digest/retry.py`)
    - 설계에 명시된 상수(MAX_WATCHLIST_SIZE, TICKER_PATTERN, PER_TICKER_LIMIT, FRESHNESS_MINUTES, SOURCE_TIMEOUT_SECONDS, MAX_SOURCE_RETRIES, TITLE_MAX_LEN, TITLE_SIMILARITY_THRESHOLD, OTHER_GROUP_KEY, MAX_HEADLINES_PER_TICKER, NO_NEWS_MESSAGE, TELEGRAM_MAX_LEN, MAX_SEND_RETRIES, SEND_RETRY_BACKOFF_SECONDS, JOB_MAX_RETRIES, JOB_RETRY_INTERVAL_SECONDS, DEFAULT_DELIVERY_TIME)를 해당 모듈에 배치
    - `tenacity`를 래핑한 재시도 유틸(횟수·대기 간격 파라미터화) 구현
    - _Requirements: 2.5, 6.6, 7.3_

- [ ] 2. Watchlist_Manager 구현 (순수 로직) (`src/stock_digest/watchlist.py`)
  - [ ] 2.1 WatchlistStore와 WatchlistManager 구현
    - 등록 순서를 보존하는 인메모리/JSON 백엔드 `WatchlistStore` 구현
    - `validate_ticker`(정규식 `^[A-Z0-9]{1,20}$`), 티커 대문자 정규화 구현 (Req 1.7)
    - `add`: 형식 검증 → 중복 검사(정규화 후) → 최대 100개 상한 검사 → 저장·성공 메시지 (Req 1.1, 1.6, 1.7, 1.8)
    - `remove`: 존재 시 삭제·성공, 미존재 시 거부·불변 (Req 1.2, 1.3)
    - `list_entries`: 등록 순서 보존, 비어 있으면 `[]` (Req 1.4, 1.5)
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8_

  - [ ]* 2.2 Watchlist 추가·조회 일관성 property test (`tests/properties`)
    - **Property 1: 관심 종목 추가 후 조회 일관성**
    - **Validates: Requirements 1.1, 1.4**

  - [ ]* 2.3 추가–제거 round-trip property test (`tests/properties`)
    - **Property 2: 추가–제거 round-trip**
    - **Validates: Requirements 1.1, 1.2**

  - [ ]* 2.4 미존재 티커 제거 불변성 property test (`tests/properties`)
    - **Property 3: 미존재 티커 제거의 불변성**
    - **Validates: Requirements 1.3**

  - [ ]* 2.5 중복 추가 멱등성 property test (`tests/properties`)
    - **Property 4: 중복 추가 멱등성**
    - **Validates: Requirements 1.6**

  - [ ]* 2.6 무효 티커 거부 property test (`tests/properties`)
    - **Property 5: 무효 티커 거부**
    - **Validates: Requirements 1.7**

  - [ ]* 2.7 Watchlist 엣지·경계 unit test (`tests/unit`)
    - 빈 watchlist 조회 시 `[]` (Req 1.5), 100개 상한 도달 시 추가 거부 경계 (Req 1.8)
    - _Requirements: 1.5, 1.8_

- [ ] 3. Settings/Config Store 구현 (순수 로직) (`src/stock_digest/settings.py`)
  - [ ] 3.1 SettingsStore와 검증 로직 구현
    - `Settings` 로드(봇 토큰은 환경 변수 `TELEGRAM_BOT_TOKEN` 우선), JSON 지속성(봇 토큰 평문 저장 금지)
    - `validate_settings`(봇 토큰·chat id 1~100자, 비어있지 않음) (Req 8.2)
    - `save`: 검증 위반 시 거부·오류 기록·기존 값 유지 (Req 8.3)
    - `validate_schedule`(delivery_time HH:MM 00:00~23:59, timezone은 `zoneinfo.available_timezones()`) 및 `set_schedule`(위반 시 거부·기존 일정 유지) (Req 6.1, 6.2)
    - `require_send_config`(토큰/chat id 누락 시 `MissingConfigError`) (Req 8.4)
    - Delivery_Time 미지정 시 기본 08:00, 무효 시각 입력 시 기본값 사용 (Req 8.5, 8.6)
    - _Requirements: 6.1, 6.2, 8.1, 8.2, 8.3, 8.4, 8.5, 8.6_

  - [ ]* 3.2 설정 저장 round-trip property test (`tests/properties`)
    - **Property 21: 설정 저장 round-trip**
    - **Validates: Requirements 6.1, 8.1, 8.2**

  - [ ]* 3.3 설정 검증 거부·기존 값 보존 property test (`tests/properties`)
    - **Property 22: 설정 검증 거부와 기존 값 보존**
    - **Validates: Requirements 6.2, 8.3**

  - [ ]* 3.4 설정 오류 경로 unit test (`tests/unit`)
    - 자격증명 누락 시 전송 중단 유발(`require_send_config`) (Req 8.4), 기본 08:00 적용 (Req 8.5), 무효 시각 → 기본값 (Req 8.6)
    - _Requirements: 8.4, 8.5, 8.6_

- [ ] 4. Deduplicator 구현 (순수 로직) (`src/stock_digest/dedup.py`)
  - [ ] 4.1 URL 정규화·제목 유사도·중복 제거 구현
    - `normalize_url`(스킴·호스트 소문자화, 쿼리·프래그먼트 제거, 말미 슬래시 정규화) (Req 3.1)
    - `title_similarity`(rapidfuzz 1차, difflib 폴백, 제목 정규화 후 0~100) (Req 3.2)
    - `_survivor`(이른 발행시각 우선, 동시각이면 정규화 URL 사전순) (Req 3.3)
    - `deduplicate`: `(published_at, normalized_url)` 안정 정렬 후 URL 기준 → 유사도 85% 기준 결정적 클러스터링 (Req 3.1, 3.2, 3.3)
    - _Requirements: 3.1, 3.2, 3.3_

  - [ ]* 4.2 중복 없음 property test (`tests/properties`)
    - **Property 10: 중복 제거 결과에는 중복이 없다**
    - **Validates: Requirements 3.1, 3.2**

  - [ ]* 4.3 생존자 선택 결정성 property test (`tests/properties`)
    - **Property 11: 중복 제거 생존자 선택의 결정성**
    - **Validates: Requirements 3.1, 3.2, 3.3**

  - [ ]* 4.4 중복 제거 멱등성 property test (`tests/properties`)
    - **Property 12: 중복 제거 멱등성** — `dedup(dedup(x)) == dedup(x)`
    - **Validates: Requirements 3.1, 3.2**

- [ ] 5. Grouper/Sorter 구현 (순수 로직) (`src/stock_digest/grouping.py`)
  - [ ] 5.1 그룹화·정렬 구현
    - `sort_key`: `(-published_at.timestamp(), title)` — 최신순, 동시각 제목 사전순 (Req 3.6)
    - `group_and_sort`: 각 기사를 associated ticker로 매핑, watchlist 미일치 시 `OTHER_GROUP_KEY`("기타"), 그룹 내 정렬, 결과 키 순서는 watchlist 순서 + "기타" (Req 3.4, 3.5, 3.6)
    - _Requirements: 3.4, 3.5, 3.6_

  - [ ]* 5.2 그룹화 완전 분할 property test (`tests/properties`)
    - **Property 13: 그룹화는 완전한 분할이다**
    - **Validates: Requirements 3.4**

  - [ ]* 5.3 미일치 기사 "기타" 그룹 property test (`tests/properties`)
    - **Property 14: 미일치 기사는 "기타" 그룹으로**
    - **Validates: Requirements 3.5**

- [ ] 6. Headline_Compiler 구현 (순수 로직) (`src/stock_digest/compiler.py`)
  - [ ] 6.1 헤드라인 컴파일 구현
    - `to_headline_item`(제목·클릭 가능한 링크·출처·발행시각 포함) (Req 4.2)
    - `compile`: 정렬된 상위 10개로 `HeadlineList` 구성 (Req 4.1, 4.3, 4.4)
    - 뉴스 없음 시 `status=NO_NEWS`, 빈 목록, "관련 뉴스 없음" note (Req 4.5)
    - 컴파일 실패 시 `status=FAILED`, 원본 제목·링크(최대 10) 대체·원본 보존 (Req 4.6)
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6_

  - [ ]* 6.2 정렬·상위 10개 선택 property test (`tests/properties`)
    - **Property 15: 정렬 및 상위 10개 선택 불변식**
    - **Validates: Requirements 3.6, 4.1, 4.3, 4.4**

  - [ ]* 6.3 헤드라인 항목 필드 완전성 property test (`tests/properties`)
    - **Property 16: 헤드라인 항목의 필드 완전성**
    - **Validates: Requirements 4.2**

  - [ ]* 6.4 뉴스 없음·컴파일 실패 unit test (`tests/unit`)
    - 뉴스 없음 안내 (Req 4.5), 컴파일 실패 대체 내용·원본 보존 (Req 4.6)
    - _Requirements: 4.5, 4.6_

- [ ] 7. Digest Builder 구현 (순수 로직) (`src/stock_digest/digest.py`)
  - [ ] 7.1 다이제스트 구성 구현
    - `build`: `target_date`(YYYY-MM-DD) 포함, watchlist 순서대로 종목별 `HeadlineList` 섹션 구성 (Req 5.2, 5.3)
    - 빈 watchlist면 `empty_watchlist_note` 안내 Digest (Req 5.4)
    - 미컴파일 종목은 "이용 불가" 문구 포함, 나머지 섹션 유지 (Req 5.5)
    - _Requirements: 5.2, 5.3, 5.4, 5.5_

  - [ ]* 7.2 섹션 순서 보존 property test (`tests/properties`)
    - **Property 17: 다이제스트 섹션 순서 보존**
    - **Validates: Requirements 5.3**

  - [ ]* 7.3 날짜 포맷 property test (`tests/properties`)
    - **Property 18: 날짜 포맷** (YYYY-MM-DD)
    - **Validates: Requirements 5.2**

  - [ ]* 7.4 빈 watchlist·부분 실패 unit test (`tests/unit`)
    - 빈 watchlist 안내 (Req 5.4), 부분 컴파일 실패 시 나머지 유지 (Req 5.5)
    - _Requirements: 5.4, 5.5_

- [ ] 8. Checkpoint - 순수 로직 컴포넌트 테스트 통과 확인
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. News_Scraper 구현 (정규화·검증·필터·상한, 순수 부분) (`src/stock_digest/scraper.py`)
  - [ ] 9.1 정규화·검증·신선도·상한 로직 구현
    - `normalize_and_validate`(필수 필드 검증·정규화, 제목 300자 초과 무효, 누락/무효 시 `ExclusionRecord`) (Req 2.2, 2.3)
    - `is_fresh`(now_utc 기준 직전 1440분 이내) (Req 2.4)
    - `scrape`의 순수 부분: 종목당 최대 20건 제한, 24h 필터, 0건 시 `no_news_tickers` 기록 (Req 2.1, 2.4, 2.6)
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.6_

  - [ ]* 9.2 종목당 수집 상한 property test (`tests/properties`)
    - **Property 6: 종목당 수집 상한** (≤ 20건)
    - **Validates: Requirements 2.1**

  - [ ]* 9.3 정규화 필드 제약 property test (`tests/properties`)
    - **Property 7: 정규화 결과의 필드 제약**
    - **Validates: Requirements 2.2**

  - [ ]* 9.4 무효 기사 제외 property test (`tests/properties`)
    - **Property 8: 무효 기사 제외**
    - **Validates: Requirements 2.3**

  - [ ]* 9.5 24시간 신선도 필터 property test (`tests/properties`)
    - **Property 9: 24시간 신선도 필터**
    - **Validates: Requirements 2.4**

  - [ ]* 9.6 종목별 0건 no-news unit test (`tests/unit`)
    - 종목별 유효 기사 0건 → no news 표시·count 0 (Req 2.6)
    - _Requirements: 2.6_

- [ ] 10. 뉴스 소스 어댑터 구현 (I/O) (`src/stock_digest/sources/`)
  - [ ] 10.1 NewsSource Protocol과 RawArticle 정의 (`sources/base.py`)
    - `NewsSource` Protocol(`fetch(ticker, timeout) -> list[RawArticle]`) 정의
    - _Requirements: 2.1, 2.2_

  - [ ] 10.2 YFinanceSource 1차 소스 구현 (`sources/yfinance_source.py`)
    - `yfinance` `Ticker.news` 응답을 `RawArticle`로 정규화, 타임아웃/오류 시 예외
    - _Requirements: 2.1, 2.2, 2.5_

  - [ ] 10.3 YahooRSSSource 폴백 소스 구현 (`sources/rss_source.py`)
    - `feeds.finance.yahoo.com` RSS 응답을 `RawArticle`로 파싱, 타임아웃/오류 시 예외
    - _Requirements: 2.1, 2.2, 2.5_

  - [ ] 10.4 NewsScraper에 소스·재시도·폴백 wiring
    - `scrape`에서 소스 호출 → 최대 3회 재시도 → 폴백(RSS) 시도 → 실패 시 `SourceFailureRecord` 기록 후 나머지 종목 계속 (Req 2.5)
    - 9.1의 순수 로직(정규화·필터·상한)과 결합
    - _Requirements: 2.1, 2.5_

  - [ ]* 10.5 소스 어댑터 파싱 unit test (`tests/unit`)
    - 대표 샘플 응답으로 yfinance/RSS → `RawArticle` 파싱 검증
    - _Requirements: 2.2_

  - [ ]* 10.6 소스 실패·재시도·폴백 integration test (`tests/integration`, mock)
    - mock 소스로 실패 → 3회 재시도 → 폴백 → 나머지 종목 계속 진행 검증 (Req 2.5)
    - _Requirements: 2.5_

- [ ] 11. Telegram_Sender 구현 (렌더·분할 순수 + 전송 I/O) (`src/stock_digest/telegram_sender.py`)
  - [ ] 11.1 렌더링·메시지 분할 구현 (순수)
    - `render`(Digest → parse mode 텍스트, 각 기사 클릭 가능 링크 마크업 포함) (Req 7.5)
    - `split_message`(각 chunk ≤ 4096, 순서 보존, 가능하면 줄 경계 분할) (Req 7.2)
    - _Requirements: 7.2, 7.5_

  - [ ]* 11.2 메시지 분할 재구성·길이 상한 property test (`tests/properties`)
    - **Property 19: 메시지 분할 재구성 및 길이 상한**
    - **Validates: Requirements 7.2**

  - [ ]* 11.3 클릭 가능한 링크 포함 property test (`tests/properties`)
    - **Property 20: 클릭 가능한 링크 포함**
    - **Validates: Requirements 7.5**

  - [ ] 11.4 TelegramClient·전송 로직 wiring (I/O)
    - `TelegramClient` Protocol과 Bot API `sendMessage` 구현
    - `send_digest`: `require_send_config` 확인 (Req 8.4), 뉴스 0건 시 "전송할 뉴스 없음" (Req 7.6), 렌더→분할→순서 전송 (Req 7.1, 7.2), 실패 시 2초 이상 간격 3회 재시도 (Req 7.3), 최종 실패 시 `SendFailureRecord` 기록·Digest 보존 (Req 7.4)
    - _Requirements: 7.1, 7.3, 7.4, 7.6, 8.4_

  - [ ]* 11.5 전송 재시도·실패 기록 integration test (`tests/integration`, mock)
    - mock TelegramClient로 전송 호출 (Req 7.1), 실패 시 2초 간격 3회 재시도 (Req 7.3), 최종 실패 기록·Digest 보존 (Req 7.4)
    - 뉴스 0건 → "전송할 뉴스 없음" (Req 7.6), 자격증명 누락 시 전송 중단 (Req 8.4)
    - _Requirements: 7.1, 7.3, 7.4, 7.6, 8.4_

- [ ] 12. Scheduler 구현 (I/O) (`src/stock_digest/scheduler.py`)
  - [ ] 12.1 APScheduler 기반 Scheduler 구현
    - `start`: 저장된 Delivery_Time·시간대로 cron trigger 등록 후 `BlockingScheduler` 실행, 미지정 시 기본 08:00 (Req 6.3, 6.4, 8.5)
    - `run_job`: `max_instances=1`, `coalesce=True`로 오버랩 방지·건너뜀 기록 (Req 6.5), 실패 시 60초 간격 3회 재시도·최종 실패 기록 (Req 6.6)
    - _Requirements: 6.3, 6.4, 6.5, 6.6, 8.5_

  - [ ]* 12.2 스케줄러 트리거·오버랩·재시도 integration test (`tests/integration`, mock)
    - cron trigger 등록·다음 실행 시각 (Req 6.3, 6.4), 오버랩 시 건너뜀 (Req 6.5), 잡 실패 3회 재시도·기록 (Req 6.6)
    - _Requirements: 6.3, 6.4, 6.5, 6.6_

- [ ] 13. DigestPipeline 오케스트레이터 wiring (`src/stock_digest/pipeline.py`)
  - [ ] 13.1 파이프라인 조립·실행 구현
    - `run`: scrape → deduplicate → group_and_sort → compile → build → send_digest 순서로 컴포넌트 조립·실행
    - Watchlist_Manager·SettingsStore 입력 결합, Scheduler에서 호출 가능한 진입점 노출
    - _Requirements: 5.1, 2.1, 3.1, 3.4, 4.1, 5.3, 7.1_

  - [ ]* 13.2 end-to-end 스모크 integration test (`tests/integration`, mock)
    - mock Yahoo Finance + mock Telegram으로 `DigestPipeline.run` 전 구간 1회 완주 검증 (Req 5.1)
    - _Requirements: 5.1_

- [ ] 14. Final checkpoint - 전체 테스트 통과 확인
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- `*`로 표시된 하위 작업은 선택적(테스트)이며 빠른 MVP를 위해 건너뛸 수 있다. 코딩 에이전트는 `*` 하위 작업을 구현하지 않는다.
- 각 작업은 추적성을 위해 특정 요구사항(및/또는 설계 프로퍼티)을 참조한다.
- 체크포인트(작업 8, 14)에서 점진적 검증을 수행한다.
- property-based test는 `hypothesis`로 작성하며 최소 100회 반복(`@settings(max_examples=100)` 이상)하고, 태그 형식은 **Feature: stock-news-telegram-digest, Property {number}: {property_text}**를 따른다.
- 설계의 22개 Correctness Property는 각각 하나의 property-based test 하위 작업으로 매핑된다: P1–5(작업 2), P21–22(작업 3), P10–12(작업 4), P13–14(작업 5), P15–16(작업 6), P17–18(작업 7), P6–9(작업 9), P19–20(작업 11).

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2"] },
    { "id": 1, "tasks": ["2.1", "3.1", "4.1", "5.1", "6.1", "7.1", "9.1", "10.1", "11.1"] },
    { "id": 2, "tasks": ["2.2", "2.3", "2.4", "2.5", "2.6", "2.7", "3.2", "3.3", "3.4", "4.2", "4.3", "4.4", "5.2", "5.3", "6.2", "6.3", "6.4", "7.2", "7.3", "7.4", "9.2", "9.3", "9.4", "9.5", "9.6", "10.2", "10.3", "11.2", "11.3"] },
    { "id": 3, "tasks": ["10.4", "11.4", "12.1"] },
    { "id": 4, "tasks": ["10.5", "10.6", "11.5", "12.2", "13.1"] },
    { "id": 5, "tasks": ["13.2"] }
  ]
}
```
