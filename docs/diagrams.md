# 構成図（ドラフト）

> 各章へ組み込む前の確認用。方向性が固まったら該当章へ移す。

---

## A. インフラ全体構成

```mermaid
flowchart TB
    subgraph client["利用者"]
        U["ユーザー<br/>（スマートフォン中心）"]
        OP["運営スタッフ"]
    end

    subgraph vercel["Vercel（有料）"]
        WEB["ajipita-web<br/>アプリ本体<br/><small>Next.js 16 / App Router</small>"]
    end

    subgraph netlify["Netlify（無料枠・商用可）"]
        SERVICE["ajipita-service<br/>サービスサイト（LP）"]
        ADMIN["ajipita-admin<br/>運営コンソール"]
    end

    subgraph supabase["Supabase"]
        DB[("PostgreSQL<br/><small>RLS で行レベル制御</small>")]
        AUTH["Auth<br/><small>マジックリンク / Google</small>"]
        STORAGE["Storage<br/><small>料理写真</small>"]
        CRON["pg_cron + Edge Functions<br/><small>バッチ・健全性チェック</small>"]
    end

    subgraph external["外部 API"]
        HP["HotPepper<br/><small>無料</small>"]
        PLACES["Google Places<br/><small>従量課金</small>"]
        MAPS["Google Maps<br/><small>地図表示</small>"]
    end

    SENTRY["Sentry<br/><small>エラー + Web Vitals</small>"]

    U --> SERVICE
    U --> WEB
    OP --> ADMIN

    SERVICE -.->|"アプリへ誘導"| WEB

    WEB -->|"BFF 経由・RLS で保護"| DB
    WEB --> AUTH
    WEB --> STORAGE
    WEB -->|"主経路"| HP
    WEB -->|"0 件のときだけ"| PLACES
    WEB --> MAPS

    ADMIN -->|"管理者権限・RLS バイパス"| DB

    CRON --> DB
    CRON -->|"監視経路は<br/>Vercel を通さない"| SENTRY
    WEB --> SENTRY

    classDef paid fill:#fde8e8,stroke:#e04747,color:#7a1a1a
    classDef free fill:#e8f4ea,stroke:#3d9c52,color:#14471f
    classDef data fill:#e8eefc,stroke:#3b6fd4,color:#12305e
    class PLACES paid
    class HP,SERVICE,ADMIN free
    class DB,AUTH,STORAGE,CRON data
```

---

## B. 店舗検索のコスト防壁

6 章の 8 層を、リクエストの流れとして表したもの。

```mermaid
flowchart TB
    START(["店舗検索リクエスト"])

    G1{"① 登録店のみ検索？"}
    G2{"② HotPepper で<br/>結果が得られた？"}
    G3{"③ キャッシュに<br/>ヒットした？"}
    G4{"④ 上限内？<br/><small>IP ハッシュ + 全体</small>"}

    DB_ONLY["自前 DB だけを引く<br/><small>外部 API を呼ばない</small>"]
    HP_RES["HotPepper の結果を返す"]
    CACHE_RES["キャッシュから返す"]
    DEGRADE["Places を呼ばず<br/>HotPepper の結果を返す<br/><small>黙って縮退</small>"]
    CALL["Google Places を呼ぶ<br/><strong>ここで初めて課金</strong>"]
    METER["⑦ サーバー側で回数を記録"]

    START --> G1
    G1 -->|"はい"| DB_ONLY
    G1 -->|"いいえ"| G2
    G2 -->|"1 件以上"| HP_RES
    G2 -->|"0 件"| G3
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

```mermaid
flowchart LR
    subgraph local["ローカル"]
        DOCKER["Docker Supabase<br/><small>日常開発の主軸</small>"]
    end

    subgraph github["GitHub"]
        FEATURE["feature/*"]
        DEVELOP["develop"]
        MAIN["main"]
        CI["GitHub Actions<br/><small>lint / E2E / pgTAP</small>"]
    end

    subgraph cloud["クラウド"]
        DEV_SB["Supabase dev<br/><small>プレビュー用</small>"]
        PROD_SB["Supabase production"]
        PREVIEW["プレビューデプロイ"]
        PROD["本番"]
    end

    DOCKER -.->|"ローカルで検証"| FEATURE
    FEATURE --> DEVELOP
    DEVELOP --> MAIN

    DEVELOP -.->|"CI 実行"| CI
    MAIN -.->|"CI 実行"| CI

    DEVELOP --> PREVIEW
    PREVIEW --> DEV_SB
    MAIN --> PROD
    PROD --> PROD_SB

    classDef env fill:#e8eefc,stroke:#3b6fd4,color:#12305e
    classDef prod fill:#fde8e8,stroke:#e04747,color:#7a1a1a
    class DOCKER,DEV_SB,PREVIEW env
    class MAIN,PROD,PROD_SB prod
```
