# mac-optimize

macOS 시스템을 점검하고 정리하는 Claude Code 스킬.

병렬 에이전트 5개가 캐시 정리, Homebrew 업데이트, 서비스 점검, 디스크 분석, 시스템 건강 점검을 나눠 맡고, 이전·이후 수치를 표로 보고한다.

## 설치

```bash
npx skills add seunggabi/mac-optimize
```

### 수동 설치

```bash
git clone https://github.com/seunggabi/mac-optimize.git
ln -s "$(pwd)/mac-optimize/skills/mac-optimize" ~/.claude/skills/mac-optimize
```

## 사용

```
/mac-optimize
```

"맥 느려", "디스크 정리", "시스템 점검" 같은 말에도 반응한다.

## 하는 일

| 단계 | 내용 |
|---|---|
| Phase 1 | CPU·메모리·스왑·디스크·캐시·자동시작 항목 진단 |
| Phase 2 | 에이전트 5개 병렬 실행 |
| Phase 3 | 결과 수집, `/tmp` 이동분과 sudo 명령 안내 |
| Phase 4 | Before → After 종합 보고서 |

| 에이전트 | 맡은 일 |
|---|---|
| cache-cleaner | pip·npm 캐시, 오래된 로그, 브라우저·IDE 캐시 |
| brew-updater | formula 업그레이드, autoremove, cleanup, doctor |
| service-optimizer | brew 서비스, 오래 떠 있는 프로세스 집계 |
| disk-analyzer | 큰 폴더, Docker, node_modules, 대용량 파일 (읽기 전용) |
| system-health | macOS 업데이트, 볼륨 검사, 스왑, 가동 시간 (읽기 전용) |

## 안전 규칙

- `rm -rf` 를 쓰지 않는다. 캐시 매니저 명령을 쓰거나 `/tmp$HOME/...` 로 옮긴다.
- 보안 소프트웨어(VPN, 백신, DLP, 공인인증)는 건드리지 않는다.
- `sudo`, macOS 업데이트 설치, cask 업그레이드, Docker prune, 프로세스 종료는 직접 하지 않고 목록이나 명령으로 안내한다.
- `/tmp` 는 같은 볼륨이라 옮기기만 해서는 디스크 여유가 늘지 않는다. 비울지는 사용자에게 묻는다.

## License

MIT
