# 日本語マルチバイトUTF-8対応開発計画書

## 実装ステータス: Phase 1-4完了 (2026-02-03)

- ✅ Phase 1: UTF-8文字分割関数実装
- ✅ Phase 2: trigram_unicode関数実装
- ✅ Phase 3: tokenizer統合
- ✅ Phase 4: テスト・ドキュメント更新
- ⏳ Phase 5: upstream PR準備（未実施）

## 1. 目標

DuckDB FTS拡張のtrigramトークナイザーを日本語（マルチバイトUTF-8）に対応させ、部分一致検索を可能にする。

## 2. 現状の課題

### 2.1 技術的課題

| 項目 | 現状 | 課題 |
|------|------|------|
| trigram方式 | バイトレベル（3バイト固定） | UTF-8文字単位への変更が必要 |
| 日本語対応 | ❌ 不正UTF-8生成でエラー | ✅ 文字単位trigram実装 |
| UTF-8デコード | 未実装 | DuckDBのUTF-8機能を活用 |

### 2.2 問題詳細

```sql
-- 現在の実装の問題点
SELECT generate_trigrams('東京');  -- 6バイト → バイト単位で4trigram生成
-- 結果: 不正なUTF-8シーケンス（エラー）

-- 理想的な動作（実装後）
SELECT generate_trigrams('東京');  -- 2文字 → 文字単位で0trigram（3文字未満）
-- 結果: []

SELECT generate_trigrams('こんにちは');  -- 5文字 → 文字単位で3trigram
-- 結果: ['こんに', 'んにち', 'にちは']
```

## 3. 技術仕様

### 3.1 UTF-8文字単位trigram

**入力**: UTF-8エンコードされた文字列  
**出力**: UTF-8文字単位で生成されたtrigram配列

**アルゴリズム**:
1. UTF-8文字列を文字単位に分割（UTF-8デコード）
2. 連続する3文字ごとにスライディングウィンドウ
3. 各trigramを有効なUTF-8文字列として結合

### 3.2 UTF-8文字コードポイント処理

| 文字種 | UTF-8バイト数 | 例 |
|--------|--------------|----|
| ASCII | 1バイト | 'a', '1' |
| ラテン拡張 | 2バイト | 'é', 'ñ' |
| ひらがな・カタカナ | 3バイト | 'あ', 'ア' |
| 漢字（基本多言語面） | 3バイト | '東', '京' |

## 4. 実装方針

### 4.1 アプローチ選択

| オプション | メリット | デメリット | 採用 |
|-----------|----------|-----------|-----|
| (A) DuckDB組み込みUTF-8関数活用 | 簡潔、メンテナンス性高 | 依存度増 | ✅ 採用候補 |
| (B) 自前UTF-8デコーダ実装 | 完全制御 | 複雑化、バグリスク | - |
| (C) 外部ライブラリ導入 | 機能豊富 | ビルド複雑化 | - |

**推奨**: オプションA（DuckDBのUTF-8処理機能を活用）

### 4.2 使用するDuckDB機能

```cpp
// DuckDBのUTF-8処理関数候補
- duckdb::string_t::GetString()  // UTF-8文字列取得
- duckdb::TextSearcher           // テキスト検索
- duckdb::string::length()       // 文字列長（バイト）
```

### 4.3 実装フロー

```
入力: "こんにちは" (UTF-8)
  ↓
UTF-8文字抽出: ['こ', 'ん', 'に', 'ち', 'は'] (5文字)
  ↓
スライディングウィンドウ（n=3）:
  - [0:3] → 'こんに'
  - [1:4] → 'んにち'
  - [2:5] → 'にちは'
  ↓
出力: ['こんに', 'んにち', 'にちは']
```

## 5. 実装ステップ

### ステップ1: UTF-8文字分割関数実装

**ファイル**: `extension/fts/fts_extension.cpp`

```cpp
// UTF-8文字を1文字ずつ分割する関数
vector<duckdb::string_t> ExtractUTF8Characters(duckdb::string_t input) {
    vector<duckdb::string_t> chars;
    // UTF-8デコード処理
    // 各文字コードポイントを抽出
    return chars;
}
```

### ステップ2: UTF-8文字単位trigram生成

**ファイル**: `extension/fts/fts_extension.cpp`

```cpp
// generate_trigrams_unicode()関数追加
static void GenerateTrigramsUnicode(DataChunk &args, ExpressionState &state, Vector &result) {
    // 1. UTF-8文字分割
    // 2. スライディングウィンドウでtrigram生成
    // 3. 結果をリストとして返す
}
```

### ステップ3: tokenizerパラメータ拡張

**新オプション**: `tokenizer='trigram_unicode'`

| 値 | 説明 |
|----|------|
| `word` | 単語ベース（既存） |
| `trigram` | バイト単位trigram（既存） |
| `trigram_unicode` | **UTF-8文字単位trigram（新規）** |

### ステップ4: マクロ生成ロジック更新

**ファイル**: `extension/fts/fts_indexing.cpp`

```cpp
// trigram_unicodeモードのマクロ生成
if (tokenizer == "trigram_unicode") {
    // UTF-8文字単位trigram生成ロジック
    macro_sql = "...generate_trigrams_unicode(...)...";
}
```

### ステップ5: テスト追加

**ファイル**: `test/sql/fts/test_trigram_unicode.test`

```sql
-- 日本語trigram生成テスト
SELECT generate_trigrams_unicode('こんにちは');
-- 期待: ['こんに', 'んにち', 'にちは']

-- インデックス作成・検索テスト
CREATE TABLE jp_docs(id VARCHAR, text VARCHAR);
INSERT INTO jp_docs VALUES ('j1', '東京タワー');
PRAGMA create_fts_index('jp_docs', 'id', 'text', tokenizer='trigram_unicode');

-- 部分一致検索: '東京' で '東京タワー' を検索
SELECT * FROM jp_docs WHERE fts_main_jp_docs.match_bm25(id, '東京') IS NOT NULL;
-- 期待: j1行がヒット
```

## 6. テスト計画

### 6.1 ユニットテスト

| テストケース | 入力 | 期待出力 |
|-------------|------|---------|
| ASCII文字 | 'hello' | ['hel', 'ell', 'llo'] |
| ひらがな | 'あいう' | ['あいう'] |
| カタカナ | 'アキハバラ' | ['アキハ', 'キハバ', 'ハバラ'] |
| 漢字 | '東京' | [] (3文字未満) |
| 混在 | '東京TOWER' | ['東京T', '京TO', 'TOW', 'OWE', 'WER'] |
| 空文字 | '' | [] |
| 1文字 | 'あ' | [] |
| 2文字 | 'あい' | [] |

### 6.2 インテグレーションテスト

```sql
-- 日本語部分一致検索
-- クエリ: '東京' → ドキュメント: '東京タワー' (ヒット)
-- クエリ: 'タワー' → ドキュメント: '東京タワー' (ヒット)
-- クエリ: 'タワ' → ドキュメント: '東京タワー' (ヒット)
```

### 6.3 パフォーマンステスト

| データ量 | インデックス作成時間 | 検索時間 |
|---------|------------------|---------|
| 1,000件 | < 1秒 | < 10ms |
| 10,000件 | < 10秒 | < 100ms |
| 100,000件 | < 2分 | < 1秒 |

## 7. マイルストーン

| フェーズ | 期間 | 目標 | 完了条件 |
|---------|------|------|---------|
| **Phase 1** | 1週間 | UTF-8文字分割関数実装 | ユニットテスト通過 |
| **Phase 2** | 1週間 | trigram_unicode関数実装 | ユニットテスト通過 |
| **Phase 3** | 1週間 | tokenizer統合 | インテグレーションテスト通過 |
| **Phase 4** | 3日間 | テスト・ドキュメント更新 | 全テスト通過、README更新 |
| **Phase 5** | 2日間 | upstream PR準備 | コードレビュー対応完了 |

**合計**: 約3.5週間

## 8. リスク管理

| リスク | 影響 | 緩和策 |
|--------|------|--------|
| UTF-8デコードの複雑性 | 高 | DuckDB組み込み関数活用 |
| パフォーマンス低下 | 中 | キャッシュ・最適化検討 |
| 既存機能への影響 | 中 | 後方互換性維持（新オプション追加） |
| upstream却下 | 低 | 先にドキュメントでPR意向確認 |

## 9. 今後の拡張可能性

- [ ] 4-gram, bigram対応（可変n-gram）
- [ ] パディング付きtrigram（`$東京`, `$東`, `東京`, ...）
- [ ] 正規化オプション（半角/全角、ひらがな/カタカナ統一）
- [ ] 中国語・韓国語など他マルチバイト言語対応

## 10. 参考資料

- [UTF-8エンコーディング仕様](https://en.wikipedia.org/wiki/UTF-8)
- [DuckDB String Functions](https://duckdb.org/docs/sql/functions/char)
- [sqlite-fts5-trigram](https://github.com/simonw/sqlite-fts5-trigram)
