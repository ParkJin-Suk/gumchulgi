# SoftReel — 광과민성(PSE) 보호 숏폼 플랫폼 MVP

영상을 올리면 서버가 광과민성 발작 유발 자극(플래시·적색·패턴·컷)을 검출하고
보정본을 만든다. 앱은 세로 스와이프 피드에서 원본/보정본을 재생하며,
사용자의 하루 위험 노출량을 대시보드로 보여준다.

## 저장소 구조

```
app/                 Flutter 앱 (Android / iOS / web)
server/
  app/               FastAPI — 인증, 업로드, 피드, 스트리밍, 대시보드, 관리자, 웹 스튜디오
  worker/            큐 워커 — ffmpeg 정규화 → 검출 → 보정 사다리
  webstudio/         /studio 로 서빙되는 정적 웹 스튜디오
psepipe_v3_seam/     검출기·필터 (현행 검출기는 이 폴더 하나)
deploy/azure/        Azure VM + Flutter 웹 HTTPS 배포
scripts/             실행·점검 스크립트 (run_api, run_worker, seed_admin_demo, check_env, check_setup)
docs/                설계 스펙, 실행 가이드, 로드맵, 검출기 명세서
legacy_detectors/    구세대 검출기 보관소 — 비교·회귀용, 수정 금지
research/            서버가 쓰지 않는 실험·검증 자료
  harness/ results/ outputs/   검출기 실험 스크립트와 결과
  pse_deflicker.py, All-In-One-Deflicker/   Blind Deflickering 실험
  refs/                        참고 논문 PDF
blazebvd-training/   BlazeBVD 학습 코드 (루트의 src/, tests/, configs/ 와 중복 — 정리 예정)
```

### 무엇이 서버에서 실제로 쓰이나
서버 컨테이너는 `server/` 와 `psepipe_v3_seam/` 만 복사해 실행한다
(`server/Dockerfile`). 그중 서버가 직접·간접으로 import 하는 검출기 모듈은 아래 9개다.

`impact` `pse_bt1702` `pse_cut` `pse_pattern` `psecore` `pseenv` `psegpu_full` `pselive3` `rawmeasure`

`psepipe_v3_seam/` 의 나머지 모듈(`psepipe`, `tier`, `pse_migraine`, `pse_comfort` 등)과
`validation/` 은 검증·연구용 도구다. 서로 import 하므로 폴더 안에서 함께 둔다.

## 빠른 시작 (노트북 = 서버, 폰 = 클라이언트)

### 준비 (1회)
- Python 3.10+, `ffmpeg`/`ffprobe` PATH 등록, Flutter SDK
- CUDA torch가 있으면 워커가 자동으로 GPU 필터(`psegpu_full`)를 쓰고, 없으면 CPU(`pselive3`)로 떨어진다 — 동작은 같고 느릴 뿐.

```powershell
pip install -r server\requirements.txt
copy server\.env.example server\.env      # JWT_SECRET 을 긴 임의 문자열로 변경
```

폰에서 접속하려면 방화벽 8000 포트를 열어야 한다 (관리자 PowerShell):

```powershell
netsh advfirewall firewall add rule name="gumchulgi-api" dir=in action=allow protocol=TCP localport=8000
```

### 서버 실행

한 번에 (API + 워커, `server\.env` 로드, 데이터는 `data\`):

```powershell
powershell -ExecutionPolicy Bypass -File run_server.ps1
```

또는 터미널 2개로 따로 (`scripts\run_api.ps1`, `scripts\run_worker.ps1`).
`http://localhost:8000/health` 가 `{"ok":true}` 면 준비 완료.

### 앱 빌드 → 폰 설치

서버 주소가 APK 안에 박히므로 노트북 IP(`ipconfig` → IPv4)를 넣어 빌드한다.
폰과 노트북은 **같은 와이파이**여야 하고, IP가 바뀌면 다시 빌드해야 한다.

```powershell
cd app
flutter pub get
flutter build apk --dart-define=API_BASE=http://192.168.0.7:8000
```

`app\build\app\outputs\flutter-apk\app-release.apk` 를 폰에 복사해 설치.
에뮬레이터는 `--dart-define` 없이 빌드하면 기본값 `http://10.0.2.2:8000` 으로 호스트에 붙는다.

> 앱의 `linux/`, `windows/`, `macos/` 스캐폴드는 저장소에서 제거했다 (빌드 대상 아님).
> 필요하면 `cd app && flutter create --platforms=windows,linux,macos .` 로 다시 만든다.

### 사용 흐름
1. 회원가입 → 업로드 탭에서 갤러리 영상 선택(500MB·3분 제한) → 업로드
2. 워커가 검출·보정을 끝내면 홈 피드와 내 페이지에 표시 (홈 아이콘 재탭 / 당겨서 새로고침)
3. 화면 탭 = 일시정지/재생. 우상단 필터 아이콘 → **필터 기능 켜기** 토글로 보던 자리에서 원본↔보정본 전환
4. 위험 영상 원본 시청이 하루 예산의 80%를 넘고 필터가 꺼져 있으면 경고 배너 → 대시보드에서 노출 추이 확인

## 데이터 위치
`data\db.sqlite3` (계정·영상 메타·시청 기록) + `data\media\{id}\` (원본/보정본/썸네일/리포트).
서버를 꺼도 유지되며, 초기화하려면 `data\` 폴더를 지운다. `.env` 와 `data\` 는 커밋하지 않는다.

## 테스트
```powershell
cd server; python -m pytest tests -q
cd app;    flutter analyze; flutter test
```
CI(`.github/workflows/deploy-azure.yml`)가 서버 테스트를 돌린 뒤 배포한다.

## 연구·검증 도구 (선택)

서버·앱과 무관하며 검출기를 실험·검증할 때만 쓴다.

- `research/harness/` — 검출기 비교·튜닝 스크립트. `psepipe_v3_seam` 모듈을 import 하므로
  `PYTHONPATH=psepipe_v3_seam` 을 잡고 실행한다. 결과는 `research/results/`.
- `psepipe_v3_seam/validation/` — 검증 기록과 GPU 하네스(`gpu_harness/`), AI 이식 PoC(`ai_poc/`).
- `scripts/check_env.py` — 측정을 시작해도 되는 환경인지 점검 (`PYTHONPATH=psepipe_v3_seam` 필요).
- `research/pse_deflicker.py` — Blind Deflickering 실행 + 재검증. 현재 작성자 PC 경로가 코드에 하드코딩돼 있어 그대로는 다른 PC에서 동작하지 않는다.

## 더 읽기
- `docs/superpowers/specs/2026-08-20-platform-mvp-design.md` — MVP 설계 스펙(노출 규칙, API)
- `docs/아키텍처-개요.md`, `docs/검출기-통합.md` — 구조와 검출기 통합 경위 (현행 검출기 근거)
- `docs/노트북-서버-실행.md`, `docs/GPU-노트북-실행-가이드.md` — GPU 노트북 실행 상세
- `server/README.md` — Docker / EC2 배포(CPU 경로)
- `deploy/azure/README.md` — Azure VM + Flutter 웹 HTTPS 배포(CPU 경로)
- `docs/검출기_명세서.pdf` — 검출기 명세
- `legacy_detectors/README.md` — 구세대 검출기를 남겨둔 이유

## 브랜치 규칙
<!-- TODO(팀 합의 필요): 아래는 제안입니다. 팀에서 정한 규칙으로 바꿔주세요. -->
`main` 은 서버·앱 배포 기준 통합본이다. 큰 구조 변경이나 대량 삭제는
브랜치에서 작업하고 PR 로 합치는 것을 권장한다.
