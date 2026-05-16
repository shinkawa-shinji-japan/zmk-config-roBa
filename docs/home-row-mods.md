# Home Row Mods カスタマイズ

## 概要

ホームポジション 10 キーに**修飾キー (Ctrl / Shift / GUI / Alt) とレイヤ切替**を兼ねる
Home Row Mods (HRM) を仕込みつつ、**Vim 系エディタでの長押し連打 (`jjjjj`, `5j` 等) も
成立させる**ためのカスタム hold-tap behavior。

組込みの `&mt` / `&lt` をそのまま使うと、`j` `k` `l` を長押ししても hold (Ctrl / Layer / Shift)
が優先されて連打が発生しない。これを `quick-tap-ms` と `require-prior-idle-ms` を持つ
専用 behavior `&hrm_mt` / `&hrm_lt` を導入することで解決している。

## なぜ素の `&mt` / `&lt` ではダメか

ZMK の組込み `&mt` (mod-tap) / `&lt` (layer-tap) は、長押し時に
**修飾 (hold) 側が優先**される設計。`flavor = "balanced"` でも、tapping-term-ms
(既定 200ms) を超える長押しは hold とみなされ、連打にならない。

具体的に、本キーマップで連打したいケースが 2 パターンある:

1. **同じキーを連打したい** (`jjjjj` で下移動を加速)
2. **直前のキー押下から間を置かずに連打したい** (`5j` のように数値プレフィクス直後)

それぞれに対応するパラメータを使う。

## 使用 behavior

[`config/roBa.keymap`](../config/roBa.keymap) の `behaviors {}` ブロックに以下を定義:

```dts
hrm_mt: hrm_mt {
    compatible = "zmk,behavior-hold-tap";
    label = "HRM_MT";
    bindings = <&kp>, <&kp>;

    #binding-cells = <2>;
    flavor = "balanced";
    tapping-term-ms = <200>;
    quick-tap-ms = <175>;
    require-prior-idle-ms = <125>;
};

hrm_lt: hrm_lt {
    compatible = "zmk,behavior-hold-tap";
    label = "HRM_LT";
    bindings = <&mo>, <&kp>;

    #binding-cells = <2>;
    flavor = "balanced";
    tapping-term-ms = <200>;
    quick-tap-ms = <175>;
    require-prior-idle-ms = <125>;
};
```

- `hrm_mt`: 修飾キー兼用 (旧 `&mt LCTRL J` 相当)
- `hrm_lt`: レイヤ切替兼用 (旧 `&lt 2 K` 相当)

## パラメータの役割

| プロパティ | 値 | 意味 |
|---|---|---|
| `flavor` | `"balanced"` | tap/hold の判定モード。組込み `&mt` のグローバル設定 (`flavor = "balanced"`) と同じ |
| `tapping-term-ms` | `200` | この時間を超える長押しは hold 候補。組込みと同値 |
| `quick-tap-ms` | `175` | **直前タップから 175ms 以内** の長押しは hold ではなく **tap** として扱う (= 連打成立) |
| `require-prior-idle-ms` | `125` | **直前のキー押下から 125ms 経過するまで** hold を発動させない (= タイピング中の hold 誤発火を防ぐ) |

### 動作早見表 (`j` = `&hrm_mt LCTRL J` の例)

| 操作 | 結果 | 効くパラメータ |
|---|---|---|
| `j` をぽつんと長押し | **Ctrl** (hold) | (通常の hold-tap) |
| `j` をタップ → 即長押し | **`j` 連打** | `quick-tap-ms` |
| `5` を押した直後に `j` 長押し | **`j` 連打** | `require-prior-idle-ms` |
| `j` 長押しのまま `c` | **Ctrl+C** | (通常の hold-tap) |
| `hello` のような通常タイピング | **そのまま文字入力** | `require-prior-idle-ms` |

## 適用キー

[`config/roBa.keymap`](../config/roBa.keymap) の `default_layer` の以下 10 binding を
`&hrm_mt` / `&hrm_lt` に差し替えている。

| 物理キー | binding | 役割 |
|---|---|---|
| O | `&hrm_mt LALT O` | tap: O / hold: Alt |
| S | `&hrm_mt LGUI S` | tap: S / hold: GUI (Win) |
| J | `&hrm_mt LCTRL J` | tap: J / hold: Ctrl |
| K | `&hrm_lt 2 K` | tap: K / hold: Layer 2 (NUM) |
| L | `&hrm_mt LSHFT L` | tap: L / hold: Shift |
| Z | `&hrm_lt 1 Z` | tap: Z / hold: Layer 1 (Move) |
| X | `&hrm_mt LSHFT X` | tap: X / hold: Shift |
| C | `&hrm_lt 2 C` | tap: C / hold: Layer 2 (NUM) |
| V | `&hrm_mt LALT V` | tap: V / hold: Alt |
| M | `&hrm_mt LALT M` | tap: M / hold: Alt |

### 触っていないキー

親指列の `&lt 1 ENTER` / `&lt 3 TAB` / `&lt 1 MINUS` は素の `&lt` のまま。
親指で長押し連打したいユースケースがほぼ無いため。

## チューニング指針

実機の感触に合わせて 25ms 単位で調整する。

### 連打が出にくい / hold に化けやすい

- `quick-tap-ms` を **大きく** する (175 → 200, 225)
- `require-prior-idle-ms` を **大きく** する (125 → 150, 175)

### Vim でモーションコマンド入力中に修飾が誤発火する

- `require-prior-idle-ms` を **大きく** する

### 修飾キーの発火感が重い / hold に入りにくくなった

- `tapping-term-ms` を **小さく** する (200 → 175)
- `flavor = "tap-preferred"` への変更も検討 (ただし「素の長押し = hold」の判定が遅くなる)

### より精度を上げたい場合

`hold-trigger-key-positions` (= 他のキーを押したときだけ hold 発動) の併用を検討。
ホームロウ間の同時押しを抑制したいときに有効。

## ビルド・反映

ローカル Docker ビルドの手順とポート / 書き込み方法は、案件フォルダ側の
[`AGENTS.md`](../../AGENTS.md) (= `roba-work/AGENTS.md`) 参照。要点だけ:

- central は **右 (`roBa_R`)**。keymap オンリーの変更は **R のみ書き込みで反映**される
- 両側焼く場合は **左 (peripheral) → 右 (central)** の順が再ペアリングに優しい

## 参考

- ZMK hold-tap 公式: <https://zmk.dev/docs/keymaps/behaviors/hold-tap>
- Home Row Mods 一般論 (precondition / Urob 氏): <https://github.com/urob/zmk-config>
