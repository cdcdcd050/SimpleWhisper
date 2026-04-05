# CH00의 WoW 애드온 모음

## 개요
- **제작자**: CH00
- **UI 언어**: 영어 기본, 한글(koKR) 자동 전환 (다국어 구조)

## 레포지토리 구조

애드온별로 **독립된 GitHub 레포**로 관리한다. 로컬에서는 하나의 작업 폴더(`C:\Claude\WOW_Addon\`) 아래 각 애드온 폴더가 자체 `.git`을 가진다.

| 애드온 | 레포 | 설명 |
|--------|------|------|
| SimpleWhisper | `cdcdcd050/SimpleWhisper` (이 레포) | 귓속말 메신저 |
| SimpleRepu | `cdcdcd050/SimpleRepu` | TBC 평판 가이드 |
| SimpleRepItem | `cdcdcd050/SimpleRepItem` | TBC 평판 아이템 브라우저 (BCC 전용) |

> **커밋/푸시/릴리즈는 반드시 해당 애드온 폴더 안에서 수행한다.** 다른 애드온의 레포에 영향을 주지 않도록 주의.

### 이 레포 (SimpleWhisper) 파일 구성
```
(repo root)
├── SimpleWhisper/               -- 귓속말 메신저 애드온
│   ├── SimpleWhisper.toc        -- TOC (멀티 인터페이스)
│   ├── SimpleWhisper_*.toc      -- 클라이언트별 TOC
│   ├── SimpleWhisper.lua        -- 단일 메인 파일 (전체 로직)
│   ├── icon/
│   │   └── icon.PNG             -- 애드온 아이콘 이미지
│   ├── Sounds/                  -- 커스텀 알림 소리 폴더
│   └── libs/                    -- LibStub, CallbackHandler, LDB
├── CLAUDE.md                    -- 이 문서
├── CHANGELOG.md                 -- 변경 이력
├── README.md                    -- 프로젝트 설명
└── .github/                     -- CI/CD
```

---

# SimpleWhisper (귓속말) 애드온

## 개요
- **애드온**: SimpleWhisper (귓속말) v1.5.1
- **대상 클라이언트**: WoW 클래식~리테일 (Interface: 11508, 20505, 30405, 40402, 50503, 120001)
- **용도**: 간단한 귓속말(Whisper) 메신저
- **제작 배경**: 기존 귓속말 애드온들이 과도하게 복잡하여 핵심 기능만 새로 구현

### 의존성
- **필수**: 없음 (독립 실행 가능)
- **선택**: Arcana — 데이터 바에 귓속말 브로커 표시

### SavedVariables (캐릭터별)
- **`SimpleWhisper_DB`** (`SavedVariablesPerCharacter`): 설정 + 대화 내용 저장
  ```lua
  SimpleWhisper_DB = {
      windowPos = { point, x, y },
      windowSize = { w, h },
      dividerX = 100,
      minimapPos = 220,             -- 미니맵 버튼 각도
      soundEnabled = true,          -- 수신 알림 소리
      soundChoice = 1,              -- 소리 선택 (1=귓속말, 2=경매장, 3=커스텀)
      showTime = false,             -- 시간 표시
      autoOpen = true,              -- 수신 시 자동 열기
      combatMode = 1,               -- 전투 중 수신 (1=즉시, 2=종료후, 3=안열기)
      hideFromChat = true,          -- 기본 채팅창 숨기기
      interceptWhisper = true,      -- 귓속말 여기서 열기
      escClose = true,              -- ESC 클릭 시 즉시 닫기
      fontSize = 12,                -- 글꼴 크기 (10~22)
      opacity = 0.85,               -- 창 불투명도 (0.3~1.0)
      opacityEnabled = true,        -- 불투명도 적용 여부
      toolbarHidden = false,        -- 툴바 숨김 상태
      nameListHidden = false,       -- 이름 목록 숨김 상태
      conversations = {},           -- 대화 내용 (PLAYER_LOGOUT 시 저장)
      nameList = {},                -- 이름 목록 순서 (PLAYER_LOGOUT 시 저장)
      unreadCounts = {},            -- 안 읽은 메시지 수 (PLAYER_LOGOUT 시 저장)
  }
  ```

## 기능

### 슬래시 명령어
| 명령어 | 기능 |
|--------|------|
| `/swsw` | 메인 창 열기/닫기 |
| `/swsw demo` | 데모 데이터 로드 (테스트용) |
| `/swsw t1` | 가짜 귓속말 수신 시뮬레이션 (테스트용) |

### UI 레이아웃 (450×298 팝업 창, 리사이즈 가능)
```
+--[≡]---[who정보][새로고침]------[◀]--+
| [시간][삭제][복사][초대][옵션][X]     |
+--------+----------------------------+
|        |                            |
| 이름   |   대화 내용                 |
| 목록   |   (ScrollingMessageFrame)  |
|        |                            |
|        |   — 2026-03-20 —           |
| 상대A  |   상대A: 안녕              |
| 상대B  |   내닉네임: ㅎㅇ           |
|        |                        [↓] |
|        +----------------------------+
|        | [메시지 입력...           ] |
+--------+----------------------------+
| 상대A 메모: 클릭하여 입력...    [//] |
+--------------------------------------+
```

- **햄버거 버튼 (≡)**: 툴바 표시/숨기기 토글
- **툴바** (제목 아래): who 정보 + 시간/삭제/복사/초대/옵션/닫기 버튼
- **이름 목록 토글 (◀/▶)**: 왼쪽 이름 패널 접기/펼치기
- **왼쪽 패널**: 대화 상대 이름 버튼 리스트 (최근 활동순, 드래그로 너비 조절)
  - 직업 색상 표시, BNet 친구는 청록색 표시
  - 안 읽은 메시지 수 빨간색 표시, 선택된 대화 하이라이트 (금색 좌측 바)
  - 좌클릭: 대화 선택 + `/who` 자동 조회
  - 우클릭: WoW 기본 플레이어 드롭다운 메뉴 (초대/거래/무시 등)
- **오른쪽 상단**: ScrollingMessageFrame — 대화 내용, 날짜 구분선, 마우스 휠 스크롤
  - 상대 이름은 클릭 가능한 하이퍼링크 (`|Hplayer:이름|h`)
  - URL 자동 링크화 (클릭 시 복사 팝업)
  - 맨 아래로 스크롤 버튼 (↓)
- **오른쪽 하단**: EditBox — 메시지 입력, Enter로 전송, ESC로 창 닫기
- **메모란**: 창 하단에 대화 상대별 메모 입력 가능
- **삭제 버튼**: 확인 팝업 후 선택 대화 삭제
- **복사 버튼**: 선택 대화 전체를 텍스트 팝업으로 복사
- **초대 버튼**: 선택 대화 상대 파티 초대 (BNet 대화에서는 비활성화)
- **옵션 버튼**: 드롭다운 설정 패널 토글
- **리사이즈**: 우하단 핸들 (300×200 ~ 800×600)
- **닫기**: X 버튼 또는 ESC 키

### 옵션 패널 (옵션 버튼 클릭 시 토글)
| 옵션 | 기본값 | 설명 |
|------|--------|------|
| 수신 알림 소리 | 켜짐 | 3종 소리 선택 가능 (귓속말/경매장/커스텀sw3.ogg) |
| 수신 시 자동 열기 | 켜짐 | 귓속말 수신 시 창 자동 팝업 |
| ↳ 전투중 | 즉시 | 3단 선택: 즉시(전투 중 바로 열기) / 종료후(전투 끝나고 열기) / 안열기 |
| 기본 채팅창에서 귓속말 숨기기 | 켜짐 | 귓속말을 SimpleWhisper에만 표시 (`ChatFrame_AddMessageEventFilter` 사용, BNet 포함) |
| 귓속말 보낼 때 여기서 열기 | 켜짐 | 채팅창 이름 클릭/귓속말 시작 시 SimpleWhisper로 가로채기 |
| ESC 클릭 시 즉시 닫기 | 켜짐 | 켜짐: ESC로 즉시 창 닫기, 꺼짐: ESC로 포커스 해제 후 재차 ESC로 닫기 |
| 글꼴 크기 | 12pt | 슬라이더 (10~22pt), 채팅/입력/이름 목록/메모 모두 적용 |
| 불투명도 | 85% | 체크박스 + 슬라이더 (30~100%), 체크 해제 시 100% |
| 전체삭제 | — | 모든 대화 삭제 (설정 유지) |
| 초기화 | — | 모든 설정 기본값 복원 (대화 유지) |

### /who 조회
- 대화 상대 선택 시 자동으로 `/who` 쿼리 전송
- 결과(레벨, 직업, 길드)를 툴바에 표시 + conversations에 저장
- 오프라인이면 빨간색 "오프라인" 표시
- 새로고침 버튼 + 5초 쿨다운
- `/who` 시스템 메시지가 채팅창에 표시되지 않도록 필터링

### 채팅창 이름 클릭 연동
- `SetItemRef` 후킹으로 기본 채팅창의 플레이어 이름 좌클릭 시 SimpleWhisper 대화 창을 열고 해당 상대 선택
- `BNplayer` 링크 좌클릭 시 BNet 대화 열기
- 우클릭 등 다른 버튼은 WoW 기본 동작 유지
- `interceptWhisper` 옵션이 꺼져 있으면 가로채기 비활성화

### 귓속말 가로채기 (ChatEdit_UpdateHeader)
- WoW의 모든 귓속말 시작 경로(초상화/우클릭/파티/채팅입력 등)를 `ChatEdit_UpdateHeader` (또는 12.0+ `ChatFrameUtil.ActivateChat`) 후킹으로 가로채기
- 귓속말 모드 전환 시 SimpleWhisper 창을 열고 기본 채팅 귓속말 모드 취소

### BNet (배틀넷) 귓속말 지원
- `CHAT_MSG_BN_WHISPER` / `CHAT_MSG_BN_WHISPER_INFORM` 이벤트 처리
- 배틀태그에서 `#` 뒤를 제거하여 간결한 이름 표시
- BNet 전용 색상 (수신: 청록 `|cff00b4d8`, 발신: 파랑 `|cff2ca2ff`)
- BNet 메시지 발송: `C_BattleNet.SendWhisper` 또는 `BNSendWhisper` 호환
- BNet 대화는 이름 목록에서 청록색 표시, 초대 버튼 비활성화

### 미니맵 버튼
- 미니맵 주변에 드래그 가능한 버튼 표시
- 안 읽은 메시지가 있으면 숫자 배지 + 아이콘 색상 변경 (핑크)
- 좌클릭: 창 토글
- 우클릭 드래그: 위치 이동
- 툴팁: 제목 + 안 읽은 메시지 수 + 힌트

### 시스템 메시지 처리
- "플레이어를 찾을 수 없습니다" / "무시하고 있습니다" 메시지를 해당 대화에 시스템 메시지(`sys`)로 기록
- `ERR_CHAT_PLAYER_NOT_FOUND_S`, `ERR_CHAT_IGNORED_S` 패턴 매칭

### 직업 캐시 시스템
- 채팅 이벤트(일반/외침/길드/파티/공격대/전장 등)에서 `GetPlayerInfoByGUID`로 직업 정보 수집
- `target`/`focus`/`mouseover`/파티·공격대 유닛에서도 직업 탐색
- `/who` 결과에서도 직업 추출
- 캐시된 직업 정보로 이름 목록에 클래스 색상 표시

### 동작 방식

#### 귓속말 수신 시 (일반 + BNet)
1. 대화 기록에 메시지 추가
2. 이름 목록에서 해당 상대를 최상단으로 이동 (`BumpName`)
3. 알림 소리 재생 (옵션)
4. 창이 닫혀 있으면 자동 열기 (옵션, 전투 중: 즉시/종료후/안열기 3단 선택)
5. 선택된 대화가 없으면 수신된 대화를 자동 선택
6. 다른 대화 선택 중이면 해당 상대의 안 읽은 수 증가 (대화 전환 안 됨)

#### 귓속말 발신 시
1. 입력창에서 Enter → `SendChatMessage(text, "WHISPER", nil, fullName)` 또는 BNet은 `SendBNetWhisper(bnID, text)` 호출
2. `CHAT_MSG_WHISPER_INFORM` / `CHAT_MSG_BN_WHISPER_INFORM` 이벤트로 발신 메시지 자동 기록
3. 현재 선택된 대화면 채팅 표시 즉시 갱신

#### 대화 저장/복원
- `PLAYER_LOGOUT` 시 `conversations`, `nameList`, `unreadCounts`를 `SimpleWhisper_DB`에 저장
- `ADDON_LOADED` 시 복원

### 메시지 표시 형식
- **수신 (일반)**: `상대이름: 메시지` (핑크 `|cffff88ff`, 이름은 `|Hplayer:|h` 하이퍼링크)
- **수신 (BNet)**: `상대이름: 메시지` (청록 `|cff00b4d8`)
- **발신 (일반)**: `내닉네임: 메시지` (연핑크 `|cffffbbdd`) — `UnitName("player")`로 실제 캐릭터명 표시
- **발신 (BNet)**: `내닉네임: 메시지` (파랑 `|cff2ca2ff`)
- **/who 결과**: 녹색 `|cff00ff00`
- **시스템 메시지**: 빨간색 `|cffff4444`
- **시간 표시 활성화 시**: `[12:34:56] 상대이름: 메시지`
- **날짜 구분선**: `— 2026-03-20 —` (회색)
- **URL**: 자동 링크화 (파란색 `|cff4488ff`, 클릭 시 복사 팝업)
- **색상**: 시간 `|cffaaaaaa` (회색), 안 읽은 수 `|cffff3333` (빨강)

### LDB (LibDataBroker) 연동
- LDB data source 이름: `"SimpleWhisper"`
- 아이콘: `Interface\CHATFRAME\UI-ChatIcon-Chat-Up` (채팅 아이콘)
- 텍스트: `"귓속말"` — 안 읽은 메시지 있으면 `"귓속말(3)"` 형태로 표시, 없으면 `"귓속말(0)"`
- 좌클릭: 창 토글
- 툴팁: 제목 + 안 읽은 메시지 수 + 힌트

## 코드 구조 (SimpleWhisper.lua)

### 다국어 문자열 테이블 `L`
영어 기본값으로 초기화 후 `GetLocale() == "koKR"` 시 한글 오버라이드. 다국어 확장 시 같은 패턴으로 추가.

### 세션 데이터
| 변수 | 타입 | 용도 |
|------|------|------|
| `conversations` | `table` | `["이름"] = { fullName, class, isBN, bnID, guid, whoLevel, whoGuild, memo, {who, msg, time, date}, ... }` |
| `nameList` | `table` | 최근 활동순 이름 배열 |
| `unreadCounts` | `table` | `["이름"] = 숫자` |
| `selectedName` | `string/nil` | 현재 선택된 대화 상대 |
| `mainFrame` | `Frame/nil` | 메인 UI 프레임 (lazy 생성) |
| `ldbObject` | `table/nil` | LDB 데이터 오브젝트 |
| `minimapBadge` | `Frame/nil` | 미니맵 안 읽은 수 배지 |
| `pendingWhoName` | `string/nil` | `/who` 조회 대기 중인 이름 |
| `pendingWhoTimer` | `Timer/nil` | `/who` 타임아웃 타이머 |
| `whoFilterUntil` | `number` | `/who` 시스템 메시지 필터 만료 시간 |
| `classCache` | `table` | 채팅에서 수집한 직업 캐시 `["이름"] = "CLASS_TOKEN"` |

### 핵심 함수
| 함수 | 역할 |
|------|------|
| `ShortName(fullName)` | `Ambiguate`로 서버명 제거 |
| `ResolveBNetName(bnID)` | BNet ID → 배틀태그 또는 계정명 해석 |
| `GetBNetToonName(bnID)` | BNet 친구의 현재 캐릭터명 조회 |
| `SendBNetWhisper(bnID, text)` | BNet 메시지 발송 (Retail/Classic 호환) |
| `ResolveClass(name)` | 캐시/유닛/파티에서 직업 탐색 |
| `EnsureConversation(name, fullName, isBN, bnID)` | 대화 테이블 초기화, fullName/BNet 정보 보존 |
| `BumpName(name)` | 이름을 목록 최상단으로 이동 |
| `PlayWhisperSound()` | 수신 알림 소리 재생 (3종 선택) |
| `UpdateLDBText()` | LDB 텍스트 + 미니맵 배지 갱신 (안 읽은 수) |
| `AddMessage(name, dir, text, fullName)` | 대화 기록 추가 + 이름 순서 갱신 + 안 읽은 수 처리 |
| `LinkifyURLs(text)` | URL 자동 하이퍼링크 변환 |
| `RefreshNameList()` | 왼쪽 이름 버튼 리스트 갱신 |
| `RefreshChatDisplay()` | 오른쪽 대화 내용 갱신 (하이퍼링크, 시간, 날짜 구분선, URL) |
| `SelectConversation(name, noFocus)` | 대화 선택 + 안 읽은 수 초기화 + who 정보/메모 표시 |
| `DeleteConversation(name)` | 대화 삭제 + UI 갱신 |
| `SendWhoQuery(charName)` | `/who` 쿼리 전송 + 결과 파싱 (쿨다운 관리) |
| `CreateMainFrame()` | UI 전체 생성 (lazy, 최초 1회) |
| `ToggleMainFrame()` | 창 표시/숨기기 토글 |
| `OpenWhisperTo(name, fullName)` | 외부에서 일반 대화 열기 (채팅창 이름 클릭용) |
| `OpenBNetWhisperTo(bnID)` | 외부에서 BNet 대화 열기 |

### SetItemRef 후킹
- `player:이름` 링크의 좌클릭을 가로채 `OpenWhisperTo()` 호출
- `BNplayer:이름:bnID` 링크의 좌클릭을 가로채 `OpenBNetWhisperTo()` 호출
- `interceptWhisper` 옵션이 꺼져 있으면 가로채지 않음
- 우클릭 및 기타 링크는 `origSetItemRef`로 전달 (WoW 기본 동작 유지)

### ChatEdit_UpdateHeader 후킹
- WoW의 귓속말 모드 전환(`chatType == "WHISPER"`)을 감지하여 SimpleWhisper 창으로 리다이렉트
- 12.0+ (`ChatFrameUtil.ActivateChat`) / 이전 버전 (`ChatEdit_UpdateHeader`) 분기 처리

### 이벤트 처리
| 이벤트 | 처리 내용 |
|--------|-----------|
| `ADDON_LOADED` | DB 초기화, 대화/안읽음 복원, LDB 생성, 미니맵 버튼, 채팅 필터 등록, /who 필터, 슬래시 명령어 등록 |
| `PLAYER_LOGOUT` | 대화 내용 + 이름 목록 + 안 읽은 수 저장 |
| `CHAT_MSG_WHISPER` | 수신 메시지 기록, GUID/직업 저장, 소리 재생, 자동 열기 |
| `CHAT_MSG_WHISPER_INFORM` | 발신 메시지 기록, 채팅 표시 갱신 |
| `CHAT_MSG_BN_WHISPER` | BNet 수신 메시지 기록, 소리 재생, 자동 열기 |
| `CHAT_MSG_BN_WHISPER_INFORM` | BNet 발신 메시지 기록, 채팅 표시 갱신 |
| `CHAT_MSG_SYSTEM` | `/who` 결과 파싱, 오프라인 감지, 발송 실패 메시지 기록 |
| `CHAT_MSG_*` (채팅 캐시) | 일반/외침/길드/파티/공격대/전장 채팅에서 직업 정보 수집 |

## 호환성 처리
- **BackdropTemplate**: `BackdropTemplateMixin and "BackdropTemplate"` 패턴으로 9.0+ / 이전 버전 모두 지원
- **이름 정규화**: `Ambiguate(name, "none")`으로 서버명 제거, `fullName` 별도 보존하여 `SendChatMessage`에 사용
- **멀티 인터페이스 TOC**: 클래식(11508) ~ 리테일(120001) 6개 버전 동시 지원
- **플레이어 드롭다운**: `FriendsFrame_ShowDropdown` 존재 시 사용, 없으면 `ChatFrame_SendTell` 폴백
- **BNet API**: `C_BattleNet.GetAccountInfoByID` (Retail) / `BNGetFriendInfoByID` (Classic) 분기
- **BNet 발송**: `C_BattleNet.SendWhisper` / `BNSendWhisper` 분기
- **파티 초대**: `C_PartyInfo.InviteUnit` / `InviteUnit` 분기
- **리사이즈**: `SetResizeBounds` (10.0+) / `SetMinResize`+`SetMaxResize` (이전) 분기
- **채팅 후킹**: `ChatFrameUtil.ActivateChat` (12.0+) / `ChatEdit_UpdateHeader` (이전) 분기
- **/who API**: `C_FriendList.SendWho` / `SlashCmdList["WHO"]` 분기
- **직업 이름 매핑**: 영어/한글 직업명 → CLASS_TOKEN 테이블 별도 관리

## 수정 시 참고사항
- 단일 파일(`SimpleWhisper/SimpleWhisper.lua`) 구조이므로 모든 수정은 이 파일에서 이루어짐
- UI 프레임은 lazy 생성 — `CreateMainFrame()`이 호출되기 전까지 프레임 없음
- `nameButtons`는 동적 생성 (풀링 없음) — 대화 상대가 매우 많을 경우 성능 고려 필요
- `SetItemRef` 전역 덮어쓰기 방식 — 다른 애드온과 충돌 가능성 있음
- `ChatEdit_UpdateHeader` 후킹으로 모든 귓속말 경로를 가로채므로 관련 애드온과 충돌 가능성 있음
- `AddMessage`, `RefreshNameList`, `RefreshChatDisplay`, `SelectConversation`, `DeleteConversation`은 forward declare 패턴 사용 (함수 참조가 순서에 의존)
- `/who` 조회 시 `SetWhoToUI(false)`로 누구 목록 UI 표시를 억제하고 결과 수신 후 복원
- 채팅 캐시 프레임(`classCacheFrame`)은 애드온 로드 즉시 생성되어 채팅 이벤트를 상시 수집

## 릴리즈 절차

모든 애드온은 동일한 자동화 파이프라인을 사용한다:

1. TOC 파일에서 `## Version:` 버전 올리기
2. 커밋 → 태그(`v{버전}`) 푸시 → GitHub release 생성
3. GitHub Actions (`BigWigsMods/packager@v2`)가 CurseForge에 자동 업로드

**필요 파일 (각 애드온 레포):**
- `.github/workflows/release.yml` — 태그 푸시 트리거, `-g` 플래그로 대상 클라이언트 지정
- `.pkgmeta` — 패키지 이름, ignore 목록

**CurseForge API 키:** CurseForge 웹에서 페이로드 URL로 관리 (GitHub Secrets 불필요)

**TOC 파일명 규칙:** 클라이언트별 접미사 사용 (예: `_TBC.toc` = BCC 전용)

### 클라이언트 지정
| 애드온 | `-g` 플래그 | TOC |
|--------|-----------|-----|
| SimpleWhisper | `classic bcc wrath cata retail` | `SimpleWhisper.toc` + `_*.toc` (멀티) |
| SimpleRepu | `bcc` | `SimpleRepu_TBC.toc` (TBC 전용) |
| SimpleRepItem | `bcc` | `SimpleRepItem_TBC.toc` (TBC 전용) |

---

# SimpleRepu (평판 가이드) 애드온

## 개요
- **애드온**: SimpleRepu v1.0.0
- **대상 클라이언트**: WoW BCC (Interface: 20505)
- **용도**: TBC 평판과 관련 던전을 표시하는 가이드
- **제작 배경**: TBC 던전 평판 관계를 한눈에 파악하기 위해 제작

### 의존성
- **필수**: 없음 (독립 실행 가능, 라이브러리 번들 포함)
- **선택**: Arcana — 데이터 바에 평판 브로커 표시

### SavedVariables
- **`SimpleRepuDB`**: 설정 저장
  ```lua
  SimpleRepuDB = {
      minimapPos = 180,       -- 미니맵 버튼 각도
      popupPos = nil,         -- 팝업 창 위치 { point, relPoint, x, y }
  }
  ```

## 기능

### 미니맵 버튼
- WoW 내장 텍스처를 사용한 자체 미니맵 버튼 (LibDBIcon 미사용)
- 아이콘: `Achievement_Reputation_01` (평판 아이콘)
- 좌클릭/우클릭: 팝업 창 토글
- 우클릭 드래그: 미니맵 주변 위치 이동
- 마우스오버: 평판 현황 툴팁 표시

### 팝업 창
- `GameTooltipTemplate` 기반 — 툴팁과 동일한 형태
- 화면 중앙에 표시, 좌클릭 드래그로 이동 가능
- 위치 기억 (`SimpleRepuDB.popupPos`)
- ESC로 닫기 (`UISpecialFrames` 등록)

### 평판 표시
- 진영명: Blizzard API (`GetFactionInfoByID`) 사용 — 클라이언트 로캘 자동 적용
- 등급: 색상 코딩 (적대~확고한 동맹, 8단계)
- 수치: 현재 등급 내 진행량 / 최대값
- 던전: 만렙 미달 시 관련 던전 표시 (레이드는 빨간색, 일반 던전은 회색)
- 진영 필터링: 얼라이언스/호드 전용 진영 자동 필터

### LDB (LibDataBroker) 연동
- LDB data source 이름: `"SimpleRepu"`
- 아이콘: `Interface\Icons\Achievement_Reputation_01`
- 마우스오버: 팝업과 동일한 평판 툴팁
- 클릭: 팝업 창 토글

## 수록 진영 (11개)

### 던전 진영
| 진영 | Faction ID | 타입 | 관련 던전 |
|------|-----------|------|----------|
| Honor Hold | 946 | 얼라 전용 | 지옥불 성루, 피의 용광로, 으스러진 손의 전당 |
| Thrallmar | 947 | 호드 전용 | 지옥불 성루, 피의 용광로, 으스러진 손의 전당 |
| Cenarion Expedition | 942 | 공통 | 강제노역소, 지하수렁, 증기 저장고 |
| Lower City | 1011 | 공통 | 마나 무덤, 아키나이 납골당, 세데크 전당, 그림자 미궁 |
| The Sha'tar | 935 | 공통 | 메카나르, 식물원 |
| Keepers of Time | 989 | 공통 | 옛 힐스브래드 구릉지, 검은늪 |

### 샤트라스 진영
| 진영 | Faction ID | 비고 |
|------|-----------|------|
| The Aldor | 932 | 납품 아이템으로 평판 획득 |
| The Scryers | 934 | 납품 아이템으로 평판 획득 |

### 레이드 진영
| 진영 | Faction ID | 관련 레이드 |
|------|-----------|------------|
| Ashtongue Deathsworn | 1012 | 검은 사원 |
| The Scale of the Sands | 990 | 하이잘 산 전투 |
| The Violet Eye | 967 | 카라잔 |

## 코드 구조

### Data.lua
- `SR.FACTIONS` — 진영 목록 테이블
  ```lua
  { id = 946, name_en = "Honor Hold", alliance = true, dungeons = {
      { en = "Hellfire Ramparts", kr = "지옥불 성루" },
  }}
  ```
  - `id`: Blizzard 진영 ID (`GetFactionInfoByID`에 사용)
  - `name_en`: 영어 이름 (API 실패 시 폴백)
  - `alliance`/`horde`: 진영 필터 (없으면 공통)
  - `dungeons`: 관련 던전 목록, `raid = true`로 레이드 구분
- `SR.STANDING` — 평판 등급 라벨 (영어/한글)
- `SR.STANDING_COLORS` — 등급별 색상 (1=적대 ~ 8=확고한 동맹)

### Core.lua
| 함수/변수 | 역할 |
|----------|------|
| `L(en, kr)` | 로캘 헬퍼 (`koKR`이면 한글 반환) |
| `GetFactionData(factionID)` | Blizzard API로 평판 데이터 조회 |
| `IsFactionRelevant(factionData)` | 플레이어 진영에 맞는 팩션 필터 |
| `ShowRepTooltip(tooltip)` | 평판 정보 툴팁 렌더링 |
| `TogglePopup()` | 팝업 창 생성/토글 |

## 수정 시 참고사항
- 평판/던전 데이터 추가는 `Data.lua`의 `SR.FACTIONS` 테이블에 항목 추가
- 진영 이름은 API가 제공하므로 `name_en`은 폴백용 — `name_kr` 불필요
- 던전 이름은 API 미제공이므로 `en`/`kr` 수동 관리 필요
- UI/로직 수정은 `Core.lua`에서 이루어짐
- 팝업은 `GameTooltipTemplate` 기반 — 커스텀 프레임이 아님

---

# SimpleRepItem (평판 아이템) 애드온

## 개요
- **애드온**: SimpleRepItem v1.0.0
- **대상 클라이언트**: WoW BCC (Interface: 20505)
- **용도**: 평판 진영별 구매 가능 아이템 브라우저
- **제작 배경**: ATT(AllTheThings) 데이터를 파싱하여 평판 등급별 아이템을 한눈에 조회
- **데이터 출처**: AllTheThings (MIT License, https://github.com/ATTWoWAddon/AllTheThings)

### 의존성
- **필수**: 없음 (독립 실행 가능, 라이브러리 번들 포함)
- **선택**: Arcana — 데이터 바에 평판 아이템 브로커 표시

### SavedVariables (캐릭터별)
- **`SimpleRepItem_DB`** (`SavedVariablesPerCharacter`): 설정 저장
  ```lua
  SimpleRepItem_DB = {
      windowPos = { point, x, y },  -- 창 위치
      windowSize = { w, h },         -- 창 크기
      dividerX = 180,                -- 좌우 패널 구분선 위치
      minimapPos = 200,              -- 미니맵 버튼 각도
      opacity = 0.85,                -- 창 불투명도
  }
  ```

## 기능

### 슬래시 명령어
| 명령어 | 기능 |
|--------|------|
| `/sri` | 메인 창 열기/닫기 |

### UI 레이아웃 (520×420 팝업 창, 리사이즈 가능)
```
+--[Simple RepItem v1.0.0]-------------[X]--+
| [진영 검색...]  |                          |
+------------------+                          |
| ▶ 불타는 성전   | ▶ 우호적      보급 장교 울리크|
| ▶ 오리지널      | ▶ 존경        보급 장교 울리크|
|                  | ▶ 숭배        보급 장교 울리크|
|   Honor Hold     | ▼ 확고한 동맹  보급 장교 울리크|
|   Thrallmar      |   [아이콘] 아이템명1      |
|   Cenarion Exp.  |   [아이콘] 아이템명2      |
|   ...            |   [아이콘] 아이템명3      |
|                  |                          |
| 34개 진영        |                          |
+------------------+--------------------------+
```

- **왼쪽 패널**: 진영 트리 (폴더 접기/펼치기)
  - 최상위: 불타는 성전 / 오리지널
  - 불타는 성전: 하위 폴더 없이 진영 직접 나열 (SimpleRepu 순서)
  - 오리지널: Alliance / Horde / Neutral 하위 폴더
  - 폴더 클릭으로 ▶/▼ 토글
  - 검색창에서 진영명 필터링 (검색 시 자동 펼침)
  - 최초 실행 시 모든 폴더 닫힌 상태, 이후 세션 중 유지
- **오른쪽 패널**: 선택된 진영의 아이템 목록
  - 진영 선택 시 0.5초 로딩 텍스트 표시 후 아이템 목록 렌더링
  - 평판 등급별 접기/펼치기 헤더 (우호적/존경/숭배/확고한 동맹)
  - 헤더 좌측: 등급명 (WoW API `FACTION_STANDING_LABEL`, 로캘 자동 적용)
  - 헤더 우측: NPC 이름 (회색, `ns.NPC_NAMES` 테이블에서 로캘별 조회)
  - 같은 등급이라도 NPC가 다르면 별도 헤더로 구분
  - 헤더 클릭 시 해당 등급만 펼치고 나머지는 접힘
  - 진영 선택 시 모든 등급 접힌 상태로 시작
  - 아이템: 아이콘 + 이름 (품질 색상), 마우스오버 시 GameTooltip
  - Shift+클릭: 아이템 링크 채팅 삽입, Ctrl+클릭: 미리보기
- **리사이즈**: 우하단 핸들 (400×300 ~ 900×700)
- **닫기**: X 버튼 또는 ESC 키

### 미니맵 버튼
- 아이콘: `Achievement_Reputation_01` (평판 아이콘)
- 좌클릭: 창 토글
- 우클릭 드래그: 위치 이동
- 마우스오버: 제목 + 힌트 툴팁

### LDB (LibDataBroker) 연동
- LDB data source 이름: `"SimpleRepItem"`
- 텍스트: `"평판 아이템"` (koKR) / `"Quarter"` (기타)
- 좌클릭: 창 토글

## 파일 구성
```
SimpleRepItem/
├── SimpleRepItem_TBC.toc  -- TOC (BCC 전용, Interface: 20505)
├── SimpleRepItem.lua      -- 메인 UI + 로직 (~800줄)
├── Data.lua               -- ATT에서 파싱한 아이템 데이터
│                             34개 진영, 159개 NPC 엔트리
│                             ns.FACTION_NAMES: 폴백 진영 이름
│                             ns.NPC_NAMES: NPC 이름 (en/kr, 수동 관리)
│                             ns.TREE: 트리 구조
│                             ns.DATA: 아이템 데이터
│                             MIT License (AllTheThings)
├── parse_att.js           -- Node.js 파싱 스크립트 (데이터 재생성용)
├── npc_links.html         -- NPC 이름 검수용 링크 페이지 (개발용)
├── .github/workflows/release.yml -- 태그 푸시 → CurseForge 자동 업로드
├── .pkgmeta               -- 패키지 설정
├── CHANGELOG.md           -- 변경 이력
└── libs/                  -- LibStub, CallbackHandler, LDB
```

## 코드 구조 (SimpleRepItem.lua)

### 다국어 문자열 테이블 `L`
영어 기본값으로 초기화 후 `GetLocale() == "koKR"` 시 한글 오버라이드.

### 세션 데이터
| 변수 | 타입 | 용도 |
|------|------|------|
| `DATA` | `table` | `[factionID] = { {npcID, standing, items={...}}, ... }` |
| `FACTION_NAMES` | `table` | `[factionID] = "영어이름"` (API 폴백) |
| `NPC_NAMES` | `table` | `[npcID] = {en="영어", kr="한글"}` (수동 관리, 76개) |
| `TREE` | `table` | 왼쪽 패널 트리 구조 |
| `collapsed` | `table` | 왼쪽 트리 폴더 접기 상태 |
| `itemCollapsed` | `table` | 오른쪽 평판 등급 접기 상태 (`"factionID_npcID_standing"`) |
| `selectedFactionID` | `number/nil` | 현재 선택된 진영 |
| `searchText` | `string` | 검색 필터 텍스트 |

### 핵심 함수
| 함수 | 역할 |
|------|------|
| `GetFactionName(factionID)` | Blizzard API → 폴백 이름 순서로 진영명 해석 |
| `GetNPCName(npcID)` | `NPC_NAMES` 테이블에서 로캘별 NPC 이름 조회 |
| `StandingLabel(val)` | ATT 평판값(42000 등) → WoW standingID → `FACTION_STANDING_LABEL` |
| `StandingColor(val)` | 평판 등급별 색상 반환 |
| `BuildFactionList()` | TREE 구조에서 플랫 표시 리스트 생성 (검색 필터 적용) |
| `RefreshFactionList()` | 왼쪽 트리 패널 갱신 |
| `RefreshItemDisplay()` | 오른쪽 아이템 패널 갱신 (접기/펼치기 포함) |
| `SelectFaction(factionID)` | 진영 선택 + 아이템 접기 초기화 |
| `CreateMainFrame()` | UI 전체 생성 (lazy, 최초 1회) |
| `ToggleMainFrame()` | 창 표시/숨기기 토글 |
| `CreateMinimapButton()` | 미니맵 버튼 생성 |
| `CreateLDB()` | LDB 데이터 오브젝트 생성 |

### Standing 매핑
| ATT 값 | WoW standingID | 의미 |
|---------|---------------|------|
| 0 | 4 | 중립 |
| 3000 | 5 | 우호적 |
| 9000 | 6 | 존경 |
| 21000 | 7 | 숭배 |
| 42000 | 8 | 확고한 동맹 |
| 1~30 | — | 명성 레벨 (Renown) |

## 데이터 파싱 (parse_att.js)

### 사용법
```bash
node parse_att.js [ATT_PATH] [GAME_VERSION]
# 기본: node parse_att.js ../AllTheThings TBC
```

### 파싱 로직
1. ATT DB 파일 (Zones.lua, ExpansionFeatures.lua, Instances.lua 등) 로드
2. `n(NPC_ID, { minReputation=..., g={...} })` 패턴 탐색
3. 아이템별 `minReputation` 추출 (NPC 레벨 폴백)
4. 진영ID + 평판등급별 그룹핑
5. Lua 데이터 파일 출력 (폴백 이름 + 트리 구조 + MIT 라이선스 포함)

### 지원 아이템 타입
| ATT 함수 | 아이템ID 추출 |
|----------|--------------|
| `i(itemID)` | 첫 번째 인자 |
| `s(sourceID, itemID)` | 두 번째 인자 |
| `r(recipeID, {itemID=X})` | itemID 필드 |
| `mnt(spellID, {itemID=X})` | itemID 필드 |
| `toy(itemID)` | 첫 번째 인자 |
| `p(speciesID, {itemID=X})` | itemID 필드 |
| `heir(itemID)` | 첫 번째 인자 |

### minReputation 형식
- 인라인: `minReputation={factionID, standing}` (TBC/Classic)
- 참조: `minReputation=a[idx]` (Standard/Retail, lookup table)

## 수정 시 참고사항
- 아이템 데이터 업데이트: `node parse_att.js`로 Data.lua의 `ns.DATA` 재생성
- Data.lua의 `ns.DATA`, `ns.FACTION_NAMES`, `ns.TREE`는 parse_att.js가 자동 생성
- **`ns.NPC_NAMES`는 수동 관리** — parse_att.js 재생성 시 별도 보존 필요
  - NPC 이름 검수: `npc_links.html`을 브라우저에서 열어 Wowhead 한글 페이지와 대조
  - 새 NPC 추가 시: Data.lua에서 npcID 확인 후 `ns.NPC_NAMES`에 `{en="", kr=""}` 추가
- 진영 추가/순서 변경: parse_att.js 내 `ns.TREE` 및 `ns.FACTION_NAMES` 수정
- UI/로직 수정: SimpleRepItem.lua
- TBC 클라이언트 전용 — `C_Item` API 미사용, `GetItemInfo` 직접 호출
- 아이템 캐시 미보유 시 툴팁/아이콘 불완전 표시 (TBC API 제한)
- `GET_ITEM_INFO_RECEIVED` 이벤트로 아이템 데이터 로딩 후 자동 갱신

## 릴리즈 절차
- 태그 푸시 → GitHub Actions → BigWigsMods/packager → CurseForge 자동 업로드
- 수동 배포: zip 압축
  ```powershell
  Compress-Archive -Path 'SimpleRepItem' -DestinationPath 'SimpleRepItem-{버전}.zip' -Force
  ```
