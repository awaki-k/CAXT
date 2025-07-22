# Cadence ASIC Execution Tutorial


## 🎯 目標

`idct23_DI.tar.gz` に含まれる設計サンプル（IDCT）を使って、

1. **RTLの動作確認**
2. **論理合成（回路化）**
3. **レイアウト設計（配置配線）**
4. **静的タイミング解析（STA）**
5. **電力評価**

を順番に体験します。

---

## ✅ Step 0: 準備

### `.cshrc` に追記するコード全文

```bash
# === Cadence tools path ===
set path = ( \
    /usr/local/cadence/XCELIUM2303/tools.lnx86/bin \
    /usr/local/cadence/XCELIUM2303/tools.lnx86/inca/bin \
    /usr/local/cadence/GENUS211/tools.lnx86/bin \
    /usr/local/cadence/INNOVUS211/tools.lnx86/bin \
    /usr/local/cadence/QUANTUS211/tools.lnx86/bin \
    /usr/local/cadence/SSV231/tools.lnx86/bin \
    $path \
)

setenv LD_LIBRARY_PATH /usr/local/cadence/XCELIUM2303/tools.lnx86/lib:/usr/local/cadence/XCELIUM2303/tools.lnx86/inca/lib:$LD_LIBRARY_PATH

setenv CDS_LIC_FILE /usr/local/cadence/license/license.dat
```

---

## ▶ Step 1: RTLシミュレーション（動作確認）

### 1-1. 実行コマンド

```bash
./script/run_xrun_rtl
```

### 1-2. 波形を確認（GUI）

```bash
simvision &
```

* メニューから `File > Open Database` を選び
* `sim/waves_rtl.shm` を開く
* 信号を選んで波形を見る

🔎 これは「正しく動くVerilogか？」を確認するステップです。

---

## ▶ Step 2: 論理合成（回路に変換）

Genusを使って、Verilogをゲート回路に変換します。

### 2-1. 基本合成を実行

```bash
genus -legacy_ui -files script/genus_basic.tcl
```

※ 合成終了後、以下を確認：

* `outputs_basic/final.rpt` に `Slack > 0` であること（タイミングOK）
* `genus.log` にエラーがないこと

🔎 「設計が論理的に正しいか？」「遅延や面積は問題ないか？」を確認します。

---

## ▶ Step 3: ゲートレベルシミュレーション（GLシミュ）

合成後のNetlist（回路）で、RTLと同じ動作をするか確認。

### 3-1. 実行

```bash
./script/run_xrun_gate
```

### 3-2. 波形確認（SimVision）

```bash
simvision &
```

* `sim/waves_gate.shm` を開いて
* Step1と同じ出力になっていればOK

---

## ▶ Step 4: レイアウト設計（配置配線）

### 4-1. 実行

```bash
innovus -files script/innovus.tcl
```

自動で以下を行います：

* セルの配置
* 配線（Routing）
* パワーネット作成
* DRC（設計ルールチェック）

実行後に `summaryReport.rpt` や `final.def` が生成されます。

---

## ▶ Step 5: 静的タイミング解析（STA）

### 5-1. 実行

```bash
tempus -files script/tempus.tcl
```

* Slackやタイミング違反がないかチェック
* `tempus/reports/coverage.rpt` にて確認

---

## ▶ Step 6: レイアウト後シミュレーション

```bash
./script/run_xrun_layout
simvision &
```

* `sim/waves_layout.shm` を開く
* Step3と同じ動作ならOK

---

## ▶ Step 7: 電力評価

いつか追記します。
流れは、tbにvcdファイル保存して、joulesで電力評価すればok
