# Requirements Document

## Introduction

이 기능은 사용자가 관심 있는 미국 상장 주식(관심 종목)의 주요 뉴스를 Yahoo Finance에서 매일 자동으로 수집하고, 중복을 제거하여 종목별로 정리한 뒤, 지정된 시간에 텔레그램(Telegram)으로 전달하는 프로그램을 정의한다. 여기서 다이제스트(digest)는 AI가 생성한 산문형 요약이 아니라, 종목별로 정리된 헤드라인(제목·링크·출처·발행 시각) 목록을 컴파일한 결과물이다. 사용자는 관심 종목 목록과 전달 시간을 설정할 수 있으며, 시스템은 매일 예약된 시간에 정리된 뉴스 다이제스트(digest)를 텔레그램 봇을 통해 사용자에게 전송한다. 구현 언어는 Python이다.

주요 목표:
- 관심 종목 목록 관리
- Yahoo Finance에서 관심 종목 관련 주요 뉴스 수집(스크래핑)
- 수집된 뉴스의 중복 제거 및 정리
- 종목별 헤드라인 목록 컴파일
- 지정된 시간에 텔레그램으로 다이제스트 전송

## Glossary

- **Digest_System**: 뉴스 수집, 정리, 요약, 전달을 총괄하는 전체 시스템.
- **Watchlist_Manager**: 사용자의 관심 종목 목록(Watchlist)을 저장하고 관리하는 구성 요소.
- **Watchlist**: 사용자가 뉴스를 받고자 하는 주식 종목의 집합. 각 항목은 종목 식별자(ticker symbol)와 표시명(display name)을 포함한다.
- **News_Scraper**: 정해진 뉴스 출처(News_Source)에서 관심 종목 관련 뉴스 기사(News_Article)를 수집하는 구성 요소.
- **News_Source**: 뉴스를 제공하는 외부 출처. 본 시스템에서는 Yahoo Finance(미국 상장 주식 뉴스)를 News_Source로 사용한다.
- **News_Article**: 수집된 개별 뉴스 항목. 제목(title), 원문 링크(URL), 발행 시각(publication timestamp), 출처(source), 관련 종목(associated ticker)을 포함한다.
- **Deduplicator**: 수집된 News_Article 집합에서 중복 기사를 식별하고 제거하는 구성 요소.
- **Headline_Compiler**(헤드라인 정리기): 정리된 News_Article로부터 종목별 헤드라인 목록(Headline_List)을 컴파일하는 구성 요소. 산문형 요약을 생성하지 않고 선별된 헤드라인을 정리한다.
- **Headline_List**(종목별 정리된 헤드라인 목록): 하나의 관심 종목에 대해 정리된 헤드라인의 목록으로, 각 항목은 제목, 링크, 출처, 발행 시각을 포함한다.
- **Digest**: 특정 날짜에 대해 관심 종목별로 정리된 헤드라인 묶음. 텔레그램으로 전송되는 최종 결과물.
- **Scheduler**: 지정된 전달 시간(Delivery_Time)에 다이제스트 생성 및 전송 작업을 실행시키는 구성 요소.
- **Delivery_Time**: 사용자가 지정한 매일 다이제스트를 전달받을 시각(시간대 포함).
- **Telegram_Sender**: 텔레그램 봇 API를 통해 Digest를 사용자에게 전송하는 구성 요소.
- **User**: 관심 종목을 설정하고 텔레그램으로 다이제스트를 수신하는 사용자.
- **"기타" 그룹**: 수집된 News_Article이 Watchlist의 어느 관심 종목과도 일치하지 않을 때 해당 기사를 분류하는 기본 종목 그룹.

## Requirements

### Requirement 1: 관심 종목 목록 관리

**User Story:** 사용자로서, 나는 관심 있는 주식 종목 목록을 설정하고 싶다. 그래야 내가 관심 있는 종목의 뉴스만 받을 수 있다.

#### Acceptance Criteria

1. WHEN 사용자가 관심 종목을 추가하는 요청을 제출하면, THE Watchlist_Manager SHALL 해당 종목의 종목 식별자(1~20자)와 표시명(1~100자)을 Watchlist에 저장하고 추가 성공을 알리는 메시지를 반환한다.
2. WHEN 사용자가 Watchlist에서 특정 종목을 제거하는 요청을 제출하면, THE Watchlist_Manager SHALL 해당 종목을 Watchlist에서 삭제하고 제거 성공을 알리는 메시지를 반환한다.
3. IF 사용자가 Watchlist에 존재하지 않는 종목 식별자를 제거하려 하면, THEN THE Watchlist_Manager SHALL 제거를 거부하고 해당 종목이 존재하지 않음을 알리는 메시지를 반환하며 Watchlist를 변경하지 않는다.
4. WHEN 사용자가 Watchlist 조회를 요청하면, THE Watchlist_Manager SHALL 현재 저장된 모든 종목의 종목 식별자와 표시명을 반환한다.
5. WHILE Watchlist에 저장된 종목이 하나도 없는 상태에서, WHEN 사용자가 Watchlist 조회를 요청하면, THE Watchlist_Manager SHALL 빈 목록을 반환한다.
6. IF 사용자가 이미 Watchlist에 존재하는 종목 식별자를 추가하려 하면, THEN THE Watchlist_Manager SHALL 중복 추가를 거부하고 해당 종목이 이미 존재함을 알리는 메시지를 반환하며 기존 항목을 변경하지 않는다.
7. IF 사용자가 유효하지 않은 형식(1~20자의 영문 대문자 및 숫자 이외의 값, 또는 빈 값)의 종목 식별자를 추가하려 하면, THEN THE Watchlist_Manager SHALL 추가를 거부하고 형식 오류를 설명하는 메시지를 반환하며 Watchlist를 변경하지 않는다.
8. WHERE Watchlist에 저장된 종목 수가 최대 허용치(100개)에 도달한 상태에서, IF 사용자가 새로운 종목을 추가하려 하면, THEN THE Watchlist_Manager SHALL 추가를 거부하고 최대 개수 초과를 알리는 메시지를 반환한다.

### Requirement 2: 뉴스 수집(스크래핑)

**User Story:** 사용자로서, 나는 관심 종목에 대한 주요 뉴스가 매일 자동으로 수집되기를 원한다. 그래야 내가 직접 뉴스를 찾아보지 않아도 된다.

#### Acceptance Criteria

1. WHEN Scheduler가 다이제스트 생성 작업을 실행하면, THE News_Scraper SHALL Watchlist에 있는 각 종목에 대해 News_Source에서 관련 News_Article을 종목당 최대 20건까지 수집한다.
2. THE News_Scraper SHALL 각 수집된 News_Article에 대해 제목(최대 300자), 원문 링크(유효한 URL), 발행 시각(UTC 기준 타임스탬프), 출처 이름, 관련 종목 식별자를 기록한다.
3. IF 수집된 News_Article에서 제목, 원문 링크, 발행 시각, 출처, 관련 종목 중 하나라도 누락되거나 유효하지 않으면, THEN THE News_Scraper SHALL 해당 News_Article을 수집 결과에서 제외하고 제외 사유를 기록한다.
4. WHEN News_Scraper가 뉴스를 수집하면, THE News_Scraper SHALL 발행 시각이 작업 실행 시점 기준 직전 24시간(1,440분) 이내인 News_Article만 포함한다.
5. IF 특정 News_Source에 대한 요청이 실패하면(응답 시간이 30초를 초과하거나 오류 응답을 수신하는 경우 포함), THEN THE News_Scraper SHALL 실패한 News_Source와 실패 사유를 기록하고, 최대 3회까지 재시도한 후에도 실패하면 나머지 News_Source에 대한 수집을 계속 진행한다.
6. IF 특정 종목에 대해 24시간 이내 조건을 만족하는 유효한 News_Article이 0건이면, THEN THE News_Scraper SHALL 해당 종목을 뉴스 없음(no news) 상태로 표시하고 수집 건수를 0으로 기록한다.

### Requirement 3: 뉴스 정리 및 중복 제거

**User Story:** 사용자로서, 나는 동일한 뉴스가 여러 번 반복되지 않고 종목별로 정리된 상태로 받기를 원한다. 그래야 다이제스트를 읽기 쉽다.

#### Acceptance Criteria

1. WHEN News_Scraper가 News_Article 수집을 완료하면, THE Deduplicator SHALL 원문 링크(정규화된 URL 기준, 쿼리 스트링과 프래그먼트 제외)가 동일한 News_Article 중 발행 시각이 가장 이른 하나만 남기고 나머지를 제거한다.
2. WHEN Deduplicator가 링크 기준 중복 제거를 완료하면, THE Deduplicator SHALL 제목 유사도가 85% 이상(0~100% 범위, 정규화된 편집 거리 기반)인 News_Article을 동일 기사로 간주하여 발행 시각이 가장 이른 하나만 남기고 나머지를 제거한다.
3. IF 두 News_Article의 발행 시각이 동일하여 제거 대상을 결정할 수 없으면, THEN THE Deduplicator SHALL 원문 링크의 사전순(오름차순)으로 먼저 오는 News_Article을 남기고 나머지를 제거한다.
4. THE Digest_System SHALL 정리된 각 News_Article을 정확히 하나의 관심 종목 그룹에 매핑하여 종목별로 그룹화한다.
5. IF News_Article이 어느 관심 종목과도 일치하지 않으면, THEN THE Digest_System SHALL 해당 News_Article을 "기타" 그룹으로 분류한다.
6. THE Digest_System SHALL 각 종목 그룹 내 News_Article을 발행 시각 내림차순(최신순)으로 정렬하며, 발행 시각이 동일한 경우 제목의 사전순(오름차순)으로 정렬한다.

### Requirement 4: 뉴스 헤드라인 정리

**User Story:** 사용자로서, 나는 각 종목의 뉴스가 헤드라인 목록으로 정리되어 받기를 원한다. 그래야 제목과 링크를 훑어보며 관심 있는 기사를 바로 열어볼 수 있다.

#### Acceptance Criteria

1. WHEN Deduplicator가 뉴스 정리를 완료하면, THE Headline_Compiler SHALL 각 종목 그룹에 대해 종목당 최대 10개의 헤드라인으로 구성된 Headline_List를 5초 이내에 컴파일한다.
2. THE Headline_Compiler SHALL Headline_List의 각 헤드라인 항목에 News_Article의 제목, 클릭 가능한 원문 링크, 출처 이름, 발행 시각을 포함한다.
3. THE Headline_Compiler SHALL Headline_List의 헤드라인을 발행 시각 내림차순(최신순)으로 정렬하며, 발행 시각이 동일한 경우 제목의 사전순(오름차순)으로 정렬한다.
4. IF 특정 종목 그룹의 유효한 News_Article 수가 10개를 초과하면, THEN THE Headline_Compiler SHALL 정렬 순서상 상위 10개의 헤드라인만 Headline_List에 포함하고 나머지를 제외한다.
5. IF 특정 종목이 뉴스 없음으로 표시되면, THEN THE Headline_Compiler SHALL 해당 종목에 대해 "관련 뉴스 없음" 안내 문구와 빈 헤드라인 목록으로 구성된 Headline_List를 생성한다.
6. IF 특정 종목에 대한 Headline_List 컴파일이 실패하면, THEN THE Headline_Compiler SHALL 해당 종목에 대해 실패 상태를 표시하고 원본 News_Article의 제목과 링크 목록(최대 10개)을 대체 내용으로 제공하며 원본 데이터를 보존한다.

### Requirement 5: 다이제스트 구성

**User Story:** 사용자로서, 나는 관심 종목별 헤드라인 목록이 하나의 읽기 좋은 다이제스트로 묶여 전달되기를 원한다. 그래야 하루의 뉴스를 한 번에 확인할 수 있다.

#### Acceptance Criteria

1. WHEN Headline_Compiler가 Watchlist의 모든 종목에 대한 Headline_List 컴파일을 완료하면, THE Digest_System SHALL 해당 날짜의 Digest를 30초 이내에 구성한다.
2. THE Digest_System SHALL Digest에 다이제스트 대상 날짜를 YYYY-MM-DD 형식으로 포함한다.
3. THE Digest_System SHALL Digest에 각 관심 종목의 표시명과 해당 종목의 Headline_List를 Watchlist에 등록된 순서대로 포함한다.
4. IF Watchlist가 비어 있으면, THEN THE Digest_System SHALL 관심 종목이 설정되지 않았음을 알리는 안내 문구를 포함한 Digest를 구성한다.
5. IF 하나 이상의 종목에 대한 Headline_List가 컴파일되지 않았으면, THEN THE Digest_System SHALL 해당 종목의 표시명과 함께 헤드라인 목록을 이용할 수 없음을 알리는 문구를 Digest에 포함하고, 컴파일된 나머지 종목의 Headline_List는 유지한다.

### Requirement 6: 예약된 시간에 전달

**User Story:** 사용자로서, 나는 매일 내가 정한 시간에 다이제스트를 받고 싶다. 그래야 일정한 시간에 뉴스를 확인할 수 있다.

#### Acceptance Criteria

1. WHEN 사용자가 Delivery_Time(HH:MM 24시간 형식)과 시간대(IANA time zone 식별자)를 설정하면, THE Scheduler SHALL 해당 시각과 시간대를 다이제스트 전달 일정으로 저장하고 저장 성공을 사용자에게 표시한다.
2. IF 사용자가 입력한 Delivery_Time이 HH:MM 24시간 형식(00:00~23:59)이 아니거나 시간대 식별자가 유효한 IANA time zone 목록에 없으면, THEN THE Scheduler SHALL 해당 입력을 거부하고 기존 일정을 변경하지 않으며 형식 오류를 나타내는 오류 메시지를 표시한다.
3. WHEN 설정된 시간대 기준 현재 시각이 Delivery_Time과 같아지는 순간(초 단위 정밀도, 오차 60초 이내)에 도달하면, THE Scheduler SHALL 다이제스트 생성 및 전송 작업을 시작한다.
4. THE Scheduler SHALL 설정된 시간대 기준으로 매일 1회 Delivery_Time에 다이제스트 생성 및 전송 작업을 반복 실행한다.
5. IF 설정된 Delivery_Time에 이전 작업이 아직 실행 중이면, THEN THE Scheduler SHALL 새 작업을 시작하지 않고 이번 실행이 건너뛰어졌다는 사실과 건너뛴 시각을 기록한다.
6. IF 다이제스트 생성 또는 전송 작업이 실패하면, THEN THE Scheduler SHALL 최대 3회까지 60초 간격으로 재시도하고, 모든 재시도가 실패하면 실패 사실과 실패 시각을 기록한다.

### Requirement 7: 텔레그램 전송

**User Story:** 사용자로서, 나는 다이제스트를 텔레그램으로 받고 싶다. 그래야 평소 사용하는 메신저에서 뉴스를 확인할 수 있다.

#### Acceptance Criteria

1. WHEN Digest_System이 Digest 구성을 완료하면, THE Telegram_Sender SHALL 텔레그램 봇 API를 통해 사용자에게 Digest를 전송한다.
2. IF Digest의 길이가 텔레그램 단일 메시지 최대 길이인 4096자를 초과하면, THEN THE Telegram_Sender SHALL Digest를 각 메시지가 4096자 이하가 되도록 여러 메시지로 분할하여 원래 순서대로 전송한다.
3. IF 텔레그램 전송이 실패하면, THEN THE Telegram_Sender SHALL 각 재시도 사이에 최소 2초의 대기 시간을 두고 최대 3회까지 전송을 재시도한다.
4. IF 3회 재시도 후에도 텔레그램 전송이 실패하면, THEN THE Telegram_Sender SHALL 실패 시각, 실패 사유, 대상 사용자 식별자를 포함하는 전송 실패 기록을 남기고, 실패한 Digest 데이터를 폐기하지 않고 보존한다.
5. THE Telegram_Sender SHALL 각 News_Article의 원문 링크를 사용자가 탭하여 원문 페이지로 이동할 수 있는 클릭 가능한 형태로 Digest에 포함한다.
6. WHEN Digest에 포함된 News_Article이 0건이면, THE Telegram_Sender SHALL 전송할 뉴스가 없음을 알리는 메시지를 사용자에게 전송한다.

### Requirement 8: 설정 관리

**User Story:** 사용자로서, 나는 텔레그램 봇 토큰과 수신 대상 등 필요한 설정을 지정하고 싶다. 그래야 시스템이 나에게 정확히 메시지를 보낼 수 있다.

#### Acceptance Criteria

1. THE Digest_System SHALL 텔레그램 봇 토큰(bot token)과 수신 대상 식별자(chat identifier)를 설정 값으로 저장한다.
2. WHEN 사용자가 설정 값을 저장하면, THE Digest_System SHALL 텔레그램 봇 토큰이 비어 있지 않은 문자열(1자 이상 100자 이하)이고 수신 대상 식별자가 비어 있지 않은 값(1자 이상 100자 이하)인지 검증한다.
3. IF 텔레그램 봇 토큰 또는 수신 대상 식별자가 비어 있거나 길이 제한(1자 이상 100자 이하)을 벗어나면, THEN THE Digest_System SHALL 해당 설정 값의 저장을 거부하고 어떤 항목이 유효하지 않은지 설명하는 오류를 기록하며 기존 설정 값을 변경 없이 유지한다.
4. IF 텔레그램 봇 토큰 또는 수신 대상 식별자가 설정되지 않은 상태에서 전송 작업이 실행되면, THEN THE Telegram_Sender SHALL 전송을 중단하고 어떤 설정 항목이 누락되었는지 설명하는 오류를 기록하며 전송 대상 데이터를 변경 없이 유지한다.
5. WHEN 사용자가 Delivery_Time을 지정하지 않으면, THE Scheduler SHALL 기본 전달 시각을 오전 8시 0분 0초(사용자 시간대 기준)로 사용한다.
6. IF 사용자가 지정한 Delivery_Time이 24시간 형식(00:00부터 23:59)의 유효한 시각이 아니면, THEN THE Scheduler SHALL 해당 값을 거부하고 유효하지 않은 시각 형식임을 설명하는 오류를 기록하며 기본 전달 시각인 오전 8시 0분 0초(사용자 시간대 기준)를 사용한다.
