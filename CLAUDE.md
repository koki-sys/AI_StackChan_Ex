# CLAUDE.md - AI Stack-chan Ex (M5Stack Core2)

このドキュメントは、Claude Code が本プロジェクト（AI Stack-chan Ex）の開発コンテキスト、ビルドコマンド、設計仕様を理解するためのガイドラインです。

---

## 🚀 ビルド＆書き込みコマンド

コマンドを実行する際は、原則として `firmware/` ディレクトリ内で実行します。

- **ファームウェアのビルド**:
  ```bash
  cd firmware
  ~/.platformio/penv/bin/pio run
  # パスが通っている場合:
  pio run
  ```

- **特定の環境指定ビルド**:
  ```bash
  cd firmware
  pio run -e m5stack-core2-realtime
  ```

- **M5Stack 実機への書き込み（フラッシュ）**:
  ```bash
  cd firmware
  pio run -t upload -e m5stack-core2-realtime
  ```

- **シリアルモニターの起動**:
  ```bash
  cd firmware
  pio device monitor -b 115200
  ```

---

## 🧩 プロジェクト構成と概要

本リポジトリ（`AI_StackChan_Ex`）は、ESP32 / M5Stack（主に M5Stack Core2 向けにチューニング）上で動作する **AIスタックチャン（AI Stack-chan）** の機能拡張版ファームウェアです。

### 主要ディレクトリ構造
- `firmware/`: PlatformIO プロジェクトのルート
  - `firmware/platformio.ini`: PlatformIO のビルド設定、ライブラリ依存関係、環境定義
  - `firmware/src/main.cpp`: エントリーポイント（初期化、WiFi/NTP接続、メインループ `loop()`）
  - `firmware/src/Robot.cpp`: ロボット管理クラス（サーボ、TTS、STT、LLMの初期化および制御）
  - `firmware/src/mod/`: モジュール化された動作モード（`RealtimeAiMod`, `AiStackChanMod`, `StatusMonitorMod` など）
  - `firmware/src/llm/`: LLM連携モジュール（ChatGPT, Gemini Live, Module-LLM）
  - `firmware/src/tts/` & `firmware/src/stt/`: 音声合成（TTS）および音声認識（STT）ドライバ
- `Copy-to-SD/`: SDカード配置用のデフォルト設定ファイル（YAML設定、フォント、画像）

---

## 🛠️ 設計および設定上の重要事項

### 1. サーボモーター制御 (`-DUSE_SERVO`)
- サーボ制御コードは `platformio.ini` の `-DUSE_SERVO` ビルドフラグで切り替えられます。
- **サーボなし（顔だけ）構成**: モーターを接続せず画面表示・対話のみで動作させる場合、`platformio.ini` 内の `;-DUSE_SERVO` をコメントアウトします。これにより未接続サーボ基板の検出処理によるフリーズを完全に回避できます。

### 2. WiFi & NTP時刻同期 (`time_sync`)
- `main.cpp` の NTP同期処理は `sntp_get_sync_status()` による最大5秒間の同期待機ループと、`getLocalTime(&timeInfo, 2000)` による2秒間のタイムアウトを設定しています。
- これにより、WiFiやNTP通信が不安定な環境でも起動時に無限にフリーズすることなく安全に進みます。

### 3. タッチUIとイベントハンドリング (`display_touched`)
- タッチイベントは `main.cpp` の `loop()` から `mod->display_touched(x, y)` へ配信されます。
- `RealtimeAiMod` および `AiStackChanMod` では、機能ボタン（QRコード表示等）以外の画面全体（アバターの顔の中央など）へのタッチで直感的に音声対話（録音/STT）が開始するよう設計されています。

### 4. M5Unified ライブラリの利用
- ハードウェア制御全般には `M5Unified` を使用しています (`M5.begin()`, `M5.Display`, `M5.Touch`, `M5.Speaker`, `M5.Power`)。

---

## 📝 コードスタイルおよび開発ガイドライン

- **言語**: C++ (Arduino Framework / ESP-IDF / M5Unified)
- **ログ出力方針**: 起動プロセスや主要イベントの可視化のため、`Serial.println` に加えて `M5.Lcd.println` による液晶画面へのログ出力を並行して行います。
- **マルチスレッド・排他制御**: 音声再生やバックグラウンドタスク処理には FreeRTOS タスクおよびミューテックス (`enterMutexAudio()`, `exitMutexAudio()`) を使用して安全に排他制御を行います。
