# Prospector ドングル統合ガイド（日本語版）

## 概要

このガイドは、任意のZMKキーボード設定にProspectorドングル機能を統合するためのステップバイステップの手順を提供します。moNa2キーボードでの成功事例に基づき、よくある落とし穴を回避し、ZMK Studio互換性を持つ動作するドングルモードを実現できます。

## 前提条件

- 既存のZMKキーボード設定リポジトリ
- ZMK設定構造の基本的な理解
- キーボードプロジェクトのGitリポジトリアクセス権

## ハードウェア要件

- **マイコン**: Seeed Studio XIAO nRF52840（または互換品）
- **ディスプレイ**: 1.69インチ円形ST7789V LCD（240x240ピクセル）
- **配線構成**:
  ```
  LCD_DIN  -> Pin 10 (MOSI)
  LCD_CLK  -> Pin 8  (SCK)
  LCD_CS   -> Pin 9  (CS)
  LCD_DC   -> Pin 7  (Data/Command)
  LCD_RST  -> Pin 3  (Reset)
  LCD_BL   -> Pin 6  (Backlight PWM)
  ```

## ステップバイステップ統合手順

### ステップ1: West ManifestにProspectorモジュールを追加

**ファイル**: `config/west.yml`

west.ymlマニフェストにprospector-zmk-moduleを追加します：

```yaml
manifest:
  remotes:
    # 既存のremotes...
    - name: carrefinho                            # このリモートを追加
      url-base: https://github.com/carrefinho     # prospectorモジュール用

  projects:
    # 既存のprojects...
    - name: prospector-zmk-module                 # このプロジェクトを追加
      remote: carrefinho
      revision: main

  self:
    path: config
```

**完全な追加例**:
```diff
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
+   - name: carrefinho
+     url-base: https://github.com/carrefinho
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
+   - name: prospector-zmk-module
+     remote: carrefinho
+     revision: main
  self:
    path: config
```

### ステップ2: ビルド設定を更新

**ファイル**: `build.yaml`

ビルドマトリックスにドングルビルド設定を追加します：

```yaml
include:
  # 既存のキーボードビルド...
  - board: seeeduino_xiao_ble
    shield: [YOUR_KEYBOARD]_L rgbled_adapter
  - board: seeeduino_xiao_ble
    shield: [YOUR_KEYBOARD]_R rgbled_adapter
  
  # このドングル設定を追加
  - board: seeeduino_xiao_ble
    shield: [YOUR_KEYBOARD]_dongle prospector_adapter
    snippet: studio-rpc-usb-uart  # ZMK Studio互換性に必須
```

### ステップ3: 物理レイアウトをZMK Studio対応にする

**ファイル**: `config/boards/shields/[YOUR_KEYBOARD]/[YOUR_KEYBOARD].dtsi`

**重要な変更**: ZMK Studio互換性のため、chosenブロックから`zmk,matrix_transform`を削除：

```diff
/ {
    chosen {
        zmk,kscan = &kscan0;
-       zmk,matrix_transform = &default_transform;  // この行を削除！
        zmk,physical-layout = &default_layout;
    };
```

**ZMK Studioに必要なヘッダーとレイアウトを追加**:

```diff
+ #include <dt-bindings/zmk/matrix_transform.h>
+ #include <physical_layouts.dtsi>

/ {
    chosen {
        zmk,kscan = &kscan0;
        zmk,physical-layout = &default_layout;  // これは保持
    };

+   default_layout: default_layout {
+       compatible = "zmk,physical-layout";
+       display-name = "Default Layout";
+       transform = <&default_transform>;
+
+       keys  //                     w   h    x    y     rot    rx    ry
+           = <&key_physical_attrs 100 100    0   61       0     0     0>
+           , <&key_physical_attrs 100 100  100   28       0     0     0>
+           // ... すべてのキー位置をここに追加
+           ;
+   };

    default_transform: keymap_transform_0 {
        compatible = "zmk,matrix-transform";
        columns = <11>;  // キーボードに合わせて調整
        rows = <4>;      // キーボードに合わせて調整
        map = <
            // 既存のキーマッピング
        >;
    };
```

### ステップ4: ドングル用Kconfig定義を追加

**ファイル**: `config/boards/shields/[YOUR_KEYBOARD]/Kconfig.shield`

```kconfig
# 既存のキーボード設定...

config SHIELD_[YOUR_KEYBOARD]_DONGLE
    def_bool $(shields_list_contains,[YOUR_KEYBOARD]_dongle)
```

**ファイル**: `config/boards/shields/[YOUR_KEYBOARD]/Kconfig.defconfig`

```kconfig
# ドングル設定ブロックを追加
if SHIELD_[YOUR_KEYBOARD]_DONGLE

config ZMK_KEYBOARD_NAME
    default "[YOUR_KEYBOARD]"

config ZMK_SPLIT_ROLE_CENTRAL
    default y

config ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS
    default 2

endif

# 既存のキーボード設定...

if SHIELD_[YOUR_KEYBOARD]_DONGLE || SHIELD_[YOUR_KEYBOARD]_L || SHIELD_[YOUR_KEYBOARD]_R

config ZMK_SPLIT
    default y

endif
```

### ステップ5: ドングル用Device Tree Overlayを作成

**ファイル**: `config/boards/shields/[YOUR_KEYBOARD]/[YOUR_KEYBOARD]_dongle.overlay`

```dts
#include "[YOUR_KEYBOARD].dtsi"

/ {
    chosen {
        zmk,kscan = &mock_kscan;  // kscanのみオーバーライド
    };

    mock_kscan: kscan_1 {
        compatible = "zmk,kscan-mock";
        columns = <0>;
        rows = <0>;
        events = <0>;
    };
};

// 不完全な設定の元のkscanを無効化
&kscan0 {
    status = "disabled";
};
```

### ステップ6: ドングル設定ファイルを作成

**ファイル**: `config/boards/shields/[YOUR_KEYBOARD]/[YOUR_KEYBOARD]_dongle.conf`

```conf
# forager実装に基づくドングル設定
# CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS=2
CONFIG_ZMK_STUDIO=y
```

### ステップ7: ドングル用キーマップを作成

**ファイル**: `config/boards/shields/[YOUR_KEYBOARD]/[YOUR_KEYBOARD]_dongle.keymap`

```dts
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>

/ {
    keymap {
        compatible = "zmk,keymap";

        default_layer {
            bindings = <&none>;  // ドングル用の最小限のキーマップ
        };
    };
};
```

### ステップ8: .gitignoreを更新（オプション）

**ファイル**: `.gitignore`

```gitignore
# まだ存在しない場合はビルド成果物を追加
*.uf2
build/
.vscode/
```

## 成功の決定的要因

### 1. ZMK Studio互換性

**最も重要な変更**: メインの`.dtsi`ファイルのchosenブロックから`zmk,matrix_transform`を削除します。これはZMK Studioの`USE_PHY_LAYOUTS`アサーションを通過するために必要です。

**なぜこれが重要か**: ZMK Studioは、chosen matrix transformsなしの物理レイアウトを必要とします。`zmk,matrix_transform`がchosenブロックにある場合、アサーション`!IS_ENABLED(CONFIG_ZMK_STUDIO) || USE_PHY_LAYOUTS`は失敗します。

### 2. 物理レイアウト定義

以下を含む適切な物理レイアウト定義を確認：
- `compatible = "zmk,physical-layout"`
- 実際のキー位置を持つ`keys`プロパティ
- `transform = <&default_transform>`参照

### 3. ビルド設定

- ZMK Studio互換性のため`studio-rpc-usb-uart`スニペットを使用
- `[YOUR_KEYBOARD]_dongle`と`prospector_adapter`の両方のシールドを含める
- `seeeduino_xiao_ble`ボードをターゲット

## 検証手順

### 1. ビルド成功の確認

変更後、GitHubにプッシュして以下を確認：
- GitHub Actionsビルドが正常に完了
- アーティファクト`[YOUR_KEYBOARD]_dongle-seeeduino_xiao_ble-zmk.uf2`が生成される
- ビルドログにZMK Studioアサーションエラーがない

### 2. ハードウェアテスト

1. 生成された`.uf2`ファイルをSeeed Studio XIAO nRF52840にフラッシュ
2. 配線構成に従って1.69インチ円形LCDディスプレイを接続
3. デバイスの電源を入れる
4. ディスプレイにキーボード情報（レイヤー名、ステータスなど）が表示されることを確認

### 3. 期待される結果

- ディスプレイが初期化され、prospectorインターフェースを表示
- キーボードのレイヤー名が画面に表示される
- ビルドエラーやアサーション失敗がない
- クリーンなGitHub Actionsビルドプロセス

## よくある問題のトラブルシューティング

### 問題1: ZMK Studioアサーションエラー

**エラー**: `"ISSUE FOUND: Keyboards require additional configuration to allow for firmware with ZMK Studio enabled"`

**解決策**: メインの`.dtsi`ファイルのchosenブロックから`zmk,matrix_transform = &default_transform;`を削除。

### 問題2: Devicetreeコンパイルエラー

**エラー**: 様々なdevicetree解析エラー

**解決策**: 
- すべてのincludeが存在することを確認（`#include <physical_layouts.dtsi>`）
- 物理レイアウトの`keys`プロパティが適切にフォーマットされていることを確認
- すべての参照（`&default_transform`）が定義されていることを確認

### 問題3: シールドが見つからないエラー

**エラー**: ビルド中にシールドが見つからない

**解決策**:
- `Kconfig.shield`に適切なシールド定義があることを確認
- `Kconfig.defconfig`に対応するシールド設定があることを確認
- すべての必要なファイルが作成され、適切に命名されていることを確認

### 問題4: ビルド成果物が生成されない

**エラー**: ビルドは成功するがドングルファームウェアが生成されない

**解決策**:
- `build.yaml`に正しいシールドの組み合わせがあることを確認
- `seeeduino_xiao_ble`ボードが指定されていることを確認
- `studio-rpc-usb-uart`スニペットが含まれていることを確認

## ファイル構造の例

このガイドに従った後、リポジトリは以下の構造になるはずです：

```
your-zmk-config/
├── .gitignore
├── build.yaml                                    # 更新済み
├── config/
│   ├── west.yml                                 # 更新済み
│   └── boards/shields/[YOUR_KEYBOARD]/
│       ├── Kconfig.defconfig                    # 更新済み
│       ├── Kconfig.shield                       # 更新済み  
│       ├── [YOUR_KEYBOARD].dtsi                 # 更新済み（重要！）
│       ├── [YOUR_KEYBOARD]_dongle.conf          # 新規
│       ├── [YOUR_KEYBOARD]_dongle.keymap        # 新規
│       ├── [YOUR_KEYBOARD]_dongle.overlay       # 新規
│       └── ... (既存のキーボードファイル)
└── modules/                                     # 新規（gitサブモジュール）
    └── prospector-zmk-module/
```

## 成功基準

統合が成功したことは以下で確認できます：

1. ✅ GitHub Actionsビルドがエラーなしで完了
2. ✅ アーティファクトにドングルファームウェア（.uf2）が生成される
3. ✅ ビルドログにZMK Studioアサーションエラーがない
4. ✅ ハードウェアがキーボード情報を正しく表示
5. ✅ レイヤー名とステータスが円形LCDに表示される

## リファレンス実装

このガイドは成功したmoNa2実装に基づいています：
- **リポジトリ**: https://github.com/t-ogura/zmk-config-moNa2
- **成功コミット**: `9205d0a`
- **参考**: https://github.com/carrefinho/forager-zmk-module-dongle

## サポートと追加リソース

- **ZMKドキュメント**: https://zmk.dev/docs/features/studio
- **Prospectorモジュール**: https://github.com/carrefinho/prospector-zmk-module
- **ZMK Studioガイド**: https://zmk.dev/docs/features/studio#adding-zmk-studio-support-to-a-keyboard

---

**注記**: このガイドは、数ヶ月にわたるデバッグとテストから得られた知識を凝縮したものです。重要な洞察は、ZMK Studio互換性には、chosenデバイスツリーブロックから`zmk,matrix_transform`を削除する必要があるということです - この1行の変更が複雑なビルドアサーションエラーを解決します。