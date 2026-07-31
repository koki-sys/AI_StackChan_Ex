# ステアリングファイル: 対話開始時の電子音の調整と、録音停止時の「ブブ」音の追加

対話開始時（音声認識開始時）に鳴る電子音を、標準の高さの「ソ・ラ（G5: 784Hz、A5: 880Hz）」の2音上昇メロディ（テンポ速め）にし、さらにその音量を現在システム設定の5割（50%）に調整します。
また、リアルタイム対話中に画面タッチで音声認識（録音）を停止した際には、拒否音として「ブ・ブ（400Hz・100ms x 2）」を5割の音量で鳴らします。

## 目的
対話開始の操作時には「ソラッ♪」、録音停止（対話終了・キャンセル）の操作時には「ブブッ」と鳴るようにし、機器の状態変化を直感的に分かりやすくします。いずれの音も現在システム音量の5割で鳴らし、喋り声の大きさには影響を与えません。

## 対象範囲
* `src/main.cpp`
* `src/mod/AiStackChan/RealtimeAiMod.cpp`

## 主な変更ファイル
* [main.cpp](file:///Users/kokisys/dev/AI_StackChan_Ex/firmware/src/main.cpp)
* [RealtimeAiMod.cpp](file:///Users/kokisys/dev/AI_StackChan_Ex/firmware/src/mod/AiStackChan/RealtimeAiMod.cpp)

## 実装方針

### `main.cpp` への `err_tone()` の追加
「ブブ」という音を鳴らすための `err_tone()` 関数を新規追加します。

```cpp
void err_tone()
{
  enterMutexAudio();
  M5.Mic.end();
  M5.Speaker.begin();
  delay(300);     // AtomS3Rはこのdelayがないと鳴らないときがある

  // 現在の音量を退避し、一時的に5割に下げます
  uint8_t org_vol = M5.Speaker.getVolume();
  M5.Speaker.setVolume(org_vol * 0.5);

  M5.Speaker.tone(400, 100);  // 低い音（400Hz）を0.1秒
  delay(120);                 // わずかな隙間
  M5.Speaker.tone(400, 100);  // 低い音（400Hz）を0.1秒
  delay(150);                 // 音が鳴り終わるのを待つ

  // 音量を元の値に戻します
  M5.Speaker.setVolume(org_vol);

  M5.Speaker.end();
  M5.Mic.begin();
  exitMutexAudio();
}
```

### `RealtimeAiMod.cpp` での条件分岐
録音中かどうかを `pRtLLM->isRealtimeRecording()` で判断し、`true` なら `err_tone()`、`false` なら `sw_tone()` を呼ぶように変更します。

```cpp
extern void sw_tone();
extern void err_tone(); // 追加

// ...

void RealtimeAiMod::display_touched(int16_t x, int16_t y)
{
  // ...
  if (!touched_button)
  {
    if (pRtLLM->isRealtimeRecording()) {
      err_tone(); // 録音停止（Listening終了）時はブブ
    } else {
      sw_tone();  // 録音開始時はソラ
    }
    toggleRealtimeRecord();
  }
}
```

## 確認方法
* PlatformIO で `m5stack-core2-realtime` 環境のビルド確認を行います。
* 実機に書き込み、リアルタイム対話モードにおいて：
  * 通常時に画面タッチした際、「ソラッ♪」と鳴ることを確認します。
  * 「Listening...」と表示されている（録音中）時に画面タッチした際、「ブブッ」と鳴って録音が止まることを確認します。
