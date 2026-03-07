---
emoji: 🏗️
published: true
title: Producer / Consumer 分離 ― なぜ PLAO / adsb-eval / ARENA
  は3リポジトリなのか
topics:
- ADSB
- RaspberryPi
- 設計思想
- データエンジニアリング
- Python
type: tech
---

# Producer / Consumer 分離

## なぜ PLAO / adsb-eval / ARENA は3リポジトリなのか

## この章の位置づけ

第1章では、ADS-B受信評価の指標として **有効受信機体数の時間積算（AUC）**
を定義しました。\
第2章では、その評価を成立させるための **Failure‑First
なログ基盤（PLAO）** を解説しました。

ここで自然な疑問が生まれます。

**このログはどのように解析エンジンへ渡されるのか？**

PLAOは **ログ収集装置** であり、統計解析エンジンではありません。

この章では、評価基盤を構成する次の3コンポーネントの役割を説明します。

-   **PLAO**：位置ログ生成（Position Log Producer）
-   **adsb-eval**：デコーダ指標生成（Runtime Metrics Producer）
-   **ARENA**：統計解析エンジン（Statistical Consumer）

そして、この構造が **Producer / Consumer アーキテクチャ**
として設計されている理由を解説します。

------------------------------------------------------------------------

# システム全体構造

``` mermaid
flowchart TD

    subgraph Edge["Raspberry Pi (Edge)"]

        SDR["SDR Hardware"]
        READSB["Decoder (readsb)"]

        PLAO["PLAO<br>Position Logger"]
        EVAL["adsb-eval<br>Runtime Metrics"]

        SDR --> READSB
        READSB --> PLAO
        READSB --> EVAL
    end

    subgraph Transfer["Data Sync"]
        RSYNC["rsync"]
    end

    subgraph Analysis["ARENA (Workstation)"]

        ARENA["ARENA<br>Statistical Evaluation Engine"]

        METRICS["AUC / Derived Metrics"]

        TEST["Frequentist Tests"]
        GLM["Negative Binomial GLM"]
        BAYES["Bayesian Inference"]
        CPD["Change‑point Detection"]

        ARENA --> METRICS
        METRICS --> TEST
        METRICS --> GLM
        METRICS --> BAYES
        METRICS --> CPD
    end

    PLAO --> RSYNC
    EVAL --> RSYNC
    RSYNC --> ARENA
```

この構造は大きく3層に分かれます。

  層         役割
  ---------- ----------------
  Edge       観測データ収集
  Transfer   ログ同期
  Analysis   統計解析

------------------------------------------------------------------------

# なぜモノリポジトリにしないのか（設計思想）

一見すると、このシステムは1つのリポジトリにまとめることも可能です。

しかし実際には、各コンポーネントの性質は大きく異なります。

  コンポーネント   実行環境            目的
  ---------------- ------------------- ----------------
  PLAO             Raspberry Pi        観測ログ生成
  adsb-eval        Raspberry Pi        ランタイム指標
  ARENA            Workstation / GPU   統計解析

これらは

-   実行環境
-   依存関係
-   計算負荷

が完全に異なります。

もしこれを1つのリポジトリに統合すると

-   Raspberry Pi向け依存
-   GPU向け依存
-   統計解析ライブラリ

が混在し、開発と運用が複雑になります。

このため本システムでは **Producer / Consumer 分離** を採用しています。

------------------------------------------------------------------------

# なぜ3つのリポジトリになるのか（実装）

Producer / Consumer
分離を実装すると、自然に3つのコンポーネントが生まれます。

  Repo        役割
  ----------- --------------------------
  PLAO        Position Log Producer
  adsb-eval   Runtime Metrics Producer
  ARENA       Statistical Consumer

つまり

**3リポジトリ構成は設計思想の結果です。**

最初から「3つに分けよう」と決めたわけではありません。

責務分離を進めた結果として、この構造になります。

------------------------------------------------------------------------

# PLAOの責務

PLAOの責務は極めて単純です。

    aircraft.json
    ↓
    append‑only JSONL

出力例

    pos_YYYYMMDD.jsonl

PLAOは

-   統計計算をしない
-   集計をしない
-   フィルタリングもしない

**ただ正確に書くだけです。**

この単純さが、エッジ環境での長期稼働を支えます。

------------------------------------------------------------------------

# adsb-evalの役割

adsb-evalは **デコーダのランタイム指標を収集するツール** です。

PLAOが

「どの機体がどこにいたか」

を記録するのに対し、adsb-evalは

「デコーダがどのように動作していたか」

を記録します。

主な入力

    /run/readsb/aircraft.json
    /run/readsb/stats.json
    /run/airspy_adsb/stats.json

主な出力

    dist_1m.jsonl
    stats_history.jsonl
    dist_signal_stats_1m.jsonl

つまり

    decoder runtime
    ↓
    structured telemetry logs

を生成する役割です。

------------------------------------------------------------------------

# なぜエッジで統計解析をしないのか

統計解析は **ARENA** で行います。

理由はシンプルです。

**エッジコンピュータの役割はリアルタイム観測だからです。**

Raspberry Pi は

-   CPU
-   RAM
-   I/O

すべてが限られています。

もしここで

-   Bayesian inference
-   GLM
-   change‑point detection

のような重い処理を実行すると、ログ収集のリアルタイム性が失われます。

------------------------------------------------------------------------

# ワークステーション解析

ARENAは統計解析専用の環境で動作します。

主な解析手法

-   Mann‑Whitney U test
-   Bootstrap confidence intervals
-   Negative Binomial GLM
-   Bayesian inference (NumPyro / NUTS)
-   Change‑point detection
-   External traffic normalization

これらの処理は

-   CPU
-   GPU
-   大量メモリ

を消費します。

この種の計算は Raspberry Pi
ではなく、**ワークステーションで実行する方が圧倒的に効率的**です。

------------------------------------------------------------------------

# 再現性という設計目標

この分離設計の最大の目的は **再現性 (Reproducibility)** です。

ARENAは

    pos_YYYYMMDD.jsonl
    dist_1m.jsonl

があれば、いつでも再解析できます。

つまり

**ログ = 実験データ**

です。

ハードウェア変更の評価では

-   モデル変更
-   検定変更
-   仮説検証

を何度も繰り返します。

ログと解析を疎結合にすることで、この再評価が可能になります。

------------------------------------------------------------------------

# まとめ

本章では PLAO / adsb-eval / ARENA の役割分離を説明しました。

重要な設計思想は次の通りです。

-   デコーダとロガーを分離する
-   Producer / Consumer を分離する
-   エッジでは観測だけを行う
-   統計解析はワークステーションで実行する
-   ログを解析から独立させ再現性を担保する

この設計により、評価基盤は **信頼性・再現性・拡張性**
を同時に獲得します。

次章では、このログを用いて
**ノイズ環境下で構成変更の効果を分離する統計モデル** を解説します。
