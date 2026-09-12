# 00_Newbee Obsidian + OCI + MCP 완전 정리 v2.0

> Mac(편집) → OCI Gitea(중앙) → OCI 서버(mcp-obsidian 8001 / mcpo 8002 / Open WebUI 8090) → NVIDIA NIM Muse Glimmer 30B
> 최종 검증일: 2026-05-11 / Open WebUI v0.11.3 / MCP SDK 1.17.3 / Node 24

---

## 전체 아키텍처

```
┌──────────────────────────────────┐
│ Mac ~/00_Newbee                  │  Obsidian 원본 + obsidian-git
└──────────────┬───────────────────┘
               │ git push/pull (obsidian-git)
               ▼
┌──────────────────────────────────┐
│ OCI Gitea                        │  ocigit.duckdns.org:3000/david/00_newbee.git
│ 중앙 저장소                       │  729MB (히스토리 리셋 후)
└──────────────┬───────────────────┘
               │ git clone
               ▼
┌──────────────────────────────────────────────────────┐
│ OCI 서버 /mnt/build-data/oci-vaults/00_newbee  729M │
│  ┌──────────────┐  :8001  Streamable HTTP  ┌────────┐│
│  │ mcp-obsidian │  ─────────────────────► │  mcpo  ││ :8002 OpenAPI
│  │ Node.js      │  systemd                 │ Python ││
│  └──────────────┘                          └────┬───┘│
│         ▲ host.docker.internal:host-gateway  OpenAPI │
│  ┌──────┴───────┐ :8090 Docker               ▼      │
│  │ Open WebUI   │ ◄────────────────  Obsidian Tools │
│  │ v0.11.3      │  Python 함수 방식 (GET 검증 우회)  │
│  └──────┬───────┘                                   │
└─────────┼───────────────────────────────────────────┘
          │ HTTPS https://integrate.api.nvidia.com/v1
          ▼
  ┌─────────────────┐
  │ Muse Glimmer 30B│  무료 엔드포인트, Native Tool Calling
  └─────────────────┘
```

---

## 1단계: Mac 볼트 경량화 (18GB → 729MB)

### 1-1. 히스토리 리셋
```bash
cd ~/00_Newbee
du -sh .git  # 18G 확인
mv .git .git.old-$(date +%Y%m%d-%H%M)
git init
git branch -M main
```

### 1-2. .gitignore (수정본 - v2)
```gitignore
# macOS
.DS_Store
.AppleDouble
.LSOverride

# Obsidian - 동적 UI/캐시 (반드시 제외)
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/hotkeys.json
.obsidian/appearance.json
.obsidian/graph.json
.obsidian/cache/
.obsidian/indexed-db/
.obsidian/app.json

# Lumina AI - 540MB 주범
.obsidian/plugins/lumina/storage/
.obsidian/plugins/lumina/data.json
.obsidian/plugins/lumina/*.json
obsidian-lumina-cache/

# Sync / Backup
.stfolder
.stfolder.removed-*
.stversions
.stignore
.trash/
.smart-env/
.obsidian-trash/

# 대용량 바이너리 - LFS 고려
*.bin
*.tibx
*.tib
*.img
*.iso
*.zip
*.7z
*.tar
*.gz
*.apk
*.ttc
*.itb
*.dmg
*.exe
*.mov
*.mp4
**/assets/*.bin
**/assets/*.tibx
**/assets/*.mp4

# 로그/임시
*.tmp
*.log
*.bak
```

### 1-3. 첫 커밋
```bash
git add -A
git status | head -n 50
git ls-files -z | xargs -0 du -ch 2>/dev/null | tail -1
git commit -m "init: 00_Newbee vault reset (18GB -> 729MB, lumina cache excluded)"
git remote add origin https://ocigit.duckdns.org/david/00_newbee.git
git push -u origin main --force
```

---

## 2단계: OCI 서버 클론

```bash
sudo mkdir -p /mnt/build-data/oci-vaults
sudo chown ubuntu:ubuntu /mnt/build-data/oci-vaults
cd /mnt/build-data/oci-vaults
git clone https://ocigit.duckdns.org/david/00_newbee.git 00_newbee
du -sh 00_newbee 00_newbee/.git
# 729M / 333M 예상
```

### 자동 동기화 cron (v2 추가)
```bash
crontab -e
# 5분마다 pull, 변경 있으면 push (충돌은 Mac 우선)
*/5 * * * * cd /mnt/build-data/oci-vaults/00_newbee && git pull --rebase --autostash >/dev/null 2>&1
```

---

## 3단계: mcp-obsidian :8001 (수정 - 완전한 server.js)

### 3-1. 설치
```bash
sudo mkdir -p /opt/mcp-obsidian
sudo chown ubuntu:ubuntu /opt/mcp-obsidian
cd /opt/mcp-obsidian
cat > package.json <<'JSON'
{
  "type": "module",
  "dependencies": {
    "@modelcontextprotocol/sdk": "1.17.3",
    "express": "4.21.2",
    "cors": "2.8.5",
    "zod": "3.23.8"
  }
}
JSON
npm install --save-exact
```

### 3-2. server.js v2 - 검색/읽기/목록/쓰기 완전 구현
```javascript
#!/usr/bin/env node
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";
import cors from "cors";
import fs from "fs/promises";
import path from "path";
import { z } from "zod";

const VAULT = "/mnt/build-data/oci-vaults/00_newbee";
const PORT = 8001;

const app = express();
app.use(cors({ exposedHeaders: ["mcp-session-id", "mcp-protocol-version"] }));
app.use(express.json({ limit: "10mb" }));

const mcpServer = new McpServer(
  { name: "obsidian-00_newbee", version: "2.0.0" },
  { capabilities: { tools: {} } }
);

const safePath = (p) => {
  const cleaned = (p || "").replace(/^\//, "").replace(/\.\./g, "");
  return path.join(VAULT, cleaned);
};

async function walkFiles(dir, results = []) {
  const entries = await fs.readdir(dir, { withFileTypes: true });
  for (const e of entries) {
    if (e.name.startsWith(".")) continue;
    const full = path.join(dir, e.name);
    if (e.isDirectory()) await walkFiles(full, results);
    else if (e.name.endsWith(".md")) results.push(full);
  }
  return results;
}

mcpServer.registerTool("search", {
  description: "00_newbee Obsidian vault md 파일 내용 검색. vault 전용. knowledge base 아님.",
  inputSchema: { query: z.string().describe("검색어") }
}, async ({ query }) => {
  const files = await walkFiles(VAULT);
  const hits = [];
  for (const f of files) {
    try {
      const txt = await fs.readFile(f, "utf8");
      if (txt.toLowerCase().includes(query.toLowerCase())) {
        hits.push(path.relative(VAULT, f));
        if (hits.length >= 50) break;
      }
    } catch {}
  }
  return { content: [{ type: "text", text: hits.join("\n") || "결과 없음" }] };
});

mcpServer.registerTool("read", {
  description: "Obsidian vault 파일 읽기",
  inputSchema: { path: z.string() }
}, async ({ path: p }) => {
  const txt = await fs.readFile(safePath(p), "utf8");
  return { content: [{ type: "text", text: txt.slice(0, 15000) }] };
});

mcpServer.registerTool("list", {
  description: "Obsidian vault 폴더 목록. dir='' 이면 루트. 절대 '00_newbee'를 dir로 넣지 마세요.",
  inputSchema: { dir: z.string().optional().default("") }
}, async ({ dir }) => {
  const entries = await fs.readdir(safePath(dir), { withFileTypes: true });
  const out = entries.map(e => (e.isDirectory() ? e.name + "/" : e.name)).join("\n");
  return { content: [{ type: "text", text: out }] };
});

mcpServer.registerTool("write", {
  description: "Obsidian vault 파일 쓰기 (vault에 직접 저장)",
  inputSchema: { path: z.string(), content: z.string() }
}, async ({ path: p, content }) => {
  const fp = safePath(p);
  await fs.mkdir(path.dirname(fp), { recursive: true });
  await fs.writeFile(fp, content, "utf8");
  return { content: [{ type: "text", text: `written: ${p}` }] };
});

app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({ sessionIdGenerator: undefined });
  await mcpServer.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.get("/health", (req, res) => res.json({ ok: true, vault: VAULT, tools: ["search","read","list","write"] }));
app.listen(PORT, "0.0.0.0", () => console.log(`MCP on ${PORT}/mcp`));
```

### 3-3. systemd
```bash
sudo tee /etc/systemd/system/mcp-obsidian.service <<'EOF'
[Unit]
Description=MCP Obsidian 00_newbee v2 - SDK 1.17.3 stateless
After=network.target
[Service]
Type=simple
User=ubuntu
WorkingDirectory=/opt/mcp-obsidian
ExecStart=/usr/bin/node /opt/mcp-obsidian/server.js
Restart=on-failure
RestartSec=3
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now mcp-obsidian
curl -s http://localhost:8001/health | jq
```

---

## 4단계: mcpo :8002

```bash
python3 -m venv ~/mcpo-venv
~/mcpo-venv/bin/pip install mcpo "mcp>=1.17.0"

sudo tee /etc/systemd/system/mcpo.service <<'EOF'
[Unit]
Description=mcpo bridge 00_newbee
After=network.target mcp-obsidian.service
Requires=mcp-obsidian.service
[Service]
Type=simple
User=ubuntu
ExecStart=/home/ubuntu/mcpo-venv/bin/mcpo --port 8002 --server-type streamable-http -- http://localhost:8001/mcp
Restart=on-failure
RestartSec=3
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now mcpo
curl -s http://localhost:8002/openapi.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(list(d['paths'].keys()))"
# ['/search', '/read', '/list', '/write', '/openapi.json', '/docs']
```

---

## 5단계: Open WebUI v0.11.3 (수정본)

```yaml
# ~/open-webui/docker-compose.yaml v2
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:v0.11.3
    container_name: open-webui
    ports: ["8090:8080"]
    volumes:
      - open-webui:/app/backend/data
      - /mnt/build-data/oci-vaults/00_newbee:/mnt/build-data/oci-vaults/00_newbee:ro  # 쓰기 필요하면 :ro 제거
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - WEBUI_SECRET_KEY=CHANGE_ME_32chars
      - ENABLE_CODE_INTERPRETER=true
    restart: unless-stopped
volumes:
  open-webui:
```
```bash
docker compose pull && docker compose down && docker compose up -d
docker logs -f open-webui --tail 50 | grep -i "v0.11.3\|Uvicorn\|tools"
```

---

## 6단계: NVIDIA NIM 연결

1. build.nvidia.com → API Keys → `nvapi-...` 생성
2. Open WebUI → 관리자 → 연결 → OpenAI 호환 → `https://integrate.api.nvidia.com/v1` + Bearer 키
3. 모델 선택: `meta/muse-glimmer-30b`

---

## 7단계: 도구 등록 - 함수 방식이 정답 (외부 도구 서버 실패 원인 정리)

**실패 원인**: Open WebUI 0.11.3은 외부 도구 서버 검증 시 `GET /search`를 보냄. mcpo는 `POST` 전용이라 405. 결과 도구 목록 0개.
→ 해결: Python 함수로 우회. 함수는 GET 검증 없음.

### 경로
워크스페이스 → 도구 → + 새 도구

### 코드 v2 (수정 - timeout, 에러처리, 설명 강화)
```python
"""
title: Obsidian Vault Tools
author: david
version: 2.0.0
description: 00_newbee Obsidian vault 전용 - search, read, list, write. 지식베이스 아님.
"""
import requests

class Tools:
    def __init__(self):
        self.base = "http://host.docker.internal:8002"

    def search(self, query: str) -> str:
        """00_newbee Obsidian vault 729MB에서 md 파일 내용 검색. vault 전용 검색기. knowledge base 도구를 절대 사용하지 마세요. :param query: 검색어"""
        r = requests.post(f"{self.base}/search", json={"query": query}, timeout=30)
        r.raise_for_status()
        return r.text or "결과 없음"

    def read(self, path: str) -> str:
        """Obsidian vault 파일 읽기. search 결과의 상대 경로를 그대로 사용. :param path: 파일 경로"""
        r = requests.post(f"{self.base}/read", json={"path": path}, timeout=30)
        r.raise_for_status()
        return r.text[:12000]

    def list(self, dir: str = "") -> str:
        """Obsidian vault 폴더 목록. dir=''이면 루트. '00_newbee'를 dir로 넣지 마세요. :param dir: 폴더 경로"""
        r = requests.post(f"{self.base}/list", json={"dir": dir}, timeout=30)
        r.raise_for_status()
        return r.text

    def write(self, path: str, content: str) -> str:
        """Obsidian vault 파일 쓰기. :param path: 저장 경로 :param content: 내용"""
        r = requests.post(f"{self.base}/write", json={"path": path, "content": content}, timeout=30)
        r.raise_for_status()
        return r.text
```

저장 → 활성화 → 모델 편집에서 `Obsidian Vault Tools` 체크

### 모델 시스템 프롬프트 v2 (Glimmer 과추론 방지)
```
당신은 00_newbee Obsidian vault 검색 도우미입니다.

[도구 사용 규칙 - 반드시 준수]
1. 00_newbee 볼트 관련 요청은 무조건 search, read, list만 사용. list_knowledge_bases, 지식베이스 검색 절대 금지.
2. search는 최대 1회만 호출.
3. dir에 "00_newbee"를 넣지 마세요. 루트는 "" 입니다.
4. 파일 내용은 read로 필요한 1~3개만 읽기.
5. 응답은 한국어로, 파일 경로는 그대로 표시.

[여러 파일 요약이 필요하면]
AI 방식으로 파일 10개를 read하지 말고, 사용자에게 "스크립트 방식 요약(batch) 또는 지식 베이스(RAG) 전환이 필요하다"고 제안하세요.
```

고급 매개변수 → 함수 호출 → Native

---

## 8단계: 검증 및 여러 파일 요약 전략

### 검증
```
질문: 00_newbee 볼트에서 "MCP" 포함된 노트를 search 도구로 검색해줘
예상 mcpo 로그: POST /search 200
예상 Glimmer: View Result from search → 16개 파일 목록
```

### 여러 파일 요약이 필요하면 (v2 추가)

**문제**: Glimmer가 10개 파일 read하면 토큰 30k+ → 느리고 비용 증가.

**해결책 A - 스크립트 방식 (추천, 무료)**
```bash
# OCI 서버에서 배치 요약 파일 생성
rg -l "MCP" /mnt/build-data/oci-vaults/00_newbee --type md | head -10 | xargs -I {} sh -c 'echo "=== {} ==="; sed -n "1,80p" "{}"' > /tmp/mcp_digest.md
# /tmp/mcp_digest.md를 Open WebUI에 업로드 후 "이거 요약해줘" 1회 호출
```

mcpo에 batch 도구 추가 예정:
```
POST /batch_search {query, limit, preview_lines}
→ 여러 파일 80줄씩 잘라서 한 번에 반환
```

**해결책 B - 지식 베이스(RAG)**
1. 관리자 → 문서 → 지식 베이스 `00_newbee` 생성
2. `23. obsidian with AI`, `00.4 HA server on OCI` 폴더만 먼저 업로드 (임베딩 모델: bge-m3 or nvidia/nv-embed-v2)
3. 채팅에서 `#00_newbee MCP 요약해줘` → RAG로 상위 5개 청크만 Glimmer에 전달 → tool 호출 없이 답변

| 방식 | 장점 | 단점 |
|---|---|---|
| AI tool 16회 read | 구현 쉬움 | 토큰 폭증, 느림 |
| 스크립트 배치 | 빠르고 무료, 1회 호출 | 사전 스크립트 필요 |
| RAG 지식 베이스 | 대량 요약 최적, 재사용 | 초기 임베딩 시간 |

---

## 최종 요약표 v2

| 컴포넌트 | 주소 | 실행 | 상태 |
|---|---|---|---|
| mcp-obsidian | :8001/mcp | systemd Node.js | ✅ |
| mcpo | :8002/docs | systemd Python venv | ✅ |
| Open WebUI | :8090 | Docker v0.11.3 | ✅ |
| NVIDIA NIM | integrate.api.nvidia.com | HTTPS | ✅ |
| Glimmer 30B | meta/muse-glimmer-30b | 무료 | ✅ |
| Obsidian Tools | 함수 방식 | Open WebUI 내부 | ✅ |
| Git 동기화 | 5분 cron pull | OCI | ⏳ 설정 중 |
| 쓰기 | :ro 제거 필요 | docker-compose | ⚠️ |

---

## 다음 할 일 체크리스트

- [ ] 1. docker-compose에서 :ro 제거 → 쓰기 테스트
- [ ] 2. cron git pull 자동화
- [ ] 3. batch_search 도구 추가 (여러 파일 요약용)
- [ ] 4. 지식 베이스에 핵심 폴더 2개만 RAG 등록
- [ ] 5. 0.12+ 업그레이드 시 mcpo 제거, 네이티브 MCP로 전환 (https://docs.openwebui.com/features/mcp)

## 핵심 교훈 v2

1. .git 18GB 원인은 obsidian-git + Lumina storage + loose objects. .gitignore + 히스토리 리셋이 근본 해결.
2. Open WebUI 외부 도구 서버는 GET 검증 때문에 mcpo와 궁합 안 맞음 → 함수 방식이 현재 안정.
3. Glimmer는 "볼트"를 "knowledge base"로 오해 → 함수 description에 "지식베이스 아님, vault 전용" 명시 필수.
4. 여러 파일 요약은 AI에게 10번 read 시키지 말고 스크립트 배치 또는 RAG로 전환.
