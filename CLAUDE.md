# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 프로젝트 개요

한국 동행복권 로또 6/45 자동 구매 플러그인 시스템. Flask 기반 웹 인터페이스와 Selenium을 이용한 브라우저 자동화로 로또 자동 구매를 스케줄링 및 관리.

## 핵심 아키텍처

### 3계층 구조
1. **Web Layer** (`setup.py`, `mod_basic.py`)
   - Flask 플러그인 시스템 기반
   - 설정 UI, 스케줄러 제어, 구매 이력 조회
   - `PluginModuleBase` 상속받는 `ModuleBasic`이 핵심 모듈

2. **Business Logic Layer** (`mod_basic.py`)
   - `do_action()`: 구매 프로세스 전체 오케스트레이션
   - `get_buy_data()`: 구매 모드별 번호 생성 (auto/manual/plugin_auto)
   - `scheduler_function()`: 크론 스케줄러 실행 함수
   - Discord 알림 연동 (`ToolNotify`, `SupportDiscord`)

3. **Automation Layer** (`ff_pylotto.py`)
   - `DhLottery`: Selenium 기반 동행복권 사이트 자동화
   - 드라이버 모드: local (Chrome) / remote (Firefox)
   - 핵심 메서드: `login()`, `buy_lotto()`, `check_deposit()`, `check_history()`

### 데이터 모델
- `ModelLottoItem` (model.py): 구매 이력 저장
  - round (회차), count (구매 수), data (JSON), img (스크린샷 URL)

## 주요 개발 명령어

### 환경 구성
```bash
# Selenium Remote 드라이버 실행 (권장)
# AMD64
docker run -d --name selenium_firefox -p 4444:4444 -p 5900:5900 -p 7900:7900 --shm-size 2g selenium/standalone-firefox:latest

# ARM (오라클 A1)
docker run -d --name selenium_firefox -p 4444:4444 -p 5900:5900 -p 7900:7900 --shm-size 2g seleniarm/standalone-firefox:latest
```

### noVNC 접속
- URL: `http://localhost:7900`
- 비밀번호: `secret`
- 용도: 브라우저 자동화 화면 실시간 모니터링

### 포트 구분 (중요!)
- **4444**: Selenium WebDriver 포트 (코드에서 사용)
- **7900**: noVNC 웹 인터페이스 포트 (사람이 브라우저로 접속)
- **5900**: VNC 포트 (VNC 클라이언트 접속용)

## 구매 모드 설정 (buy_data)

```python
# 완전 자동
auto

# 수동 6개
manual : 2, 3, 4, 8, 45, 3

# 반자동 (3개 수동 + 3개 자동)
manual : 2, 3, 4

# 플러그인 자동 (세션당 동일 번호)
plugin_auto
plugin_auto
plugin_auto

# 주석 지원
#manual : 8, 28, 12, 9
```

## 중요 구현 사항

### 드라이버 모드 분기
- **local**: Chrome + ChromeDriverManager (윈도우/맥 GUI 환경)
- **remote**: Firefox + WebDriver Remote (VPS/Docker 환경)
- User-Agent 위장: Windows 10 Chrome으로 설정 (remote 모드)

### 구매 프로세스 플로우
```
login() → check_deposit() → check_history()
→ 구매 가능 여부 검증 (예치금, 주당 5건 제한)
→ buy_lotto(buy_data, dry=False)
→ 스크린샷 저장 + DB 기록 + Discord 알림
```

### 에러 핸들링 패턴
- 팝업 처리: `remove_popup()` - 다중 윈도우 핸들 정리
- Alert 예외: `UnexpectedAlertPresentException` 캐치
- iframe 전환: 구매 페이지는 iframe 구조
- WebDriverWait: 동적 요소 로딩 대기 (최대 10초)

### 알림 전략 (notify_mode)
- `always`: 모든 스케줄 실행 시 알림
- `real_buy`: 실제 구매 성공 또는 예치금 부족 시만 알림
- `none`: 알림 비활성화

## 데이터베이스
- SQLAlchemy 기반 ORM
- 바인드키: `P.package_name` (lotto)
- 자동 삭제 기능: `db_auto_delete` + `db_delete_day` 설정

## 보안 및 주의사항
- 사용자 ID/비밀번호는 `P.ModelSetting`에 평문 저장
- 동행복권 사이트는 주당 5건 구매 제한 (하드코딩)
- `buy_mode_one_of_week=True`: 이번 주 구매 이력 있으면 스킵
- dry run 모드: `test_buy` 커맨드로 구매 전 미리보기

## 플러그인 시스템 컨벤션
- 모든 설정값은 `db_default` 딕셔너리로 초기화
- 메뉴 구조는 `setting` 딕셔너리의 `menu` 키로 정의
- 템플릿 파일명: `{package_name}_{module_name}_{sub}.html`
- 스케줄러: cron 표현식 (`interval` 설정값)

## 트러블슈팅

### Selenium 연결 실패 (501 Unsupported method)
**증상**: `WebDriverException: Message: Error code: 501, Unsupported method ('POST')`

**원인**: 잘못된 포트 사용 (7900 noVNC 포트를 Selenium 포트로 착각)

**해결**:
1. 웹 UI → 설정 → Selenium 드라이버 모드 → "리모트" 선택
2. ID 필드를 다음 중 하나로 설정:
   - Docker 내부: `http://172.17.0.1:4444/wd/hub`
   - 외부 접속: `http://서버IP:4444/wd/hub` 또는 `http://서버IP:4444`
3. 설정 저장 후 "구입 테스트" 버튼으로 확인

### KeyError: 'data' 에러
**증상**: `process_command`에서 `KeyError: 'data'` 발생

**원인**: `do_action()` 예외 발생 시 'log' 키만 설정하고 'data' 키는 없음

**해결**: 코드에서 `data.get('log', str(data))` 사용 (이미 수정됨)
