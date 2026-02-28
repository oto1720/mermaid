# 権限設計書 v5

> 達成目標：最小権限 / 誤付与防止 / 監査可能性 / 運用可能性
> **v5変更点：複数チーム所属に対応。グローバルロール × チームスコープロールの2軸設計に変更。**

---

## 1. ロール2軸設計の概念

```mermaid
flowchart LR
    subgraph GLOBAL["🌐 グローバルロール（アカウント単位）"]
        direction TB
        GA["🔴 SYSTEM_ADMIN\n全システム管理者\n※ 手動付与のみ"]
        GB["🟣 LOGIN_USER\n認証済みユーザー\nどのチームにも未所属 or\n全チーム脱退後に戻る"]
        GC["⚪ GUEST\n未ログイン"]
    end

    subgraph TEAM_SCOPE["👥 チームスコープロール（チーム単位・チームごとに独立）"]
        direction TB
        TA["🟠 TEAM_OWNER\nそのチームの作成者\nPM兼務"]
        TB["🟢 TEAM_MEMBER\nそのチームの一般メンバー"]
    end

    subgraph EXAMPLE["💡 複数チーム所属の例"]
        direction TB
        E1["ユーザーA\n├ チームX → 🟠 TEAM_OWNER\n├ チームY → 🟢 TEAM_MEMBER\n└ チームZ → 未所属（🟣 LOGIN_USER扱い）"]
    end

    GLOBAL -->|"チーム参加・作成で\nチームロールを追加付与"| TEAM_SCOPE
    TEAM_SCOPE --> EXAMPLE

    style GLOBAL     fill:#e8f4fd,stroke:#4d96ff
    style TEAM_SCOPE fill:#fff3cd,stroke:#ffc107
    style EXAMPLE    fill:#d4edda,stroke:#6bcb77
```

---

## 2. ロール定義一覧

```mermaid
flowchart TB
    subgraph GLOBAL_ROLES["🌐 グローバルロール"]
        ADMIN["🔴 SYSTEM_ADMIN\n【付与条件】手動付与のみ\n─────────────\n全データフルアクセス\nユーザー強制削除\nシステム設定変更\n監査ログ閲覧\n全チーム横断での操作"]
        LUSER["🟣 LOGIN_USER\n【付与条件】新規登録後に自動付与\n─────────────\n自プロフィール・スキル編集\n新規チーム作成\n公開チームの概要閲覧\nチーム内部データへのアクセス不可"]
        GUEST["⚪ GUEST\n【付与条件】未ログイン状態\n─────────────\nLP・登録・ログイン画面のみ\n一切の操作不可"]
    end

    subgraph TEAM_ROLES["👥 チームスコープロール（チームごとに独立）"]
        OWNER["🟠 TEAM_OWNER\n【付与条件】チーム作成時に自動付与\n─────────────\nチーム解散・設定変更\nメンバーキック・ロール変更\nAI生成・ロードマップ確定\nタスク作成・削除・担当変更\n招待リンク発行 ※PMを兼務"]
        MEMBER["🟢 TEAM_MEMBER\n【付与条件】招待リンクでチーム参加時\n─────────────\n自分のタスクステータス更新\nコメント投稿・自コメント削除\n学習ログ記録\n要件定義ウィザード入力\nチーム情報・ロードマップ閲覧"]
    end

    ADMIN -.- LUSER -.- GUEST
    OWNER -.- MEMBER

    style ADMIN  fill:#ff6b6b,color:#fff,stroke:#cc0000
    style LUSER  fill:#9b59b6,color:#fff,stroke:#6c3483
    style GUEST  fill:#adb5bd,color:#fff,stroke:#6c757d
    style OWNER  fill:#fd7e14,color:#fff,stroke:#c45e00
    style MEMBER fill:#6bcb77,color:#fff,stroke:#339944
```

---

## 3. ロール取得・遷移フロー（複数チーム対応）

```mermaid
flowchart TD
    GUEST(["⚪ GUEST\n未ログイン"])
    LUSER["🟣 LOGIN_USER\n認証済み・グローバル"]
    OWNER["🟠 TEAM_OWNER\nチームXのオーナー"]
    MEMBER["🟢 TEAM_MEMBER\nチームYのメンバー"]
    ADMIN["🔴 SYSTEM_ADMIN"]

    GUEST  -->|"登録 / ログイン"| LUSER
    LUSER  -->|"チームX作成\n→ チームXでOWNER付与"| OWNER
    LUSER  -->|"チームY招待リンクで参加\n→ チームYでMEMBER付与"| MEMBER
    OWNER  -->|"チームX解散 or 譲渡\n→ チームXのロール削除"| LUSER
    MEMBER -->|"チームY脱退 or キック\n→ チームYのロール削除"| LUSER
    LUSER  -.->|"SYSTEM_ADMINのみ手動付与"| ADMIN

    NOTE1["💡 同一ユーザーが\nチームXでOWNER かつ\nチームYでMEMBER\nを同時に持てる"]
    NOTE2["💡 全チームを脱退すると\nLOGIN_USERに戻る\n（アカウントは存在し続ける）"]

    OWNER  -.- NOTE1
    MEMBER -.- NOTE1
    LUSER  -.- NOTE2

    subgraph RULES["⚠️ 誤付与防止ルール"]
        R1["🚫 自己ロール昇格は不可"]
        R2["🚫 SYSTEM_ADMINは手動付与のみ"]
        R3["📝 ロール変更は操作ログに全記録"]
        R4["👑 TEAM_OWNERは1チームにつき1名まで"]
        R5["🔄 脱退・キック時はそのチームの\n担当タスクを自動で未アサインに変更"]
        R6["✅ 他チームのロールには一切影響しない\nチームロールは完全独立"]
    end

    style ADMIN  fill:#ff6b6b,color:#fff,stroke:#cc0000
    style OWNER  fill:#fd7e14,color:#fff,stroke:#c45e00
    style MEMBER fill:#6bcb77,color:#fff,stroke:#339944
    style LUSER  fill:#9b59b6,color:#fff,stroke:#6c3483
    style GUEST  fill:#adb5bd,color:#fff,stroke:#6c757d
    style RULES  fill:#fff8e1,stroke:#ffc107
```

---

## 4. 権限マトリクス（機能別）

### 4-1. 認証・ユーザー管理

```mermaid
block-beta
    columns 6
    H1["操作 / ロール"]:1 H2["🔴 ADMIN"]:1 H3["🟠 OWNER"]:1 H4["🟢 MEMBER"]:1 H5["🟣 LOGIN_USER"]:1 H6["⚪ GUEST"]:1

    A1["新規登録"]:1               A2["✅"]:1 A3["✅"]:1 A4["✅"]:1 A5["✅"]:1 A6["✅"]:1
    B1["ログイン / ログアウト"]:1   B2["✅"]:1 B3["✅"]:1 B4["✅"]:1 B5["✅"]:1 B6["✅"]:1
    C1["自プロフィール編集"]:1      C2["✅"]:1 C3["✅"]:1 C4["✅"]:1 C5["✅"]:1 C6["❌"]:1
    D1["自スキル登録・編集"]:1      D2["✅"]:1 D3["✅"]:1 D4["✅"]:1 D5["✅"]:1 D6["❌"]:1
    E1["他ユーザー詳細閲覧"]:1      E2["✅"]:1 E3["🔶 同チーム内"]:1 E4["🔶 同チーム内"]:1 E5["❌"]:1 E6["❌"]:1
    F1["ユーザー検索"]:1            F2["✅"]:1 F3["❌"]:1 F4["❌"]:1 F5["❌"]:1 F6["❌"]:1
    G1["ユーザー削除"]:1            G2["✅"]:1 G3["❌"]:1 G4["❌"]:1 G5["❌"]:1 G6["❌"]:1

    style H1 fill:#333,color:#fff
    style H2 fill:#ff6b6b,color:#fff
    style H3 fill:#fd7e14,color:#fff
    style H4 fill:#6bcb77,color:#fff
    style H5 fill:#9b59b6,color:#fff
    style H6 fill:#adb5bd,color:#fff
```

### 4-2. チーム管理

```mermaid
block-beta
    columns 6
    H1["操作 / ロール"]:1 H2["🔴 ADMIN"]:1 H3["🟠 OWNER"]:1 H4["🟢 MEMBER"]:1 H5["🟣 LOGIN_USER"]:1 H6["⚪ GUEST"]:1

    A1["チーム新規作成"]:1         A2["✅"]:1 A3["✅"]:1 A4["✅"]:1 A5["✅"]:1 A6["❌"]:1
    B1["チーム設定変更"]:1         B2["✅"]:1 B3["✅ 自チームのみ"]:1 B4["❌"]:1 B5["❌"]:1 B6["❌"]:1
    C1["チーム解散"]:1             C2["✅"]:1 C3["✅ 自チームのみ"]:1 C4["❌"]:1 C5["❌"]:1 C6["❌"]:1
    D1["招待リンク発行"]:1         D2["✅"]:1 D3["✅ 自チームのみ"]:1 D4["❌"]:1 D5["❌"]:1 D6["❌"]:1
    E1["招待リンクで参加"]:1       E2["✅"]:1 E3["✅"]:1 E4["✅"]:1 E5["✅"]:1 E6["❌"]:1
    F1["メンバーキック"]:1         F2["✅"]:1 F3["✅ 自チームのみ"]:1 F4["❌"]:1 F5["❌"]:1 F6["❌"]:1
    G1["ロール変更"]:1             G2["✅"]:1 G3["✅ 自チームのみ"]:1 G4["❌"]:1 G5["❌"]:1 G6["❌"]:1
    H1b["メンバー一覧閲覧"]:1     H2b["✅"]:1 H3b["✅ 自チームのみ"]:1 H4b["✅ 自チームのみ"]:1 H5b["🔶 公開チームのみ"]:1 H6b["❌"]:1
    I1["チームダッシュボード"]:1   I2["✅"]:1 I3["✅ 自チームのみ"]:1 I4["✅ 自チームのみ"]:1 I5["🔶 公開チームのみ"]:1 I6["❌"]:1

    style H1 fill:#333,color:#fff
    style H2 fill:#ff6b6b,color:#fff
    style H3 fill:#fd7e14,color:#fff
    style H4 fill:#6bcb77,color:#fff
    style H5 fill:#9b59b6,color:#fff
    style H6 fill:#adb5bd,color:#fff
```

### 4-3. 要件定義 / AI生成 / タスク管理

```mermaid
block-beta
    columns 6
    H1["操作 / ロール"]:1 H2["🔴 ADMIN"]:1 H3["🟠 OWNER"]:1 H4["🟢 MEMBER"]:1 H5["🟣 LOGIN_USER"]:1 H6["⚪ GUEST"]:1

    S1["─── 要件定義 ───"]:6
    A1["ウィザード入力"]:1          A2["✅"]:1 A3["✅"]:1 A4["✅"]:1 A5["❌"]:1 A6["❌"]:1
    B1["要件編集・更新"]:1          B2["✅"]:1 B3["✅"]:1 B4["❌"]:1 B5["❌"]:1 B6["❌"]:1
    C1["要件削除"]:1                C2["✅"]:1 C3["✅"]:1 C4["❌"]:1 C5["❌"]:1 C6["❌"]:1
    D1["MVP範囲変更"]:1             D2["✅"]:1 D3["✅"]:1 D4["❌"]:1 D5["❌"]:1 D6["❌"]:1

    S2["─── AI生成 ───"]:6
    E1["AI分析実行"]:1              E2["✅"]:1 E3["✅"]:1 E4["❌"]:1 E5["❌"]:1 E6["❌"]:1
    F1["生成結果の編集"]:1          F2["✅"]:1 F3["✅"]:1 F4["❌"]:1 F5["❌"]:1 F6["❌"]:1
    G1["再生成・やり直し"]:1        G2["✅"]:1 G3["✅"]:1 G4["❌"]:1 G5["❌"]:1 G6["❌"]:1
    I1["ロードマップ確定"]:1        I2["✅"]:1 I3["✅"]:1 I4["❌"]:1 I5["❌"]:1 I6["❌"]:1
    J1["担当割り振り変更"]:1        J2["✅"]:1 J3["✅"]:1 J4["❌"]:1 J5["❌"]:1 J6["❌"]:1
    JR["ロードマップ閲覧"]:1        JR2["✅"]:1 JR3["✅"]:1 JR4["✅"]:1 JR5["❌"]:1 JR6["❌"]:1

    S3["─── タスク管理 ───"]:6
    K1["タスク作成・削除"]:1        K2["✅"]:1 K3["✅"]:1 K4["❌"]:1 K5["❌"]:1 K6["❌"]:1
    L1["タスク仕様編集"]:1          L2["✅"]:1 L3["✅"]:1 L4["❌"]:1 L5["❌"]:1 L6["❌"]:1
    M1["自タスクステータス更新"]:1   M2["✅"]:1 M3["✅"]:1 M4["✅"]:1 M5["❌"]:1 M6["❌"]:1
    N1["他タスクステータス更新"]:1   N2["✅"]:1 N3["✅"]:1 N4["❌"]:1 N5["❌"]:1 N6["❌"]:1
    O1["コメント投稿"]:1            O2["✅"]:1 O3["✅"]:1 O4["✅"]:1 O5["❌"]:1 O6["❌"]:1
    P1["他人コメント削除"]:1        P2["✅"]:1 P3["✅"]:1 P4["❌"]:1 P5["❌"]:1 P6["❌"]:1

    style H1 fill:#333,color:#fff
    style H2 fill:#ff6b6b,color:#fff
    style H3 fill:#fd7e14,color:#fff
    style H4 fill:#6bcb77,color:#fff
    style H5 fill:#9b59b6,color:#fff
    style H6 fill:#adb5bd,color:#fff
    style S1 fill:#eee,color:#666
    style S2 fill:#eee,color:#666
    style S3 fill:#eee,color:#666
```

---

## 5. スコープ設計（複数チーム対応）

```mermaid
flowchart TB
    subgraph GLOBAL["🌐 グローバルスコープ　SYSTEM_ADMIN のみ"]
        GA["全ユーザーデータ"]
        GB["全チームデータ"]
        GC["全監査ログ / システム設定"]
    end

    subgraph USER_SCOPE["🟣 ログインスコープ　LOGIN_USER"]
        UA["自プロフィール・スキル"]
        UB["公開チームの概要のみ"]
        UC["新規チーム作成"]
    end

    subgraph TEAM_A["👥 チームAスコープ　OWNERまたはMEMBERとして所属"]
        AA["チームAのタスク・ロードマップ"]
        AB["チームAの要件定義・AI結果"]
        AC["チームAのメンバー情報"]
    end

    subgraph TEAM_B["👥 チームBスコープ　OWNERまたはMEMBERとして所属"]
        BA["チームBのタスク・ロードマップ"]
        BB["チームBの要件定義・AI結果"]
        BC["チームBのメンバー情報"]
    end

    GLOBAL -->|"スコープ縮小"| USER_SCOPE
    USER_SCOPE -->|"チームA参加"| TEAM_A
    USER_SCOPE -->|"チームB参加"| TEAM_B

    RULE1["⚠️ チームAのデータはチームBから見えない\nチームスコープは完全独立"]
    TEAM_A -.- RULE1
    TEAM_B -.- RULE1

    style GLOBAL    fill:#ffe0e0,stroke:#ff6b6b
    style USER_SCOPE fill:#ede7f6,stroke:#9b59b6
    style TEAM_A    fill:#fff3cd,stroke:#ffc107
    style TEAM_B    fill:#d4edda,stroke:#6bcb77
    style RULE1     fill:#e8f4fd,stroke:#4d96ff
```

---

## 6. DB設計（2軸ロール対応）

```mermaid
flowchart LR
    subgraph TABLES["🗃️ テーブル構成"]
        direction TB
        U["users\nid / email / name\npassword_hash\ncreated_at"]
        GR["global_roles\nid / name / level\n─────────────\nSYSTEM_ADMIN: 100\nLOGIN_USER: 10\nGUEST: 0"]
        UGR["user_global_roles\nuser_id / global_role_id\n─────────────\n※ グローバルロール管理\n※ 1ユーザー1レコード"]
        T["teams\nid / name / description\nis_public / created_at"]
        TR["team_roles\nid / name / level\n─────────────\nTEAM_OWNER: 20\nTEAM_MEMBER: 10"]
        UTR["user_team_roles\nuser_id / team_id / team_role_id\n─────────────\n※ チームロール管理\n※ チームごとに独立\n※ 複数チーム所属はここで実現"]
    end

    U --> UGR --> GR
    U --> UTR --> TR
    T --> UTR

    NOTE["💡 複数チーム所属の実現方法\nuser_team_rolesに\n（user_id, team_id）の\n組み合わせで複数レコード持つ\n例：\n(userA, teamX, OWNER)\n(userA, teamY, MEMBER)\n(userA, teamZ, MEMBER)"]

    UTR -.- NOTE

    style TABLES fill:#f8f9fa,stroke:#999
    style NOTE   fill:#d4edda,stroke:#6bcb77
```

---

## 7. 権限設計サマリー

```mermaid
flowchart LR
    subgraph G1["🔐 最小権限"]
        G1A["登録直後はLOGIN_USER\n最小権限からスタート"]
        G1B["チーム参加でMEMBER昇格\nチーム作成でOWNER昇格"]
        G1C["チームロールはチームごとに独立\n他チームへの影響ゼロ"]
    end

    subgraph G2["🚫 誤付与防止"]
        G2A["自己ロール昇格は不可"]
        G2B["SYSTEM_ADMINは手動付与のみ"]
        G2C["チームロール変更は\nそのチームのOWNERのみ\n他チームのOWNERは操作不可"]
    end

    subgraph G3["📋 監査可能性"]
        G3A["WHO / WHAT / WHEN / RESULT\n4軸でログ記録"]
        G3B["チームIDも記録し\nどのチームの操作かを追跡可能"]
        G3C["SYSTEM_ADMINのみ閲覧可\n改ざん不可・追記のみ"]
    end

    subgraph G4["⚙️ 運用可能性"]
        G4A["グローバルロール × チームロールの\n2軸設計で複数チーム対応"]
        G4B["user_team_rolesテーブルで\nチームごとのロールを一元管理"]
        G4C["新チーム追加・新ロール追加は\nDBレコード追加のみ\nコード変更不要"]
    end

    style G1 fill:#ffe0e0,stroke:#ff6b6b
    style G2 fill:#fff3cd,stroke:#ffc107
    style G3 fill:#d4edda,stroke:#6bcb77
    style G4 fill:#e8f4fd,stroke:#4d96ff
```
