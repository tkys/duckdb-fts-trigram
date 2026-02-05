# ベンチマーク・テストスクリプト

## スクリプト一覧

### 1. benchmark_fts.py

**用途**: 100万件データセットでのパフォーマンステスト

**機能**:
- データロード（`wiki_jp.parquet` から）
- インデックス作成（`trigram_unicode` tokenizer）
- 検索テスト（4つの検索語）
- 時間計測とDBサイズ確認

**実行方法**:
```bash
# データセット準備（別途実行が必要）
python3 prepare_data.py

# ベンチマーク実行
python3 benchmark_fts.py
```

**検索語**:
- 東京タワー
- 人工知能
- 鬼滅の刃
- 徳川家康

**出力**:
- データロード時間
- インデックス作成時間
- 検索語ごとの実行時間とヒット件数
- DBサイズ（MB）

---

### 2. test_trigram_fts.py

**用途**: trigram_unicode機能の基本的なテスト

**機能**:
- `generate_trigrams_unicode` 関数テスト
- 日本語FTSインデックステスト
- 検索テスト（部分一致、完全一致）
- 英語trigram mode vs word modeの比較テスト

**テスト内容**:

#### 1. generate_trigrams_unicode 関数テスト
| 入力 | 期待結果 | 説明 |
|------|---------|------|
| 'hello' | ['hel', 'ell', 'llo'] | ASCII文字 |
| 'こんにちは' | ['こんに', 'んにち', 'にちは'] | ひらがな |
| '東京タワー' | ['東京タ', '京タワ', 'タワー'] | 漢字+カタカナ |
| '東京' | [] | 2文字（空配列期待） |
| 'ab' | [] | 2文字ASCII（空配列期待） |

#### 2. 日本語FTSインデックステスト
**テストデータ**:
```sql
('1', '東京タワー'),
('2', '東京駅'),
('3', '大阪城'),
('4', '京都タワー')
```

**検索パターン**:
- '東京タ': 3文字 - '東京タワー' がヒット期待
- 'タワー': 3文字 - '東京タワー', '京都タワー' がヒット期待
- '大阪城': 完全一致
- '京都': 2文字 - ヒットなし期待（trigram最小3文字）

#### 3. 英語trigram vs word mode比較
**テストデータ**: 英語商品名（iPhone, Samsung, Google）

**検索クエリ**: 'iPho'
- trigram mode: ヒットする（部分一致）
- word mode: ヒットしない（完全一致のみ）

**実行方法**:
```bash
python3 test_trigram_fts.py
```

**出力例**:
```
============================================================
DuckDB Trigram FTS (Python) テスト
============================================================

[1] Extension ロード: /Users/m1macmini/playground/duckdb-fts-trigram/duckdb-fts/build/release/extension/fts/fts.duckdb_extension
    ✅ ロード成功

[2] generate_trigrams_unicode 関数テスト
    'hello' (ASCII文字) → ['hel', 'ell', 'llo']
    'こんにちは' (ひらがな) → ['こんに', 'んにち', 'にちは']
    '東京タワー' (漢字+カタカナ) → ['東京タ', '京タワ', 'タワー']
    '東京' (2文字（空配列期待）) → []
    'ab' (2文字ASCII（空配列期待）) → []

[3] 日本語 FTS インデックステスト
    ✅ trigram_unicode インデックス作成成功

[4] 検索テスト
    '東京タ' (3文字 - '東京タワー' がヒット期待)
        → [1:0000] 東京タワー
    'タワー' (3文字 - '東京タワー', '京都タワー' がヒット期待)
        → [2:0000] 京都タワー
        → [4:0000] 東京タワー
    '大阪城' (完全一致)
        → [3:0000] 大阪城
    '京都' (2文字 - ヒットなし期待（trigram最小3文字）)
        → ヒットなし

[5] 英語 trigram テスト（word mode との比較）
    trigram mode: 'iPho' → [1:0000] iPhone 15 Pro
    word mode:    'iPho' → ヒットなし
```

---

## 使用上の注意点

### benchmark_fts.py

**データセットの準備**:
- `wiki_jp.parquet` ファイルが必要
- Wikipedia要約データセットのダウンロードと変換が必要
- 手順は `performance-test-plan.md` を参照

**拡張機能のパス設定**:
- EXT_PATH 変数を現在の環境に合わせて変更
- Mac環境: `/Users/m1macmini/playground/duckdb-fts-trigram/duckdb-fts/build/release/extension/fts/fts.duckdb_extension`

### test_trigram_fts.py

**前提条件**:
- DuckDB拡張機能がビルド済みであること
- `build/release/extension/fts/fts.duckdb_extension` が存在すること

**実行方法**:
```bash
# スクリプト実行権限を付与
chmod +x test_trigram_fts.py

# 実行
python3 test_trigram_fts.py
```

**予期されるテスト結果**:
1. generate_trigrams_unicode 関数が正しく動作すること
2. 日本語trigramインデックスが作成できること
3. 日本語部分一致検索が機能すること
4. 3文字未満の検索キーワードがヒットしないこと
5. trigram modeの方がword modeより多くヒットすること

---

## 既知の問題

### benchmark_fts.py

1. **データセットが存在しない**
   - `wiki_jp.parquet` が見つからない
   - 解決策: データセットの準備手順を実行

2. **大規模データセットの拡張**
   - 現在のWikipedia要約データセットは数万件
   - 100万件への拡張は複製または他データセットの追加が必要

### test_trigram_fts.py

なし（基本的な機能テストは正常に動作）

---

## 今後の改善

### benchmark_fts.py

- データ準備スクリプトの統合（`performance-test-plan.md` の手順）
- 並列検索テストの追加
- スループット（QPS: queries per second）の計測
- メモリ使用量の計測
- インデックスサイズの計測

### test_trigram_fts.py

- エラーハンドリングの強化
- テスト結果のJSON/CSV形式での出力
- ベンチマークとの統合

---

**作成日**: 2026-02-04
**更新日**: 2026-02-04
**作成者**: Claude Sonnet 4.5
