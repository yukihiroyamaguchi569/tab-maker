# 音源からギター TAB 譜を作るツール — 設計計画

## Context

- 既存の `index.html`（alphaTab 1.8.4 で alphaTex を表示・再生する試作ページ）を土台に、**mp3 からギターの TAB 譜の下書きを自動生成する**ツールを作る。目的は「完全自動」ではなく「人が直して仕上げる前提の下書き」。
- 入力は市販曲の mp3（ユーザーが用意する。YouTube からの取得はツールに含めない）。主に単音フレーズ、ときどきダブルストップ。
- 開発は MacBook（Apple Silicon、RAM 16 GB、空き 19 GB、Homebrew Python 3.14.7、pyenv / ffmpeg あり、uv なし）。本番は Mac mini（RAM 48 GB）。どちらでも同じ手順で入ること。
- 最初の形は **CLI + 既存ページで表示**。Web UI での編集は後の段階。

## 調査で確定した前提（2026-10-25、PyPI / GitHub / 公式ドキュメントで確認）

| 部品 | 採用 | 根拠・注意 |
|---|---|---|
| 音源分離 | `audio-separator` 0.47.1（`[cpu]` extras、Python ≥3.10） | Demucs `htdemucs_6s`（guitar stem）を内蔵。Demucs 本体はメンテ停止のため入れない。becruily のギター専用 Mel-Band RoFormer（HF: `becruily/mel-band-roformer-guitar`）は未登録で読込可否は未確認 → Phase 0 で試す |
| 採譜 | `basic-pitch` 0.4.0 | **Python ≤3.11 でしか素直に入らない**（`tensorflow-macos<2.15.1` の pin）。`--no-deps` + onnxruntime で 3.13 に入る可能性はあるが未検証。`predict()` の `note_events` は `(start_s, end_s, midi, amplitude, bends)` |
| 拍・テンポ | `librosa` 1.0.0（**Python ≥3.12**） | `beat.beat_track`。madmom / BeatNet は wheel 無し・メンテ停止で不採用 |
| MIDI | `pretty_midi` | Basic Pitch の戻り値型 |
| 出力 | alphaTex 文字列を Python で生成 | alphaTab は MIDI 直接読込なし。`api.tex()` がそのまま使える。保存が要るときは `Gp7Exporter` |
| Python | **3.13** | torch 2.14.1 の arm64 wheel あり、audio-separator / librosa 対応。3.14 は依存の wheel が未確認 |
| パッケージ管理 | **uv**（`brew install uv`） | Python 3.13 の取得と依存の lock を 1 ツールで行え、Mac mini へ `uv sync` で再現できる。pyenv はビルドが要るので使わない |

Basic Pitch と librosa 1.0 の Python 要件が衝突する。**工程間をファイルで受け渡す構成**にして、Basic Pitch だけ別環境（3.11）で `basic-pitch` CLI を subprocess 実行する逃げ道を残す（Phase 0 で単一環境に収まるか確定）。

## アーキテクチャ

```
mp3 ──separate──▶ work/<song>/stems/guitar.wav
                 ──transcribe──▶ work/<song>/notes.csv   (start, end, midi, amplitude)
                 ──beats──▶      work/<song>/beats.json  (tempo, beat_times)
                 ──tab──▶        work/<song>/<song>.alphatex
index.html（ファイル選択 or ?tex=URL で読み込み）──▶ alphaTab で表示・再生
```

- `separate` / `transcribe` は外部ツールの薄い adapter（テストではモック）。
- `tab` は **副作用のない純粋関数の塊**（量子化・運指・alphaTex 書き出し）。ここに単体テストを集中させる。
- 中間ファイルは全部 `work/` に置き、`.gitignore`（mp3 は著作物、stem は巨大）。

### リポジトリ構成（既存 `tab-maker` に同居）

```
tab-maker/
  index.html              # 表示ページ（読み込み機能を追加）
  README.md
  pyproject.toml          # uv が生成。requires-python = ">=3.13,<3.14"
  uv.lock
  Makefile                # verify: ruff check / ruff format --check / pytest
  tabmaker/
    __init__.py
    cli.py                # argparse: separate / transcribe / tab / run（全工程）
    separate.py           # audio-separator adapter
    transcribe.py         # basic-pitch adapter（API 直叩き or subprocess）
    beats.py              # librosa でテンポ・拍列
    quantize.py           # 純粋関数: note_events + beats → 拍グリッド上の音符列
    fingering.py          # 純粋関数: MIDI 列 → (string, fret) 列（DP）
    alphatex.py           # 純粋関数: 音符列 → alphaTex 文字列
  tests/
    test_quantize.py  test_fingering.py  test_alphatex.py  test_cli.py
  work/                   # gitignore
```

### 中核アルゴリズム（`tab` 工程）

1. **フィルタ**: ギター音域（MIDI 40=E2 〜 88=E6、`--max-fret` で可変）外と、`amplitude < --min-confidence`（既定 0.3 あたり、Phase 4 で調整）の音を捨てる。`--min-length-ms` 未満も捨てる。
2. **量子化**: `beats.json` の拍列から 16 分グリッドを作り、開始・終了を最寄りのグリッドへ丸める（最小 16 分）。同じ開始グリッドの音は和音（ダブルストップ）にまとめる。隙間は休符、重なりは前の音を切る。
3. **小節割り**: 拍子は 4/4 固定（`--time-signature` で変更可）。小節をまたぐ音はタイ `-` で分ける。先頭の拍のズレは `--offset-beats` で補正（librosa は小節頭を検出しないため）。テンポは拍間隔の中央値。
4. **運指（DP）**: 標準チューニング固定（`--tuning` は後回し）。各音の候補 (string, fret) を列挙し、コスト = 前の押さえ位置からのフレット移動 + 高フレットの罰則 + 弦の飛びの罰則、を動的計画法で最小化。和音は候補の組（弦が重複せず、フレット幅 ≤ 4）で同じことをする。
5. **alphaTex 書き出し**: `\title` `\tempo` `\ts` `\tuning` → 小節ごとに `:16` を基準に長さを切り替えつつ `fret.string` / `(a.b c.d)` / `r` / `-` を並べる。alphaTex の弦番号の向き（1 弦 = 高音 E か）は **Phase 0 で描画して実測**し、確定値を `alphatex.py` の定数コメントに記す。

## 実装フェーズ（各フェーズ 1 Issue + 1 PR、Opus サブエージェントに委譲、codex レビュー後に報告）

### Phase 0: 環境の確定（スパイク。成否で以降の設計が変わる）
1. `brew install uv` → `uv init` → `uv python pin 3.13` → `uv add "audio-separator[cpu]" librosa pretty_midi` → `uv add --dev pytest ruff`。
2. Basic Pitch の入れ方を順に試し、**最初に動いた方式を採用**:
   - (a) `uv add basic-pitch --no-deps` + `uv add onnxruntime resampy` → `predict()` が動くか
   - (b) 動かなければ `uv python install 3.11` で別 venv（`tools/basic-pitch-env/`）を作り、`basic-pitch` CLI（`--save-note-events`）を subprocess で呼ぶ。`transcribe.py` はこの差を吸収する。
3. 動作確認は著作物を使わず、**numpy で生成した正弦波（E2・A2・D3 を 0.5 秒ずつ）** を wav にして Basic Pitch に通し、MIDI 40/45/50 が出ることを確認。audio-separator は同じ wav を `htdemucs_6s` で通して stem が 6 本出ることだけ確認（分離品質は Phase 4）。
4. becruily の ckpt + yaml を audio-separator に読ませられるか 1 回試す（不可なら Phase 4 の比較候補から外す）。
5. alphaTex の弦番号の向きを `index.html` に 1 音書いて実測。
6. 結果（採用方式、所要時間、ディスク使用量）を `docs/environment.md` に記す。

### Phase 1: 中核ロジック（純粋関数 + テスト）
- `quantize.py` `fingering.py` `alphatex.py` を TDD で実装。テストは正常系・境界（小節境界のタイ、同時発音、音域外、極短音、空入力）・エラー（拍列が空）。
- `make verify` が通ること。

### Phase 2: adapter と CLI
- `separate.py` `transcribe.py` `beats.py` `cli.py`。外部ツールは差し替え可能にし、`test_cli.py` では Fake に置き換えて `run` が alphaTex を書き出すことを確認。
- `tabmaker run song.mp3` で `work/song/song.alphatex` が出る。

### Phase 3: 表示ページ
- `index.html`: `<input type="file">` と `?tex=<URL>` の 2 経路で alphaTex を読み込み `api.tex()`。読み込みがないときは今のデモ譜面を表示。既存の CSS・再生は変えない。
- README に使い方（`uv sync` → `uv run tabmaker run ...` → `python3 -m http.server` → ページで開く）を追記。

### Phase 4: ベンチマークとパラメータ調整
- ユーザーが用意する mp3 1〜2 曲で `htdemucs_6s`（可能なら becruily も）を比較。stem を耳で確認、生成 TAB を目視し、しきい値（confidence、min length）と `--offset-beats` の既定値を決める。
- 結果と限界（キーボードの混入、奏法は手付け、など）を `docs/benchmark.md` に記す。mp3 と stem はコミットしない。

## 検証（end-to-end）
1. `make verify`（ruff + pytest）が緑。
2. 合成音 wav → `tabmaker run` → 期待どおりの alphaTex（E2 A2 D3 が 0.6 / 0.5 / 0.4 のいずれかの弦・フレットで、4 分音符として並ぶ）。
3. 生成した `.alphatex` を `http://localhost:8000/?tex=work/<song>/<song>.alphatex` で開き、エラーなく描画・再生できる（ブラウザで確認）。
4. Phase 4: 実曲でユーザーが聞き比べ、下書きとして使えるかを判断。

## 規約・運用
- `~/.claude/rules/code-quality.md` に従う。依存のバージョンは `uv add` に決めさせ、手書きしない。
- Python は型ヒント必須、`ruff` で lint/format。ファイル 400 行 / 関数 60 行を目安に分割。
- 各 Phase を feature ブランチ → Issue → PR（User Prompt セクション付き、末尾に `work in tab-maker`）。マージ済みブランチは残す。
- 想定外（Basic Pitch が両方式とも動かない等）の場合は止まって再計画する。代替候補は torchcrepe / pesto（単音のみ、ノート分割を自作）。
