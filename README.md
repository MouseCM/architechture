# Deviantly — Mermaid architecture diagrams

These diagrams are derived from `deviantly-architecture-map.html`. They preserve the HTML's distinction between **code-derived paths**, **database-verified objects**, and **inferred or unverified handoffs**. A dashed edge is not proof of a production connection.

## 1. System overview

```mermaid
flowchart LR
  U[Product users / external clients] --> WEB[React application UI]
  U --> LAND[Landing site<br/>SvelteKit static + Express]
  WEB -->|HTTP API calls| API[Go Chi API<br/>port 8081]

  API --> VIDEO[Video query handler]
  VIDEO -->|HTTP SQL| CH[(ClickHouse)]
  API --> LIVE[Live Feed client + cache]
  LIVE -->|ClickHouse HTTP| CH
  API --> TREND[Trend handler]
  TREND --> RAM[In-memory MasterClusters<br/>and generated telemetry]
  TREND -->|Leaderboard / hashtags| ECHO[(Separate EchoTik PostgreSQL)]
  API --> COM[Product / shop / brand / creator repositories]
  COM --> PG[(Primary PostgreSQL<br/>deviantly.*)]
  API --> LS[Live Selling handler]
  LS --> PG
  API --> INS[Insight / hooks / Top Ads handlers]
  INS --> PG
  API --> AUTH[Auth / admin handlers]
  AUTH --> PG
  AUTH --> GOOGLE[Google identity provider]
  API --> MEDIA[Media handler]
  MEDIA --> AI[External AI / media providers]
  MEDIA --> FILES[(Local task state / image files)]
  API --> MCP[External MCP endpoint]
  API --> REPORT[External product-report service]

  LAND -->|POST /demo, /waitlist| LPG[(Landing PostgreSQL<br/>bookings + waitlist)]
  LAND -.->|Optional webhook| HOOK[Webhook target]

  classDef app fill:#e8f1ff,stroke:#3b6ea8,color:#102a43;
  classDef store fill:#f1edff,stroke:#7b61a8,color:#2b1b4b;
  classDef ext fill:#fff5e5,stroke:#b7791f,color:#5c3a00;
  class WEB,LAND,API,VIDEO,LIVE,TREND,COM,LS,INS,AUTH,MEDIA app;
  class CH,ECHO,PG,LPG,RAM,FILES store;
  class GOOGLE,AI,MCP,REPORT,HOOK ext;
```

## 2. Read paths: discovery and trends

```mermaid
flowchart LR
  VE[Video Explorer UI] -->|GET videos, rewrite, detail by :id, thumbnail| API[Go Chi API]
  API --> VH[Video handler / service<br/>filters, search, sort, pagination]
  VH -->|Platform-specific SQL| CH[(ClickHouse)]
  CH --- VT[Tables named in the HTML:<br/>videos_500plus / videos_flat<br/>youtube_shorts_1000plus<br/>instagram_reels + metrics]
  VH -.-> FALLBACK[Empty/error fallback can show<br/>sample cards and fixed totals]

  TL[Trend leaderboard UI] -->|/trend/leaderboard, /dates, /stats| TH[Trend handler]
  TAG[Trending hashtags UI] -->|/trend/hashtags| TH
  TH --> ECHO[(Separate EchoTik PostgreSQL<br/>leaderboard_videos JSON snapshots<br/>and hashtag query path)]

  TM[Trend matrix UI] -->|/trend and poll /trend/live every 3.2 sec| TH2[Trend handler path]
  TH2 --> RAM[In-memory MasterClusters<br/>and wall-clock-derived telemetry]
  TMT[(ClickHouse trend_metrics<br/>table exists)] -.->|No serving read path found| TH2

  LF[Live Feed UI] -->|/live-feed and /stats| LFS[Livefeed client + in-process cache]
  LFS -->|ClickHouse HTTP| CH
  classDef ui fill:#e8f1ff,stroke:#3b6ea8,color:#102a43;
  classDef store fill:#f1edff,stroke:#7b61a8,color:#2b1b4b;
  classDef warn fill:#fff0f0,stroke:#be5555,color:#6f1d1b;
  class VE,TL,TAG,TM,LF ui;
  class CH,ECHO,RAM,TMT store;
  class FALLBACK warn;
```

## 3. Product research, commerce, creator and intelligence pages

```mermaid
flowchart LR
  subgraph COMMERCE[Product / shop / brand pages]
    PRODUCTS[Trending Products + Product Detail + Market Explorer]
    SHOPS[Trending Shops + Shop Detail]
    BRANDS[Trending Brands + Brand Detail]
    PRODUCTS --> PREPO[Product repository]
    SHOPS --> SREPO[Shop repository]
    BRANDS --> BREPO[Brand repository]
  end
  PREPO --> PG[(Primary PostgreSQL<br/>deviantly.* schemas)]
  SREPO --> PG
  BREPO --> PG

  subgraph CREATOR[Creator and content intelligence]
    CREATORS[Creators + Creator Detail]
    HOOKS[Viral Hooks + Hook Detail]
    INSIGHT[Viral Content + Insight Spotlight<br/>Audience Interests + Demographics]
    ADS[Top Ads]
    CREATORS --> CREPO[Creator repository]
    HOOKS --> HKREPO[Hookest repository]
    INSIGHT --> INREPO[Insight repository]
    ADS --> ADREPO[Top Ads repository]
  end
  CREPO --> PG
  HKREPO --> PG
  INREPO --> PG
  ADREPO --> PG

  VI[Viral Ideas UI] -->|normalize / search| VISVC[Viral Ideas search service]
  VISVC -->|Video search| CH[(ClickHouse / Live Feed data)]
  VISVC -->|Product lookups| PG
  VISVC -.-> MCP[External MCP endpoint<br/>used in some paths]

  PR[Product Research UI] -->|reports / jobs / job status| PRS[Product Research service]
  PRS --> RFILES[(Local JSON reports / in-memory maps)]
  PRS -->|Job create / status proxy| EXTREPORT[External report service<br/>PRODUCT_RESEARCH_URL]
  PR -.->|UI requests /suppliers; checked-in handler does not mount it| GAP[Endpoint mismatch]

  MS[Media Studio UI] --> MSAPI[Media API handler<br/>inspect / generate / video task / image]
  MSAPI --> AIP[External AI/media providers]
  MSAPI --> MFILES[(Local uploads, task state,<br/>generated image files)]

  classDef ui fill:#e8f1ff,stroke:#3b6ea8,color:#102a43;
  classDef service fill:#eaf7ed,stroke:#3a8754,color:#153b22;
  classDef store fill:#f1edff,stroke:#7b61a8,color:#2b1b4b;
  classDef warn fill:#fff0f0,stroke:#be5555,color:#6f1d1b;
  class PRODUCTS,SHOPS,BRANDS,CREATORS,HOOKS,INSIGHT,ADS,VI,PR,MS ui;
  class PREPO,SREPO,BREPO,CREPO,HKREPO,INREPO,ADREPO,VISVC,PRS,MSAPI service;
  class PG,CH,RFILES,MFILES store;
  class GAP warn;
```

## 4. Crawler and event-ingestion pipelines

```mermaid
flowchart LR
  subgraph YOUTUBE[YouTube Shorts — strongest source-to-staging path in code]
    DISC[Go discovery crawler] --> FILES[Shard text / JSONL files<br/>channel IDs]
    FILES --> EXTRACT[Rust extractor<br/>scan / dedupe / enrich]
    EXTRACT --> YTPG[(PostgreSQL<br/>youtube_shorts + creators<br/>publish markers)]
    YTPG -->|Publish, acknowledge, then mark published| YTK{{Kafka topics<br/>youtube.shorts / youtube.creators}}
    YTK -.->|Downstream consumer mapping unverified| YTCH[(ClickHouse<br/>youtube_shorts_1000plus)]
  end

  subgraph GENERIC[Kafka-shaped ClickHouse ingestion — DDL wiring unverified]
    PUB[Kafka topics / publishers<br/>video, creator, Instagram events]
    PUB -.->|Topic-to-table mapping unverified| RAW[(ClickHouse kafka_* raw tables)]
    RAW -.->|MV names suggest parsing / normalization| MV[(ClickHouse mv_kafka_* resources)]
    MV -.->|Target relationship needs DDL| TARGET[(Serving resources<br/>videos / shorts / metrics)]
    ERR[(kafka_errors<br/>at, topic, error, raw payload)]
    ERR -.->|Per-topic error MV path| ERRMV[(mv_errors_kafka_*)]
  end

  subgraph TIKTOK[TikTok Shop]
    TTCRAWL[Crawler + enrichers<br/>seller / product / video / creator] --> TTPG[(PostgreSQL<br/>tiktok_shop.* + kafka_outbox)]
    TTPG -.->|Publisher script exists;<br/>runtime caller not located| TTPUB[Outbox publisher script]
    TTPUB -.->|Production topic/consumer not verified| PUB
  end

  subgraph INSTAGRAM[Instagram]
    IGCRAWL[Instagram harvester / UA pipeline] -->|PostgreSQL upsert| IGP[(PostgreSQL<br/>instagram_reels + creators)]
    IGP -.->|PG-to-ClickHouse bridge not located| IGCH[(ClickHouse<br/>instagram_reels + metrics)]
  end

  subgraph AMAZON[Amazon]
    AMAZONSCAN[Batch scanner] --> AMZPG[(PostgreSQL amazon_product)]
    AMZPG -.->|Schema name / join mapping unverified| AMZAPI[API query target<br/>deviantly.amazon_products]
  end

  classDef code fill:#eaf7ed,stroke:#3a8754,color:#153b22;
  classDef store fill:#f1edff,stroke:#7b61a8,color:#2b1b4b;
  classDef warn fill:#fff0f0,stroke:#be5555,color:#6f1d1b;
  class DISC,FILES,EXTRACT,TTCRAWL,TTPUB,IGCRAWL,AMAZONSCAN code;
  class YTPG,YTK,RAW,MV,TARGET,ERR,ERRMV,YTCH,TTPG,IGP,IGCH,AMZPG,AMZAPI store;
```

## 5. Account, landing and fallback paths

```mermaid
flowchart LR
  LOGIN[Authentication UI] -->|login / register / Google / logout / me| AUTH[Auth handler]
  ADMIN[Admin UI] -->|users / approve / block / role / delete| AUTH
  AUTH --> PG[(Primary PostgreSQL<br/>user / account metadata)]
  AUTH --> GOOGLE[Google identity provider]
  ADMIN -.->|Passkey currently in query parameter| RISK[Credential may enter URL logs/history]

  LIVESELL[Live Selling UI] -->|list / detail / timeline / products| LSH[Live Selling handler]
  LSH --> LSPG[(PostgreSQL deviantly.live_streams)]
  LSH -.->|DB error or empty unfiltered query| FIX[Fixture-data fallback]

  LAND[Landing site form] -->|POST /demo /waitlist| EXPRESS[Express form endpoints<br/>validation + rate-limit + deduplication]
  EXPRESS --> LPG[(Landing PostgreSQL<br/>bookings / waitlist)]
  EXPRESS -.->|Optional fire-and-forget| WH[Webhook]

  classDef ui fill:#e8f1ff,stroke:#3b6ea8,color:#102a43;
  classDef service fill:#eaf7ed,stroke:#3a8754,color:#153b22;
  classDef store fill:#f1edff,stroke:#7b61a8,color:#2b1b4b;
  classDef warn fill:#fff0f0,stroke:#be5555,color:#6f1d1b;
  class LOGIN,ADMIN,LIVESELL,LAND ui;
  class AUTH,LSH,EXPRESS service;
  class PG,GOOGLE,LSPG,FIX,LPG,WH store;
  class RISK warn;
```

## 6. Issues / verification checklist represented in the HTML

```mermaid
flowchart TD
  ROOT[Architecture review findings]

  ROOT --> TREND[Trend matrix does not read trend_metrics]
  TREND --> TRENDIMPACT[Matrix uses in-memory cluster definitions<br/>and wall-clock-derived live telemetry]

  ROOT --> FALLBACK[Fallbacks can obscure data-source failures]
  FALLBACK --> VF[Video pages may show sample cards / fixed totals]
  FALLBACK --> LSF[Live Selling may return fixture data<br/>when DB reads fail or return no rows]

  ROOT --> PIPE[Ingestion lineage gaps]
  PIPE --> KAFKA[Kafka topic → raw table → MV → serving table<br/>relationships need DDL confirmation]
  PIPE --> YT[YouTube publish is at-least-once;<br/>downstream consumer mapping unverified]
  PIPE --> TT[TikTok outbox publisher exists,<br/>but runtime caller/scheduler not located]
  PIPE --> IG[Instagram PostgreSQL → ClickHouse<br/>synchronization bridge unknown]
  PIPE --> AMZ[amazon_product vs deviantly.amazon_products<br/>schema mapping is unverified]

  ROOT --> API[API / integration mismatches]
  API --> SUP[Product Research UI requests /suppliers;<br/>checked-in handler does not mount the route]
  API --> REPORT[Product research uses local JSON / in-memory state;<br/>persistence and deployed proxy destination need confirmation]

  ROOT --> SECURITY[Security concern]
  SECURITY --> PASSKEY[Admin passkey sent in query parameter;<br/>HTML recommends protected header and credential rotation]

  ROOT --> EVIDENCE[Evidence boundary]
  EVIDENCE --> CATALOG[Valv PostgreSQL catalog is partial;<br/>absence from catalog does not prove absence on server]
  EVIDENCE --> DEPLOY[Deployment, runtime health, physical DB identity<br/>and exact ClickHouse lineage are not fully confirmed]

  classDef root fill:#e8f1ff,stroke:#3b6ea8,color:#102a43;
  classDef issue fill:#fff0f0,stroke:#be5555,color:#6f1d1b;
  classDef detail fill:#fff5e5,stroke:#b7791f,color:#5c3a00;
  class ROOT root;
  class TREND,FALLBACK,PIPE,API,SECURITY,EVIDENCE issue;
  class TRENDIMPACT,VF,LSF,KAFKA,YT,TT,IG,AMZ,SUP,REPORT,PASSKEY,CATALOG,DEPLOY detail;
```

## Evidence / notation

- **Solid arrow**: the HTML describes a code-derived request path or an explicitly verified database object.
- **Dashed arrow**: the HTML flags the handoff as inferred, incomplete, conditional, or requiring runtime / DDL verification.
- The diagrams show logical relationships from the document, not confirmed deployment topology, host boundaries, or a guarantee that each source has fresh production data.
