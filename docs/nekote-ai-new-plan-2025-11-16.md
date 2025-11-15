# Nekote AI 再構築計画 | 2025-11-16

## 1. 指導原則：「アンチ・コラプション・レイヤー」の厳格な適用

前回の失敗（`nekote-ai-failure-analysis-2025-11-16.md`）は、ドメインモデルが外部の仕様（OpenAI の API 仕様）によって「汚染」されたことに起因する。これは `PLAYBOOK.md` で定義された「**ドメイン・ファースト（アンチ・コラプション・レイヤー）**」 および「**関心の分離 (SoC)**」 という中核原則への明確な違反であった。

本計画は、この失敗を正し、アーキテクチャの境界を厳格に再定義する。中核となる原則は「**アンチ・コラプション・レイヤー (ACL)**」 の徹底である。

この原則は、アーキテクチャを以下の 3 つの明確に分離されたコンポーネントとして扱うことを**強制**する：

1.  **DTO (Data Transfer Object):**
    * **責務:** 外部 API（OpenAI, Gemini 等）の JSON 仕様と**完全に**一致するデータ専用クラス。
    * **規則:** シリアライズ属性（`[JsonPropertyName]`）やカスタム `JsonConverter` を持つ**唯一**の場所である。ビジネスロジックを**一切**含まない。

2.  **ドメインモデル (Domain Model):**
    * **責務:** `Nekote.AI.Domain` の純粋な内部ビジネス概念（例: `Conversation`）。
    * **規則:** 外部仕様から**完全に**隔離された純粋な POCO でなければならない。DTO への参照やシリアライズ属性を**一切**持ってはならない。

3.  **マッパー / リポジトリ (Mapper / Repository):**
    * **責務:** これら 2 つの世界（ドメインと DTO）を翻訳する「橋渡し」。
    * **規則:** `Infrastructure` レイヤーに属し、すべての「汚い」ベンダー固有ロジック（例: `"assistant"` という文字列を `NekoteChatRole.Assistant` enum に変換するロジック）を**カプセル化**する。

## 2. 構築戦略：「スケルトン・ファースト」

我々は「**スケルトン・ファースト（骨格を先に、肉付けは後に）**」アプローチを採用する。これは、AI アシスタントが実装の詳細（筋肉）を埋める前に、人間がアーキテクチャの境界（骨格）を定義するプロセスである。

このアプローチは、AI への指示を「曖昧な設計タスク」から「明確に型指定されたメソッド実装タスク」へと変換し、コード生成の精度を最大化することを目的とする。

### ステップ 1. 外部コントラクトの定義 (DTO)
`Infrastructure` レイヤーに、OpenAI と Gemini の API 仕様に 1:1 で対応する DTO クラス群を定義する。

### ステップ 2. 内部ドメインの定義 (Domain Model)
`Domain` レイヤーに、プロバイダから完全に独立した、純粋なドメインモデルを定義する。

### ステップ 3. ドメイン契約の定義 (Interface)
`Domain` レイヤーに、`IChatCompletionService` などのインターフェースを定義する。これらは純粋なドメインモデルのみを参照する。

### ステップ 4. インフラの骨格作成 (Skeleton)
`Infrastructure` レイヤーに、ステップ 3 のインターフェースを実装する具象リポジトリクラス（`OpenAiChatRepository` 等）の骨格を `throw new NotImplementedException()` のスタブメソッドと共に定義する。

## 3. ステップ 1：外部コントラクトの定義 (DTO)

`Infrastructure` レイヤー（例: `Nekote.AI.Infrastructure.OpenAI`）に、外部 API の仕様と 1:1 で対応する DTO クラス群を定義する。これらは `internal` としてカプセル化し、`Domain` レイヤーから参照できないようにすることが推奨される。

### 3.0. DTO 設計戦略：「全 Nullable」と JsonExtensionData

外部 API の実際の応答を完全には予測できないため、以下の防御的設計戦略を採用する：

1.  **全プロパティを Nullable にする:**
    * 参照型 (e.g., `string`, `OpenAiMessageDto`) は `?` を付ける。
    * 値型 (e.g., `int`, `long`, `bool`) も `?` を付けて nullable にする。
    * これにより、API がフィールドを返さない場合でも、デフォルト値 (0, false, 空のリスト) にフォールバックする代わりに `null` として明示的に表現できる。

2.  **`[JsonExtensionData]` を追加する:**
    * 各 DTO に `Dictionary<string, JsonElement>` プロパティを追加し、`[JsonExtensionData]` 属性を付与する。
    * これにより、DTO に定義されていない任意の JSON フィールド（例えば、将来 OpenAI が追加する新しいフィールド）もデシリアライズ時に保持され、エラーを発生させずに向先互換性を確保できる。
    * 必須フィールドのみを DTO に明示的に定義し、オプションのパラメータ (temperature, max_tokens 等) や使用しない機能 (tool calling 等) に関連するフィールドは定義しない。

この戦略により、コードは堅牢になり、API の予期せぬ変更や不完全なレスポンスに対しても耐性を持つ。

### 3.1. OpenAI Chat DTO (リクエスト)

OpenAI への Chat API リクエストを表現する DTO。

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace Nekote.AI.Infrastructure.OpenAI.Dtos
{
    /// <summary>
    /// OpenAI Chat API へのリクエストボディ DTO。
    /// </summary>
    internal class OpenAiChatRequestDto
    {
        [JsonPropertyName("model")]
        public string? Model { get; set; }

        [JsonPropertyName("messages")]
        public List<OpenAiMessageDto>? Messages { get; set; }

        [JsonPropertyName("stream")]
        public bool? Stream { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    /// <summary>
    /// OpenAI Chat API のメッセージ DTO (リクエスト/レスポンス共通部)。
    /// </summary>
    internal class OpenAiMessageDto
    {
        [JsonPropertyName("role")]
        public string? Role { get; set; }

        /// <summary>
        /// リクエスト送信時は、常に単純な文字列 content を使用する。
        /// (Nekote.AI はマルチモーダル入力をサポートしないため)
        /// レスポンス受信時は JsonConverter がこれを処理する。
        /// </summary>
        [JsonPropertyName("content")]
        [JsonConverter(typeof(OpenAiContentConverter))]
        public OpenAiContentBase? Content { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }
}
```

### 3.2. OpenAI Chat DTO (レスポンス)

OpenAI からのレスポンスを表現する DTO。

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace Nekote.AI.Infrastructure.OpenAI.Dtos
{
    /// <summary>
    /// OpenAI Chat API からのレスポンスボディ DTO。
    /// </summary>
    internal class OpenAiChatResponseDto
    {
        [JsonPropertyName("id")]
        public string? Id { get; set; }

        [JsonPropertyName("object")]
        public string? Object { get; set; }

        [JsonPropertyName("created")]
        public long? Created { get; set; }

        [JsonPropertyName("model")]
        public string? Model { get; set; }

        [JsonPropertyName("choices")]
        public List<OpenAiChoiceDto>? Choices { get; set; }

        [JsonPropertyName("usage")]
        public OpenAiUsageDto? Usage { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    /// <summary>
    /// レスポンスの "choice" オブジェクト DTO。
    /// </summary>
    internal class OpenAiChoiceDto
    {
        [JsonPropertyName("index")]
        public int? Index { get; set; }

        [JsonPropertyName("message")]
        public OpenAiMessageDto? Message { get; set; }

        [JsonPropertyName("finish_reason")]
        public string? FinishReason { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    /// <summary>
    /// トークン使用量 DTO。
    /// </summary>
    internal class OpenAiUsageDto
    {
        [JsonPropertyName("prompt_tokens")]
        public int? PromptTokens { get; set; }

        [JsonPropertyName("completion_tokens")]
        public int? CompletionTokens { get; set; }

        [JsonPropertyName("total_tokens")]
        public int? TotalTokens { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }
}
```

### 3.3. OpenAI Content ポリモーフィック処理 (DTO と Converter)

OpenAI の `content` プロパティ（文字列または配列）のどちらにも対応できる、型安全な DTO 階層とカスタム `JsonConverter` を定義する。これは ACL の「汚い処理」をカプセル化する中核的なロジックである。

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace Nekote.AI.Infrastructure.OpenAI.Dtos
{
    // --- 1. DTO 階層 ---

    /// <summary>
    /// "content" プロパティのポリモーフィックな
    /// 値を表現するための抽象基底 DTO。
    /// </summary>
    [JsonConverter(typeof(OpenAiContentConverter))]
    internal abstract class OpenAiContentBase
    { }

    /// <summary>
    /// "content" が単純な文字列の場合の具象 DTO。
    /// </summary>
    internal class OpenAiContentString : OpenAiContentBase
    {
        public string? Text { get; set; }
    }

    /// <summary>
    /// "content" が "parts" の配列の場合の具象 DTO。
    /// </summary>
    internal class OpenAiContentArray : OpenAiContentBase
    {
        public List<OpenAiContentPartDto>? Parts { get; set; }
    }

    /// <summary>
    /// "content" 配列内の "part" オブジェクト DTO。
    /// </summary>
    internal class OpenAiContentPartDto
    {
        [JsonPropertyName("type")]
        public string? Type { get; set; }

        /// <summary>
        /// type が "text" の場合にのみ使用する。
        /// </summary>
        [JsonPropertyName("text")]
        public string? Text { get; set; }

        // "image_url" DTO は Nekote.AI のスコープ外のため、
        // デシリアライズ対象に含めない。
    }

    // --- 2. カスタム JsonConverter ---

    /// <summary>
    /// OpenAI の "content" プロパティをデシリアライズするカスタム コンバーター。
    /// JSON の型（string, array, null）に応じて、
    /// OpenAiContentBase の適切な派生クラスをインスタンス化する。
    /// </summary>
    internal class OpenAiContentConverter : JsonConverter<OpenAiContentBase>
    {
        public override OpenAiContentBase Read(
            ref Utf8JsonReader reader,
            Type typeToConvert,
            JsonSerializerOptions options)
        {
            switch (reader.TokenType)
            {
                // Case 1: "content": "Hello world"
                case JsonTokenType.String:
                    return new OpenAiContentString { Text = reader.GetString() };

                // Case 2: "content": [ { "type": "text", ... } ]
                case JsonTokenType.StartArray:
                    // 標準デシリアライザーに "parts" のリストとして処理させる
                    var parts = JsonSerializer.Deserialize<List<OpenAiContentPartDto>>(ref reader, options);
                    return new OpenAiContentArray { Parts = parts ?? new() };

                // Case 3: "content": null (例: ツール呼び出し応答時)
                case JsonTokenType.Null:
                    // null の場合は、空の文字列コンテンツとして扱う
                    return new OpenAiContentString { Text = "" };

                default:
                    throw new JsonException(
                        $"Cannot deserialize 'content'. Expected string, array, or null, but got {reader.TokenType}.");
            }
        }

        // --- 書き込み (Write) の実装 ---
        // リクエストをシリアライズする際に使用する。
        public override void Write(
            Utf8JsonWriter writer,
            OpenAiContentBase value,
            JsonSerializerOptions options)
        {
            switch (value)
            {
                // Nekote.AI はテキストリクエストのみを送信するため、
                // 常に OpenAiContentString をシリアライズする。
                case OpenAiContentString stringContent:
                    writer.WriteStringValue(stringContent.Text);
                    break;

                // 配列のシリアライズはスコープ外（だが、念のため実装）
                case OpenAiContentArray arrayContent:
                    JsonSerializer.Serialize(writer, arrayContent.Parts, options);
                    break;

                case null:
                    writer.WriteNullValue();
                    break;
            }
        }
    }
}
```

### 3.4. Gemini Chat DTO (仮)

同様に、Gemini の API 仕様に対応する DTO も `Nekote.AI.Infrastructure.Gemini.Dtos` 名前空間に定義する。これらは OpenAI の DTO とは**全く異なる構造**を持つことになる（例: `contents` と `parts` の階層、`role` が `model` と `user` のみ）。

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace Nekote.AI.Infrastructure.Gemini.Dtos
{
    // (Gemini API の仕様に基づき、
    //  OpenAiChatRequestDto とは *異なる* DTO を定義する)

    internal class GeminiChatRequestDto
    {
        [JsonPropertyName("contents")]
        public List<GeminiContentDto>? Contents { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    internal class GeminiContentDto
    {
        [JsonPropertyName("role")]
        public string? Role { get; set; } // "user" または "model"

        [JsonPropertyName("parts")]
        public List<GeminiPartDto>? Parts { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    internal class GeminiPartDto
    {
        [JsonPropertyName("text")]
        public string? Text { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    // (レスポンス DTO も同様に定義)
}
```

### 3.5. OpenAI 詳細使用量 DTO

OpenAI の最新 API は、トークン使用量に関する詳細情報を返す。これらはセクション 3.2 の `OpenAiUsageDto` に統合される。

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace Nekote.AI.Infrastructure.OpenAI.Dtos
{
    /// <summary>
    /// トークン使用量詳細 DTO（最新 API 対応）。
    /// </summary>
    internal class OpenAiUsageDto
    {
        [JsonPropertyName("prompt_tokens")]
        public int? PromptTokens { get; set; }

        [JsonPropertyName("completion_tokens")]
        public int? CompletionTokens { get; set; }

        [JsonPropertyName("total_tokens")]
        public int? TotalTokens { get; set; }

        /// <summary>
        /// プロンプトトークンの詳細（キャッシュヒット等）。
        /// </summary>
        [JsonPropertyName("prompt_tokens_details")]
        public OpenAiPromptTokensDetailsDto? PromptTokensDetails { get; set; }

        /// <summary>
        /// 完了トークンの詳細（推論トークン等）。
        /// </summary>
        [JsonPropertyName("completion_tokens_details")]
        public OpenAiCompletionTokensDetailsDto? CompletionTokensDetails { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    /// <summary>
    /// プロンプトトークンの内訳詳細 DTO。
    /// </summary>
    internal class OpenAiPromptTokensDetailsDto
    {
        /// <summary>
        /// キャッシュから読み込まれたトークン数。
        /// </summary>
        [JsonPropertyName("cached_tokens")]
        public int? CachedTokens { get; set; }

        /// <summary>
        /// オーディオ入力に使用されたトークン数。
        /// </summary>
        [JsonPropertyName("audio_tokens")]
        public int? AudioTokens { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    /// <summary>
    /// 完了トークンの内訳詳細 DTO。
    /// </summary>
    internal class OpenAiCompletionTokensDetailsDto
    {
        /// <summary>
        /// 推論（reasoning）に使用されたトークン数。
        /// </summary>
        [JsonPropertyName("reasoning_tokens")]
        public int? ReasoningTokens { get; set; }

        /// <summary>
        /// オーディオ出力に使用されたトークン数。
        /// </summary>
        [JsonPropertyName("audio_tokens")]
        public int? AudioTokens { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }
}
```

### 3.6. OpenAI ストリーミング DTO

OpenAI の Chat API はストリーミングレスポンスをサポートする。`stream=true` を設定すると、API は Server-Sent Events (SSE) 形式でチャンクを送信する。

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace Nekote.AI.Infrastructure.OpenAI.Dtos
{
    /// <summary>
    /// OpenAI Chat API のストリーミングレスポンスチャンク DTO。
    /// </summary>
    internal class OpenAiChatStreamChunkDto
    {
        [JsonPropertyName("id")]
        public string? Id { get; set; }

        [JsonPropertyName("object")]
        public string? Object { get; set; }

        [JsonPropertyName("created")]
        public long? Created { get; set; }

        [JsonPropertyName("model")]
        public string? Model { get; set; }

        [JsonPropertyName("choices")]
        public List<OpenAiStreamChoiceDto>? Choices { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    /// <summary>
    /// ストリーミングレスポンスの "choice" オブジェクト DTO。
    /// </summary>
    internal class OpenAiStreamChoiceDto
    {
        [JsonPropertyName("index")]
        public int? Index { get; set; }

        /// <summary>
        /// ストリーミング時は "message" ではなく "delta" が使用される。
        /// </summary>
        [JsonPropertyName("delta")]
        public OpenAiStreamDeltaDto? Delta { get; set; }

        [JsonPropertyName("finish_reason")]
        public string? FinishReason { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    /// <summary>
    /// ストリーミングチャンクの差分 (delta) DTO。
    /// </summary>
    internal class OpenAiStreamDeltaDto
    {
        [JsonPropertyName("role")]
        public string? Role { get; set; }

        [JsonPropertyName("content")]
        public string? Content { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }
}
```

### 3.7. OpenAI エラー DTO

OpenAI API がエラーを返す場合、標準化されたエラーレスポンス構造を使用する。リポジトリクラスはこれらの DTO を使用してエラーを処理する必要がある。

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace Nekote.AI.Infrastructure.OpenAI.Dtos
{
    /// <summary>
    /// OpenAI API エラーレスポンス DTO。
    /// </summary>
    internal class OpenAiErrorResponseDto
    {
        [JsonPropertyName("error")]
        public OpenAiErrorDto? Error { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }

    /// <summary>
    /// OpenAI API エラー詳細 DTO。
    /// </summary>
    internal class OpenAiErrorDto
    {
        [JsonPropertyName("message")]
        public string? Message { get; set; }

        [JsonPropertyName("type")]
        public string? Type { get; set; }

        [JsonPropertyName("param")]
        public string? Param { get; set; }

        [JsonPropertyName("code")]
        public string? Code { get; set; }

        /// <summary>
        /// API から返される未知のフィールドを保持する。
        /// </summary>
        [JsonExtensionData]
        public Dictionary<string, JsonElement>? ExtensionData { get; set; }
    }
}
```

## 4. ステップ 2 & 3：内部ドメインと契約の定義 (モデル & インターフェース)

「外部の世界」(DTO) が定義された今、それとは**完全に独立**した「内部の世界」(Domain) を定義する。

### 4.1. 内部ドメインモデルの定義 (ステップ 2)

これらは `Nekote.AI.Domain` レイヤーに配置される、純粋な POCO (Plain Old C# Object) である。これらは外部 API の仕様やシリアライズ属性を**一切含まない**。

```csharp
namespace Nekote.AI.Domain
{
    /// <summary>
    /// Nekote.AI が内部で統一的に使用する、
    /// プロバイダから独立したチャットロール。
    /// </summary>
    public enum NekoteChatRole
    {
        /// <summary>
        /// システム指示 (例: OpenAI の "system")
        /// </summary>
        System,

        /// <summary>
        /// ユーザー入力 (例: OpenAI の "user", Gemini の "user")
        /// </summary>
        User,

        /// <summary>
        /// AI の応答 (例: OpenAI の "assistant", Gemini の "model")
        /// </summary>
        Assistant
    }

    /// <summary>
    /// Nekote.AI の純粋なチャットメッセージ・ドメインモデル。
    /// 外部 API の DTO とは一切の結合を持たない。
    /// </summary>
    public class NekoteChatMessage
    {
        /// <summary>
        /// メッセージのロール。
        /// </summary>
        public NekoteChatRole Role { get; }

        /// <summary>
        /// メッセージのテキスト内容。
        /// </summary>
        public string Content { get; }

        /// <summary>
        /// コンストラクタ。
        /// </summary>
        /// <param name="role">ロール。</param>
        /// <param name="content">テキスト内容。</param>
        public NekoteChatMessage(NekoteChatRole role, string content)
        {
            // (Enum 値の検証はここ、またはファクトリメソッドで行う)
            // (string.IsNullOrWhiteSpace の検証もここで行う)
            Role = role;
            Content = content ?? string.Empty; // null セマンティクス
        }
    }
}
```

### 4.2. ドメイン契約の定義 (ステップ 3)

これらも `Domain` レイヤーに配置される「契約」であり、リポジトリパターンを定義する。これらのインターフェースは、ステップ 4.1 で定義した**純粋なドメインモデルのみ**を参照する。

```csharp
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;

namespace Nekote.AI.Domain.Contracts
{
    /// <summary>
    /// チャット補完サービス（リポジトリ）のドメイン契約。
    /// </summary>
    /// <remarks>
    /// Infrastructure レイヤーがこのインターフェースを実装する。
    /// </remarks>
    public interface IChatCompletionService
    {
        /// <summary>
        /// チャットの履歴に基づき、次の AI 応答を取得する。
        /// </summary>
        /// <param name="history">
        /// これまでの会話履歴（純粋なドメインモデルのリスト）。
        /// </param>
        /// <param name="ct">キャンセレーショントークン。</param>
        /// <returns>AI からの応答（純粋なドメインモデル）。</returns>
        Task<NekoteChatMessage> GetCompletionAsync(
            IEnumerable<NekoteChatMessage> history,
            CancellationToken ct);
    }

    /// <summary>
    /// テキストエンベディングサービス（リポジトリ）のドメイン契約。
    /// </summary>
    public interface ITextEmbeddingService
    {
        /// <summary>
        /// 単一のテキスト入力に対するエンベディング ベクトルを生成する。
        /// </summary>
        /// <param name="text">ベクトル化するテキスト。</param>
        /// <param name="ct">キャンセレーショントークン。</param>
        /// <returns>エンベディングベクトル。</returns>
        Task<float[]> GetEmbeddingAsync(
            string text,
            CancellationToken ct);
    }
}
```

## 5. ステップ 4：インフラストラクチャの骨格作成 (Skeleton)

「外部 (DTO)」と「内部 (Interface)」という 2 つの固定された境界ができた。最後に、これらを繋ぐ「橋」となる `Infrastructure` レイヤーの具象クラス（リポジトリ）の骨格を定義する。

これらのクラスは、`PLAYBOOK.md` (セクション 3.3) に従い、`Infrastructure/OpenAI` や `Infrastructure/Gemini` といった「**垂直スライス (Vertical Slice)**」ディレクトリに配置される。

### 5.1. OpenAI リポジトリの骨格

`Nekote.AI.Infrastructure.OpenAI` プロジェクト内に配置される。

```csharp
using Microsoft.Extensions.Configuration;
using Nekote.AI.Domain;
using Nekote.AI.Domain.Contracts;
using Nekote.AI.Infrastructure.OpenAI.Dtos; // 内部 DTO を参照
using System.Net.Http; // HttpClient を使用
using System.Text.Json; // JsonSerializer を使用

namespace Nekote.AI.Infrastructure.OpenAI
{
    /// <summary>
    /// OpenAI の Chat API と通信する具象リポジトリ。
    /// IChatCompletionService のドメイン契約を実装する。
    /// </summary>
    internal class OpenAiChatRepository : IChatCompletionService
    {
        private readonly HttpClient _httpClient;
        private readonly IConfiguration _configuration;

        // DI (依存性注入) により HttpClient と設定が提供される
        public OpenAiChatRepository(
            IHttpClientFactory httpClientFactory,
            IConfiguration configuration)
        {
            _httpClient = httpClientFactory.CreateClient("OpenAI");
            _configuration = configuration;
        }

        /// <summary>
        /// ドメイン契約の実装。
        /// ここに AI (アシスタント) が実装（筋肉）を書き込む。
        /// </summary>
        public async Task<NekoteChatMessage> GetCompletionAsync(
            IEnumerable<NekoteChatMessage> history,
            CancellationToken ct)
        {
            // --- 1. マップ (Domain -> DTO) ---
            // 純粋なドメインモデルを、OpenAI の DTO に変換する。
            var requestDto = MapToOpenAiRequestDto(history);

            // --- 2. API 呼び出し ---
            // (HttpClient を使用した実際の POST リクエストと
            //  レスポンス DTO (OpenAiChatResponseDto) の受信)

            // (このスケルトンでは NotImplementedException を throw する)
            throw new NotImplementedException(
                "AI must implement HttpClient POST request and JSON deserialization logic here.");

            // --- 3. マップ (DTO -> Domain) ---
            // OpenAiChatResponseDto から、応答メッセージ DTO を取得
            // var responseMessageDto = responseDto.Choices.First().Message;
            //
            // DTO を純粋なドメインモデルに変換して返す
            // return MapToNekoteDomain(responseMessageDto);
        }

        // --- 5.2. マッパー (アンチ・コラプション・レイヤー) ---
        //
        // マッパーは、リポジトリの責務の一部として
        // プライベートメソッドで実装する。

        private OpenAiChatRequestDto MapToOpenAiRequestDto(
            IEnumerable<NekoteChatMessage> history)
        {
            var requestDto = new OpenAiChatRequestDto
            {
                Model = _configuration["Nekote:Ai:OpenAI:Model"] ?? "gpt-5-turbo"
            };

            foreach (var message in history)
            {
                requestDto.Messages.Add(new OpenAiMessageDto
                {
                    Role = MapRoleToOpenAi(message.Role),
                    Content = new OpenAiContentString { Text = message.Content }
                });
            }
            return requestDto;
        }

        private NekoteChatMessage MapToNekoteDomain(OpenAiMessageDto responseDto)
        {
            // DTO の content からテキストを抽出する
            // (このロジックは AI が実装する)
            string textContent = ExtractTextFromContent(responseDto.Content);

            // DTO のロール文字列を、純粋な Enum に変換する
            NekoteChatRole role = responseDto.Role.ToLowerInvariant() switch
            {
                "assistant" => NekoteChatRole.Assistant,
                "user" => NekoteChatRole.User,       // (通常レスポンスでは発生しないが念のため)
                "system" => NekoteChatRole.System,   // (同様)
                _ => throw new InvalidDataException(
                    $"OpenAI API から未知のロール '{responseDto.Role}' が返されました。")
            };

            return new NekoteChatMessage(role, textContent);
        }

        /// <summary>
        /// DTO のポリモーフィックな content からテキストを抽出する。
        /// (このスケルトンでは NotImplementedException を throw する)
        /// </summary>
        private string ExtractTextFromContent(OpenAiContentBase content)
        {
            // AI はここに、OpenAiContentString と OpenAiContentArray を
            // switch パターンで処理し、"text" type のみを取り出す
            // クリーンなマッピングロジックを実装する必要がある。
            switch (content)
            {
                case OpenAiContentString stringContent:
                    // return stringContent.Text;
                    break;
                case OpenAiContentArray arrayContent:
                    // return string.Join("\n", arrayContent.Parts
                    //     .Where(p => p.Type == "text")
                    //     .Select(p => p.Text));
                    break;
            }

            throw new NotImplementedException(
                "AI must implement text extraction logic here based on OpenAiContentBase concrete type (String/Array).");
        }

        private string MapRoleToOpenAi(NekoteChatRole role)
        {
            return role switch
            {
                NekoteChatRole.System => "system",
                NekoteChatRole.User => "user",
                NekoteChatRole.Assistant => "assistant",
                // Enum 検証はドメインモデルのコンストラクタで行うため、
                // _ (default) ケースは理論上不要
                _ => throw new ArgumentOutOfRangeException(nameof(role),
                    $"予期しない NekoteChatRole '{role}' です。")
            };
        }
    }
}
```

### 5.2. Gemini リポジトリの骨格

`Nekote.AI.Infrastructure.Gemini` プロジェクト内に配置される。
OpenAI のリポジトリと**同じドメイン契約**（`IChatCompletionService`）を実装するが、**全く異なる内部 DTO とマッピングロジック**を使用することが重要である。

```csharp
using Nekote.AI.Domain;
using Nekote.AI.Domain.Contracts;
using Nekote.AI.Infrastructure.Gemini.Dtos; // Gemini 固有の DTO を参照

namespace Nekote.AI.Infrastructure.Gemini
{
    /// <summary>
    /// Gemini の API と通信する具象リポジトリ。
    /// </summary>
    internal class GeminiChatRepository : IChatCompletionService
    {
        // (HttpClient, IConfiguration 等のインジェクション...)

        public async Task<NekoteChatMessage> GetCompletionAsync(
            IEnumerable<NekoteChatMessage> history,
            CancellationToken ct)
        {
            // --- 1. マップ (Domain -> DTO) ---
            //
            // ★ここが OpenAI の実装と *完全に異なる* 部分★
            //
            // Gemini はロール ("user", "model") が交互に
            // 出現することしかサポートしない。
            // また、"system" ロールを持たない。
            // この「汚い」変換ロジックを、このリポジトリの
            // マッパーが *すべて* 引き受ける。
            //
            var geminiRequestDto = MapToGeminiRequestDto(history);

            // (このスケルトンでは NotImplementedException を throw する)
            throw new NotImplementedException(
                "AI must implement Gemini-specific DTO mapping (to GeminiChatRequestDto), API call, and response DTO to domain remapping logic here.");

            // (API 呼び出し...)

            // (DTO -> Domain へのマッピング...)
        }

        private GeminiChatRequestDto MapToGeminiRequestDto(
            IEnumerable<NekoteChatMessage> history)
        {
            // AI はここに、Nekote.AI の履歴 (System ロールを含む可能性がある) を、
            // Gemini の "user"/"model" の交互の履歴に変換するロジックを実装する必要がある。
            // (例: System プロンプトを、最初の User メッセージに結合する等)
            throw new NotImplementedException();
        }

        private NekoteChatMessage MapToNekoteDomain(
            GeminiChatResponseDto responseDto)
        {
            // AI はここに、Gemini のレスポンス DTO
            // (例: response.candidates[0].content.parts[0].text) から
            // テキストを抽出し、NekoteChatMessage を生成するロジックを実装する必要がある。
            // ロールは常に NekoteChatRole.Assistant となる。
            throw new NotImplementedException();
        }
    }
}
```

---
*[nao7sep](https://github.com/nao7sep) | Nekote AI New Plan | 2025-11-16*
