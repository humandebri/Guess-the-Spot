# Guess-the-Spot 詳細実装計画

## 🎯 プロジェクト目標
ICP上で動作する分散型位置推理ゲームのMVPを12週間で開発

## 📊 技術スタック
- **Backend**: Motoko (ICP Canisters)
- **Frontend**: React + TypeScript (PWA)
- **Storage**: ICP Stable Memory
- **Map**: Mapbox GL JS
- **Auth**: Internet Identity

## 🏗️ アーキテクチャ概要

```
┌─────────────┐     ┌───────────────┐     ┌─────────────────┐
│   Frontend  │────▶│  GameEngine   │────▶│  RewardMint     │
│  (React PWA)│     │   Canister    │     │  (SPOT Token)   │
└─────────────┘     └───────┬───────┘     └─────────────────┘
                            │                       ▲
                            ▼                       │
                    ┌───────────────┐               │
                    │   PhotoNFT    │               │
                    │   Canister    │               │
                    └───────┬───────┘               │
                            │                       │
                            ▼                       │
                    ┌───────────────┐               │
                    │ ReputationOracle │───────────┘
                    └───────────────┘
```

## 📅 12週間実装スケジュール

### 🚀 Phase 1: 基礎インフラ (Week 1-4)

#### Week 1-2: SPOTトークン実装
**目標**: ICRC-1準拠の報酬トークン基盤構築

```motoko
// 実装内容
- mint(), transfer(), balance() 基本関数
- approve(), transferFrom() 委任機能
- 手数料メカニズム (0.01 SPOT/transaction)
- エラーハンドリングとイベント通知
```

**技術的考慮事項**:
- Principalベースの残高管理
- stable変数でのデータ永続化
- GameEngineのみmint権限を持つアクセス制御

#### Week 3-4: PhotoNFT & ストレージ
**目標**: 写真のオンチェーン保存とNFT化

```motoko
// 実装内容
- ICRC-7準拠NFT発行機能
- 256KBチャンク分割アルゴリズム
- Stable Memory直接操作でコスト最適化
- メタデータ管理 (lat/lon/azim/timestamp/quality)
```

**技術的考慮事項**:
- Region APIを使用した効率的なメモリ管理
- 1MB写真 = 4チャンク = 約$0.005/年のコスト
- perceptual hashによる重複検出準備

### 🎮 Phase 2: ゲームコア (Week 5-8)

#### Week 5-6: GameEngine実装
**目標**: 位置推理とスコアリングの中核ロジック

```motoko
// Vincenty formula実装
func calculateDistance(lat1: Float, lon1: Float, lat2: Float, lon2: Float) : Float {
    // 地球楕円体モデルによる高精度距離計算
    // 精度: ±0.5mm
};

// スコア計算
func calculateScore(distance: Float, azimuthError: Float) : Nat {
    let Sd = 1.0 - (distance - 25.0) / (1000.0 - 25.0);
    let Sphi = 1.0 - azimuthError / 30.0;
    return Int.abs(Float.toInt(100.0 * (Sd ** 1.3) * (Sphi ** 0.7)));
};
```

**実装詳細**:
- ラウンド生成ロジック（写真ランダム選択）
- プレイヤー推理の受付と検証
- 報酬計算（漸減係数B(t)適用）
- RewardMint呼び出しによる自動報酬分配

#### Week 7: ReputationOracle
**目標**: 写真品質管理とユーザー信頼性評価

```motoko
// Quality Score更新
func updateQualityScore(photoId: Nat, hitRate: Float, badRatio: Float) : Float {
    let F = 1.0 - 0.7 * hitRate - 0.3 * badRatio;
    let alpha = 0.8;
    return alpha * oldScore + (1 - alpha) * F;
};
```

**BAN判定ロジック**:
- soft-ban: Q < 0.15 && 出題数 ≥ 30
- hard-ban: Q < 0.05 && bad_ratio > 0.5

#### Week 8: Canister間通信
**目標**: 各Canisterの連携実装

```motoko
// Inter-canister calls
- GameEngine → RewardMint: mint報酬
- GameEngine → ReputationOracle: 品質スコア更新
- GameEngine → PhotoNFT: メタデータ取得
- ReputationOracle → PhotoNFT: qualityフィールド更新
```

### 🌐 Phase 3: フロントエンド (Week 9-11)

#### Week 9-10: 基盤構築
**技術スタック**:
```json
{
  "build": "vite",
  "framework": "react",
  "language": "typescript",
  "auth": "@dfinity/auth-client",
  "agent": "@dfinity/agent",
  "map": "mapbox-gl",
  "state": "zustand",
  "style": "tailwindcss"
}
```

**実装内容**:
- Internet Identity統合
- Canister接続層（Actor生成）
- 地図UI（Mapbox GL JS）
- PWA基本設定

#### Week 11: 画面実装
**主要画面**:
1. **写真アップロード**
   - EXIF位置情報抽出
   - 方位入力UI
   - チャンク分割アップロード
   - プログレス表示

2. **ゲームプレイ**
   - 写真表示
   - 地図ピン配置
   - 方位推定入力
   - リアルタイムスコア計算

3. **統計・ギャラリー**
   - リーダーボード
   - NFTコレクション表示
   - 個人統計

### 🔒 Phase 4: 品質保証 (Week 12)

#### セキュリティ実装
- GPS/方位データ検証基盤
- レート制限（1分5回まで）
- Sybil攻撃対策（Internet Identity + Device Fingerprint）

#### テスト戦略
```typescript
// テストカバレッジ目標
- Unit Tests: 80%+ (Motoko: vessel, TS: vitest)
- Integration Tests: Canister間通信の全パス
- E2E Tests: 主要ユーザーフロー (Playwright)
- Load Tests: 1000並行ユーザー想定
```

## 🚨 リスクと対策

| リスク | 影響度 | 対策 |
|--------|--------|------|
| Stable Memory容量制限 | 高 | 8GBまで拡張可能、古い写真のアーカイブ化 |
| Cycle消費過多 | 中 | バッチ処理、キャッシュ戦略 |
| 不正GPS投稿 | 高 | SafetyNet/App Attest統合（Phase 2） |
| UI/UXの複雑さ | 中 | プロトタイプによる早期検証 |

## 📈 成功指標（MVP）

- [ ] 100枚以上の写真アップロード成功
- [ ] 1000ラウンド以上のゲームプレイ
- [ ] 平均レスポンス時間 < 2秒
- [ ] Cycle効率: 1ゲーム < 0.001 USD
- [ ] Quality Score機能による不正写真除外率 > 90%

## 🔄 継続的改善（Post-MVP）

1. **Month 4**: DAO移行準備、Treasury管理
2. **Month 5**: モバイルネイティブアプリ
3. **Month 6**: ARヒント機能、マルチ言語対応

## 💻 開発環境セットアップ

```bash
# Backend
dfx start --clean
dfx deploy --network local

# Frontend
npm install
npm run dev

# Testing
npm run test
dfx canister call reward_mint balance '(principal "test-user")'
```

---

**Next Step**: Week 1-2のSPOTトークン実装から開始