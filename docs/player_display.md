# player_display の仕組み

`assets/player_display/` は、**プレイヤースキンを身体パーツ単位で世界に表示する**ための
カスタムアイテムモデルを定義する名前空間です（Minecraft 1.21.4 以降のアイテムモデル定義機能を利用）。

頭キューブ 1 個を 8 通りに変形して各身体パーツに見せ、`item_display` エンティティで
組み立てることで、等身大のプレイヤー像を再現します。

## ファイル構成

```
assets/player_display/
├── items/
│   └── core.json          エントリーポイント（アイテムモデル定義）
└── models/
    ├── head.json          頭
    ├── torso.json         胴
    ├── left_arm.json      左腕（通常）
    ├── right_arm.json     右腕（通常）
    ├── slim_left.json     左腕（Slim スキン用）
    ├── slim_right.json    右腕（Slim スキン用）
    ├── left_leg.json      左脚
    └── right_leg.json     右脚
```

## 1. エントリーポイント `items/core.json`

アイテムに `minecraft:item_model = "player_display:core"` コンポーネントを付けると、
このファイルが使用されます。

```
type: select  →  property: custom_model_data
```

アイテムの `minecraft:custom_model_data` の **文字列値** によって 8 つの `cases` を分岐します。

| `when` 値                | 描画されるパーツ          |
| ------------------------ | ------------------------- |
| `head`                   | 頭                        |
| `torso`                  | 胴                        |
| `left_arm` / `right_arm` | 通常腕（左右）            |
| `slim_left` / `slim_right` | 細腕（Slim スキン用、左右） |
| `left_leg` / `right_leg` | 脚（左右）                |

どの case にも一致しない場合は `fallback` の `item/flower_banner_pattern`
（バニラの旗の模様）が表示されます。設定ミスを検知するためのマーカーです。

各 case の中身は共通パターンです。

```json
{
  "type": "special",
  "base":  "player_display:<part>",
  "model": { "type": "player_head" }
}
```

- `model` … `player_head` 特殊レンダラ。アイテムの `minecraft:profile` コンポーネントから
  スキンを取得し、**頭のキューブ**を描画する。
- `base` … 形状・配置を決めるベースモデル。描画された頭をパーツの形・位置に変形する。

## 2. ベースモデル `models/*.json` — 変形のからくり

8 ファイルすべて `parent: builtin/entity` で、中身は `display.thirdperson_righthand` の
**`scale` / `translation` のみ**です。つまり「頭キューブ」を引き伸ばして
各パーツに化けさせています。

`head` を 8×8×8 px の基準とすると、各パーツのパラメータは次の通りです。

| パーツ              | `scale` (X, Y, Z)             | 実寸法 (px) | `translation` (X, Y, Z) |
| ------------------- | ----------------------------- | ----------- | ----------------------- |
| head                | 0.9375, 0.9375, 0.9375        | 8×8×8       | 0, 7.5, 0               |
| torso               | 0.9375, 1.40625, 0.46875      | 8×12×4      | 0, 0, 0（原点＝基準）   |
| left_arm / right_arm | 0.46875, 1.40625, 0.46875     | 4×12×4      | ∓5.63, 2.5, 0           |
| slim_left / slim_right | 0.3515625, 1.40625, 0.46875 | 3×12×4      | ∓5.15, 2.5, 0           |
| left_leg / right_leg | 0.46875, 1.40625, 0.46875     | 4×12×4      | ∓1.88, 0, 0             |

### scale の検証

`scale` の比はスキンモデルの実寸と完全に一致します。

- Y: `1.40625 / 0.9375 = 1.5` → 胴の高さ 12 = 頭 8 × 1.5
- Z: `0.46875 / 0.9375 = 0.5` → 胴の奥行 4 = 頭 8 × 0.5
- X（通常腕）: `0.46875 / 0.9375 = 0.5` → 腕幅 4 = 頭 8 × 0.5
- X（細腕）: `0.3515625 / 0.9375 = 0.375` → 細腕幅 3 = 頭 8 × 0.375

### translation の検証

胴 (`torso`) を原点に置き、他パーツを立ち姿の人型として相対配置しています。

- 頭: +7.5 上
- 腕: ±5.63 横、+2.5 上（肩の位置に合わせて高め）
- 細腕: ±5.15 横（腕より細いぶん内側）、+2.5 上
- 脚: ±1.88 横、上下オフセットなし

### thirdperson_righthand のみである理由

`display` に `thirdperson_righthand` しか定義されていないのは、これらのモデルが
**`item_display: "thirdperson_righthand"` を指定したアイテム表示エンティティ
（`item_display`）で使われる**ことを前提としているためです。

## 3. 利用フロー

サーバー / データパック側で、頭・胴・腕×2・脚×2 の **計 6〜8 個の `item_display`
エンティティ**を立ち姿になるよう配置し、各エンティティのアイテムに次のコンポーネントを
設定します。

- `minecraft:item_model = player_display:core`
- `minecraft:custom_model_data = {"strings": ["torso"]}` など、パーツ名の文字列
- `minecraft:profile = 対象プレイヤーのスキン`

リソースパックが各アイテムを `scale` / `translation` でパーツ形状に変形し、
それらが合体して**等身大のプレイヤー像**として表示されます。

## 注意点（リポジトリ外の前提）

- `player_head` レンダラは常にスキンの**「頭」キューブ**を描画するため、胴・腕・脚は
  「頭テクスチャを引き伸ばしたもの」になります。各パーツを正しい見た目にするには、
  サーバー / データパック側で、パーツのピクセルを頭領域に焼き込んだ
  **専用の `profile` スキン**を各 `item_display` に渡す設計が前提です。
  このリポジトリは**形状・配置（ジオメトリ）を担当**しており、テクスチャ生成は含みません。
- `pack.mcmeta` は `supported_formats: [15, 2147483647]` と広い範囲を宣言していますが、
  `items/` 形式・`select`・`special`・`player_head` は **Minecraft 1.21.4 以降**でのみ
  機能します。
- CI（`.github/workflows/main.yml`）は `main` への push で全体を `resource.zip` 化し、
  日時タグで Release を自動作成します。`docs/` は `assets/` 外のため、ゲームからは
  無視されます。

## まとめ

player_display は「**1 個の頭キューブを 8 通りに変形して人体パーツに見せ、
`item_display` で組み立てる**」リソースパック側の仕掛けです。

| 役割               | 担当                                      |
| ------------------ | ----------------------------------------- |
| 形状・配置         | このリソースパック（`base` モデルの変形） |
| スキン / テクスチャ | 各アイテムの `minecraft:profile` コンポーネント |
| パーツの組み立て   | サーバー / データパック（`item_display` の配置） |
