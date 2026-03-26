---
title: Raspberry Pi 5 + Claude Code でローカルAIエージェントを動かす最小構成
emoji: 🍓
type: tech
topics: ["RaspberryPi", "AI", "ClaudeCode"]
published: false
---

## はじめに

本記事は、Raspberry Pi 5（またはPi 4）上でClaude Code（MCP経由）とOllamaを組み合わせ、クラウド依存を最小化したローカルAIエージェントを動かすための最小構成をまとめたものです。

筆者は非エンジニア（工学部出身・40代）で、Raspberry Pi 4台のクラスター「Genesis」を自宅で運用しています。

## 環境

| 項目 | 内容 |
|------|------|
| ハードウェア | Raspberry Pi 5 (8GB) |
| OS | Raspberry Pi OS Bookworm (64bit) |
| Python | 3.11.x |
| Ollama | 最新版 |
| モデル | gemma3:4b, qwen2.5:3b |

## Step 1: Ollamaのインストールと起動確認

```bash
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl status ollama
ollama pull gemma3:4b
ollama pull qwen2.5:3b
ollama run gemma3:4b 'こんにちは'
```

外部ノードからアクセスする場合:

```bash
sudo systemctl edit ollama --force
```

```ini
[Service]
Environment='OLLAMA_HOST=0.0.0.0:11434'
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart ollama
curl http://192.168.0.203:11434/api/tags
```

## Step 2: FastMCPサーバーの実装

```bash
pip install fastmcp httpx
```

```python
# mcp_server.py
from fastmcp import FastMCP
import httpx, subprocess
from datetime import datetime

mcp = FastMCP('pi-edge-agent')
OLLAMA_URL = 'http://localhost:11434'

@mcp.tool()
async def query_local_llm(prompt: str, model: str = 'gemma3:4b') -> str:
    async with httpx.AsyncClient(timeout=60.0) as client:
        response = await client.post(
            f'{OLLAMA_URL}/api/generate',
            json={'model': model, 'prompt': prompt, 'stream': False}
        )
        return response.json().get('response', '応答なし')

@mcp.tool()
async def get_system_status() -> dict:
    temp = subprocess.run(['vcgencmd', 'measure_temp'], capture_output=True, text=True)
    mem = subprocess.run(['free', '-m'], capture_output=True, text=True)
    vals = mem.stdout.strip().split('\n')[1].split()
    return {'temperature': temp.stdout.strip(),
            'memory_total_mb': int(vals[1]), 'memory_used_mb': int(vals[2])}

if __name__ == '__main__':
    mcp.run(transport='stdio')
```

## Step 3: Claude CodeへのMCP登録

```json
{
  "mcpServers": {
    "pi-edge-agent": {
      "command": "python3",
      "args": ["/home/pi/mcp_server.py"]
    }
  }
}
```

## Step 4: LLMルーター最小構成

```python
# llm_router.py
import httpx, os
from enum import Enum

class LLMBackend(Enum):
    LOCAL = 'local'
    CLOUD = 'cloud'

OLLAMA_URL = os.getenv('OLLAMA_URL', 'http://localhost:11434')
DEEPSEEK_API_KEY = os.getenv('DEEPSEEK_API_KEY', '')

def select_backend(prompt: str, max_tokens: int = 500) -> LLMBackend:
    if max_tokens <= 200 or len(prompt) < 100:
        return LLMBackend.LOCAL
    return LLMBackend.CLOUD

async def query(prompt: str, max_tokens: int = 500) -> str:
    backend = select_backend(prompt, max_tokens)
    if backend == LLMBackend.LOCAL:
        return await _query_ollama(prompt)
    return await _query_deepseek(prompt, max_tokens)

async def _query_ollama(prompt: str, model: str = 'gemma3:4b') -> str:
    async with httpx.AsyncClient(timeout=60.0) as client:
        try:
            r = await client.post(f'{OLLAMA_URL}/api/generate',
                json={'model': model, 'prompt': prompt, 'stream': False})
            return r.json().get('response', '')
        except Exception as e:
            return await _query_deepseek(prompt, 500)

async def _query_deepseek(prompt: str, max_tokens: int) -> str:
    async with httpx.AsyncClient(timeout=30.0) as client:
        r = await client.post('https://api.deepseek.com/chat/completions',
            headers={'Authorization': f'Bearer {DEEPSEEK_API_KEY}'},
            json={'model': 'deepseek-chat',
                  'messages': [{'role': 'user', 'content': prompt}],
                  'max_tokens': max_tokens})
        return r.json()['choices'][0]['message']['content']
```

## Genesisクラスター構成

| ノード | 役割 | 主な処理 |
|--------|------|----------|
| nodeA (Pi 4) | オーケストレーター | Discord Bot、タスク振り分け |
| nodeB (Pi 4) | LLMワーカー | Ollama推論（gemma3:4b, qwen2.5:3b） |
| nodeC (Pi 4) | モニター | 死活監視、自動復旧 |
| nodeD (Pi 4) | DNS | Pi-hole v6 |

## おわりに

月額AI推論コストをほぼゼロにできます。Raspberry Pi AI HAT+ 2（40 TOPS、約$150）が実用化されれば、画像認識レイヤーも追加可能です。
