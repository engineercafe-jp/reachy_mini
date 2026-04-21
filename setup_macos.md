# Reachy Mini シミュレーション環境構築 (macOS)

## 前提条件の確認

| ツール | 確認コマンド | 備考 |
|--------|-------------|------|
| uv | `uv --version` | 未インストールの場合: `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Homebrew | `brew --version` | 未インストールの場合: https://brew.sh |
| Git LFS | `git lfs version` | Step 1 でインストール |

---

## Step 1: Git LFS のインストール

STL などのバイナリアセットは Git LFS で管理されている。先にインストールしておく。

```bash
brew install git-lfs
git lfs install
```

## Step 2: Homebrew で Python 3.12 をインストール

> **重要:** `uv` で管理した Python は `libpython3.12.dylib`（共有ライブラリ）を含まないため、`mjpython` が起動できない。macOS では **Homebrew の Python** を使う必要がある。

```bash
brew install python@3.12
```

インストール後に確認:

```bash
/opt/homebrew/bin/python3.12 --version
```

## Step 3: 仮想環境の作成・有効化

```bash
cd /path/to/reachy_mini   # リポジトリのルートに移動
uv venv .venv --python /opt/homebrew/bin/python3.12
source .venv/bin/activate
```

プロンプトに `(.venv)` が表示されれば成功。

## Step 4: SDK のインストール（MuJoCo 込み）

```bash
uv sync --extra mujoco
```

## Step 5: Git LFS アセットの取得

リポジトリを LFS インストール前にクローンした場合、STL ファイルがポインタのままになっている。以下で実ファイルに差し替える。

```bash
git lfs pull
```

確認（ファイルサイズが数百 KB〜 MB あれば OK）:

```bash
ls -lh src/reachy_mini/descriptions/reachy_mini/mjcf/scenes/../assets/body_down_3dprint.stl
```

## Step 6: macOS GStreamer セグフォルト対策

`mjpython` 起動時に以下のようなクラッシュが発生する既知の問題がある:

```
ERROR: Caught a segmentation fault while loading plugin file:
.../gstreamer-1.0/libgstpython.dylib
```

事前に対策しておく（音声・映像機能への影響なし）:

```bash
mv $(python -c "import gstreamer_python, pathlib; print(pathlib.Path(gstreamer_python.__file__).parent / 'lib/gstreamer-1.0/libgstpython.dylib')") \
   $(python -c "import gstreamer_python, pathlib; print(pathlib.Path(gstreamer_python.__file__).parent / 'lib/gstreamer-1.0/libgstpython_.dylib')")
```

---

## シミュレーションモード比較

| モード | 起動コマンド | 特徴 |
|--------|-------------|------|
| **MuJoCo** (`--sim`) | `mjpython -m reachy_mini.daemon.app.main --sim` | 物理エンジン、3D GUI、カメラシミュレーション |
| **Mockup** (`--mockup-sim`) | `reachy-mini-daemon --mockup-sim` | 軽量・GUI なし、ロジックテスト向け |

---

## 接続エンドポイント

| サービス | URL |
|---------|-----|
| REST API | `http://localhost:8000/api` |
| API ドキュメント | `http://localhost:8000/docs` |
| WebSocket | `ws://localhost:8000` (20 Hz) |

SDK コードの変更は不要。`ReachyMini()` はデフォルトで `localhost` に接続する。

```python
from reachy_mini import ReachyMini
from reachy_mini.utils import create_head_pose

with ReachyMini() as mini:
    mini.goto_target(
        head=create_head_pose(z=20, roll=10, mm=True, degrees=True),
        duration=1.0
    )
    mini.goto_target(antennas=[0.6, -0.6], duration=0.3)
    mini.goto_target(antennas=[-0.6, 0.6], duration=0.3)
```

---

## トラブルシューティング

### `libpython3.12.dylib` が見つからない

```
failed to dlopen path '.../.venv/bin/python3': dlopen(...): Library not loaded: @rpath/libpython3.12.dylib
```

`uv` 管理の Python を使っている場合に発生する。Step 2 に従い Homebrew の Python で venv を再作成する。

### STL ファイルのエラー (`number of faces should be between 1 and 200000`)

```
Error: number of faces should be between 1 and 200000 in STL file '...body_down_3dprint.stl'
```

Git LFS のポインタファイルが実ファイルに差し替わっていない。Step 5 の `git lfs pull` を実行する。

### GStreamer 警告 (`External plugin loader failed`)

起動時に表示される下記警告は無害なので無視してよい:

```
GStreamer-WARNING: External plugin loader failed.
```

---

## 参考ドキュメント

- インストールガイド: `reachy_mini/docs/source/SDK/installation.md`
- シミュレーションガイド: `reachy_mini/docs/source/platforms/simulation/get_started.md`
- トラブルシューティング: `reachy_mini/docs/source/troubleshooting.md`
