# BLE 遅延問題 調査ノート

## やりたかったこと

Eyelash Corne（ZMK スプリットキーボード）を Bluetooth 接続で使用した際に、
USB 接続と同等のレスポンスで入力したい。

---

## 観察された症状

- BLE 接続時に体感できるキー入力の遅延
- 誤入力（誤った文字が入力される）
- 左右両半分ともに同じ症状が発生

---

## 調査・検証の経緯

### Step 1：BLE 接続インターバルの確認

**仮説**：Linux ホスト側が BLE 接続インターバルを大きな値に設定しており、
それがレイテンシの原因になっている。

**検証**：`btmon` で接続インターバルを確認

```
Connection interval: 180.000 msec  ← 初期状態（非常に大きい）
```

BLE の接続インターバルは 1.25ms 単位で、最小 7.5ms まで設定可能。
180ms はキーボードとして致命的な遅延。

**対応**：ZMK ファームウェアに接続インターバル要求を追加

```conf
# config/eyelash_corne.conf
CONFIG_BT_PERIPHERAL_PREF_MIN_INT=6   # 6 × 1.25ms = 7.5ms
CONFIG_BT_PERIPHERAL_PREF_MAX_INT=6
CONFIG_BT_PERIPHERAL_PREF_LATENCY=0
CONFIG_BT_PERIPHERAL_PREF_TIMEOUT=400
```

**結果**：`btmon` で 7.5ms に改善されたことを確認 ✓

```
Connection interval: 7.50 msec  ← 改善後
```

しかし**遅延は依然として残った**。

---

### Step 2：ディスプレイが原因との特定

**重要な発見**：
`my-setting-01` ブランチ（設定ミスでディスプレイが映らない状態）では
遅延がまったくなく正常動作することが判明。

| 状態 | ディスプレイ | BLE 遅延 |
|------|------------|---------|
| my-setting-01（設定ミス） | 表示なし | **なし** ✓ |
| 現ブランチ | カスタム OLED 表示 | あり |

**仮説**：ディスプレイ処理が BLE スタックに干渉している。

---

### Step 3：LVGL リフレッシュレート削減（失敗）

**仮説**：30fps の LVGL レンダリング + I2C 転送が nRF52840 の
スケジューラを圧迫して BLE 送信タイミングを乱している。

**対応**：

```conf
CONFIG_LV_DISP_DEF_REFR_PERIOD=500  # 30fps → 2fps
```

右側（ペリフェラル）のディスプレイも無効化：

```conf
CONFIG_ZMK_DISPLAY=n
```

**結果**：
- `CONFIG_LV_DISP_DEF_REFR_PERIOD` は効果なし（遅延変わらず）
- 右側 OLED が `=n` にもかかわらず依然として表示されていた

→ **config が上書きされている**ことが発覚。

---

### Step 4：`zmk-nice-oled` モジュールが真の原因と特定

**調査**：`west.yml` を確認し、外部モジュールの存在を確認

```yaml
- name: zmk-nice-oled
  remote: mctechnology17
  revision: main
```

**根本原因の構造**：

```
zmk-nice-oled モジュールの問題点：

1. Kconfig 内で select ZMK_DISPLAY を使用
   → conf ファイルの CONFIG_ZMK_DISPLAY=n を無効化していた

2. カスタムステータス画面がキー入力イベントのたびに
   LVGL オブジェクト更新を同期実行
   → ZMK イベントコールバック内で重い処理
   → HID レポート送信が遅延する
```

`CONFIG_LV_DISP_DEF_REFR_PERIOD` が効かなかった理由も判明：
リフレッシュ頻度ではなく、**イベントコールバック内の処理**が
レイテンシの原因だったため。

**なぜ my-setting-01 では問題なかったか**：
`my-setting-01` は `nice_view` ウィジェット（別ハードウェア向け）を
SSD1306 に誤って適用していたため、カスタム画面が描画されず、
結果としてイベントコールバックの重い処理も実行されていなかった。

---

### Step 5：zmk-nice-oled 削除（解決）

**対応**：

1. `west.yml` から `zmk-nice-oled` モジュールを削除
2. `eyelash_corne_left.conf` でカスタムスクリーンを無効化
3. `build.yaml` から `nice_oled` シールドを削除

```yaml
# west.yml（削除後）
projects:
  - name: zmk
    remote: zmkfirmware
    revision: v0.3.0
    import: app/west.yml
  - name: zmk-board-eyelash   # ← これのみ残す
    remote: eyelash
    revision: main
```

```conf
# eyelash_corne_left.conf
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=n  # ZMK 標準スクリーンを使用
```

```yaml
# build.yaml（修正後）
shield: eyelash_corne_left   # nice_oled を削除
```

**結果**：**BLE 遅延が解消** ✓

---

## 最終的な構成

| 設定項目 | 値 | 理由 |
|---------|-----|------|
| BLE 接続インターバル | 7.5ms | CONFIG_BT_PERIPHERAL_PREF_MIN/MAX_INT=6 |
| ディスプレイ（左） | ZMK 標準スクリーン | zmk-nice-oled 削除 |
| ディスプレイ（右） | ZMK 標準スクリーン | 同上 |
| zmk-nice-oled | 削除 | イベントコールバック内重処理がラグ原因 |

---

## 教訓

- **BLE 接続インターバルと HID レイテンシは別問題**。
  インターバルが 7.5ms でも、ファームウェア側の処理が重ければラグは残る。

- **外部 ZMK モジュールは Kconfig で `select` を使って設定を上書きできる**。
  `CONFIG_X=n` が conf ファイルに書いてあっても効かないことがある。

- **ZMK のカスタム表示モジュールはイベントドリブン**。
  LVGL のリフレッシュレートを下げても、イベントコールバック内の処理は
  リフレッシュレートとは無関係に毎キー入力ごとに実行される。

- **動かないと思っていた設定が実は正解のヒントになることがある**。
  my-setting-01 の「ディスプレイ映らない不具合」が根本原因を特定する
  決定的な手がかりになった。
