# Nekote AI 統合戦略

## 1. 基本哲学：「AI サポート」から「AI ベース」への移行

本文書は、Nekote の中核である `Nekote.Core` に、生成 AI の機能を直接かつ深く統合するための戦略を定義するものです。

ここでの根本的な思想的転換は、従来の「AI サポート」モデル（AI 機能をコアライブラリの**上に**構築する）から、「AI ベース」モデル（AI の基本機能をコアライブラリの**基盤として**内部に組み込む）へと移行することです。

この移行を実現するため、`Nekote.Core` は**「関心の分離 (SoC)」**および**「ドメイン・ファースト」**の原則を徹底します。`Nekote.Core` 自体が `Domain`（純粋なビジネスロジック・モデル）と `Infrastructure`（外部システムへのアクセス）の層に分離され、AI 機能は `Infrastructure` レイヤーの責務として実装されます。

---

## 2. Nekote.Core の責務範囲（スコープ）

Nekote の中核原則である「軽量性」と「依存関係の最小化」を最大限に維持するため、AI 機能の取り扱いに関して責務範囲（スコープ）を明確に定義します。

### A. コア機能（Nekote.Core に自前実装）

`Nekote.Core` が直接サポートする機能は、**「テキストベース」**かつ**「汎用的」**で**「実装が比較的単純」**なものに限定します。

* **対象機能:**
    1.  **チャット (Chat):** これが中核となります。
    2.  **テキストエンベディング (Embedding):** RAG の基盤として必須です。
* **実装方針:**
    * `Nekote.Core` の `Infrastructure` レイヤー内部に、`IHttpClientFactory` から取得した `HttpClient` を用い、REST API の直接呼び出しコードを**自前で実装**します。
    * これらの中核機能のために、**いかなるサードパーティ製の SDK アセンブリにも依存しません**。

### B. 責務範囲外（Nekote.Core が扱わない領域）

以下の機能は、`Nekote.Core` の責務範囲外とします。

1.  **バイナリベース API (画像, 音声等)**
    * **理由:** すべてのアプリケーションが必要とするわけではなく、SDK 依存をコアに持ち込むべきでないため。これらが必要な場合は、アプリケーション層で直接 SDK を利用してください。
2.  **ツール呼び出し (Tool Calling / Function Calling) の実行**
    * **理由:** API からの「ツールを使え」というリクエスト（JSON）を解釈し、対応するクライアント側の C# メソッドを実行し、その結果を送り返す「オーケストレーション」ロジックは、非常に複雑かつアプリケーション固有のコードを大量に要求します。`Nekote.Core` はこのロジックを実装しません。

---

## 3. アーキテクチャ：リポジトリパターンと DI

`Nekote.Core` のアーキテクチャは、「リポジトリパターン」および「DI（依存性注入）」を前提とします。

### A. ドメイン・ファーストとリポジトリパターン

`Nekote.Core` は、AI プロバイダを「データソース」の一種として扱います。

1.  **Domain レイヤー (Nekote.Core の契約):**
    * `Nekote.Core` は、純粋なドメインインターフェース（契約）を定義します。これらは `Domain` レイヤーに属します。
    * 例: `IChatCompletionService`, `ITextEmbeddingService`, `ITranslationService`
2.  **Infrastructure レイヤー (Nekote.Core の実装):**
    * 各 AI プロバイダへの実際の API 呼び出しは、`Infrastructure` レイヤーに属する具象クラス（リポジトリ）として実装されます。
    * 例: `GeminiChatRepository` (implements `IChatCompletionService`), `DeepLTranslationRepository` (implements `ITranslationService`)
    * この設計により、利用者は `ITranslationService` というインターフェースにのみ依存し、背後にある実装が `Gemini` なのか `DeepL` なのかを意識する必要がありません。

### B. 厳格なドメイン vs DTO (アンチ・コラプション・レイヤー)

ドメインモデルを外部の仕様変更から厳格に保護するため、「アンチ・コラプション・レイヤー」の設計を採用します。

1.  **Nekote ドメインモデル:**
    * `Nekote.Core` が公開するモデル（例: `ChatMessage`, `EmbeddingResult`）は、**純粋な POCO** です。これらには `[JsonPropertyName]` などのシリアライズ属性を**一切含みません**。
2.  **内部 DTO (Data Transfer Object):**
    * 各リポジトリ（例: `GeminiChatRepository`）は、その**内部でのみ**使用する**プライベートな DTO** を持ちます（例: `GeminiApiRequestDto`）。
    * これらは API の JSON 仕様と完全に一致し、`[JsonPropertyName]` などのシリアライズ属性を持ちます。
3.  **マッパー (Mapper):**
    * 各リポジトリの**唯一の責務**は、「Nekote ドメインモデル」と「内部 DTO」を相互に変換する「マッパー（アンチ・コラプション・レイヤー）」として機能することです。

### C. DI, 設定, ロギング

`Nekote.Core` は `Microsoft.Extensions.DependencyInjection` を標準の DI コンテナとして前提とします。

* **DI:** `NKote.Core` は `services.AddNekoteCore()` のような拡張メソッドを提供し、`IChatCompletionService` 等のインターフェースと実装（リポジトリ）を `Scoped` ライフタイム（標準）で DI コンテナに自動登録します。
* **設定:** API キーやエンドポイント URL は、`IConfiguration` インターフェース経由でのみ読み込まれます。ハードコードされた文字列は存在しません。
* **ロギング:** すべての内部ログ（API 呼び出しのトレース等）は `ILogger<T>` インターフェース経由で出力されます。`Console.WriteLine()` は使用しません。

---

## 4. サポート対象 AI とその特性

`Nekote.Core` は、現在以下の 6 つの主要 AI プロバイダをサポート対象として計画しています。（2025年11月現在）

* **OpenAI (GPT シリーズ, 例: GPT-5):**
    業界のデファクトスタンダード。コーディング、複雑な論理推論、創造性など、あらゆるタスクを高水準でこなす「万能型」の基準点です。
* **Google (Gemini シリーズ, 例: 2.5 Pro):**
    OpenAI の最大の対抗馬。**100 万トークン**（最大 200 万）を超える超巨大なコンテキストウィンドウが最大の特徴です。 リポジトリ全体や数百ページに及ぶドキュメントの分析など、「ビッグドキュメント」タスクにおいて最強の選択肢です。
* **Anthropic (Claude シリーズ, 例: Sonnet 4.5):**
    元来の「文章の質」や「安全性」 に加え、最新の Sonnet 4.5 は**コーディング**および**エージェントタスク**においてトップクラスの性能を発揮し、GPT-5 を凌駕するベンチマークも報告されています。
* **xAI (Grok 3):**
    **X (旧 Twitter) へのリアルタイムアクセス**という独自の能力を持ちます。 他の AI よりも制限が少なく、ウィットに富んだ「反骨精神」のある回答が特徴で、最新のトレンド分析や異なる視点を得るために有用です。
* **Mistral (Mistral Large 2 等):**
    ヨーロッパ（パリ拠点）製。**「プライバシーファースト」**な設計（オンプレミスデプロイ対応など） を特徴とし、GDPR コンプライアンスの観点で重要です。また、アルゴリズムの改善により、軽量なモデルでも高い性能を発揮するコストパフォーマンスに優れます。
* **DeepSeek (R1, V3 等):**
    **コーディングと数学の「特化型」**として、特定のベンチマーク（SWE-bench 等）で他を圧倒する性能を持ちます。 R1 は特に数学的・論理的な推論に特化しており、技術的な問題解決において高いコストパフォーマンスを誇ります。

---

## 5. 中核機能

### A. ネイティブ RAG サポート

`Nekote.Core` に AI 機能を基盤として組み込む最大の目的は、**RAG (Retrieval-Augmented Generation) のネイティブサポート**です。

1.  **スマート・チャンキング:** `IChatCompletionService` を利用し、長文テキストを「意味的な概念」に基づいてインテリSジェントに分割・要約させます。
2.  **エンベディング:** 分割された各チャンクを `ITextEmbeddingService` でベクトル化します。
3.  **検索:** `Nekote.Core` が持つコサイン類似度計算機能と組み合わせ、高精度なセマンティック検索を実現します。

### B. キャッシュ戦略

キャッシュは `Infrastructure` の関心事として扱います。特にエンベディング（`ITextEmbeddingService`）のような決定論的な（入力が同じなら出力も同じ）操作は、キャッシュの恩恵を強く受けます。

* **実装:** 「デコレーターパターン」を採用します。
* **例:**
    1.  `OpenAiEmbeddingRepository` (実際の実装)
    2.  `CachedEmbeddingRepository` (デコレーター): `IMemoryCache` を持ち、`OpenAiEmbeddingRepository` をラップします。DI 登録時にこのデコレーターを注入することで、キャッシュを透過的に有効化します。

### C. Chat API：Responses API への対応

OpenAI の次世代標準である**「Responses API」**をサポート対象とします。これは従来の Chat Completions API と、2026 年に廃止が予定されている Assistants API を統合・簡素化するものです。

* **ステートレス:** 翻訳や要約など、単発のタスク（最初の API 呼び出し）。
* **ステートフル:** 連続した会話。最初の呼び出しで得た `response_id` を、次のリクエストに `previous_response_id` として含める ことで、クライアント側で会話履歴全体を管理・送信する必要がなくなり、トークンとコストを削減します。

---

## 6. 品質と堅牢性（内部規約）

`Nekote.Core` の開発においては、以下の規約を厳格に適用します。

* **言語規約:**
    * `/// <summary>` などの XML ドキュメントコメントは**日本語**で記述します。
    * `throw new Exception(...)` のメッセージや `ILogger<T>` で出力するログは、すべて**英語**で記述します。
* **非同期処理:**
    * すべての `await` 呼び出しは `.ConfigureAwait(false)` を使用します。
    * すべての非同期メソッドは `CancellationToken` を受け入れます。
* **診断とトレーサビリティ:**
    * デバッグを容易にするため、`IDiagnosticDataCollector` (仮称) インターフェースを定義します。
    * これは「キーの重複を許容する」「LINQ でクエリ可能な」「スレッドセーフな」時系列データコレクターです。
    * API メソッドは `CancellationToken` と同様にこのコレクターをオプション引数として受け取り、例外発生時などに、呼び出し側が「送信した生 JSON」などを `catch` 節で取得できるようにします。

---
*[nao7sep](https://github.com/nao7sep) | Nekote AI Integration Strategy | 2025-11-08*
