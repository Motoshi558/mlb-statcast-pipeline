# MLB Statcast Analytics Pipeline on Databricks

**Databricks Free Edition**上に構築した、MLB Statcast(投球データ)のエンドツーエンド分析パイプライン兼ダッシュボード。**Medallionアーキテクチャ**(Bronze → Silver → Gold)に沿ってデータを段階的に整形しています。

## 制作背景

DatabricksのコアとなるツールセットDelta Lake、PySpark、Lakeviewダッシュボード、Genie)を、実データ(MLB Statcast)を使って実践的に習得することを目的としたプロジェクトです。データアナリスト/データエンジニア職向けのポートフォリオとして作成しました。

## アーキテクチャ

```
pybaseball (Statcast API)
        │
        ▼
┌───────────────┐   生データをそのまま、全列string型で保存
│  Bronze層     │   mlb.bronze.statcast_raw
└───────┬───────┘
        │  型変換・欠損/異常値フィルタ・重複排除
        ▼
┌───────────────┐   分析可能な状態にクレンジング済み
│  Silver層     │   mlb.silver.statcast_clean
└───────┬───────┘
        │  groupByによる集計
        ▼
┌───────────────┐   投手/打者別サマリー + 選手名結合済み
│  Gold層       │   mlb.gold.pitcher_summary_named
│               │   mlb.gold.batter_summary_named
│               │   mlb.gold.pitcher_category_summary
└───────┬───────┘
        ▼
┌───────────────────────┐
│  Lakeviewダッシュボード │  投手シート / 打者シート
│  (+ Genieによる自然言語クエリ) │
└───────────────────────┘
```

## 技術スタック

- **プラットフォーム**: Databricks Free Edition(サーバーレスコンピュート、Unity Catalog)
- **データソース**: [pybaseball](https://github.com/jldbc/pybaseball)(Statcast投球データ)
- **処理**: PySpark、Delta Lake
- **可視化**: Databricks Lakeviewダッシュボード、Genie(自然言語によるクエリ生成)
- **言語**: Python、SQL

## パイプライン詳細

### Bronze層 — 生データの取り込み
`pybaseball.statcast()`で取得したデータを、**全列をstring型に変換した状態でそのままDeltaに保存**しています。これはpandas側のNaN/object混在によるSparkの型推論エラーを避けるためであり、同時に「Bronze層は元データを一切加工せず保存する」という設計方針にも沿っています。型の解釈は、このあとのSilver層で明示的に行います。

### Silver層 — クレンジング
- 主要カラム(`game_date`、`release_speed`、`launch_speed`、`launch_angle`、`release_spin_rate`など)を本来の型に変換
- 1球を一意に特定するキー(`game_pk` + `at_bat_number` + `pitch_number`)で重複排除
- 欠損値・範囲外の値を除外(例:`release_speed`が30〜110mphの範囲外)
- 球種コードの表記ゆれを統一(例:`FT`→`SI`、`FA`→`FF`)
- Statcastの生の球種コードを`Fastball`/`Breaking Ball`/`Offspeed`/`Other`に分類した`pitch_category`列を追加

### Gold層 — 集計
- `pitcher_summary`/`batter_summary`:投手視点・打者視点それぞれの選手別集計(平均Exit Velocity、平均回転数、投球数など)
- `pitcher_category_summary`:投手別・`pitch_category`別の集計
- `player_lookup`:`pybaseball.playerid_reverse_lookup`によるMLBAM ID→選手名の対応表。集計テーブルとleft joinし(`*_named`)、名前が引けない選手がいても集計行自体は欠落しないようにしている

## ダッシュボード

野球データ分析における2つの自然な視点に合わせ、ダッシュボードは2ページ構成にしています。

- **投手シート**:投球数、球種構成比(`pitch_category`別)、回転数×球速の散布図、球速ランキング
- **打者シート**:Exit Velocityランキング、打球角度の分布、打球数

ランキング系のウィジェットには最小サンプル数フィルター(例:`batted_ball_count >= 20`)を設定し、サンプル数が極端に少ない選手が上位に出てくることを防いでいます。

一部のウィジェットは**Genie**(自然言語からの可視化生成)で作成しました。曖昧さを減らすため、集計方法・グラフ種類・軸・並び順・表示形式まで明示的にプロンプトへ指定しています。

## 設計上の主な判断

| 判断 | 理由 |
|---|---|
| Bronze層をstring型で統一 | 上流データのスキーマ/型変化に強くするため。型に関する判断は全てSilver層に集約 |
| Medallionアーキテクチャの採用 | 業界標準のパターンであり、生データ保存・クレンジング・ビジネス集計を監査可能な形で分離できる |
| 選手名結合にleft joinを使用 | 選手IDが名前解決できない場合でも、集計行自体は失わない |
| 最小サンプル数フィルター | 打球数1回などの統計的に信頼できない結果がランキング上位を占めることを防ぐ |
| カテゴリ名を英語表記に統一 | 日本語話者以外のレビュワーにもダッシュボードを利用可能にするため |

## スクリーンショット

_(`docs/dashboard_pitcher.png`、`docs/dashboard_batter.png`をここに追加)_

## 今後の展望

- Bronze→Silver→Goldパイプラインを Databricks Jobsでスケジュール実行し自動化
- MLflowによる実験管理を伴った、打球結果予測モデルの追加
- フルシーズンのデータセットへの拡張、および全件洗い替えではなく差分ロードへの移行
