# ESM Interruptibility 앱 PRD

## 문서 상태

- 상태: 초안(Draft)
- 최초 정리일: 2026-08-05
- 실험 프로토콜 및 세부 수치는 연구팀 확인에 따라 변경될 수 있음
- 원문에서 `x` 또는 `확인 필요`로 표시된 항목은 이 문서에서 `TBD`로 표기함
- 요구사항과 구현 현황을 분리하여 관리함

## 1. 제품 개요

### 1.1 목적

실험 참여자의 스마트폰에서 정해진 시간에 ESM(Ecological Momentary Assessment) 세션을 실행하고 질문, 음성 응답, 오디오 및 주변 정보를 수집하여 서버에 안전하게 업로드한다.

### 1.2 사용자 및 기기 규모

- 대상: 총 50가구
- 가구당 기기: 3~5대
- 예상 기기 수: 150~250대
- 최대 운영 목표: 약 250대
- 실험 기간: 28일(4주)

### 1.3 이번 버전의 제외 범위

- TBD: 현재 명시적으로 제외하기로 확정된 기능이 없음

## 2. 핵심 기능 요구사항

| ID | 기능 | 요구사항 |
| --- | --- | --- |
| FR-001 | 기기 인증 | Supabase Auth의 기기별 이메일과 비밀번호를 사용해 인증한다. |
| FR-002 | 실험 설정 조회 | 인증된 기기에 해당하는 Participant 및 실험 설정을 서버에서 조회한다. |
| FR-003 | 다중 기기 동기화 | 같은 Participant에 속한 여러 기기가 동일한 ESM 예정 시각을 사용한다. |
| FR-004 | 로컬 스케줄링 | 서버 설정을 바탕으로 앱이 다음 ESM 시각을 계산하고 로컬에서 실행한다. |
| FR-005 | 세션 실행 | 예정 시각에 질문 재생, 응답 녹음, 판정 및 세션 종료 절차를 수행한다. |
| FR-006 | 오디오 경로 | 질문 재생과 녹음에 Jabra 오디오 경로를 사용한다. |
| FR-007 | 응답 판정 | 녹음 파일을 OpenAI transcription으로 변환하고 의미 있는 응답인지 판정한다. |
| FR-008 | 오프라인 저장 | 세션 데이터와 업로드 상태를 원격 요청 전에 로컬 영속 저장소에 보존한다. |
| FR-009 | 서버 업로드 | 세션 ZIP은 Supabase Storage에, 세션 레코드는 Database에 저장한다. |
| FR-010 | 순차 재시도 | 업로드 실패 데이터는 로컬 Queue에 유지하고 연결 복구 후 생성 순서대로 하나씩 업로드한다. |
| FR-011 | Heartbeat | 최대 5분 간격으로 기기 상태를 서버에 갱신한다. |
| FR-012 | 재부팅 복구 | 기기 재부팅 후 Foreground Service, ESM 스케줄 및 Heartbeat 실행을 복구한다. |
| FR-013 | 놓친 세션 처리 | 이미 예정 시각이 지난 세션은 즉시 실행하지 않고 폐기한다. |
| FR-014 | 실험 전 테스트 | 실험 시작일 전에도 Test Mode로 스케줄링부터 Heartbeat까지 전체 흐름을 검증할 수 있어야 한다. |

## 3. 관리자 요구사항

관리자 기능의 구체적인 화면과 권한 정책은 별도 설계가 필요하다.

- 기기 등록
- Participant별 Operating Hours 설정
- 마지막 세션 시각을 기준으로 다음 ESM 랜덤 시각 계산 및 확인
- 기기 상태 확인
- Heartbeat 확인
- Session 업로드 확인
- 오류 Alert 확인
- 수동 테스트 실행 여부는 TBD

## 4. 실험 프로토콜

### 4.1 실험 기간과 운영시간

- 실험 기간: 28일
- Operating Hours: Participant별 설정
- 일반적인 예시: 09:00~22:00
- 평일과 주말은 Participant 요청에 따라 별도 설정할 필요가 있음
- 시간대: TBD. 현재 개발 기본값은 `Asia/Seoul`

### 4.2 최초 트리거

- 실험 시작 후 첫 Operating Hours 안에서 랜덤 시각으로 계산한다.
- 정확한 최초 계산 규칙은 TBD다.

### 4.3 다음 트리거 간격

- 간격 유형: 랜덤
- 최소 간격: 30분
- 최대 간격: 90분
- 목표 평균: 60분
- 확률 분포: TBD
- 기준 시각: 같은 Participant의 여러 기기 중 가장 최근 Session Start Time

### 4.4 Operating Hours 이월 규칙

운영시간 종료 전까지 경과한 시간을 랜덤 간격에서 차감하고, 남은 시간을 다음 운영일 시작 시점부터 이어서 계산한다.

원문 예시:

- 마지막 트리거: 21:40
- 운영 종료: 22:00
- 당일 경과 처리: 20분
- 다음 날 운영 시작: 09:00
- 원문에 기재된 다음 시작 범위: 09:10~10:00

예시의 최대 시각은 계산상 10:10이 될 수 있어 연구팀 확인이 필요하다.

### 4.5 다중 기기 동기화

- 같은 Participant의 모든 기기는 동시에 울려야 한다.
- 공유하는 것은 동일한 예정 ESM 시각이다.
- 세션 시작 후 각 기기의 녹음, 전사, 완료 및 업로드는 독립적으로 처리한다.
- 기기 간 허용 시간 오차는 TBD다.
- 실제 Session Start Time 중 어느 값을 다음 스케줄 기준으로 확정하는지 서버 조정 규칙이 필요하다.

### 4.6 세션 전 주변정보 수집

- 시작 시각: ESM 예정 시각 2분 전
- 수집 대상:
  - 주변 오디오
  - 조도 센서
  - 가속도 센서
- 센서 샘플링 주기와 오디오 경로는 TBD다.

### 4.7 질문과 응답

- 질문 수: Q1, Q2 총 2개
- 질문별 녹음 Window: 약 8초
- 질문별 최대 시도 횟수: 총 2회로 해석하되 확인 필요
- 재시도 간격: 8초
- 질문 음원 또는 TTS 사용 여부: TBD
- 질문 및 녹음은 Jabra 오디오 경로를 사용한다.

### 4.8 응답 판정과 중단

- 녹음된 응답을 OpenAI transcription으로 한국어 전사한다.
- 전사 결과가 의미 있는 응답인지 로컬 또는 명시된 규칙으로 판정한다.
- 무음이나 사용할 수 없는 전사는 유효 응답으로 처리하지 않는다.
- 중단 문구를 감지하면 중단 이유를 묻고 현재 세션을 종료한다.
- 중단 문구 목록, 이유 질문 방식 및 최종 결과 코드는 TBD다.

## 5. 기능별 상세 명세

### 5.1 스케줄링

- 시작 조건: 인증 완료, 활성 실험 설정 존재, 실험 기간 및 Operating Hours 유효
- 종료 조건: 예정 세션을 시작했거나, 실험 종료 또는 설정 비활성화
- 생성 데이터: 다음 예정 시각, 계산에 사용한 기준 시각과 랜덤 간격
- 실패 처리: 오류 로그 저장 후 설정 재조회 또는 Alarm 재등록
- 앱 종료 후 복구: Foreground Service와 AlarmManager 사용
- 재부팅 후 복구: Boot Receiver에서 설정과 다음 Alarm을 재등록
- 화면 표시: 다음 ESM 예정 시각과 스케줄 상태

### 5.2 세션 녹음

- 시작 조건: 세션 활성화, 오디오 권한 허용, 요구된 Jabra 경로 확인
- 종료 조건: 녹음 Window 종료, 유효 응답, 최대 시도 소진, 사용자 중단 또는 기술 오류
- 생성 파일: 질문별 응답 오디오 파일 및 세션 전 주변 오디오 파일
- 실패 처리: 오류를 세션 메타데이터와 로컬 로그에 기록하고 기술 실패 여부를 판정
- 앱 종료 후 복구: 진행 중 녹음은 복구할 수 없으므로 불완전 세션 처리 규칙 필요
- 화면 표시: `q1_recording`, `q2_recording` 등 현재 단계

### 5.3 Session ZIP 생성

- 시작 조건: 세션이 최종 결과에 도달하고 필수 로컬 파일 기록이 완료됨
- 종료 조건: ZIP 파일 생성과 무결성 확인 완료
- 생성 파일: Session ZIP File
- 실패 처리: 원본 파일을 삭제하지 않고 ZIP 생성 작업을 재시도
- 앱 종료 후 복구: 영속 Queue에서 ZIP 미생성 세션을 조회하여 재시도
- 화면 표시: 세션 보관 또는 업로드 대기 상태

### 5.4 업로드

- 시작 조건: 로컬 Session ZIP과 Session Record가 준비되고 네트워크 사용 가능
- 종료 조건:
  - Supabase Storage ZIP 업로드 확인
  - Database Session Record 저장 확인
  - 로컬 업로드 상태 갱신
- 원격 저장:
  - Supabase Storage: Audio 및 Session Data ZIP
  - Supabase Database: Session Record
- 실패 처리: 로컬 Queue에 유지
- 재시도: 연결 복구 후 생성 순서대로 한 건씩 업로드
- 앱 종료 후 복구: WorkManager 또는 영속 Queue에서 미전송 세션 조회
- 화면 표시: `Uploaded` 또는 `Not uploaded`

### 5.5 Heartbeat

- 시작 조건: 설치 및 인증 완료 후 Foreground Service 정상 실행
- 종료 조건: 앱 제거 또는 실험 종료 후 기능 비활성화
- 실행 목표 주기: 5분
- 실행 방식: AlarmManager 기반 예정
- 생성 파일: 없음
- 서버 데이터:
  - `device_id`
  - `heartbeat_timestamp`
  - `battery_level`
  - `usb_connected`
- 실패 처리: 로컬 로그 기록 및 다음 주기 재시도. 장기 실패 정책은 TBD
- 재부팅 복구: AlarmManager와 Boot Receiver를 통해 재등록
- 화면 표시: 최근 Heartbeat 성공 상태를 Recent Logs에 표시

## 6. 데이터 명세

### 6.1 로컬 세션 레코드 최소 필드

- `session_id`
- `participant_id`
- `device_id`
- `scheduled_at`
- `started_at`
- `ended_at`
- `outcome`
- `archive_path`
- `transcript`
- `upload_status`
- `created_at`
- `updated_at`

### 6.2 Session ZIP 예상 구성

- 질문별 Audio
- 주변 Audio
- Transcript
- Session Metadata
- 이벤트 및 센서 Timestamp
- Participant, Device 및 Experiment 식별정보
- 조도 센서 데이터
- 가속도 센서 데이터
- 오류 및 실행 결과 정보

정확한 디렉터리 구조와 JSON 스키마는 TBD다.

### 6.3 식별자 표시 규칙

- Participant 표시명: `p1`, `p2`, ...
- Device 표시명: `p1_1`, `p1_2`, `p1_3`, ...
- Database 내부 기본키 형식: TBD
- Auth user와 Device 레코드 매핑 규칙: TBD

### 6.4 아직 결정되지 않은 데이터 항목

- 오디오 형식과 품질
- 파일 이름 규칙
- 필수 타임스탬프 목록과 시간 정밀도
- 서버 검색 및 인덱싱 항목
- 데이터 보존 기간과 로컬 삭제 기준
- 기존 결과 데이터 샘플

## 7. 화면 요구사항

### 7.1 참가자 앱

- Device 이름
- 서버 연결 상태
- 현재 Session 진행 상태
- 마지막 업로드 상태
- 마지막 서버 업데이트 상태
- Volume knob
- Recent Logs
- Recent Logs 최대 10줄 표시

### 7.2 관리자 대시보드

- 화면 구성: TBD
- 조회 및 변경 가능 항목: TBD
- 오류와 기기 상태 표시 방식: TBD
- 수동 테스트 버튼 필요 여부: 상충 요구사항 확인 필요

## 8. API 및 서버 요구사항

### 8.1 인증

- Supabase Auth 사용
- 물리 기기별 이메일과 비밀번호 사용

### 8.2 세션 업로드

- ZIP 업로드: Supabase Storage
- Session Record: Supabase Database
- 요청 및 응답 스키마: TBD
- 중복 업로드 방지 방식: TBD
- 실패 코드와 재시도 조건: TBD

### 8.3 Heartbeat

요청 데이터:

```json
{
  "device_id": "TBD",
  "timestamp": "TBD",
  "battery_level": 0,
  "usb_connected": false
}
```

Heartbeat를 이력으로 누적할지, 기기별 최신 행을 갱신할지는 TBD다.

## 9. 인수 조건 및 테스트 시나리오

| ID | 시나리오 | 인수 조건 |
| --- | --- | --- |
| AC-001 | 오프라인 세션 | 인터넷 없이 세션을 완료해도 ZIP과 Queue가 보존된다. |
| AC-002 | 연결 복구 | 연결 복구 후 대기 세션이 생성 순서대로 한 건씩 업로드된다. |
| AC-003 | 재부팅 복구 | Boot 완료 후 Foreground Service, ESM Alarm 및 Heartbeat가 복구된다. |
| AC-004 | 놓친 세션 | 재부팅 또는 장시간 종료 중 놓친 과거 세션을 즉시 실행하지 않는다. |
| AC-005 | 중복 방지 | 동일 Session ZIP을 재시도해도 원격에 중복 데이터가 생성되지 않는다. |
| AC-006 | 장기 동작 | 앱은 24시간 이상 동작하되 세션은 Operating Hours 안에서만 실행한다. |
| AC-007 | 다중 기기 | 같은 Participant의 기기가 합의된 허용 오차 안에서 동일 예정 시각에 시작한다. |
| AC-008 | 서버 장애 | 서버 장애 중 데이터가 유실되지 않고 복구 후 순차 업로드된다. |
| AC-009 | 실험 전 테스트 | Test Mode에서 스케줄링, 녹음, ZIP, 업로드 및 Heartbeat를 검증할 수 있다. |
| AC-010 | Jabra 분리 | Jabra 분리 시의 세션 결과와 Alert 정책에 따라 처리된다. 세부 기준은 TBD다. |
| AC-011 | 관리자 확인 | 관리자가 기기 상태, Heartbeat, Session 및 오류를 확인할 수 있다. |

## 10. 확인이 필요한 결정 사항

우선순위가 높은 미결정 사항부터 확정한다.

1. 최초 트리거 계산 규칙
2. 랜덤 간격의 정확한 확률 분포와 난수 시드 공유 방식
3. Operating Hours 이월 예시의 최대 시각이 10:00인지 10:10인지
4. 다중 기기 간 허용 시간 오차
5. 여러 기기의 실제 시작 시각 중 다음 스케줄 기준을 정하는 방식
6. 평일, 주말 및 요일별 Operating Hours 데이터 구조
7. Jabra가 없을 때 세션 실패, 대기 또는 폰 오디오 대체 여부
8. 질문별 최대 2회가 총 시도 2회인지 재시도 2회인지
9. 중단 문구 목록과 중단 이유 질문 흐름
10. 오디오 형식, bitrate, sample rate 및 파일명 규칙
11. ZIP 내부 디렉터리와 JSON 스키마
12. Session outcome 및 workflow 상태 정의
13. 중복 업로드 방지를 위한 idempotency 규칙
14. 로컬 및 서버 데이터 보존·삭제 정책
15. Heartbeat 저장 방식과 실패 Alert 임계값
16. 관리자 수동 테스트 기능과 참가자 Test Mode의 구분
17. 관리자 대시보드의 최소 화면과 권한 범위

## 11. 현재 구현 체크포인트

2026-08-05 기준:

- [x] Android Kotlin 기본 프로젝트
- [x] XML 기반 최소 화면
- [x] 마이크 권한 요청
- [x] `MediaRecorder` 기반 M4A 녹음
- [x] 앱 내부 저장소에 녹음 파일 저장
- [x] FastAPI `/transcribe` multipart 업로드
- [x] OpenAI transcription 결과 수신 및 화면 표시
- [ ] Room 기반 로컬 Session 저장
- [ ] Session ZIP 생성
- [ ] 영속 업로드 Queue 및 순차 재시도
- [ ] Device 인증과 설정 조회
- [ ] 로컬 ESM 스케줄링
- [ ] 다중 기기 시간 동기화
- [ ] 질문 재생 및 Q1/Q2 상태 머신
- [ ] 주변정보 수집
- [ ] Foreground Service
- [ ] Heartbeat
- [ ] Boot Receiver 복구
- [ ] Supabase Storage 및 Database 업로드
- [ ] 관리자 대시보드
