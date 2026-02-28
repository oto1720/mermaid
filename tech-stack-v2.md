# 技術構成設計書 v2

> v2変更点：バックエンドを **Go（Gin）** に変更、インフラを **Google Cloud Run** に変更
> サークル開発（初心者〜中級混合・6〜8名）を前提とした現実解

---

## 1. 技術スタック全体像

```mermaid
flowchart TB
    subgraph CLIENT["🖥️ クライアント層"]
        direction LR
        FE["Next.js 14\nReact 18 + TypeScript\nTailwind CSS + shadcn/ui"]
        STATE["Zustand\n状態管理"]
        SSE["Server-Sent Events\nAIストリーミング表示"]
        DND["dnd-kit\nカンバンDnD"]
        FE --- STATE
        FE --- SSE
        FE --- DND
    end

    subgraph SERVER["⚙️ サーバー層（Go）"]
        direction LR
        API["Gin\nGo 1.22"]
        MW["ミドルウェア\nJWT / RBAC / ABAC\nレートリミット / スコープ"]
        WORKER["Goroutine Worker\nAI非同期処理\nチャネル通信"]
        API --> MW --> WORKER
    end

    subgraph DATA["🗃️ データ層"]
        direction LR
        DB["Cloud SQL\nPostgreSQL 15\nsqlc + migrate"]
        CACHE["Memorystore\nRedis\nレートリミット・キュー"]
        DB --- CACHE
    end

    subgraph AI["🤖 AI層"]
        CLAUDE["Claude API\nGo HTTPクライアント\nSSEストリーミング"]
        KEYSTORE["Secret Manager\nBYOKキー管理\nAES-256暗号化"]
        CLAUDE --- KEYSTORE
    end

    subgraph INFRA["☁️ Google Cloud"]
        direction LR
        CR["Cloud Run\nコンテナ実行\nオートスケール"]
        AR["Artifact Registry\nDockerイメージ管理"]
        LB["Cloud Load Balancing\nHTTPS終端"]
        GHA["GitHub Actions\nCI/CD"]
        CR --- AR --- LB --- GHA
    end

    CLIENT  -->|"HTTPS / REST"| LB
    LB      -->|"HTTP/2"| CR
    CR      --> SERVER
    SERVER  -->|"sqlc"| DATA
    SERVER  -->|"HTTP"| AI
    SERVER & CLIENT --> INFRA

    style CLIENT fill:#dbeafe,stroke:#3b82f6
    style SERVER fill:#d4edda,stroke:#16a34a
    style DATA   fill:#fef9c3,stroke:#eab308
    style AI     fill:#ffe0e0,stroke:#ff6b6b
    style INFRA  fill:#ede7f6,stroke:#9b59b6
```

---

## 2. バックエンド構成（Go + Gin）

```mermaid
flowchart TB
    subgraph GO["⚙️ Go + Gin アーキテクチャ"]
        direction TB

        subgraph MW_CHAIN["🔗 ミドルウェアチェーン"]
            M1["① CORS / セキュリティヘッダー\ncors.New() / helmet的な設定"]
            M2["② JWT検証\ngolang-jwt/jwt\n→ 401 Unauthorized"]
            M3["③ RBACチェック\nロール × チームスコープ\n→ 403 Forbidden"]
            M4["④ ABACチェック\n5ルールを条件評価\n→ 403 or warning"]
            M5["⑤ レートリミット\ngo-redis + Sliding Window\n→ 429 Too Many Requests"]
            M6["⑥ 監査ログ記録\ngoroutineで非同期書き込み"]
            M1 --> M2 --> M3 --> M4 --> M5 --> M6
        end

        subgraph ROUTES["🛣️ ルーター（Gin）"]
            R1["POST /api/auth/register\nPOST /api/auth/login"]
            R2["POST /api/teams\nGET  /api/teams/:id\nPATCH /api/teams/:id"]
            R3["POST /api/requirements\nPATCH /api/requirements/:id"]
            R4["POST /api/ai/generate\nGET  /api/ai/stream/:jobId（SSE）"]
            R5["GET  /api/roadmaps/:teamId\nPATCH /api/roadmaps/:id/confirm"]
            R6["PATCH /api/tasks/:id/status\nPOST /api/tasks/:id/comments"]
            R7["POST /api/teams/:id/api-key\nGET  /api/teams/:id/usage"]
            R8["GET /api/admin/audit-logs"]
        end

        subgraph LAYERS["📦 レイヤー構成"]
            direction LR
            HANDLER["Handler層\nリクエスト/レスポンス変換\nバリデーション（go-playground/validator）"]
            SERVICE["Service層\nビジネスロジック\nABAC評価 / AI呼び出し"]
            REPO["Repository層\nsqlcで生成されたクエリ\nDB操作の抽象化"]
            HANDLER --> SERVICE --> REPO
        end

        subgraph AI_WORKER["🔧 AI Workerプロセス"]
            W1["Goroutine Pool\n並行AI処理"]
            W2["Channel通信\njobChannel / resultChannel"]
            W3["SSEストリーミング\nhttp.Flusher で逐次送信"]
            W1 --> W2 --> W3
        end
    end

    style GO       fill:#d4edda,stroke:#16a34a
    style MW_CHAIN fill:#ecfdf5,stroke:#6ee7b7
    style ROUTES   fill:#ecfdf5,stroke:#6ee7b7
    style LAYERS   fill:#ecfdf5,stroke:#6ee7b7
    style AI_WORKER fill:#ecfdf5,stroke:#6ee7b7
```

---

## 3. Go パッケージ構成

```mermaid
mindmap
  root((📦 Go パッケージ))
    Webフレームワーク
      gin-gonic/gin
        高速HTTPルーター
        ミドルウェア対応
    認証・セキュリティ
      golang-jwt/jwt
        JWT発行・検証
      golang.org/x/crypto
        bcryptパスワードハッシュ
      crypto/aes
        AES-256 APIキー暗号化
        標準ライブラリで対応
    DB・ORM
      sqlc
        SQL→Goコード自動生成
        型安全なクエリ
      lib/pq
        PostgreSQLドライバ
      golang-migrate/migrate
        DBマイグレーション管理
    Redis
      go-redis/redis
        レートリミット
        キャッシュ
    バリデーション
      go-playground/validator
        リクエストバリデーション
    ロギング
      uber-go/zap
        構造化ログ
        高速・JSON出力
    AI連携
      net/http 標準ライブラリ
        Claude APIクライアント
        SSEストリーミング
    設定管理
      spf13/viper
        環境変数管理
    テスト
      testify/assert
        ユニットテスト
      httptest
        APIテスト 標準ライブラリ
```

---

## 4. DBスキーマ設計（sqlc + PostgreSQL）

```mermaid
erDiagram
    users {
        uuid id PK
        string email UK
        string name
        string password_hash
        string skill_level
        timestamp created_at
    }
    global_roles {
        uuid id PK
        string name
        int level
    }
    user_global_roles {
        uuid user_id FK
        uuid global_role_id FK
    }
    teams {
        uuid id PK
        string name
        string description
        string level
        boolean is_public
        timestamp created_at
    }
    team_roles {
        uuid id PK
        string name
        int level
    }
    user_team_roles {
        uuid user_id FK
        uuid team_id FK
        uuid team_role_id FK
    }
    requirements {
        uuid id PK
        uuid team_id FK
        string product_type
        jsonb feature_checklist
        string difficulty
        text free_text
        string status
        timestamp created_at
    }
    roadmaps {
        uuid id PK
        uuid team_id FK
        string status
        timestamp confirmed_at
        timestamp created_at
    }
    phases {
        uuid id PK
        uuid roadmap_id FK
        int phase_number
        string title
    }
    tasks {
        uuid id PK
        uuid phase_id FK
        string title
        text description
        string status
        uuid task_owner_id FK
        int estimated_hours
        timestamp due_date
        timestamp created_at
    }
    task_comments {
        uuid id PK
        uuid task_id FK
        uuid user_id FK
        text content
        timestamp created_at
    }
    team_api_keys {
        uuid id PK
        uuid team_id FK
        bytea encrypted_key
        int monthly_limit
        int used_tokens
        timestamp created_at
    }
    api_usage_logs {
        uuid id PK
        uuid team_id FK
        uuid user_id FK
        string action_type
        int tokens_used
        timestamp created_at
    }
    audit_logs {
        uuid id PK
        uuid user_id FK
        uuid team_id FK
        string action
        string resource_type
        string result
        jsonb attributes_snapshot
        timestamp created_at
    }
    abac_rules {
        uuid id PK
        string name
        string target_action
        jsonb condition_json
        string effect
        int priority
    }

    users ||--o{ user_global_roles : ""
    global_roles ||--o{ user_global_roles : ""
    users ||--o{ user_team_roles : ""
    teams ||--o{ user_team_roles : ""
    team_roles ||--o{ user_team_roles : ""
    teams ||--|| requirements : ""
    teams ||--|| roadmaps : ""
    roadmaps ||--o{ phases : ""
    phases ||--o{ tasks : ""
    tasks ||--o{ task_comments : ""
    tasks }o--|| users : ""
    teams ||--o| team_api_keys : ""
    teams ||--o{ api_usage_logs : ""
    users ||--o{ audit_logs : ""
```

---

## 5. AI生成フロー（Goroutine + SSE）

```mermaid
sequenceDiagram
    actor User as 👤 ユーザー
    participant FE as 🖥️ フロント
    participant GIN as ⚙️ Gin Handler
    participant CH as 📬 Job Channel
    participant WK as 🔧 Goroutine Worker
    participant SM as 🔑 Secret Manager
    participant CL as 🤖 Claude API
    participant DB as 🗃️ Cloud SQL

    User->>FE: AI分析実行
    FE->>GIN: POST /api/ai/generate

    GIN->>GIN: JWT / RBAC / ABACチェック
    GIN->>DB: 要件・メンバースキル取得
    GIN->>CH: jobをchannelに送信
    GIN-->>FE: 202 Accepted { job_id }

    FE->>GIN: GET /api/ai/stream/{job_id}
    GIN-->>FE: SSE接続確立\nContent-Type: text/event-stream

    CH->>WK: goroutineでjobを受信
    WK->>SM: APIキー取得（BYOK or 共有枠）
    SM-->>WK: 復号済みキー

    WK->>CL: HTTPリクエスト（stream=true）
    loop ストリーミング中
        CL-->>WK: チャンク（delta）
        WK-->>FE: SSE data: {chunk}\n\n
        FE->>FE: UIにリアルタイム表示
    end

    WK->>DB: 生成結果を保存
    WK->>DB: トークン使用量を記録
    WK->>WK: キーをメモリから即破棄
    WK-->>FE: SSE event: complete

    note over WK,SM: Secret Managerで\nキーを安全に管理
```

---

## 6. Google Cloud インフラ構成

```mermaid
flowchart TB
    subgraph GCP["☁️ Google Cloud Platform"]
        direction TB

        subgraph NETWORK["🌐 ネットワーク層"]
            LB["Cloud Load Balancing\nHTTPS終端・SSL証明書自動"]
            CDN["Cloud CDN\n静的アセットキャッシュ"]
        end

        subgraph COMPUTE["⚙️ コンピューティング層"]
            CR_API["Cloud Run\n本番APIサービス\nGoコンテナ\ncpu:1 mem:512Mi\nmin:0 max:10インスタンス"]
            CR_WORKER["Cloud Run Jobs\nAI Worker\n非同期処理専用\n必要時のみ起動"]
        end

        subgraph DATA_LAYER["🗃️ データ層"]
            CSQL["Cloud SQL\nPostgreSQL 15\nプライベートIP接続\n自動バックアップ"]
            MEM["Memorystore\nRedis 7\nレートリミット\nセッション管理"]
        end

        subgraph STORAGE["📦 ストレージ・レジストリ"]
            AR["Artifact Registry\nDockerイメージ管理\nVulnerability Scan"]
            GCS["Cloud Storage\n将来：添付ファイル保管"]
        end

        subgraph SECURITY["🔐 セキュリティ層"]
            SM["Secret Manager\nAPIキー・JWT秘密鍵\nDB接続情報"]
            IAM["Cloud IAM\nサービスアカウント管理\n最小権限原則"]
        end

        subgraph OPS["📊 運用・監視層"]
            LOG["Cloud Logging\nGoのzapログ集約"]
            MON["Cloud Monitoring\nCPU/メモリ/レイテンシ"]
            TRACE["Cloud Trace\nリクエストトレーシング"]
            ALERT["Cloud Alerting\n異常検知・通知"]
        end

        LB --> CR_API
        CDN --> LB
        CR_API --> CSQL
        CR_API --> MEM
        CR_API --> SM
        CR_API --> CR_WORKER
        CR_WORKER --> CSQL
        CR_WORKER --> SM
        AR --> CR_API
        IAM --> CR_API & CR_WORKER
        CR_API --> LOG & MON & TRACE
        MON --> ALERT
    end

    subgraph FRONTEND["🖥️ フロントエンド"]
        VERCEL["Vercel\nNext.js ホスティング\n自動HTTPS・CDN"]
    end

    subgraph CICD["🔄 CI/CD"]
        GHA["GitHub Actions"]
        GHA -->|"docker build & push"| AR
        GHA -->|"gcloud run deploy"| CR_API
        GHA -->|"deploy"| VERCEL
    end

    VERCEL -->|"HTTPS"| LB

    style GCP      fill:#e8f5e9,stroke:#4caf50
    style NETWORK  fill:#f1f8e9,stroke:#8bc34a
    style COMPUTE  fill:#f1f8e9,stroke:#8bc34a
    style DATA_LAYER fill:#f1f8e9,stroke:#8bc34a
    style STORAGE  fill:#f1f8e9,stroke:#8bc34a
    style SECURITY fill:#f1f8e9,stroke:#8bc34a
    style OPS      fill:#f1f8e9,stroke:#8bc34a
    style FRONTEND fill:#dbeafe,stroke:#3b82f6
    style CICD     fill:#ede7f6,stroke:#9b59b6
```

---

## 7. Cloud Run デプロイ設計

```mermaid
flowchart LR
    subgraph DOCKERFILE["🐳 Dockerfile（マルチステージビルド）"]
        direction TB
        D1["Stage 1: builder\nFROM golang:1.22-alpine\ngo mod download\ngo build -o /app/server"]
        D2["Stage 2: runner\nFROM gcr.io/distroless/static\nCOPY --from=builder /app/server\nUSER nonroot\nENTRYPOINT [/server]"]
        D1 -->|"バイナリのみコピー\nイメージサイズ最小化"| D2
    end

    subgraph CR_CONFIG["⚙️ Cloud Run 設定"]
        direction TB
        C1["コンテナ設定\ncpu: 1\nmemory: 512Mi\nport: 8080\ntimeout: 300s（AI生成考慮）"]
        C2["スケーリング設定\nmin-instances: 0（コスト削減）\nmax-instances: 10\nconcurrency: 80"]
        C3["環境変数\n※ 実値はSecret Manager参照\nDB_URL → sm://project/db-url\nJWT_SECRET → sm://project/jwt-secret\nSERVICE_API_KEY → sm://project/api-key"]
        C4["ネットワーク設定\nVPC Connector経由でCloud SQLに接続\nプライベートIPで通信\n外部からDBへの直接アクセス不可"]
    end

    subgraph CICD_FLOW["🔄 デプロイフロー"]
        direction TB
        G1["① PRにpush\n→ lint + test + build"]
        G2["② mainマージ\n→ Docker build\n→ Artifact Registryにpush\n→ Cloud Run stgにデプロイ"]
        G3["③ タグpush（v*）\n→ 本番Cloud Runにデプロイ\n→ ゼロダウンタイムで切り替え"]
        G1 --> G2 --> G3
    end

    style DOCKERFILE fill:#d4edda,stroke:#6bcb77
    style CR_CONFIG  fill:#e8f5e9,stroke:#4caf50
    style CICD_FLOW  fill:#ede7f6,stroke:#9b59b6
```

---

## 8. Express vs Go（Gin）比較・選定理由

```mermaid
flowchart TB
    subgraph COMPARE["⚖️ Express.js vs Go（Gin）"]
        direction LR

        subgraph EXPRESS["Node.js + Express"]
            E1["✅ 初心者が多い環境では\n学習コスト低い"]
            E2["✅ npmエコシステムが豊富"]
            E3["❌ シングルスレッド\n並行処理が複雑"]
            E4["❌ 型安全性が弱い\n（TypeScriptでカバーはできる）"]
            E5["❌ メモリ消費大\nCloud Runのコスト増"]
            E6["❌ コールドスタートが遅い"]
        end

        subgraph GOLANG["Go + Gin"]
            G1["✅ Goroutineで並行処理が簡単\nAI非同期処理に最適"]
            G2["✅ コンパイル言語で型安全\n実行時エラーが少ない"]
            G3["✅ バイナリが軽量\nコールドスタート高速（Cloud Run◎）"]
            G4["✅ メモリ効率が高い\nCloud Runコスト削減"]
            G5["✅ 標準ライブラリが充実\nnet/http / crypto/aes"]
            G6["⚠️ 初心者には学習コストあり\nただしsqlc/Ginはシンプル"]
        end
    end

    subgraph REASON["💡 Go採用の決め手"]
        R1["Cloud Runとの相性\nバイナリ1つ・コールドスタート速い\nmin-instances:0でコスト最適化可能"]
        R2["AI非同期処理\nGoroutine + Channelで\nワーカープールが自然に書ける"]
        R3["セキュリティ要件\ncrypto/aesが標準で使える\n型安全でABACルール評価が安全"]
        R4["運用コスト\nメモリ512MiでCloud Run稼動\nNode.jsより大幅にコスト削減"]
    end

    style EXPRESS  fill:#fef9c3,stroke:#eab308
    style GOLANG   fill:#d4edda,stroke:#16a34a
    style REASON   fill:#dbeafe,stroke:#3b82f6
```

---

## 9. Railway vs Google Cloud Run 比較・選定理由

```mermaid
flowchart TB
    subgraph COMPARE2["⚖️ Railway vs Google Cloud Run"]
        direction LR

        subgraph RAILWAY["Railway"]
            RA1["✅ 初期設定が超簡単\nGitHub連携でワンクリック"]
            RA2["✅ 無料枠あり"]
            RA3["❌ スケールの細かい制御が難しい"]
            RA4["❌ エンタープライズ向け機能が弱い"]
            RA5["❌ 日本リージョンがない"]
            RA6["❌ GCPサービスとの連携が弱い"]
        end

        subgraph CLOUDRUN["Google Cloud Run"]
            CR1["✅ Goroutineと相性◎\nHTTP/2対応・並行処理最適"]
            CR2["✅ コールドスタートが速い\nGoバイナリは特に速い"]
            CR3["✅ Cloud SQL / Secret Manager\nとネイティブ連携"]
            CR4["✅ min-instances:0で\nリクエストがないと課金なし"]
            CR5["✅ 東京リージョン（asia-northeast1）\nレイテンシが低い"]
            CR6["⚠️ 初期設定にGCPの知識が必要\nただし学習価値が高い"]
        end
    end

    subgraph COST["💰 コスト比較（月次目安）"]
        direction TB
        CO1["Cloud Run\n無料枠: 200万リクエスト/月\n180,000 vCPU秒/月\nMVP段階ではほぼ無料"]
        CO2["Cloud SQL\n最小構成: db-f1-micro\n約$7〜10/月"]
        CO3["Memorystore\n最小1GB: 約$15/月\n※ Redis Cloudの無料枠で代替可"]
        CO4["Secret Manager\n月1万アクセスまで無料\nほぼ無料"]
    end

    style RAILWAY   fill:#fef9c3,stroke:#eab308
    style CLOUDRUN  fill:#d4edda,stroke:#16a34a
    style COST      fill:#dbeafe,stroke:#3b82f6
```

---

## 10. フェーズ別技術導入ロードマップ

```mermaid
timeline
    title 技術スタック段階的導入（Go + GCP版）
    section Phase 0 MVP（2〜3週）
        Go基盤         : Go 1.22 + Gin セットアップ
                       : sqlc + golang-migrate
                       : JWT認証 + RBAC MW
                       : Docker マルチステージビルド
        GCPセットアップ : Cloud Run（asia-northeast1）
                       : Cloud SQL PostgreSQL
                       : Secret Manager
                       : Artifact Registry
                       : GitHub Actions CI/CD
        AI基盤         : Claude API Go HTTPクライアント
                       : Goroutine Worker + Channel
                       : AES-256 APIキー暗号化
                       : Redisレートリミット
    section Phase 1（3〜4週追加）
        SSEストリーミング : http.Flusher でSSE実装
                         : AIストリーミング表示
        BYOK対応         : Secret Manager統合
                         : キー選択ロジック
                         : 月次上限・停止通知
        認証強化         : Google OAuth（golang.org/x/oauth2）
        通知             : アプリ内通知（ポーリング）
                         : Cloud Tasks（遅延通知）
    section Phase 2（中期）
        可視化           : ガントチャート（Recharts）
                         : 使用量ダッシュボード
        監視強化         : Cloud Trace 統合
                         : Cloud Alerting 設定
                         : Sentry Go SDK
        外部連携         : Slack/Discord Webhook
                         : Cloud Scheduler（週次レポート）
        スケール対応      : Cloud Run min-instances:1（本番）
                         : Cloud SQL HA構成
```

---

## 11. ディレクトリ構成（Go）

```mermaid
mindmap
  root((📁 プロジェクト構成))
    backend/
      cmd/
        server/
          main.go エントリポイント
      internal/
        handler/
          auth.go
          team.go
          ai.go
          task.go
          admin.go
        service/
          auth_service.go
          team_service.go
          ai_service.go
          key_service.go
          usage_service.go
          abac_service.go
        repository/
          sqlc/ 自動生成コード
          queries/ SQLファイル
        middleware/
          jwt.go
          rbac.go
          abac.go
          ratelimit.go
          audit.go
        model/
          user.go
          team.go
          task.go
        worker/
          ai_worker.go Goroutineプール
          job.go ジョブ定義
      pkg/
        crypto/ AES-256暗号化
        logger/ zap設定
        validator/ カスタムバリデーション
      db/
        migrations/ マイグレーションSQL
        schema.sql
      Dockerfile
      go.mod
    frontend/
      app/ Next.js App Router
      components/
      lib/
      Dockerfile
    infra/
      cloudbuild.yaml
      cloudrun.yaml サービス定義
      terraform/ 将来IaC化
    docker-compose.yml ローカル開発
    .github/workflows/ CI/CD
```

---

## 12. 技術選定サマリー

```mermaid
mindmap
  root((🛠️ 技術スタック v2))
    🖥️ フロントエンド
      Next.js 14 + TypeScript
      Tailwind CSS + shadcn/ui
      Zustand 状態管理
      dnd-kit カンバンDnD
      Recharts グラフ
      Axios APIクライアント
    ⚙️ バックエンド Go
      Go 1.22 + Gin
      golang-jwt JWT認証
      golang.org/x/crypto bcrypt
      crypto/aes AES-256標準ライブラリ
      sqlc 型安全クエリ生成
      golang-migrate マイグレーション
      go-redis レートリミット
      uber-go/zap 構造化ログ
      Goroutine + Channel AI非同期
    🗃️ データ
      Cloud SQL PostgreSQL 15
      Memorystore Redis
        or Redis Cloud無料枠
    🤖 AI
      Claude API Go HTTPクライアント
      SSE http.Flusher
      Secret Manager BYOK管理
      AES-256暗号化
    ☁️ Google Cloud
      Cloud Run コンテナ実行
      Artifact Registry イメージ管理
      Cloud SQL マネージドDB
      Secret Manager シークレット管理
      Cloud Logging ログ集約
      Cloud Monitoring 監視
      Cloud Load Balancing HTTPS終端
    🔄 CI/CD
      GitHub Actions
      Docker マルチステージビルド
      Vercel フロントデプロイ
      gcloud run deploy バックデプロイ
```
