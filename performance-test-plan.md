# 100万件日本語データでのtrigram_unicode検索パフォーマンステスト計画

## 1. 目的

- 100万件の日本語データでtrigram_unicode検索のパフォーマンステストを実施
- インデックス作成時間、検索応答時間、スループットを測定

## 2. 既存データセット調査結果

### データセット比較表

| データセット | サイズ | 取得方法 | 適合性 | 採用優先度 |
|-------------|----------|-----------|-----------|-------------|
| **青空文庫** | 約11万件 | スクレイピング | 文学テキスト | ⭐⭐⭐ |
| **livedoorニュース** | 約7,000件 | 直接ダウンロード | ニュース記事、短文 | ⭐⭐ |
| **Wikipedia日本語** | 数百万件 | ダウンロード | 前処理必要 | ⭐ |
| **Wikipedia要約** | 数万件 | 直接ダウンロード | 短文で検索に適 | ⭐⭐⭐ |
| **国立国会図書館蔵書目録** | 非常に大規模 | API取得 | 図書館データ | ⭐⭐⭐ |

**採用データセット**: Wikipedia要約データセット

### Wikipedia要約データセット詳細

- **URL**: https://github.com/simpler-github/Japanese-Wikipedia-Summary-Data
- **サイズ**: 数万件
- **特徴**:
  - Wikipedia記事の要約
  - タイトルと本文のペア
  - 比較的短い文章
- **形式**: CSV
- **取得方法**: GitHubから直接ダウンロード
- **拡張方法**: 不足分は他データセットで補完

## 3. 使用データセット

### 3.1 Wikipedia要約データセット

**ファイル名**: `japanese_wikipedia_summary.csv`

**カラム**:
- `title`: 記事タイトル
- `summary`: 要約本文

**取得コマンド**:
```bash
# GitHubからダウンロード
git clone https://github.com/simpler-github/Japanese-Wikipedia-Summary-Data.git
cd Japanese-Wikipedia-Summary-Data
cat *.csv > japanese_wikipedia_summary.csv
```

### 3.2 データ拡張（100万件への調整）

```python
# Pythonスクリプトで100万件に複製・拡張
import pandas as pd
import random

# 既存データ読み込み
df = pd.read_csv('japanese_wikipedia_summary.csv')

# 100万件に複製（必要な場合）
if len(df) < 1000000:
    # 複製してカテゴリを追加
    df_extended = pd.concat([df] * (1000000 // len(df) + 1))
    df_extended = df_extended.head(1000000)
else:
    df_extended = df.head(1000000)

# ID付与
df_extended['id'] = [f'doc{i:06d}' for i in range(len(df_extended))]

# CSV形式で保存
df_extended[['id', 'title', 'summary']].to_csv('jp_docs_1m.csv', index=False)
```

## 4. パフォーマンステストシナリオ

| ID | データ量 | キーワード | 検索パターン | 期待検索時間 |
|----|----------|-----------|--------------|--------------|
| S-1 | 1,000件 | 単語 | 完全一致（3文字以上） | < 10ms |
| S-2 | 1,000件 | 部分一致 | 完全一致（3文字以上） | < 10ms |
| M-1 | 10,000件 | 単語 | 完全一致（3文字以上） | < 100ms |
| M-2 | 10,000件 | 部分一致 | 完全一致（3文字以上） | < 100ms |
| L-1 | 100,000件 | 単語 | 完全一致（3文字以上） | < 1秒 |
| L-2 | 100,000件 | 部分一致 | 完全一致（3文字以上） | < 1秒 |
| XL-1 | 1,000,000件 | 単語 | 完全一致（3文字以上） | < 10秒 |
| XL-2 | 1,000,000件 | 部分一致 | 完全一致（3文字以上） | < 10秒 |
| XL-3 | 1,000,000件 | 複数キーワード | AND検索 | < 10秒 |

## 5. テスト手順

### 5.1 データセット準備

```bash
# 1. Wikipedia要約データセットのダウンロード
cd /Users/m1macmini/playground/duckdb-fts-trigram
mkdir -p data
cd data
git clone https://github.com/simpler-github/Japanese-Wikipedia-Summary-Data.git

# 2. データ結合・拡張スクリプト実行
python3 << 'EOF'
import pandas as pd
import glob

# 全CSVファイル結合
dfs = []
for file in glob.glob('Japanese-Wikipedia-Summary-Data/*.csv'):
    df = pd.read_csv(file)
    dfs.append(df)

df_combined = pd.concat(dfs, ignore_index=True)

# 100万件に調整
if len(df_combined) > 1000000:
    df_combined = df_combined.head(1000000)
elif len(df_combined) < 1000000:
    # 複製して調整
    repeats = (1000000 // len(df_combined)) + 1
    df_extended = pd.concat([df_combined] * repeats, ignore_index=True)
    df_combined = df_extended.head(1000000)

# ID付与
df_combined['id'] = [f'doc{i:06d}' for i in range(len(df_combined))]

# CSV形式で保存（UTF-8 BOM）
df_combined[['id', 'title', 'summary']].to_csv('jp_docs_1m.csv', 
                                                   index=False, 
                                                   encoding='utf-8-sig')
print(f"Saved {len(df_combined)} records")
EOF
```

### 5.2 DuckDB起動・拡張機能ロード

```bash
# DuckDB起動
cd /Users/m1macmini/playground/duckdb-fts-trigram/duckdb-fts
./build/release/duckdb -unsigned

# SQLコマンドで拡張機能ロード
LOAD 'build/release/extension/fts/fts.duckdb_extension';

# データ投入確認
SELECT COUNT(*) FROM read_csv_auto('data/jp_docs_1m.csv');
```

### 5.3 データ投入

```sql
-- テーブル作成
CREATE OR REPLACE TABLE jp_docs (
    id VARCHAR,
    title VARCHAR,
    text VARCHAR
);

-- データ投入
COPY jp_docs FROM 'data/jp_docs_1m.csv' (DELIMITER ',', HEADER);

-- データ確認
SELECT COUNT(*) as total_records FROM jp_docs;
SELECT * FROM jp_docs LIMIT 5;
```

### 5.4 インデックス作成（時間計測）

```sql
-- インデックス作成（時間計測）
.timer on

-- 既存インデックス削除（あれば）
PRAGMA drop_fts_index('jp_docs');

-- trigram_unicodeインデックス作成
PRAGMA create_fts_index('jp_docs', 'id', 'title', 'text', tokenizer='trigram_unicode');

.timer off

-- インデックス確認
SELECT 
    (SELECT COUNT(*) FROM fts_main_jp_docs.dict) as unique_terms,
    (SELECT COUNT(*) FROM fts_main_jp_docs.terms) as total_terms,
    (SELECT COUNT(*) FROM fts_main_jp_docs.docs) as total_docs;
```

### 5.5 検索テスト（時間計測）

```sql
-- タイマー設定
.timer on

-- テスト1: 単語検索（完全一致）
SELECT COUNT(*) FROM jp_docs 
WHERE fts_main_jp_docs.match_bm25(id, '日本') IS NOT NULL;

-- テスト2: 部分一致検索
SELECT COUNT(*) FROM jp_docs 
WHERE fts_main_jp_docs.match_bm25(id, '東京都') IS NOT NULL;

-- テスト3: 複数キーワード検索
SELECT COUNT(*) FROM jp_docs 
WHERE fts_main_jp_docs.match_bm25(id, '日本 語') IS NOT NULL;

-- テスト4: 英語混在検索
SELECT COUNT(*) FROM jp_docs 
WHERE fts_main_jp_docs.match_bm25(id, 'Tokyo') IS NOT NULL;

-- テスト5: 実際のヒット件数確認
SELECT id, title, 
       fts_main_jp_docs.match_bm25(id, '東京都') AS score
FROM jp_docs 
WHERE fts_main_jp_docs.match_bm25(id, '東京都') IS NOT NULL
LIMIT 10;

.timer off
```

## 6. 使用する検索クエリ

### 6.1 単語検索（完全一致）

```sql
-- ターゲット: '東京' (3文字)
SELECT COUNT(*) FROM jp_docs 
WHERE fts_main_jp_docs.match_bm25(id, '東京') IS NOT NULL;
```

### 6.2 部分一致検索

```sql
-- ターゲット: '東京都' (3文字以上)
-- 期待: '東京', '京都', '東京都' などを含むドキュメントがヒット
SELECT COUNT(*) FROM jp_docs 
WHERE fts_main_jp_docs.match_bm25(id, '東京都') IS NOT NULL;

-- ヒットしたドキュメントの確認
SELECT id, title,
       fts_main_jp_docs.match_bm25(id, '東京都') AS score
FROM jp_docs 
WHERE fts_main_jp_docs.match_bm25(id, '東京都') IS NOT NULL
ORDER BY score DESC
LIMIT 10;
```

### 6.3 複数キーワード検索

```sql
-- ターゲット: '日本 語' (AND検索)
SELECT COUNT(*) FROM jp_docs 
WHERE fts_main_jp_docs.match_bm25(id, '日本 語') IS NOT NULL;
```

### 6.4 英語混在検索

```sql
-- ターゲット: 'Tokyo Tower' (英語+日本語)
SELECT COUNT(*) FROM jp_docs 
WHERE fts_main_jp_docs.match_bm25(id, 'Tokyo Tower') IS NOT NULL;
```

### 6.5 スループットテスト

```sql
-- 連続検索でスループット計測
.timer on

-- クエリ1
SELECT COUNT(*) FROM jp_docs WHERE fts_main_jp_docs.match_bm25(id, '日本') IS NOT NULL;
-- クエリ2
SELECT COUNT(*) FROM jp_docs WHERE fts_main_jp_docs.match_bm25(id, '東京') IS NOT NULL;
-- クエリ3
SELECT COUNT(*) FROM jp_docs WHERE fts_main_jp_docs.match_bm25(id, '京都') IS NOT NULL;
-- クエリ4
SELECT COUNT(*) FROM jp_docs WHERE fts_main_jp_docs.match_bm25(id, '大阪') IS NOT NULL;
-- クエリ5
SELECT COUNT(*) FROM jp_docs WHERE fts_main_jp_docs.match_bm25(id, '福岡') IS NOT NULL;

.timer off
```

## 7. 結果記録テンプレート

| 実行日時 | データ量 | インデックス作成時間 | 検索1時間 | 検索2時間 | 検索3時間 | 使用メモリ | 備考 |
|-----------|----------|------------------|-----------|-----------|-----------|-----------|------|
| 2026-02-04 | 1,000,000件 | 秒 | ms | ms | MB | |

## 8. 他の環境への引き継ぎ手順

### 8.1 前提条件

- DuckDBのインストール
- trigram_unicode拡張機能のビルド
- Python 3.x (pandasライブラリ)

### 8.2 環境構築手順

```bash
# 1. リポジトリのクローン
git clone <repository-url>
cd duckdb-fts-trigram

# 2. サブモジュールの初期化とビルド
cd duckdb-fts
git submodule update --init --recursive
mkdir -p build/release
cd build/release
cmake ../.. -DCMAKE_BUILD_TYPE=Release
make -j8 fts_loadable_extension

# 3. データセットの準備
cd ../..
mkdir -p data
cd data
# Wikipedia要約データセットのダウンロード
wget https://github.com/simpler-github/Japanese-Wikipedia-Summary-Data/archive/refs/heads/main.zip
unzip main.zip
# データ結合・拡張スクリプト実行

# 4. DuckDBでのテスト実行
cd ../duckdb-fts
./build/release/duckdb -unsigned
```

### 8.3 実行スクリプト例

```bash
#!/bin/bash
# performance_test.sh

set -e

echo "=== Performance Test for trigram_unicode ==="
echo "Date: $(date)"
echo ""

# Step 1: Prepare data
echo "Step 1: Preparing data..."
cd data
python3 prepare_data.py

# Step 2: Start DuckDB
echo "Step 2: Starting DuckDB..."
cd ../duckdb-fts
./build/release/duckdb -unsigned < test_script.sql > results.log

# Step 3: Analyze results
echo "Step 3: Analyzing results..."
grep "Run Time:" results.log

echo ""
echo "=== Test Complete ==="
```

### 8.4 トラブルシューティング

| 問題 | 原因 | 解決策 |
|--------|--------|--------|
| データセットが見つからない | URL変更・ダウンロード失敗 | GitHubから直接ダウンロード、URL確認 |
| 拡張機能ロード失敗 | パスが間違っている | 相対パス/絶対パスを確認 |
| インデックス作成が遅すぎる | データ量が多すぎる、メモリ不足 | データ量を減らす、メモリ増設 |
| 検索がヒットしない | キーワードが短すぎる（3文字未満） | 3文字以上のキーワードを使用 |

## 9. 注釈・資料

### 9.1 技術的注釈

- **trigram_unicodeの仕様**: UTF-8文字単位でtrigramを生成
- **最小キーワード長**: 3文字未満は検索結果として返さない
- **BM25スコア**: trigramの重複度に基づいて計算

### 9.2 参考資料

- [Wikipedia要約データセット](https://github.com/simpler-github/Japanese-Wikipedia-Summary-Data)
- [DuckDB COPYコマンド](https://duckdb.org/docs/sql/statements/copy)
- [DuckDB FTS拡張](https://github.com/duckdb/duckdb-fts)
- [UTF-8エンコーディング](https://ja.wikipedia.org/wiki/UTF-8)

---

**作成日**: 2026-02-04
**作成者**: Claude Sonnet 4.5
**バージョン**: 1.0
**用途**: 100万件日本語データでのtrigram_unicode検索パフォーマンステスト
