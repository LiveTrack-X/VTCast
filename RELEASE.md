# VTCast 릴리즈 절차

VTCast 데스크톱 앱의 Windows 설치형, 포터블 실행 파일, 서명된 인앱 업데이트
메타데이터를 함께 발행하는 절차입니다.

## 자동업데이트 채널

- 엔드포인트:
  `https://github.com/LiveTrack-X/VTCast/releases/latest/download/latest.json`
- 앱에 포함되는 공개키:
  `crates/sender-app/tauri.conf.json`의 `plugins.updater.pubkey`
- GitHub Actions 비밀:
  `TAURI_SIGNING_PRIVATE_KEY`, `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`
- 설치 모드: NSIS `currentUser`(관리자 권한 불필요)

LiveTrack-X 전용 키페어는 발급 완료됐습니다. 개인키는 저장소에 커밋하지 않으며,
현재 로컬 작업 환경에서는 다음 무시된 파일에만 보관합니다.

- `dist/livetrack-updater.key`
- `dist/livetrack-updater.password.dpapi`

두 파일을 함께 안전한 비밀 저장소에 백업해야 합니다. DPAPI 암호 파일은 생성한
Windows 계정과 컴퓨터에서만 복호화할 수 있습니다. 개인키와 암호를 모두 잃으면
이미 배포된 앱이 신뢰하는 키로 새 업데이트에 서명할 수 없습니다.

## 핵심 규칙

- `Cargo.toml`과 `crates/sender-app/tauri.conf.json`의 버전은 항상 같습니다.
- 자동업데이트는 현재 앱보다 높은 버전에만 표시됩니다.
- 개인키, 암호, 복호화된 암호를 커밋하거나 로그에 출력하지 않습니다.
- `latest.json`은 직접 편집하지 않습니다. GitHub Actions가 서명 산출물과 일치하게
  생성합니다.
- `v1.0.6` 이하는 이전 공개키를 포함하므로 `v1.0.7`을 한 번 수동 설치해야 합니다.

## 정식 릴리스

예를 들어 `1.0.8`을 발행할 때:

1. `Cargo.toml`의 `[workspace.package].version`을 `1.0.8`로 변경합니다.
2. `crates/sender-app/tauri.conf.json`의 `version`을 `1.0.8`로 변경합니다.
3. 테스트하고 변경을 PR로 병합합니다.
4. 병합된 `main` 커밋에 태그를 만들어 푸시합니다.

```powershell
git switch main
git pull --ff-only origin main
git tag -a v1.0.8 -m "VTCast 1.0.8"
git push origin v1.0.8
```

`.github/workflows/release.yml`이 태그와 소스 버전의 일치를 확인한 뒤 다음을
자동으로 처리합니다.

1. Windows 릴리스 빌드
2. LiveTrack-X 개인키로 NSIS 설치 파일 서명
3. 초안 GitHub Release 생성
4. 설치형, 서명, `latest.json` 업로드
5. 포터블 EXE 추가
6. 모든 자산이 준비된 뒤 Release 공개

주요 자산:

- `VTCast_<VER>_x64-setup.exe`
- `VTCast_<VER>_x64-setup.exe.sig`
- `VTCast_<VER>_LiveTrack-X_x64_Portable.exe`
- `latest.json`

## 로컬 서명 빌드

GitHub Actions 전에 같은 키로 NSIS 산출물을 확인할 때만 사용합니다. 아래 명령은
복호화된 암호를 출력하지 않으며, 종료 시 서명 환경변수를 제거합니다.

```powershell
$securePassword = Get-Content -Raw "dist/livetrack-updater.password.dpapi" |
  ConvertTo-SecureString
$credential = [pscredential]::new("VTCast updater", $securePassword)
$plainPassword = $credential.GetNetworkCredential().Password

try {
  $env:TAURI_SIGNING_PRIVATE_KEY = Get-Content -Raw "dist/livetrack-updater.key"
  $env:TAURI_SIGNING_PRIVATE_KEY_PASSWORD = $plainPassword
  npx --yes @tauri-apps/cli@2 build `
    --config crates/sender-app/tauri.conf.json `
    --bundles nsis `
    --ci
} finally {
  Remove-Item Env:TAURI_SIGNING_PRIVATE_KEY -ErrorAction SilentlyContinue
  Remove-Item Env:TAURI_SIGNING_PRIVATE_KEY_PASSWORD -ErrorAction SilentlyContinue
  $plainPassword = $null
}
```

로컬 산출물은 `target/release/bundle/nsis/`에 생성됩니다.
`createUpdaterArtifacts: true`인 Tauri v2 채널은 NSIS 설치 EXE 자체를 업데이트
번들로 재사용하고 같은 이름에 `.sig`를 붙인 서명 파일을 생성합니다.

## 발행 후 검증

```powershell
$latest = Invoke-RestMethod `
  "https://github.com/LiveTrack-X/VTCast/releases/latest/download/latest.json"
$latest.version
$latest.platforms.PSObject.Properties |
  ForEach-Object { $_.Value.url }

gh release view "v$($latest.version)" `
  --repo LiveTrack-X/VTCast `
  --json url,assets
```

확인할 항목:

- `latest.json`의 버전이 태그와 같은가
- Windows 플랫폼 URL이 같은 Release의 NSIS 설치 EXE를 가리키는가
- `signature`가 비어 있지 않은가
- 설치형과 포터블 EXE가 모두 다운로드 가능한가
- 새 버전보다 낮은 버전의 앱에서 업데이트 배너가 표시되는가

## 릴레이 배포는 별도

`crates/relay/` 또는 수신 페이지 변경은 데스크톱 앱 Release만으로
`vtcast.livetrack.kr` 서버에 배포되지 않습니다. 공개 릴레이의 재빌드와
재배포는 별도로 수행해야 합니다.
