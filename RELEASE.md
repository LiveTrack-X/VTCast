# VTCast 릴리즈 절차

VTCast 데스크톱 앱(`vtcast.exe`, Tauri)의 새 버전을 내고 **인앱 자동 업데이트**가
기존 사용자에게 전달되도록 하는 절차. Claude에게 "X.Y.Z 릴리즈해줘"라고 시키면
이 문서대로 실행한다.

> 예시는 `1.0.2`를 낸다고 가정. `<VER>` = `1.0.2`, 태그 = `v1.0.2`.

---

## 0. 사전 준비

- **LiveTrack-X 업데이터 서명 키페어는 아직 발급하지 않음.**
  - 키를 준비하기 전에는 포터블 EXE만 Release에 올리고 `latest.json`은 발행하지 않는다.
  - 자동업데이트를 활성화할 때 `cargo tauri signer generate`로 전용 키를 생성하고,
    `crates/sender-app/tauri.conf.json`의 `plugins.updater.pubkey`를 새 공개키로 교체한다.
  - 개인키는 `dist/vtcast-updater.key`에 두고 **절대 커밋하거나 유출하지 않는다**.
    이 키를 잃으면 해당 공개키를 탑재한 앱의 자동업데이트가 불가능해진다.
- 업데이터 엔드포인트(앱에 박힘): `https://github.com/LiveTrack-X/VTCast/releases/latest/download/latest.json`
- `gh` CLI 인증 상태를 확인한 뒤 릴리스를 생성한다. CLI를 사용할 수 없는 환경에서만
  아래 REST API 예시를 대체 경로로 사용한다.

## 핵심 규칙
- 자동 업데이트는 **더 높은 버전**에만 뜬다. 같은 버전이면 안 뜸. 항상 버전을 올릴 것.
- 매 릴리즈에 **설치파일 + `latest.json` 둘 다** 업로드해야 한다. 빠지면 업데이트 안 됨.
- NSIS 설치 모드는 `currentUser`(관리자 권한 불필요) — 업데이트가 매끄럽게 됨.

---

## 1. 버전 올리기
두 곳을 동일 버전으로:
- `Cargo.toml` (루트) → `[workspace.package]`의 `version = "<VER>"`
- `crates/sender-app/tauri.conf.json` → `"version": "<VER>"`

## 2. 서명 환경변수 + 빌드
```bash
export TAURI_SIGNING_PRIVATE_KEY="$(cat dist/vtcast-updater.key)"
export TAURI_SIGNING_PRIVATE_KEY_PASSWORD=""   # 키에 비번 없음
cargo tauri build
```
산출물:
- `target/release/bundle/nsis/VTCast_<VER>_x64-setup.exe`
- `target/release/bundle/nsis/VTCast_<VER>_x64-setup.exe.sig`  (서명)

## 3. `latest.json` 작성
`signature`는 위 `.sig` 파일 **내용 전체**, `url`은 릴리즈 다운로드 URL.
```bash
SIG=$(cat target/release/bundle/nsis/VTCast_<VER>_x64-setup.exe.sig)
PUBDATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
cat > dist/latest.json <<EOF
{
  "version": "<VER>",
  "notes": "<릴리즈 요약 한 줄>",
  "pub_date": "$PUBDATE",
  "platforms": {
    "windows-x86_64": {
      "signature": "$SIG",
      "url": "https://github.com/LiveTrack-X/VTCast/releases/download/v<VER>/VTCast_<VER>_x64-setup.exe"
    }
  }
}
EOF
```

## 4. 버전 범프 커밋
```bash
git add Cargo.toml Cargo.lock crates/sender-app/tauri.conf.json
git commit -m "chore: bump version to <VER>"
```

## 5. 푸시 + main 동기화
```bash
git push origin <작업브랜치>
git checkout main && git merge --ff-only <작업브랜치> && git push origin main
git checkout <작업브랜치>
```
(작업을 main에서 직접 했으면 `git push origin main`만.)

## 6. 태그
```bash
git tag -a v<VER> -m "VTCast <VER>"
git push origin v<VER>
```

## 7. GitHub Release 생성 + 자산 업로드

### gh가 있으면
```bash
gh release create v<VER> --title "VTCast <VER>" --notes "<요약>" \
  target/release/bundle/nsis/VTCast_<VER>_x64-setup.exe \
  dist/latest.json
```

### gh가 없으면 (현재 환경) — git 자격증명 토큰 + REST API
```bash
TOKEN=$(printf "protocol=https\nhost=github.com\n\n" | git credential fill 2>/dev/null | grep '^password=' | cut -d= -f2-)

# 릴리즈 생성 (페이로드는 파일로 — JSON 이스케이프 문제 회피)
cat > /tmp/rel.json <<EOF
{ "tag_name": "v<VER>", "name": "VTCast <VER>",
  "body": "<요약>", "draft": false, "prerelease": false }
EOF
RID=$(curl -s -X POST -H "Authorization: token $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/LiveTrack-X/VTCast/releases --data @/tmp/rel.json \
  | grep -oE '"id": *[0-9]+' | head -1 | grep -oE '[0-9]+')

# 자산 업로드
curl -s -X POST -H "Authorization: token $TOKEN" -H "Content-Type: application/octet-stream" \
  --data-binary @target/release/bundle/nsis/VTCast_<VER>_x64-setup.exe \
  "https://uploads.github.com/repos/LiveTrack-X/VTCast/releases/$RID/assets?name=VTCast_<VER>_x64-setup.exe"
curl -s -X POST -H "Authorization: token $TOKEN" -H "Content-Type: application/json" \
  --data-binary @dist/latest.json \
  "https://uploads.github.com/repos/LiveTrack-X/VTCast/releases/$RID/assets?name=latest.json"
```

## 8. 검증
```bash
# 엔드포인트가 새 버전을 서빙하는지 (앱에 박힌 URL)
curl -sL "https://github.com/LiveTrack-X/VTCast/releases/latest/download/latest.json" | head -c 200
# 설치파일 다운로드 가능한지
curl -sL -o /dev/null -w "%{http_code}\n" -r 0-0 \
  "https://github.com/LiveTrack-X/VTCast/releases/download/v<VER>/VTCast_<VER>_x64-setup.exe"
```
둘 다 200/206이면 OK. 기존 (구버전) 앱은 다음 실행 시 "업데이트 하시겠습니까" 배너 → 자동 설치.

---

## 별개: 릴레이(수신 페이지) 배포
수신 페이지 freeze 수정 같은 `crates/relay/assets/receiver.html` 변경은 **앱 릴리즈와 무관**하며,
**공개 릴레이(`vtcast.livetrack.kr`) 재빌드·재배포**가 따로 필요하다(시그널링 로직·프로토콜은 그대로라
구버전 송출기와 호환). 앱 자동 업데이트와 별개로 처리.

## 자동화(선택)
태그 `v*` 푸시 시 GitHub Actions(`tauri-apps/tauri-action`)로 빌드·서명·릴리즈·`latest.json`
생성을 한 번에 할 수 있다. 저장소 시크릿에 `TAURI_SIGNING_PRIVATE_KEY`(키 내용),
`TAURI_SIGNING_PRIVATE_KEY_PASSWORD`(빈 값) 등록 필요. 필요 시 워크플로 추가.
