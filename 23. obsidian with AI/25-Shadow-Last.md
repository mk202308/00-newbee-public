# Shadow 공개 미러 시스템 - 최종 성공 기록
**작성일**: 2026-09-12
**상태**: ✅ 운영 중
**목적**: 비공개 Obsidian 볼트의 특정 폴더만 "필요할 때만" 공개하여 외부 AI(Claude)와 협업

---

## 1. 최종 아키텍처

```
[Mac ~/00_Newbee]
│ obsidian-git (autoSave 30s / autoPush 60s / autoPull 60s)
▼
[Gitea: 00_newbee] (비공개, ocigit.duckdns.org)
│ git pull
▼
[OCI oci-build: /mnt/build-data/oci-vaults/00_newbee]
│ 원본 clone (비공개 유지)
│
│ /opt/shadow/public-list.conf ← 공개 폴더 지정
▼
[/opt/scripts/sync-shadow.sh]
├─ 파일명 기반 위험 파일 → [REDACTED]
├─ 내용 자동 마스킹 (sed 기반)
├─ 사후 검사 (의심 패턴 리포트)
└─ Orphan commit + Force push
│
├─▶ [Gitea: 00_newbee-public] (공개, Tailscale 내부망 전용)
│
└─▶ [GitHub: 00-newbee-public] (공개, 외부 접근 가능)
      ↓
      https://github.com/mk202308/00-newbee-public
      ↓
      [Claude] ← 여기서 읽음
```

**핵심**: Gitea 공개 저장소는 Tailscale 내부망에만 노출 → Claude 접근 불가. **GitHub를 미러 대상으로 추가**하여 외부 접근 우회.

---

## 2. 디렉토리 구조

### OCI 서버 (oci-build)

```
/opt/
├── mcp-obsidian/              # MCP 서버 (Node.js, :8001)
│   ├── package.json           # SDK 1.17.3 고정
│   ├── server.js              # McpServer API, 2026-07-28 stateless
│   └── node_modules/
│
├── shadow/
│   ├── public-list.conf       # 공개할 폴더 목록
│   └── github-url.txt         # GitHub remote URL (토큰 포함)
│
└── scripts/
    └── sync-shadow.sh         # 동기화 스크립트

/mnt/build-data/
├── oci-vaults/
│   └── 00_newbee/             # 원본 볼트 (비공개)
│
└── public-shadow/
    └── shadow/                # 공개 미러 작업 디렉토리
        ├── .git/
        ├── README.md
        └── 23. obsidian with AI/  # 실제 공개 콘텐츠
```

### 관련 서비스

| 서비스 | 서버 | 포트 | 방식 |
|---|---|---|---|
| mcp-obsidian | oci-build | 8001 | systemd |
| mcpo | oci-build | 8002 | systemd |
| Open WebUI | oci-build | 8090 | Docker |
| Gitea | oci-build | 3000 | Docker |
| NPM | oci-npm (100.99.0.4) | 80/443/81 | Docker |

---

## 3. 저장소 정보

### Gitea (비공개 원본)

| 항목 | 값 |
|---|---|
| URL | `https://ocigit.duckdns.org/david/00_newbee.git` |
| 접근 | Tailscale 내부망만 |
| 상태 | Private |

### Gitea (공개 미러)

| 항목 | 값 |
|---|---|
| URL | `https://ocigit.duckdns.org/david/00_newbee-public.git` |
| 접근 | Tailscale 내부망만 |
| 상태 | Public (내부망 한정) |

### GitHub (공개 미러 — 외부 접근용)

| 항목 | 값 |
|---|---|
| URL | `https://github.com/mk202308/00-newbee-public` |
| 접근 | 인터넷 전체 |
| 상태 | Public |
| 용도 | Claude가 읽음 |

---

## 4. 워크플로우

### 평상시 (아무것도 공개 안 됨)

```bash
cat /opt/shadow/public-list.conf
# → 주석만 있거나 빈 상태
```

### 작업 시작

**1단계: 공개할 폴더 지정**

```bash
shadow-edit
# nano 편집기 열림
# 예: "23. obsidian with AI" 입력
```

**2단계: 동기화 실행**

```bash
shadow-sync
```

예상 로그:

```
[1/7] 원본 pull...
[2/7] 섀도우 초기화...
[3/7] 공개 폴더 복사...
  ✅ 23. obsidian with AI
[4/7] 위험 파일 처리...
[5/7] 내용 자동 마스킹...
  총 마스킹: 3 건
[6/7] 사후 검사...
  ✅ 사후 검사 통과
[7/7] 공개 저장소 재작성...
  ✅ Gitea push 성공
  ✅ GitHub push 성공
✅ 동기화 완료
🐙 GitHub: https://github.com/mk202308/00-newbee-public
```

**3단계: Claude에게 URL 전달**

폴더 목록:
```
https://github.com/mk202308/00-newbee-public/tree/main/23.%20obsidian%20with%20AI
```

특정 파일 (raw):
```
https://raw.githubusercontent.com/mk202308/00-newbee-public/main/23.%20obsidian%20with%20AI/12.%20ai.md
```

**4단계: 작업 완료 후 공개 중단**

```bash
shadow-clear
```

---

## 5. 편의 별칭 (~/.bashrc)

```bash
# Shadow On-Demand
alias shadow-edit='sudo nano /opt/shadow/public-list.conf'
alias shadow-sync='/opt/scripts/sync-shadow.sh'
alias shadow-list='cat /opt/shadow/public-list.conf'
alias shadow-clear='sudo tee /opt/shadow/public-list.conf <<EOL
# 전체 비공개 - 빈 목록
EOL
/opt/scripts/sync-shadow.sh'
alias shadow-log='cat /tmp/shadow-sync.log'
alias shadow-mask='cat /tmp/shadow-mask-report.txt 2>/dev/null | sort -u'
alias shadow-check='cat /tmp/shadow-postcheck.txt 2>/dev/null'
alias shadow-github='echo "https://github.com/mk202308/00-newbee-public"'
```

---

## 6. 마스킹 규칙

### 자동 치환 대상 (sed 기반)

| 패턴 | 라벨 |
|---|---|
| `nvapi-...` | NVIDIA_KEY |
| `sk-...` | OPENAI_KEY |
| `ghp_...` | GITHUB_TOKEN |
| `fc-...` (32 hex) | FIRECRAWL_KEY |
| `AKIA...` | AWS_KEY |
| `xoxb-...` | SLACK_TOKEN |
| `secret_...` | NOTION_TOKEN |
| `api_key: 값` | API_KEY |
| `token: 값` | TOKEN |
| `password: 값` | PASSWORD |
| `secret: 값` | SECRET |
| PEM 개인 키 블록 | PRIVATE_KEY |
| `*.key`, `*.pem`, `*.p12`, `*.pfx`, `*.env` | 파일명 필터 |

### 치환 형식

```
api_key: [REDACTED_API_KEY]
→ api_key: [REDACTED_NVIDIA_KEY]
```

주의: 라벨에 콜론(:) 대신 언더스코어(_) 사용 (Perl 문법 오류 방지).

---

## 7. 시스템 프롬프트 (Open WebUI — Muse Glimmer)

```
당신은 Obsidian 볼트 검색 도우미입니다.
규칙:
1. search 도구 호출은 최대 1회로 제한하세요.
2. 파일 내용을 읽지 말고 경로만 반환하세요.
3. 추가 탐색 없이 즉시 결과를 사용자에게 제시하세요.
4. 응답은 반드시 한국어로 작성하세요.
5. 볼트 이름(00_newbee)을 list의 dir로 넘기지 마세요.
```

---

## 8. 트러블슈팅 이력

| # | 문제 | 원인 | 해결 |
|---|---|---|---|
| 1 | 18GB `.git` | loose object 미정리 + Lumina 540MB + 바이너리 | 히스토리 리셋 → 729MB |
| 2 | mcpo 브릿지 실패 | `mcp` 버전 불일치 + Perl 문법 오류 | `mcp>=1.17.0` 재설치 + Perl→sed |
| 3 | Open WebUI 도구 서버 등록 실패 | GET 검증 vs POST 전용 | Python 함수로 우회 |
| 4 | Gitea 인증 실패 | remote URL에 토큰 없음 | 토큰 URL 포함 + chmod 600 |
| 5 | 공개 저장소 히스토리 잔존 | 커밋 누적 | `.git` 삭제 + force push |
| 6 | GitHub remote 소실 | 스크립트가 `.git` 재생성 | `github-url.txt` + 자동 재추가 |
| 7 | 사후 검사 오탐 | `.git/hooks/*.sample` 매칭 | `--exclude-dir=.git` |
| 8 | Tailscale 내부망만 노출 | 외부 접근 불가 | **GitHub 미러** 채택 |

---

## 9. 보안 주의사항

### 노출된 토큰 (작업 완료 후 revoke 권장)

| 토큰 | 위치 | 조치 |
|---|---|---|
| Gitea 토큰 | 이 대화 기록 | revoke 후 재발급 |
| GitHub 토큰 | 이 대화 기록 | revoke 후 재발급 |

### 토큰 파일 권한

```bash
chmod 600 /mnt/build-data/public-shadow/shadow/.git/config
chmod 600 /opt/shadow/github-url.txt
chmod 600 /mnt/build-data/oci-vaults/00_newbee/.git/config
```

### 민감 폴더는 공개 금지

- `00.4 HA server on OCI/`
- `10. 공유기/`
- `19. DNS NPM/`
- `07. Home Server/`

---

## 10. 성공 검증 기록 (2026-09-12)

### Claude가 실제로 읽은 파일

```
https://github.com/mk202308/00-newbee-public/blob/main/23.%20obsidian%20with%20AI/Shadow.md
```

결과: ✅ 성공 (10.1 KB, 304줄)

### 마스킹 동작 확인

```
23. obsidian with AI/0. 진행방향.md    [CONTENT_MASKED]
23. obsidian with AI/12. ai.md         [CONTENT_MASKED]
23. obsidian with AI/Shadow.md         [PRIVATE_KEY]
```

총 3건 마스킹, 사후 검사 통과.

---

## 11. 앞으로의 소통 방식

```
[1] shadow-edit → 공개 폴더 지정
     ↓
[2] shadow-sync 실행
     ↓
[3] Claude에게 GitHub URL 전달
     - 폴더: https://github.com/mk202308/00-newbee-public/tree/main/<폴더>
     - 파일: https://raw.githubusercontent.com/mk202308/00-newbee-public/main/<경로>
     ↓
[4] Claude가 파일 읽고 분석/답변
     ↓
[5] shadow-clear → 공개 중단 (README만 남음)
```

---

## 12. Vault as MCP (Mac) vs obsidian-mcp (OCI)

| 항목 | Vault as MCP (Mac) | obsidian-mcp (OCI) |
|---|---|---|
| 실행 위치 | Obsidian 플러그인 | Node.js 서버 |
| GUI 필요 | ✅ | ❌ |
| 포트 | 8765 | 8001 |
| 용도 | 로컬 AI 직접 연결 | 서버 AI (Open WebUI) |
| MCP 프로토콜 | 2025-06-18 | 2026-07-28 (SDK 1.17.3) |

둘 다 운영 중. Mac은 Vault as MCP, OCI는 obsidian-mcp.

---

## 13. 관련 파일 요약

| 파일 | 위치 | 용도 |
|---|---|---|
| `public-list.conf` | `/opt/shadow/` | 공개 폴더 목록 |
| `github-url.txt` | `/opt/shadow/` | GitHub remote URL |
| `sync-shadow.sh` | `/opt/scripts/` | 동기화 스크립트 |
| `shadow-sync.log` | `/tmp/` | 동기화 로그 |
| `shadow-mask-report.txt` | `/tmp/` | 마스킹 리포트 |
| `shadow-postcheck.txt` | `/tmp/` | 사후 검사 결과 |
| `server.js` | `/opt/mcp-obsidian/` | MCP 서버 |
| `mcpo.service` | `/etc/systemd/system/` | mcpo 브릿지 |
| `mcp-obsidian.service` | `/etc/systemd/system/` | MCP 서버 |

---

## 14. 최종 파일 생성 스크립트

### 방법 A: Mac에서 생성

```bash
cat > ~/00_Newbee/"23. obsidian with AI/25. Shadow 최종 성공 기록.md" <<'MARKDOWN_EOF'
(이 문서 전체를 붙여넣기)
MARKDOWN_EOF
```

### 방법 B: OCI 서버에서 생성

```bash
sudo tee "/mnt/build-data/oci-vaults/00_newbee/23. obsidian with AI/25. Shadow 최종 성공 기록.md" <<'MARKDOWN_EOF'
(이 문서 전체를 붙여넣기)
MARKDOWN_EOF
sudo chown ubuntu:ubuntu "/mnt/build-data/oci-vaults/00_newbee/23. obsidian with AI/25. Shadow 최종 성공 기록.md"
```

주의: OCI에서 직접 생성 시 Git 추적 상태가 꼬일 수 있음. Mac에서 생성 후 push 권장.

---

**시스템 구축 완료. 이제 Obsidian 볼트를 Claude와 함께 작업할 수 있습니다.**

**작성**: 2026-09-12  
**다음 업데이트**: 시스템 변경 시 이 문서 갱신
