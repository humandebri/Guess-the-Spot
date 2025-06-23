### 📑 これまでに合意した **レーティング・システム要件まとめ**

---

## 1. コア概念

| 概念                 | 説明                                                                  |
| ------------------ | ------------------------------------------------------------------- |
| **playerRating**   | プレイヤーの実力値（初期 1500）                                                  |
| **photoRating**    | 写真の難易度値（初期 1500 など）                                                 |
| **score**          | 1 回のプレイで得たスコア（距離や点数で決定）                                             |
| **avgScore (写真側)** | 写真の現在の平均スコア                                           |
| **result**         | 勝敗フラグ：`1 = score > avgScore`, `0.5 = ±許容範囲`, `0 = score < avgScore` |

---

## 2. レーティング更新式（Elo 原理）

```ts
// 期待勝率（プレイヤー視点）
const expected = 1 / (1 + 10 ** ((photoRating - playerRating) / 400));

// 動的 K 係数（上級者ほど小さく）
function dynamicK(r: number) {
  if (r < 1600) return 32;
  if (r < 2000) return 24;
  return 16;            // 高レート帯
}

// プレイヤー更新
playerRating += dynamicK(playerRating) * (result - expected);

// 写真側も逆方向で更新
const photoExpected = 1 / (1 + 10 ** ((playerRating - photoRating) / 400));
photoRating += dynamicK(photoRating) * ((1 - result) - photoExpected);
```

### 効果

* **強いプレイヤー × 簡単写真**

  * 勝っても +数 pt、負けると −数十 pt
* **高難度写真で成功**

  * 期待値が低いので +10〜20 pt など大幅増

---

## 3. インフレ抑制策

1. **K 係数段階制**

   * `1600 未満: 32`, `1600–1999: 24`, `2000 以上: 16`
2. **レート上限**

   * 例：`maxRating = 2500`
3. **期間内上昇幅制限**

   * 例：同一ユーザー `＋100 pt / day` でクリップ
4. **シーズン制／減衰**（オプション）

   * 期末に全員を平均へ収束 or 未プレイ減衰

---

## 4. 勝敗判定ロジック

```ts
// 許容幅を ±δ% にして引き分けを設定してもよい
if (score > avgScore * (1 + delta))      result = 1;
else if (score < avgScore * (1 - delta)) result = 0;
else                                     result = 0.5;
```

* **delta** 例：`0.02`（±2% 以内は引き分け）

---

テーブル設計（最小構成）

| テーブル        | 主なカラム                                                                                           |
| ----------- | ----------------------------------------------------------------------------------------------- |
| **players** | `id (UUID)`, `name`, `rating INT`, `rating_last_updated TIMESTAMP`                              |
| **photos**  | `id (UUID)`, `avg_score FLOAT`, `play_count INT`, `rating INT`, `rating_last_updated TIMESTAMP` |
| **guesses** | `id`, `player_id FK`, `photo_id FK`, `score FLOAT`, `created_at TIMESTAMP`                      |

* **DB トリガ** or **API Route** で `guesses` 追加時に

  1. `avg_score, play_count` 更新
  2. 上記 Elo 式で双方の `rating` 更新
  3. クリップ（上限・日次増分チェック）

---

## 6. API ルート (Next.js) 概要

```
POST /api/submit-guess
{
  playerId,
  photoId,
  score
}
↓
• 写真の平均点・play_count 取得・更新
• 勝敗判定 → result
• playerRating, photoRating を Elo 式で再計算
• 変動後の値を players / photos テーブルへ保存
• レスポンスで新レートと photoRating を返却
```

---

このまとめを基に **実装サンプル** や **詳細設計** が必要になったら、いつでもリクエストしてください 🚀
