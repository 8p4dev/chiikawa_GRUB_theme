# GRUB 테마 조사 (치이카와)

## 증상
- 키보드 조작 시 그래픽 깨짐(artifact/tearing).
- 부트 항목 선택 시 크래시 / OS 진입 실패.
- 초기 렌더는 정상(스크린샷 OK) → theme.txt 파싱·로딩은 통과.

## 가능 원인 (우선순위)
1) 비디오 모드 불안정: 설정한 해상도가 펌웨어/VM 드라이버에서 지원되지 않아 키 입력 시 재그리기 때 gfxterm가 깨지거나 크래시.
2) 큰 RGBA 배경(1920x1080) + 여러 PNG가 특정 VESA 모드에서 GPU 메모리를 초과해 버퍼 재할당 시 크래시/깨짐.
3) 커스텀 폰트 위험: 손상되었거나 너무 큰 `ok_mallang_24.pf2`가 네비게이션 중 새 글리프 렌더링 때 크래시 유발 가능.
4) PNG 포맷 에지 케이스: 일부 GRUB 빌드는 알파가 있는 32비트 PNG나 비인덱스 색상 selection 이미지를 싫어함 → 하이라이트 전환 시 깨짐.
5) 자산 크기 여유 부족: 메뉴 아이콘 46x46, `item_height` 48로 여유가 적음. 루트에 중복 아이콘은 불필요한 로드만 증가(크래시는 아님).

## 빠른 진단
- GRUB에서 `videoinfo`로 지원 모드 확인 후 목록에 있는 1280x720x32 또는 1024x768x32 같은 모드 선택.
- 에뮬레이터 재현: `grub-emu --directory=. --theme=./theme.txt` (Linux 환경 필요할 수 있음)으로 디스크 건드리지 않고 테스트.
- 테마 문법 검사: `grub-script-check theme.txt` (배포판에 있으면 활용).
- 폰트 재생성: `grub-mkfont -o ok_mallang_24.pf2 -s 24 /path/to/ttf` 후 파일 크기 확인(1MB 이하 권장).
- 이미지 포맷 점검: 모든 PNG를 8비트 인덱스 또는 24비트 RGB(알파 제거)로, 배경은 가로 1280px 이하, selection 아이콘은 `item_height` 이하로 맞추기.

## 시도할 완화책
1) 보수적 gfx 모드 설정(`/etc/default/grub`):
   - `GRUB_GFXMODE=1280x720x32`
   - `GRUB_GFXPAYLOAD_LINUX=keep`
   - `GRUB_TERMINAL_OUTPUT=gfxterm`
   설정 후 `update-grub` → 재부팅 테스트.
2) 자산 최적화:
   - 배경: 1280x720(또는 선택한 gfxmode 해상도)로 스케일, 24비트 RGB로 변환(알파 제거).
   - `select_c/e/w.png`: 24비트 RGB, 크기 약 48x48 이하, 투명도 불필요.
   - 메뉴 아이콘: 48 이하로 리사이즈, `icons/` 폴더에만 두고 루트 중복 삭제.
3) 폰트 재생성 후에도 문제면, 일시적으로 기본 `unicode.pf2`로 교체해 폰트 기인 여부 확인.
4) 여전히 깨지면 선택 하이라이트 이미지를 끄고 단색 사용:
   - `selected_item_pixmap_style = "select_*.png"` 주석 처리, `selected_item_color`를 더 어둡게 설정.
5) 테마 비활성화(`GRUB_THEME=`) 시 부팅이 정상인지 확인해 테마가 원인인지 확정. 정상이라면 자산 축소 후 다시 켜기.

## 추가로 필요한 데이터
- `videoinfo` 출력과 실제 설정한 `GRUB_GFXMODE`.
- 변환 후 `chiikawa_wallpaper.png`, `select_*.png`의 크기와 비트 깊이.
- 기본 `unicode.pf2` 사용 시 결과 vs 커스텀 폰트 사용 시 결과.
- 테마 비활성화 시 크래시가 사라지는지 여부(테마 원인 확정용).
