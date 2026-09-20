# 構成図の一覧

各章に載せている図をまとめたもの。**編集用の `.drawio` はこのディレクトリの `diagrams/` にあります**
（[app.diagrams.net](https://app.diagrams.net) の File → Open From → Device で開けます）。

| 図 | 掲載章 |
| --- | --- |
| A. インフラ全体構成 | [1. 全体構成](./01-architecture.md) |
| B. 店舗検索のコスト防壁 | [6. コスト](./06-cost.md) |
| C. データアクセス境界（BFF） | [4. バックエンド・データ](./04-backend.md) |
| D. 環境の 3 層とデプロイ | [5. インフラ・CI/CD](./05-infrastructure.md) |

Mermaid 版も併記しています（GitHub 上でテキストとして差分が追えるため）。

---

## A. インフラ全体構成

### draw.io 版（決定版）

![インフラ全体構成](./diagrams/infrastructure.png)

編集用ファイル: [`diagrams/infrastructure.drawio`](./diagrams/infrastructure.drawio)
（[app.diagrams.net](https://app.diagrams.net) で開ける。PNG は同ディレクトリに書き出し済み）

### Mermaid 版（GitHub 上で即読める簡易版）

```mermaid
flowchart TB
    U["ユーザー"]
    OP["運営スタッフ"]

    WEB["ajipita-web<br/><small>アプリ本体<br/>Vercel・有料</small>"]

    subgraph netlify["Netlify（無料枠・商用利用可）"]
        SERVICE["ajipita-service<br/><small>サービスサイト（LP）</small>"]
        ADMIN["ajipita-admin<br/><small>運営コンソール</small>"]
    end

    DB[("Supabase PostgreSQL<br/><small>RLS で行レベル制御</small>")]
    SBX["Supabase Auth / Storage"]
    CRON["pg_cron + Edge Functions<br/><small>バッチ・健全性チェック</small>"]

    EXT["外部 API<br/><small>HotPepper（主経路・無料）<br/>Google Places（0 件のときだけ・$32/1,000）<br/>Google Geocoding（エリア名解決・$5/1,000）<br/>Google Maps（地図表示）</small>"]
    SENTRY["Sentry<br/><small>エラー + Web Vitals</small>"]

    U --> SERVICE
    U --> WEB
    OP --> ADMIN
    SERVICE -.->|"誘導"| WEB

    WEB -->|"BFF 経由・RLS で保護"| DB
    ADMIN -->|"管理者権限・RLS バイパス"| DB
    WEB --> SBX
    WEB --> EXT
    ADMIN -.->|"運営操作時のみ"| EXT
    WEB --> SENTRY

    DB --- CRON
    CRON -->|"Vercel を経由しない"| SENTRY

    classDef paid fill:#fde8e8,stroke:#e04747,color:#7a1a1a
    classDef free fill:#e8f4ea,stroke:#3d9c52,color:#14471f
    classDef data fill:#e8eefc,stroke:#3b6fd4,color:#12305e
    class WEB paid
    class SERVICE,ADMIN free
    class DB,SBX,CRON data
```

> 課金が発生するのは赤のノードだけです。外部 API の内訳と防壁は B を参照。

---

## B. 店舗検索のコスト防壁

6 章の 8 層を、リクエストの流れとして表したもの。

### draw.io 版（決定版）

![店舗検索のコスト防壁](./diagrams/cost-barriers.png)

編集用: [`diagrams/cost-barriers.drawio`](./diagrams/cost-barriers.drawio)

### Mermaid 版（簡易版）

```mermaid
flowchart TB
    START(["店舗検索リクエスト"])

    G1{"① 登録店のみ検索？"}
    G2{"② HotPepper が 0 件？<br/>かつ 1 ページ目？"}
    G3{"③ キャッシュに<br/>ヒットした？"}
    G4{"④ 上限内？<br/><small>IP ハッシュ + 全体 / 短窓と日次</small>"}

    DB_ONLY["自前 DB だけを引く<br/><small>外部 API を呼ばない</small>"]
    HP_RES["HotPepper の結果を返す"]
    CACHE_RES["キャッシュから返す"]
    DEGRADE["Places を呼ばず<br/>HotPepper の結果を返す<br/><small>黙って縮退</small>"]
    CALL["Google Places を呼ぶ<br/><strong>ここで初めて課金</strong>"]
    METER["⑦ サーバー側で回数を記録"]

    START --> G1
    G1 -->|"はい"| DB_ONLY
    G1 -->|"いいえ"| G2
    G2 -->|"1 件以上 /<br/>2 ページ目以降"| HP_RES
    G2 -->|"0 件 かつ 1 ページ目"| G3
    G3 -->|"ヒット"| CACHE_RES
    G3 -->|"ミス"| G4
    G4 -->|"超過 / 確認不能<br/><small>fail closed</small>"| DEGRADE
    G4 -->|"枠あり"| CALL
    CALL --> METER

    classDef free fill:#e8f4ea,stroke:#3d9c52,color:#14471f
    classDef paid fill:#fde8e8,stroke:#e04747,color:#7a1a1a
    classDef guard fill:#fff4e0,stroke:#d98c1f,color:#6b4108
    class DB_ONLY,HP_RES,CACHE_RES,DEGRADE free
    class CALL paid
    class G1,G2,G3,G4 guard
```

> ⑤（台帳の権限を配らない）と ⑥（フィールド最小化）は流れの中ではなく
> 上限判定と API 呼び出しの**内側**に効くため、この図には現れません。
> ⑧（提示済み ID の allowlist）は店舗詳細の別経路です。

---

## C. データアクセス境界（BFF）

4 章の構成。ロジックの置き場所と到達方法を分けている。

### draw.io 版（決定版）

![データアクセス境界](./diagrams/data-access.png)

編集用: [`diagrams/data-access.drawio`](./diagrams/data-access.drawio)

### Mermaid 版（簡易版）

```mermaid
flowchart TB
    subgraph server["サーバー"]
        MODULE["データモジュール<br/><small>server-only・ロジックの単一実体</small>"]
        RSC["Server Component"]
        API["API ルートハンドラ<br/><small>認証・検証・整形のみ</small>"]
    end

    subgraph clients["クライアント"]
        CC["Client Component"]
        NATIVE["ネイティブアプリ<br/><small>（将来）</small>"]
    end

    DB[("PostgreSQL<br/>RLS")]

    RSC -->|"直接呼ぶ<br/><small>自己 HTTP しない</small>"| MODULE
    API -->|"呼ぶ"| MODULE
    CC -->|"fetch"| API
    NATIVE -.->|"同じ API"| API
    MODULE --> DB

    classDef core fill:#e8eefc,stroke:#3b6fd4,color:#12305e
    classDef thin fill:#f3f4f6,stroke:#9ca3af,color:#374151
    class MODULE core
    class API,RSC thin
```

---

## D. 環境の 3 層とデプロイ

### draw.io 版（決定版）

![環境の3層とデプロイ](./diagrams/environments.png)

編集用: [`diagrams/environments.drawio`](./diagrams/environments.drawio)

### Mermaid 版（簡易版）

```mermaid
flowchart TB
    DOCKER["ローカル Docker Supabase<br/><small>日常開発の主軸</small>"]

    FEATURE["feature/*"]
    DEVELOP["develop"]
    MAIN["main"]
    CIFAST["静的チェック<br/><small>lint / 型 / 単体テスト<br/>全 PR で実行</small>"]
    CISLOW["E2E / pgTAP<br/><small>develop・main 向け PR のみ</small>"]

    PREVIEW["プレビューデプロイ<br/><small>PR ごとに生成</small>"]
    DEV_SB[("Supabase dev<br/><small>Free・1 週間で自動 pause</small>")]
    PROD["本番"]
    PROD_SB[("Supabase production<br/><small>Pro</small>")]

    DOCKER -->|"ローカルで検証"| FEATURE
    FEATURE --> DEVELOP
    DEVELOP --> MAIN

    FEATURE -.-> CIFAST
    DEVELOP -.-> CIFAST
    DEVELOP -.-> CISLOW
    MAIN -.-> CISLOW

    DEVELOP --> PREVIEW --> DEV_SB
    MAIN --> PROD --> PROD_SB

    classDef local fill:#e8eefc,stroke:#3b6fd4,color:#12305e
    classDef prod fill:#fde8e8,stroke:#e04747,color:#7a1a1a
    class DOCKER,PREVIEW,DEV_SB local
    class MAIN,PROD,PROD_SB prod
```
