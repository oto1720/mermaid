# 権限設計書 v3

> 達成目標：最小権限 / 誤付与防止 / 監査可能性 / 運用可能性
> **v3変更点：TEAM_PMを廃止。TEAM_OWNERがPM業務を兼務する4ロール構成に簡素化。**

---

## 1. ロール定義・権限概要

```mermaid
flowchart TB
    subgraph ROLES["👑 ロール階層 （上位は下位の権限をすべて包含）"]
        ADMIN["🔴 SYSTEM_ADMIN\n全データフルアクセス\nユーザー強制削除\nシステム設定変更\n監査ログ閲覧"]
        OWNER["🟠 TEAM_OWNER\nチーム作成者が自動取得\nチーム解散・設定変更\nメンバーキック・ロール変更\nAI生成・ロードマップ確定\nタスク作成・削除・担当変更\n招待リンク発行 ※PMを兼務"]
        MEMBER["🟢 TEAM_MEMBER\n自分のタスクステータス更新\nコメント投稿・自分のコメント削除\n学習ログ記録\n要件定義ウィザード入力\nチーム情報・ロードマップ閲覧"]
        GUEST["🔵 GUEST\nチーム情報の閲覧のみ\n操作系すべて不可\nコメント不可"]
    end

    ADMIN -->|包含| OWNER -->|包含| MEMBER -->|包含| GUEST

    style ADMIN  fill:#ff6b6b,color:#fff,stroke:#cc0000
    style OWNER  fill:#fd7e14,color:#fff,stroke:#c45e00
    style MEMBER fill:#6bcb77,color:#fff,stroke:#339944
    style GUEST  fill:#4d96ff,color:#fff,stroke:#0055cc
```

---

## 2. 権限マトリクス（機能別）

### 2-1. 認証・ユーザー管理

```mermaid
block-beta
    columns 5
    H1["操作 / ロール"]:1 H2["🔴 ADMIN"]:1 H3["🟠 OWNER"]:1 H4["🟢 MEMBER"]:1 H5["🔵 GUEST"]:1

    A1["ユーザー登録"]:1        A2["✅"]:1 A3["✅"]:1 A4["✅"]:1 A5["✅"]:1
    B1["ログイン / ログアウト"]:1 B2["✅"]:1 B3["✅"]:1 B4["✅"]:1 B5["✅"]:1
    C1["自プロフィール編集"]:1   C2["✅"]:1 C3["✅"]:1 C4["✅"]:1 C5["❌"]:1
    D1["自スキル登録・編集"]:1   D2["✅"]:1 D3["✅"]:1 D4["✅"]:1 D5["❌"]:1
    E1["他ユーザー詳細閲覧"]:1   E2["✅"]:1 E3["🔶 チーム内"]:1 E4["🔶 チーム内"]:1 E5["❌"]:1
    F1["ユーザー検索"]:1         F2["✅"]:1 F3["❌"]:1 F4["❌"]:1 F5["❌"]:1
    G1["ユーザー削除"]:1         G2["✅"]:1 G3["❌"]:1 G4["❌"]:1 G5["❌"]:1

    style H1 fill:#333,color:#fff
    style H2 fill:#ff6b6b,color:#fff
    style H3 fill:#fd7e14,color:#fff
    style H4 fill:#6bcb77,color:#fff
    style H5 fill:#4d96ff,color:#fff
```

### 2-2. チーム管理

```mermaid
block-beta
    columns 5
    H1["操作 / ロール"]:1 H2["🔴 ADMIN"]:1 H3["🟠 OWNER"]:1 H4["🟢 MEMBER"]:1 H5["🔵 GUEST"]:1

    A1["チーム新規作成"]:1       A2["✅"]:1 A3["✅"]:1 A4["❌"]:1 A5["❌"]:1
    B1["チーム設定変更"]:1       B2["✅"]:1 B3["✅"]:1 B4["❌"]:1 B5["❌"]:1
    C1["チーム解散"]:1           C2["✅"]:1 C3["✅"]:1 C4["❌"]:1 C5["❌"]:1
    D1["招待リンク発行"]:1       D2["✅"]:1 D3["✅"]:1 D4["❌"]:1 D5["❌"]:1
    E1["メンバー招待承認"]:1     E2["✅"]:1 E3["✅"]:1 E4["❌"]:1 E5["❌"]:1
    F1["メンバーキック"]:1       F2["✅"]:1 F3["✅"]:1 F4["❌"]:1 F5["❌"]:1
    G1["ロール変更"]:1           G2["✅"]:1 G3["✅"]:1 G4["❌"]:1 G5["❌"]:1
    H1b["メンバー一覧閲覧"]:1   H2b["✅"]:1 H3b["✅"]:1 H4b["✅"]:1 H5b["🔶 公開のみ"]:1
    I1["チームダッシュボード"]:1 I2["✅"]:1 I3["✅"]:1 I4["✅"]:1 I5["🔶 公開のみ"]:1

    style H1 fill:#333,color:#fff
    style H2 fill:#ff6b6b,color:#fff
    style H3 fill:#fd7e14,color:#fff
    style H4 fill:#6bcb77,color:#fff
    style H5 fill:#4d96ff,color:#fff
```

### 2-3. 要件定義 / AI生成 / タスク管理

```mermaid
block-beta
    columns 5
    H1["操作 / ロール"]:1 H2["🔴 ADMIN"]:1 H3["🟠 OWNER"]:1 H4["🟢 MEMBER"]:1 H5["🔵 GUEST"]:1

    S1["─── 要件定義 ───"]:5
    A1["ウィザード入力"]:1       A2["✅"]:1 A3["✅"]:1 A4["✅"]:1 A5["❌"]:1
    B1["要件編集・更新"]:1       B2["✅"]:1 B3["✅"]:1 B4["❌"]:1 B5["❌"]:1
    C1["要件削除"]:1             C2["✅"]:1 C3["✅"]:1 C4["❌"]:1 C5["❌"]:1
    D1["MVP範囲変更"]:1          D2["✅"]:1 D3["✅"]:1 D4["❌"]:1 D5["❌"]:1

    S2["─── AI生成 ───"]:5
    E1["AI分析実行"]:1           E2["✅"]:1 E3["✅"]:1 E4["❌"]:1 E5["❌"]:1
    F1["生成結果の編集"]:1       F2["✅"]:1 F3["✅"]:1 F4["❌"]:1 F5["❌"]:1
    G1["再生成・やり直し"]:1     G2["✅"]:1 G3["✅"]:1 G4["❌"]:1 G5["❌"]:1
    I1["ロードマップ確定"]:1     I2["✅"]:1 I3["✅"]:1 I4["❌"]:1 I5["❌"]:1
    J1["担当割り振り変更"]:1     J2["✅"]:1 J3["✅"]:1 J4["❌"]:1 J5["❌"]:1
    JR["ロードマップ閲覧"]:1     JR2["✅"]:1 JR3["✅"]:1 JR4["✅"]:1 JR5["❌"]:1

    S3["─── タスク管理 ───"]:5
    K1["タスク作成・削除"]:1     K2["✅"]:1 K3["✅"]:1 K4["❌"]:1 K5["❌"]:1
    L1["タスク仕様編集"]:1       L2["✅"]:1 L3["✅"]:1 L4["❌"]:1 L5["❌"]:1
    M1["自タスクステータス更新"]:1 M2["✅"]:1 M3["✅"]:1 M4["✅"]:1 M5["❌"]:1
    N1["他タスクステータス更新"]:1 N2["✅"]:1 N3["✅"]:1 N4["❌"]:1 N5["❌"]:1
    O1["コメント投稿"]:1         O2["✅"]:1 O3["✅"]:1 O4["✅"]:1 O5["❌"]:1
    P1["他人コメント削除"]:1     P2["✅"]:1 P3["✅"]:1 P4["❌"]:1 P5["❌"]:1
    PR["タスク閲覧"]:1           PR2["✅"]:1 PR3["✅"]:1 PR4["✅"]:1 PR5["❌"]:1

    style H1 fill:#333,color:#fff
    style H2 fill:#ff6b6b,color:#fff
    style H3 fill:#fd7e14,color:#fff
    style H4 fill:#6bcb77,color:#fff
    style H5 fill:#4d96ff,color:#fff
    style S1 fill:#eee,color:#666
    style S2 fill:#eee,color:#666
    style S3 fill:#eee,color:#666
```

### 2-4. 学習・支援 / 運用管理

```mermaid
block-beta
    columns 5
    H1["操作 / ロール"]:1 H2["🔴 ADMIN"]:1 H3["🟠 OWNER"]:1 H4["🟢 MEMBER"]:1 H5["🔵 GUEST"]:1

    S1["─── 学習・支援 ───"]:5
    A1["学習リソース閲覧"]:1         A2["✅"]:1 A3["✅"]:1 A4["✅"]:1 A5["❌"]:1
    B1["カスタム教材追加"]:1         B2["✅"]:1 B3["✅"]:1 B4["❌"]:1 B5["❌"]:1
    C1["自分の学習ログ記録"]:1       C2["✅"]:1 C3["✅"]:1 C4["✅"]:1 C5["❌"]:1
    D1["他人の学習ログ閲覧"]:1       D2["✅"]:1 D3["✅"]:1 D4["🔶 チーム内"]:1 D5["❌"]:1
    E1["質問テンプレート送信"]:1     E2["✅"]:1 E3["✅"]:1 E4["✅"]:1 E5["❌"]:1

    S2["─── 運用・管理（ADMIN専用）───"]:5
    F1["全チーム一覧閲覧"]:1         F2["✅"]:1 F3["❌"]:1 F4["❌"]:1 F5["❌"]:1
    G1["監査ログ閲覧"]:1             G2["✅"]:1 G3["❌"]:1 G4["❌"]:1 G5["❌"]:1
    I1["ユーザー強制削除"]:1         I2["✅"]:1 I3["❌"]:1 I4["❌"]:1 I5["❌"]:1
    J1["AI APIコスト確認"]:1         J2["✅"]:1 J3["❌"]:1 J4["❌"]:1 J5["❌"]:1
    K1["システム設定変更"]:1         K2["✅"]:1 K3["❌"]:1 K4["❌"]:1 K5["❌"]:1

    style H1 fill:#333,color:#fff
    style H2 fill:#ff6b6b,color:#fff
    style H3 fill:#fd7e14,color:#fff
    style H4 fill:#6bcb77,color:#fff
    style H5 fill:#4d96ff,color:#fff
    style S1 fill:#eee,color:#666
    style S2 fill:#eee,color:#666
```

---

## 3. ロール遷移ルール（誤付与防止）

```mermaid
flowchart TD
    NEW(["👤 新規ユーザー登録"]) --> MEMBER

    MEMBER["🟢 TEAM_MEMBER\nデフォルト付与"]
    OWNER["🟠 TEAM_OWNER\nPM兼務"]
    ADMIN["🔴 SYSTEM_ADMIN"]

    MEMBER -->|"チーム作成時\n自動昇格"| OWNER
    MEMBER -->|"招待リンク経由\nチーム参加"| MEMBER
    OWNER  -->|"チーム譲渡 or 解散"| MEMBER
    OWNER  -.->|"SYSTEM_ADMINのみ\n手動付与"| ADMIN

    subgraph RULES["⚠️ 誤付与防止ルール"]
        R1["🚫 自己ロール昇格は不可"]
        R2["🚫 SYSTEM_ADMINは\n手動付与のみ"]
        R3["📝 ロール変更は\n操作ログに全記録"]
        R4["👑 TEAM_OWNERは\n1チーム1名まで"]
        R5["🔄 OWNER脱退時は\n担当タスクを自動で\n未アサインに変更"]
        R6["✅ TEAM_PMを廃止\nOWNERがPMを兼務することで\n権限の分散・混乱を防止"]
    end

    style ADMIN  fill:#ff6b6b,color:#fff,stroke:#cc0000
    style OWNER  fill:#fd7e14,color:#fff,stroke:#c45e00
    style MEMBER fill:#6bcb77,color:#fff,stroke:#339944
    style RULES  fill:#fff8e1,stroke:#ffc107
```

---

## 4. スコープ設計（リソースの見える範囲）

```mermaid
flowchart TB
    subgraph GLOBAL["🌐 グローバルスコープ　SYSTEM_ADMIN のみ"]
        GA["全ユーザーデータ"]
        GB["全チームデータ"]
        GC["全監査ログ"]
        GD["システム設定"]
    end

    subgraph TEAM["👥 チームスコープ　OWNER / MEMBER"]
        TA["所属チームの\nメンバー情報"]
        TB["チームのタスク\nロードマップ"]
        TC["チームの\n要件定義・AI結果"]
        TD["チームの\n学習リソース"]
    end

    subgraph PERSONAL["👤 パーソナルスコープ　全ロール共通"]
        PA["自分のプロフィール"]
        PB["自分のタスク"]
        PC["自分の学習ログ"]
        PD["自分のスキル情報"]
    end

    GLOBAL -->|"スコープ縮小"| TEAM -->|"スコープ縮小"| PERSONAL

    subgraph RULES2["📌 スコープルール"]
        SR1["他チームのデータは\n存在自体が見えない\n※ 404ではなく403を返す"]
        SR2["MEMBERは同チームの\nメンバー情報のみ閲覧可"]
        SR3["GUESTは公開フラグのある\nチームのみ閲覧可"]
    end

    style GLOBAL   fill:#ffe0e0,stroke:#ff6b6b
    style TEAM     fill:#fff3cd,stroke:#ffc107
    style PERSONAL fill:#d4edda,stroke:#6bcb77
    style RULES2   fill:#e8f4fd,stroke:#4d96ff
```

---

## 5. 監査ログ設計

```mermaid
flowchart LR
    subgraph TRIGGER["🎯 記録トリガー（アクション）"]
        direction TB
        T1["🔐 認証系\nログイン成功・失敗\nPWリセット / アカウント削除"]
        T2["👥 チーム系\nチーム作成・解散\nメンバー招待・キック\nロール変更"]
        T3["📋 要件・AI系\n要件定義 作成・編集・削除\nAI生成実行\nロードマップ確定・変更"]
        T4["✅ タスク系\nタスク作成・削除\nステータス変更 / 担当変更"]
        T5["⚙️ 管理系\nSYSTEM_ADMIN操作全件\n権限設定変更"]
    end

    subgraph STRUCTURE["📦 ログデータ構造"]
        direction TB
        WHO["👤 WHO\nuser_id / email\nrole_at_the_time"]
        WHAT["🔍 WHAT\naction_type\nresource_type / resource_id"]
        WHEN["⏰ WHEN\ntimestamp_utc / session_id"]
        RESULT["📊 RESULT\nsuccess / failure\nerror_code"]
    end

    subgraph POLICY["🗄️ 保持ポリシー"]
        direction TB
        P1["通常ログ\n90日間保持"]
        P2["管理者操作ログ\n1年間保持"]
        P3["セキュリティ系\n永続保持"]
    end

    subgraph ACCESS["🔒 閲覧ルール"]
        direction TB
        A1["SYSTEM_ADMINのみ閲覧可"]
        A2["ダウンロード不可"]
        A3["改ざん不可・追記のみ"]
    end

    TRIGGER -->|"イベント発火"| STRUCTURE -->|"DB書き込み"| POLICY
    POLICY  -->|"アクセス制御"| ACCESS

    style TRIGGER   fill:#fff3cd,stroke:#ffc107
    style STRUCTURE fill:#d4edda,stroke:#6bcb77
    style POLICY    fill:#e8f4fd,stroke:#4d96ff
    style ACCESS    fill:#ffe0e0,stroke:#ff6b6b
```

---

## 6. 実装アーキテクチャ（運用可能性）

```mermaid
flowchart TD
    subgraph FRONTEND["🖥️ フロント層　表示制御のみ"]
        F1["権限に応じてボタン非表示\n※ セキュリティはAPIで担保"]
        F2["403受信時\n操作不可メッセージ表示"]
        F3["401受信時\nログイン画面へリダイレクト"]
    end

    subgraph MIDDLEWARE["⚙️ ミドルウェア層　権限チェック"]
        M1["1. 認証チェック\nJWT / Session検証"]
        M2["2. ロールチェック\n必要ロール以上か確認"]
        M3["3. スコープチェック\nチームID一致確認"]
        M4["4. オーナーチェック\n自分のリソースか確認"]
        M1 --> M2 --> M3 --> M4
    end

    subgraph DB["🗃️ DB設計　RBAC（4ロール）"]
        D1["roles\nid / name / level\n※ ADMIN / OWNER / MEMBER / GUEST"]
        D2["user_roles\nuser_id / role_id\n※ 中間テーブル"]
        D3["team_roles\nteam_id / user_id / role_id\n※ チーム×ロール"]
        D1 --- D2 --- D3
    end

    subgraph CHANGE["🔄 変更対応"]
        C1["新ロール追加時\nDBレコード追加のみ\nコード変更不要"]
        C2["権限変更時\n管理画面から即時反映\n再デプロイ不要"]
    end

    FRONTEND   -->|"APIリクエスト"| MIDDLEWARE
    MIDDLEWARE -->|"権限参照"| DB
    DB --> CHANGE

    style FRONTEND   fill:#e8f4fd,stroke:#4d96ff
    style MIDDLEWARE fill:#fff3cd,stroke:#ffc107
    style DB         fill:#d4edda,stroke:#6bcb77
    style CHANGE     fill:#ffe0e0,stroke:#ff6b6b
```

---

## 7. 権限設計サマリー

```mermaid
flowchart LR
    subgraph G1["🔐 最小権限"]
        G1A["デフォルトはTEAM_MEMBER\n最も弱いロールから開始"]
        G1B["GUESTは閲覧のみ\n操作系は完全ブロック"]
        G1C["OWNERはチーム作成時のみ\n自動昇格・それ以外は手動"]
    end

    subgraph G2["🚫 誤付与防止"]
        G2A["自己ロール昇格は不可"]
        G2B["SYSTEM_ADMINは\n手動付与のみ"]
        G2C["TEAM_PMを廃止し\nロールをシンプル化\n混乱・誤付与リスク低減"]
    end

    subgraph G3["📋 監査可能性"]
        G3A["WHO / WHAT / WHEN / RESULT\n4軸でログ記録"]
        G3B["改ざん不可・追記のみ"]
        G3C["SYSTEM_ADMINのみ閲覧可"]
    end

    subgraph G4["⚙️ 運用可能性"]
        G4A["4ロールのみ\n覚えやすく管理しやすい"]
        G4B["権限チェックは\nミドルウェアに集約"]
        G4C["新ロール追加は\nDBレコード追加のみ"]
    end

    style G1 fill:#ffe0e0,stroke:#ff6b6b
    style G2 fill:#fff3cd,stroke:#ffc107
    style G3 fill:#d4edda,stroke:#6bcb77
    style G4 fill:#e8f4fd,stroke:#4d96ff
```
