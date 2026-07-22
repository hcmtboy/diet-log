# 体づくりログ（ダイエット記録アプリ）— 開発引き継ぎ資料

このリポジトリは、単一ファイルのダイエット記録アプリ `index.html` を GitHub Pages で公開するものです。
この README は開発担当（Claude Code）向けの引き継ぎ資料であり、アプリの設計意図・データ構造・デプロイ手順・iPhone連携をまとめています。

> **役割分担**: トレーニング/栄養の方針決定・データレビューは別途チャット（Cowork）側のトレーナーが担当します。ここ（Claude Code）は開発・デプロイ・環境まわりを担当します。評価点の配分・コーチのコメント・目標値・取り込む項目は"トレーニング方針"に直結するため、変更仕様はトレーナー側から降りてくる想定です。UI改善やリファクタ、バグ修正は自由に進めてOKです。

---

## 1. 基本方針（触るときの前提）

- **単一HTMLファイル**（`index.html`）で完結。CSS・JSはすべてインライン。**外部ライブラリ/CDNは使わない**（オフライン動作とプライバシーのため）。
- **データは端末内のみ**。`localStorage` にだけ保存し、サーバーへは一切送信しない。→ リポジトリを公開しても個人データは漏れない。
- **体重推移グラフは自前のcanvas描画**（Chart.js等は使わない）。
- PWA対応済み（`apple-mobile-web-app-capable`、`apple-touch-icon` はdata URIのダンベルアイコン、standalone表示、アプリ名「体づくりログ」）。
- 対象は iPhone をメイン端末とする1人運用。クロス端末の自動同期はしていない（バックアップはJSON書き出しで対応）。

---

## 2. 画面構成（下部タブ）

| タブ | id | 役割 |
|---|---|---|
| 今日 | `v-today` | 記録入力。**手入力は食事・水分・体調のみ**。体重/歩数/運動は「自動取得」エリアに分離 |
| 記録 | `v-history` | 体重推移グラフ、直近7日サマリー、記録一覧（タップで編集） |
| コーチ | `v-coach` | 連続記録・7日平均・たんぱく質/水分/ジムの達成度・見立て（冷静な分析＋要所で褒めるトーン） |
| 設定 | `v-settings` | 目標設定、スプレッドシート書き出し、ヘルスケア連携手順、JSONバックアップ、評価点の説明 |

「今日」タブの入力の考え方:
- **自動取得（ヘルスケア・Nike Run Club）**: 体重・歩数・ランニング・運動。ふだんは触らない。取れなかった時だけ折りたたみ（`#manualAuto`）を開いて手修正。
- **手入力**: 朝食/昼食/夕食/間食、推定カロリー、たんぱく質、水分、便通、便通メモ、睡眠、回復度、メモ。

---

## 3. データ構造

`localStorage` キー: `diet_log_v1`

```js
store = {
  goals: {
    targetWeight: 72.5, steps: 7000, gym: 3,
    protein: 130, water: 1.8, kcalLo: 1900, kcalHi: 2300
  },
  entries: {
    "YYYY-MM-DD": {
      date, weight, run, walk, steps, gym(bool),
      parts: { back, chest, shoulder, legs },  // 1 or undefined
      other,                                    // その他の運動（Nikeワークアウト名など）
      breakfast, lunch, dinner, snack,
      kcal, protein, water,
      bowel,      // 'あり' | 'なし' | ''
      bowelMemo, sleep, recovery, note
    }
  }
}
```

- 初期データは目標値（`goals`）のみ。記録（`entries`）は空スタート（`seed()`）。個人の実ログはコードに含めない方針。
- 空判定は `has(x)`（''・null・undefined を空とみなす）、数値化は `num(x)`。

---

## 4. 評価点ロジック（`calcScore(entry)`）— 合計100点

トレーニング方針に直結する部分。変更する場合はトレーナー側の指示に従うこと。

- **運動 25**: アクティブ15（ジム or ラン+ウォーク>0 or その他運動あり）＋ 歩数 `min(10, round(steps/goal.steps*10))`
- **食事 35**: たんぱく質 `min(15, round(protein/goal.protein*15))` ＋ カロリー20（`kcalLo〜kcalHi`で20 / ±400以内で13 / それ以外6 / 未入力0）
- **水分 20**: `min(20, round(water/goal.water*20))`
- **便通 10**: あり10 / なし5 / 未選択0
- **記録 10**: 体重 or いずれかの食事が入っていれば10

判定ラベル `ratingLabel(total)`: ≥88 非常に良い / ≥78 良い / ≥65 まずまず / <65 要調整。

コーチ文言は保存後の `coachMessage(entry, score)` と コーチタブの `renderCoach()` にある（ここもトレーナー方針に属する）。

---

## 5. ヘルスケア/Nike 取り込みの仕様（重要）

### 取り込み経路
1. **URLハッシュ（自動記録の本命）**: アプリを `#d=...;w=...` 付きで開くと `init()`→`handleIncomingData()` が発火。既存のその日の食事等を保持したままマージして**自動保存**する。
2. **クリップボード**: 「🩺 いま取り込む」ボタン→`importFromHealth()`→`applyHealthText()`。フォームに反映するだけ（保存はユーザーが行う）。

### ペイロード形式（キー=値を `;` 区切り。順不同・一部欠落OK）

```
d=2026-07-22;w=75.9;s=8900;r=3.5;k=0;o=Nike Run Club 5km
```

| キー | 意味 | 例 |
|---|---|---|
| `d` | 日付（YYYY-MM-DD、/区切りも可） | `2026-07-22` |
| `w` | 体重kg | `75.9` |
| `s` | 歩数 | `8900` |
| `r` | ランニングkm | `3.5` |
| `k` | ウォーキングkm | `0` |
| `g` | ジム（`1`/`true`/`あり`で有効） | `1` |
| `o` | 運動名（その他の運動/ワークアウト） | `Nike Run Club 5km` |

キーが一切無い素の文字列（例「体重 74.8kg、歩数 8,540歩」）でも、数字2つを体重・歩数として拾うフォールバックあり（`applyHealthText`）。

パース処理を変える場合、`applyHealthText()` と `handleIncomingData()` の両方の正規表現を合わせること。

---

## 6. 書き出し / バックアップ

- `copyRow()`: 指定日の1行をTSVでクリップボードへ（Googleスプレッドシートの日別ログにそのまま貼れる）。列順は下記。
- `downloadCSV()`: 全記録をCSV（BOM付き）でダウンロード。
- `exportJSON()` / `importJSON()`: `store` 丸ごとのバックアップ・復元。

スプレッドシートの列順（`downloadCSV` の head と一致）:
```
日付, 体重(kg), ランニング(km), ウォーキング(km), 歩数, ジム, 背中, 肩, 胸, 下半身,
その他の運動, 朝食, 昼食, 夕食, 間食・外食, 推定摂取カロリー(kcal), 推定たんぱく質(g),
水分(L), 便通, 便通・体調メモ, 睡眠(h), 回復度, 今日の評価点, 評価, 備考
```

---

## 7. デプロイ（GitHub Pages・無料プラン）

- **リポジトリは Public のままにすること**。無料プランでは private にすると Pages が公開されない（＝URLが機能しなくなる）。個人データはコードに含まれないので公開で問題なし。
- ファイル名は必ず `index.html`（ルート直下）。

初回セットアップ（gh CLI）:
```bash
gh repo create diet-log --public --source=. --remote=origin --push
gh api -X POST repos/{owner}/diet-log/pages -f 'source[branch]=main' -f 'source[path]=/'
gh api repos/{owner}/diet-log/pages --jq .html_url   # 公開URL確認
```

更新フロー:
```bash
# 新しい index.html に差し替えたら
git add index.html && git commit -m "update app" && git push
# 反映まで数十秒〜1分
```

公開URL例: `https://<ユーザー名>.github.io/diet-log/`

---

## 8. iPhone連携（参考・トレーナー側で案内済み）

### ホーム画面アプリ化
公開URLを **Safari** で開く → 共有 → 「ホーム画面に追加」。standalone表示＋ダンベルアイコンで起動。

### 自動記録ショートカット
1. OMRON connect → ヘルスケアへ体重出力ON、Nike Run Club → ヘルスケアへ書き出しON（歩数はiPhoneが自動集計）。
2. ショートカットで ヘルスケアから 体重(最新1件)・歩数(今日の合計)・当日のランニング距離・ワークアウト を取得。
3. テキストで `d=[今日];w=[体重];s=[歩数];r=[ラン];o=[運動名]` を作り、「URLを開く」で `https://<ユーザー名>.github.io/diet-log/#[そのテキスト]` を開く。
4. 個人用オートメーション（毎日23:50など・「実行の前に尋ねる」オフ）で1日の終わりに自動実行。

---

## 9. トレーナー側とのデータ橋渡し

アプリの記録は端末内のみ。トレーナー（別チャット）が最新データでレビューできるよう、**週1回**を目安に「設定タブの行コピー or CSV書き出し」で Googleスプレッドシートへ反映する運用。スプレッドシートのURLは公開リポジトリに載せず、各自の共有場所（チャット等）で管理する。

---

## 10. よく触る関数の場所（index.html 内）

- 目標・初期データ: `seed()`
- 保存: `saveEntry()` / フォーム読取 `readForm()` / 過去日編集 `editEntry(date)`
- 評価点: `calcScore(entry)` / ラベル `ratingLabel(total)`
- 取り込み: `applyHealthText(txt, notify)` / `handleIncomingData(txt)` / `importFromHealth()`
- 自動取得サマリー: `renderAutoSummary()`
- 入力漏れ導線: `renderMissingDays()` / `jumpDate(date)`
- グラフ: `renderChart()`（canvas自前描画）
- コーチ: `coachMessage()` / `renderCoach()`
- 書き出し: `rowFor(date)` / `copyRow()` / `downloadCSV()` / `exportJSON()` / `importJSON()`

---

## 変更時のチェックリスト

- [ ] 外部ライブラリ/CDNを増やしていない（オフライン維持）
- [ ] localStorage 以外にデータを送っていない
- [ ] スコア/目標/コーチ文言の変更はトレーナー方針に沿っている
- [ ] 取り込みペイロードのキーを変えたら `applyHealthText` と `handleIncomingData` 両方を更新
- [ ] `index.html` のファイル名・ルート配置を維持（Pages要件）
- [ ] リポジトリは Public のまま
