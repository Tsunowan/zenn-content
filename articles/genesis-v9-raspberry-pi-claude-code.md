---
title: "Genesis v9.0 — Raspberry PiクラスターでClaude Codeを自律運用する方法"
emoji: "🍓"
type: "tech"
topics:
  - "RaspberryPi"
  - "ClaudeCode"
  - "AI"
  - "Python"
  - "FastMCP"
published: false
---

## はじめに

Raspberry Pi 4台のクラスターに Claude Code + FastMCP を組み合わせて、完全自律型のコンテンツ生成・投稿エージェントを構築しました。

## 構成

- nodeA (192.168.0.148): Orchestrator — Claude Code + Discord Bot
- nodeB (192.168.0.203): AI Worker — Ollama / Whisper API / OpenSCAD
- nodeC (192.168.0.73): Monitor — 死活監視・自動復旧
- nodeD (192.168.0.131): Pi-hole DNS

## MCPツール



## まとめ

Claude Code の Agent Teams 機能を使うことで、researcher → writer → publisher の3エージェントが並列動作し、毎朝自律的にコンテンツを生成・投稿しています。

