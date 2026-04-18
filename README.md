# Reachy Mini

## 概要

Reachy Miniのシミュレーション環境を構築し、SDKの機能を検証する。

## シミュレーション起動

### 初回セットアップ

[setup.md](./setup.md) の手順に従って環境を構築する。

### 起動コマンド

```bash
cd reachy_mini
source .venv/bin/activate

# デフォルト（ロボットのみ）
mjpython -m reachy_mini.daemon.app.main --sim

# テーブル + オブジェクトあり
mjpython -m reachy_mini.daemon.app.main --sim --scene minimal
```

> **macOS の注意点:** `reachy-mini-daemon --sim` ではなく `mjpython` を使う必要がある。

3D ウィンドウが開いたら起動完了。マウスで視点操作できる（ドラッグ: 回転、右クリック: パン、スクロール: ズーム）。

### 接続確認

別のターミナルで:

```bash
cd reachy_mini
source .venv/bin/activate
python examples/minimal_demo.py
```

### エンドポイント

| サービス | URL |
|---------|-----|
| REST API | http://localhost:8000/api |
| API ドキュメント | http://localhost:8000/docs |
| WebSocket | ws://localhost:8000 (20 Hz) |