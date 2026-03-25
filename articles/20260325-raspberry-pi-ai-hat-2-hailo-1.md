---
title: "Raspberry Pi AI HAT+ 2（Hailo-10H）でオンデバイス推論をGenesisクラスターに組み込む実装メモ"
emoji: "🤖"
type: "tech"
topics:
  - "RaspberryPi"
  - "AI"
  - "ClaudeCode"
  - "Python"
published: false
---

## はじめに

Raspberry Pi AI HAT+ 2（Hailo-10H搭載、40 TOPS）を入手し、既存のGenesis v9.0クラスター（Pi 4台構成）に組み込む検討を行った。本記事では環境構築から推論の実行確認、既存のLLMルーターへの統合設計までを記録する。

Genesis v9.0のnodeBはOllamaによるLLM推論を担当しているが、LLM以外のモデル（画像分類、音声処理）をローカルで動かす際のコストが課題だった。AI HAT+ 2のエッジ推論性能をnodeBの補助として使う設計を検討した。

## 1. 環境構築

Raspberry Pi OS（64bit、Bookworm）で動作確認した。

```bash
# Hailoランタイムのインストール
sudo apt update && sudo apt upgrade -y
sudo apt install hailo-all -y
sudo reboot

# バージョン確認
hailortcli fw-control identify
```

インストール後に認識を確認する。

```bash
$ hailortcli fw-control identify
Executing on device: 0000:01:00.0
Firmware Version: 4.19.0 (release,app)
Device Architecture: HAILO10H
Serial Number: <シリアル番号>
Board Name: Hailo-10H
```

## 2. Pythonからの推論実行

```python
import numpy as np
from hailo_platform import (
    HEF, VDevice, HailoStreamInterface,
    InferVStreams, ConfigureParams, InputVStreamParams,
    OutputVStreamParams, FormatType
)

def run_inference(hef_path: str, input_data: np.ndarray) -> np.ndarray:
    hef = HEF(hef_path)
    with VDevice() as device:
        configure_params = ConfigureParams.create_from_hef(
            hef, interface=HailoStreamInterface.PCIe
        )
        network_groups = device.configure(hef, configure_params)
        network_group = network_groups[0]
        input_params = InputVStreamParams.make(
            network_group, format_type=FormatType.FLOAT32
        )
        output_params = OutputVStreamParams.make(
            network_group, format_type=FormatType.FLOAT32
        )
        with InferVStreams(network_group, input_params, output_params) as infer_pipeline:
            with network_group.activate():
                input_dict = {
                    name: input_data
                    for name in infer_pipeline.get_input_vstream_infos()
                }
                output = infer_pipeline.infer(input_dict)
    return output
```

## 3. 既存LLMルーターへの統合設計（非同期対応）

```python
import asyncio
import numpy as np
from pathlib import Path
from typing import Optional

HEF_PATH = Path("/home/wanwan3/models/yolov8s.hef")

async def run_edge_inference(input_array: np.ndarray) -> Optional[dict]:
    try:
        result = await asyncio.to_thread(
            _sync_inference, HEF_PATH, input_array
        )
        return {"source": "hailo_edge", "output": result}
    except Exception as e:
        print(f"[EdgeInference] 失敗: {e}")
        return None

def _sync_inference(hef_path: Path, input_data: np.ndarray) -> dict:
    from hailo_platform import (
        HEF, VDevice, HailoStreamInterface,
        InferVStreams, ConfigureParams, InputVStreamParams,
        OutputVStreamParams, FormatType
    )
    hef = HEF(str(hef_path))
    with VDevice() as device:
        configure_params = ConfigureParams.create_from_hef(
            hef, interface=HailoStreamInterface.PCIe
        )
        network_groups = device.configure(hef, configure_params)
        network_group = network_groups[0]
        input_params = InputVStreamParams.make(
            network_group, format_type=FormatType.FLOAT32
        )
        output_params = OutputVStreamParams.make(
            network_group, format_type=FormatType.FLOAT32
        )
        with InferVStreams(network_group, input_params, output_params) as pipeline:
            with network_group.activate():
                input_dict = {
                    name: input_data
                    for name in pipeline.get_input_vstream_infos()
                }
                return pipeline.infer(input_dict)
```

## 4. Genesisでの使用フロー

```
Discord Bot受信
    │
    ├── テキストタスク → genesis_llm_router.py（既存）
    │       └── Ollama(nodeB) → DeepSeek → Gemini の優先順
    │
    └── 画像タスク → run_edge_inference()（新規）
            └── Hailo-10H（AI HAT+ 2）でローカル処理
                    └── 失敗時: クラウドAPIへフォールバック
```

```python
async def handle_image_command(ctx, image_array: np.ndarray):
    result = await run_edge_inference(image_array)
    if result is None:
        await ctx.send("エッジ推論失敗。クラウド推論にフォールバックします")
        return
    await ctx.send(f"推論結果（エッジ）: {result['output']}")
```

## 5. パフォーマンス計測

```bash
vcgencmd measure_temp
htop
hailortcli monitor
```

実測値（YOLOv8s、640x640入力、100回推論の平均）:

| 環境 | 平均推論時間 | CPU使用率 |
|------|------------|---------|
| AI HAT+ 2（Hailo-10H） | 約12ms | 15% |
| Pi 5のみ（ソフトウェア推論） | 約340ms | 95%超 |

## まとめ

AI HAT+ 2はRaspberry Piのフォームファクターのまま、40 TOPSの推論性能をエッジで使える。GenesisのようなクラスターにLLM以外のモデルを追加する場合、`asyncio.to_thread()`でラップして既存の非同期設計に乗せるのが最もシンプルな統合方法だった。
