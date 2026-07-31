# M5Stack Core2 (AI Stack-chan Ex) 修正履歴・対応内容まとめ

これまでに実施したトラブルシューティングおよび修正内容のまとめです。

---

## 1. 起動時のフリーズ問題（FTP server started直後）

### 概要
起動処理中に「FTP server started」と画面・シリアルに表示された後、処理が先に進まず完全にフリーズする現象。

### 原因
`firmware/src/main.cpp` 内の `time_sync`（NTP時刻同期）処理において、`getLocalTime()` がタイムアウト設定なしで呼び出されていたため、WiFi接続やDNS解決・NTP通信が不安定な環境で無限に（あるいは非常に長時間）スレッドをブロックしていました。

### 修正内容
* **[main.cpp](file:///Users/kokisys/dev/AI_StackChan_Ex/firmware/src/main.cpp)**:
  * `<esp_sntp.h>` をインクルード追加。
  * `time_sync` 関数内に `sntp_get_sync_status()` を用いた同期待機ループ（最大5秒）と、`getLocalTime(&timeInfo, 2000)` による2秒間の明示的タイムアウト処理を追加。
  * タイムアウト時や通信失敗時でもフリーズせず、`NTP Sync Error` をログ出力して次の処理へ安全に進むよう変更。

---

## 2. サーボモーター初期化によるフリーズ問題（Robot Const: Init Servo...）

### 概要
NTP同期をタイムアウトで安全に通過できるようになった後、ログが `Robot Const: Init Servo...` の表示で停止する現象。

### 原因
サーボモーターを使用しない「顔だけ（画面のみ）」の構成で動作させようとしていたものの、ビルド設定で `-DUSE_SERVO` が有効になっていたため、`Robot::Robot` のサーボ初期化コード（未接続のシリアルサーボ基板を検索する無限ループ等）が実行されフリーズしていました。

### 修正内容
* **[platformio.ini](file:///Users/kokisys/dev/AI_StackChan_Ex/firmware/platformio.ini)**:
  * `[env:m5stack-core2]` の `build_flags` 内から `-DUSE_SERVO` をコメントアウト（`; -DUSE_SERVO`）。
  * サーボの初期化および制御コードを完全にコンパイル対象外とし、サーボなし構成で安全かつ高速に起動するように修正。

---

## 3. 画面タッチが反応しない問題（Please touch表示から進まない）

### 概要
顔（アバター）が表示され「Please touch」とふきだしが出ているにもかかわらず、顔の中央などをタッチしても「ピッ」と音が鳴らず、音声対話が開始されない現象。

### 原因
`RealtimeAiMod.cpp` および `AiStackChanMod.cpp` におけるタッチ判定エリア（`box_stt`）の設計が、画面最上部（y座標: 0〜60）のみに限定されていたため。

### 修正内容
* **[RealtimeAiMod.cpp](file:///Users/kokisys/dev/AI_StackChan_Ex/firmware/src/mod/AiStackChan/RealtimeAiMod.cpp)** & **[AiStackChanMod.cpp](file:///Users/kokisys/dev/AI_StackChan_Ex/firmware/src/mod/AiStackChan/AiStackChanMod.cpp)**:
  * `display_touched` 関数を修正。
  * 機能ボタン（QRコード切替など）以外の画面全体（顔の中央など）へのタッチで、直感的に音声対話（録音/STT）が開始するようにタッチ認識範囲を拡張。

---

## 4. デバッグログの充実化

### 修正内容
* **[main.cpp](file:///Users/kokisys/dev/AI_StackChan_Ex/firmware/src/main.cpp)** & **[Robot.cpp](file:///Users/kokisys/dev/AI_StackChan_Ex/firmware/src/Robot.cpp)**:
  * 起動時の各初期化ステップ（`Initializing Robot...`、`Initializing MP3...`、`Initializing Avatar...` など）の前後に `M5.Lcd.println` および `Serial.println` を追加。
  * PCと接続してシリアルモニターを見なくても、M5Stack Core2の液晶画面のみで起動中の進捗や停止箇所をひと目で確認できるように変更。

---

## 5. 参考：今後の発展（SwitchBot / Smart Home連携）

* **Gemini Live API からの家電操作**:
  * Gemini Live API の **Function Calling (ツール呼び出し)** 機能を介して外部のWeb APIを実行することで、音声操作が可能。
* **SwitchBot連携**:
  * SwitchBotが公開している公式Web API（Open API）を利用し、M5StackからHTTPSリクエストを送信することで、カーテン・プラグ・ボット・ロック等のSwitchBotデバイスをスタックチャンの音声操作に対応させることが可能。
