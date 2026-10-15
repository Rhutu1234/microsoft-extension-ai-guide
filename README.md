# Microsoft.Extensions.AI

*A deep-dive walkthrough of Microsoft.Extensions.AI — the modern, provider-neutral .NET abstraction layer for AI services — covering `IChatClient` and `IEmbeddingGenerator`, how AI service abstractions let you swap providers, dependency injection registration, the middleware pipeline (logging, caching, telemetry, function invocation, and custom middleware), tool/function calling with `AIFunction`, streaming, structured output, and integrating it all into ASP.NET Core.*

---

## Table of Contents

1. [Introduction](#introduction)
2. [Overview: What It Is and Why It Exists](#1-overview-what-it-is-and-why-it-exists)
3. [IChatClient](#2-ichatclient)
4. [IEmbeddingGenerator](#3-iembeddinggenerator)
5. [AI Service Abstractions](#4-ai-service-abstractions)
6. [Dependency Injection](#5-dependency-injection)
7. [Middleware](#6-middleware)
8. [Tool / Function Calling](#7-tool--function-calling)
9. [Streaming](#8-streaming)
10. [Integration with ASP.NET Core](#9-integration-with-aspnet-core)
11. [Testing and Evaluation](#10-testing-and-evaluation)
12. [Microsoft.Extensions.AI vs. the Alternatives](#11-microsoftextensionsai-vs-the-alternatives)
13. [Common Pitfalls](#12-common-pitfalls)
14. [Quick Reference Table](#quick-reference-table)
15. [Conclusion](#conclusion)

---

## Introduction

Every AI provider ships its own SDK with its own types: different message classes, different option names, different streaming shapes, different ways to define tools. Code written directly against one SDK is welded to that vendor, hard to unit-test, and re-implements the same cross-cutting concerns — logging, caching, telemetry, retries, the tool-calling loop — for every project. .NET already solved this class of problem for logging (`ILogger`), configuration (`IConfiguration`), HTTP (`HttpClient` + handlers), and caching (`IDistributedCache`): define a small interface, let providers implement it, and compose behavior with middleware. **Microsoft.Extensions.AI (MEAI)** does the same for AI.

```csharp
IChatClient client = new OpenAIClient(apiKey)
    .GetChatClient("gpt-4o-mini")
    .AsIChatClient();                                   // adapt a provider SDK to the shared interface

ChatResponse response = await client.GetResponseAsync("Explain dependency injection in one sentence.");
Console.WriteLine(response.Text);
```

Swap that first expression for Azure OpenAI, Ollama, or any other `IChatClient`, and the second statement doesn't change. That's the whole idea; the rest of this guide is about composing behavior around it (middleware), registering it properly (DI), and operating it in ASP.NET Core.

> **A note on versions.** MEAI went through a preview period in which method names changed (for example, `CompleteAsync` became `GetResponseAsync`, and `AsChatClient` became `AsIChatClient`), and it continues to gain features. Older blog posts often use the old names. The samples here target the current shape of the API; compile against the version you install and check the release notes. Model names (`gpt-4o-mini` and similar) are placeholders — keep them in configuration.

---

## 1. Overview: What It Is and Why It Exists

### Two packages, two roles

```plaintext
Microsoft.Extensions.AI.Abstractions   The CONTRACT: interfaces and types — IChatClient, IEmbeddingGenerator,
                                       ChatMessage, ChatOptions, AIFunction, and so on. Library authors and
                                       providers reference only this, so it stays small and stable.

Microsoft.Extensions.AI                The BATTERIES: ChatClientBuilder, and the middleware — logging, distributed
                                       caching, OpenTelemetry, automatic function invocation — plus DI
                                       registration helpers. Applications reference this.
```

Provider integrations then implement the interfaces:

```plaintext
OpenAI / Azure OpenAI       Microsoft.Extensions.AI.OpenAI (or Azure.AI.OpenAI) — adapters via .AsIChatClient()
                            and .AsIEmbeddingGenerator()
Ollama (local models)       OllamaSharp's client implements IChatClient directly
Azure AI Inference / GitHub Models   Microsoft.Extensions.AI.AzureAIInference
Others                      A growing list of community and vendor packages
```

### What the abstraction gives you

```plaintext
PORTABILITY       Swap providers or models by changing registration, not calling code.
COMPOSABILITY     Add logging, caching, telemetry, rate limiting, and tool invocation as middleware —
                  once, in one place, for every call.
TESTABILITY       Depend on an interface; substitute a fake in unit tests (Section 10).
A COMMON LANGUAGE Library authors can write AI-using libraries that work with any provider.
ECOSYSTEM         Higher-level frameworks — Semantic Kernel and the Microsoft Agent Framework — build on these
                  abstractions, so what you learn here carries over, and you can mix them.
```

### What it deliberately is *not*

```plaintext
It is NOT an orchestration framework. It has no prompt-template engine, plugin system, memory abstraction, or
agent loop (those are Semantic Kernel / Agent Framework territory — Section 11). It is a thin, stable foundation
of exchange types and a middleware pipeline. Use it when you want control and minimal dependencies.
```

### The layered picture

```plaintext
 Your application code  ──>  IChatClient (the same interface at every layer)
                                 │
                       ┌─────────┴──────────────────────────────┐
                       │  Middleware pipeline (DelegatingChatClient chain)
                       │    logging → telemetry → caching → function invocation → (your own) ...
                       └─────────┬──────────────────────────────┘
                                 │
                       Provider adapter (OpenAI / Azure OpenAI / Ollama / ...)
                                 │
                            The model service
```

---

## 2. IChatClient

### The core interface

```csharp
public interface IChatClient : IDisposable
{
    Task<ChatResponse> GetResponseAsync(
        IEnumerable<ChatMessage> messages, ChatOptions? options = null, CancellationToken cancellationToken = default);

    IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(
        IEnumerable<ChatMessage> messages, ChatOptions? options = null, CancellationToken cancellationToken = default);

    object? GetService(Type serviceType, object? serviceKey = null);
}
```

Three members: get a full response, get a streamed response, and ask the client for services it exposes (metadata, or the underlying native client). Everything else is extension methods and types around these.

### A single prompt, and a full conversation

```csharp
// Convenience overload: a single user message
ChatResponse quick = await client.GetResponseAsync("Name three C# 12 features.");

// A real conversation: roles and messages
List<ChatMessage> history =
[
    new(ChatRole.System, "You are a concise C# tutor."),
    new(ChatRole.User,   "What is a record?"),
];

ChatResponse response = await client.GetResponseAsync(history);
Console.WriteLine(response.Text);

history.AddRange(response.Messages);                       // keep the assistant's reply in the history
history.Add(new(ChatRole.User, "How is it different from a class?"));
response = await client.GetResponseAsync(history);
```

```plaintext
Roles: ChatRole.System | User | Assistant | Tool
The model is STATELESS — you hold and resend the history each turn, exactly as in AI APIs with .NET, Section 4.
(Some providers offer service-managed conversation state via ChatOptions.ConversationId; the default model
is "you resend everything.")
```

### `ChatResponse`: what you get back

```csharp
string text            = response.Text;                      // the combined text of the assistant message(s)
IList<ChatMessage> msgs = response.Messages;                  // may include tool-call and tool-result messages
UsageDetails? usage    = response.Usage;                     // InputTokenCount, OutputTokenCount, TotalTokenCount
ChatFinishReason? why  = response.FinishReason;              // Stop, Length, ToolCalls, ContentFilter...
string? model          = response.ModelId;
```

As with every LLM API: **log `Usage`** for cost tracking and **check `FinishReason`** — a `Length` finish means the answer was truncated (see *AI APIs with .NET*, Sections 4 and 8).

### `ChatOptions`: provider-neutral model parameters

```csharp
var options = new ChatOptions
{
    Temperature = 0.2f,
    MaxOutputTokens = 500,
    TopP = 0.9f,
    StopSequences = ["END_OF_ANSWER"],
    Tools = [ /* AIFunctions — Section 7 */ ],
    ToolMode = ChatToolMode.Auto,                            // Auto | RequireAny | None
    ResponseFormat = ChatResponseFormat.Json,                // or a JSON schema (see structured output below)
    ModelId = "gpt-4o",                                      // override the client's default model for this call
};

ChatResponse response = await client.GetResponseAsync(history, options);
```

The options are a **common denominator**: properties a provider doesn't support are ignored or mapped as closely as the adapter can. For provider-specific features, use the escape hatches below. As always, parameter meanings are in *AI APIs with .NET*, Section 9 — and reasoning models may restrict some of them.

### Multimodal input

```csharp
var message = new ChatMessage(ChatRole.User,
[
    new TextContent("What is in this image?"),
    new DataContent(imageBytes, "image/png"),                // inline bytes
    // new UriContent(new Uri("https://example.com/photo.jpg"), "image/jpeg")   // or a URL
]);

ChatResponse response = await client.GetResponseAsync([message]);
```

A `ChatMessage` holds a list of **content** items — text, data, URIs, function calls and results, usage, reasoning — which is how one message type covers text, images, and tool traffic across providers (subject to what the model supports).

### Structured output

```csharp
public record WeatherReport(string City, double TemperatureC, string Conditions);

ChatResponse<WeatherReport> response = await client.GetResponseAsync<WeatherReport>(
    "Give a plausible weather report for Pune.");

WeatherReport report = response.Result;                      // deserialized, typed
```

`GetResponseAsync<T>` derives a JSON Schema from your type, asks the model to conform to it (using schema-constrained output where the provider supports it), and deserializes the result. As always: **a schema guarantees shape, not truth** — validate the values (see *AI APIs with .NET*, Section 7), and use `TryGetResult` if you want to handle malformed output without an exception.

### Metadata and escape hatches

```csharp
ChatClientMetadata? meta = client.GetService<ChatClientMetadata>();
Console.WriteLine($"{meta?.ProviderName} / {meta?.DefaultModelId}");

// Reach the native provider client when you need a provider-specific feature
OpenAI.Chat.ChatClient? native = client.GetService<OpenAI.Chat.ChatClient>();

// Or pass native options through ChatOptions
var options = new ChatOptions
{
    RawRepresentationFactory = _ => new OpenAI.Chat.ChatCompletionOptions { /* provider-specific settings */ }
};
```

`GetService` walks through the middleware pipeline down to the provider, so you can always get at the underlying client. The abstraction is a convenience, not a cage.

### Exceptions are not unified

```plaintext
MEAI standardizes the REQUEST/RESPONSE shapes, not the errors. An OpenAI-backed client throws the OpenAI SDK's
exception type (ClientResultException), an Azure AI Inference client throws its own, an Ollama client throws
HttpRequestException. If you need provider-independent error handling (retries, circuit breakers, friendly
messages), implement it in middleware or at one boundary in your code (Section 6) rather than scattering
provider-specific catch blocks.
```

---

## 3. IEmbeddingGenerator

### The embedding counterpart

```csharp
public interface IEmbeddingGenerator<in TInput, TEmbedding> : IDisposable
    where TEmbedding : Embedding
{
    Task<GeneratedEmbeddings<TEmbedding>> GenerateAsync(
        IEnumerable<TInput> values, EmbeddingGenerationOptions? options = null, CancellationToken cancellationToken = default);

    object? GetService(Type serviceType, object? serviceKey = null);
}
```

For text embeddings the concrete type is `IEmbeddingGenerator<string, Embedding<float>>`. (Embeddings and their uses are the subject of this series' *Embeddings & Vector Search* guide.)

### Creating one

```csharp
// OpenAI
IEmbeddingGenerator<string, Embedding<float>> generator =
    new OpenAIClient(apiKey).GetEmbeddingClient("text-embedding-3-small").AsIEmbeddingGenerator();

// Azure OpenAI — the DEPLOYMENT name, with keyless Entra ID auth
IEmbeddingGenerator<string, Embedding<float>> azureGenerator =
    new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
        .GetEmbeddingClient("my-embedding-deployment")
        .AsIEmbeddingGenerator();
```

### Using it

```csharp
// One text -> one vector
ReadOnlyMemory<float> vector = await generator.GenerateVectorAsync("How do I reset my password?");

// Many texts in one request (much more efficient for ingestion)
GeneratedEmbeddings<Embedding<float>> embeddings = await generator.GenerateAsync(chunks);
for (int i = 0; i < chunks.Count; i++)
    Store(chunks[i], embeddings[i].Vector);

// Shorter vectors, where the model supports it
var options = new EmbeddingGenerationOptions { Dimensions = 512 };
var small = await generator.GenerateVectorAsync(text, options);

// Compare two vectors
float similarity = TensorPrimitives.CosineSimilarity(a.Span, b.Span);     // System.Numerics.Tensors
```

### Metadata and middleware

```csharp
var meta = generator.GetService<EmbeddingGeneratorMetadata>();
// provider name, default model, and default dimension count — handy for sizing a vector index

// Embedding generators have their own builder and middleware, just like chat clients
builder.Services.AddEmbeddingGenerator(sp =>
        new OpenAIClient(apiKey).GetEmbeddingClient("text-embedding-3-small").AsIEmbeddingGenerator())
    .UseDistributedCache()                                    // cache embeddings of repeated text
    .UseLogging()
    .UseOpenTelemetry();
```

```plaintext
Caching embeddings is one of the best uses of middleware: embedding the same chunk or the same popular query
repeatedly is pure waste. Keep the rules from the vector search guide: ONE embedding model for documents and
queries; record the model with stored vectors; embeddings derived from sensitive text are sensitive data.
```

---

## 4. AI Service Abstractions

### Same code, different providers

```csharp
// The only provider-specific lines are the construction of the inner client
IChatClient openAi = new OpenAIClient(openAiKey).GetChatClient("gpt-4o-mini").AsIChatClient();

IChatClient azure  = new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
                         .GetChatClient("my-gpt-deployment").AsIChatClient();

IChatClient local  = new OllamaApiClient(new Uri("http://localhost:11434"), "llama3.2");     // OllamaSharp

// Everything downstream depends only on IChatClient
async Task<string> SummarizeAsync(IChatClient chat, string text)
    => (await chat.GetResponseAsync($"Summarize in two sentences:\n\n{text}")).Text;
```

### Why this matters in practice

```plaintext
DEVELOP LOCALLY, DEPLOY TO THE CLOUD   Run a local model through Ollama while developing (free, offline), and use
                                       Azure OpenAI in production — selected by configuration, same code.

AVOID LOCK-IN                          Prices, quality, and availability change fast. Moving to a different provider
                                       or model is a registration change, not a rewrite.

MODEL ROUTING                          Register several clients — a small, cheap one and a strong one — and choose
                                       per task (keyed services, Section 5), as recommended in LLM Application
                                       Development, Section 9.

FALLBACKS                              Wrap a primary and a secondary client in middleware that fails over when
                                       the first is unavailable.

LIBRARIES                              Write reusable AI-using code that accepts an IChatClient and works for any host.
```

### The honest limits of the abstraction

```plaintext
COMMON DENOMINATOR      Features that exist on only one provider (a specific built-in tool, a proprietary option)
                        may need the escape hatches (GetService<T>, RawRepresentationFactory) — which then
                        reintroduce provider coupling, so isolate them behind your own interface.

BEHAVIOR DIFFERS        Different models follow instructions, call tools, and format JSON differently. The
                        interface being identical does NOT mean the results are. Re-run your evaluation set when you
                        switch models (see RAG, Section 12).

ERRORS AREN'T UNIFIED   See Section 2.

NEW FEATURES LAG        New provider capabilities arrive in the native SDK first and in the abstraction later.
```

Treat MEAI as the thing that makes the *common 90%* portable and testable, while accepting that the last 10% sometimes needs native code.

---

## 5. Dependency Injection

### Register the client in the container

```csharp
// Program.cs
builder.Services.AddChatClient(sp =>
        new OpenAIClient(builder.Configuration["OpenAI:ApiKey"]!)
            .GetChatClient(builder.Configuration["OpenAI:Model"]!)
            .AsIChatClient())
    .UseLogging()
    .UseOpenTelemetry(sourceName: "MyApp.AI")
    .UseFunctionInvocation();
```

`AddChatClient` registers an `IChatClient` and **returns a `ChatClientBuilder`**, onto which you chain middleware (Section 6). Anything in your application can now take an `IChatClient` in its constructor:

```csharp
public class SummaryService(IChatClient chat)
{
    public async Task<string> SummarizeAsync(string text, CancellationToken ct) =>
        (await chat.GetResponseAsync($"Summarize in two sentences:\n\n{text}", cancellationToken: ct)).Text;
}

builder.Services.AddScoped<SummaryService>();
```

### Lifetime

```plaintext
The default is SINGLETON, which is right for most cases: provider clients are designed to be created once and
reused (they manage connections and are thread-safe). Don't register a new chat client per request — you lose
connection reuse. Per-request VARIATION belongs in ChatOptions and in the messages you pass, not in a new client.
You can pass a different ServiceLifetime to AddChatClient when you genuinely need scoped behavior.
```

### Configuration and secrets

```csharp
public class AiOptions
{
    public string Model { get; set; } = "";
    public string Endpoint { get; set; } = "";
    public int MaxOutputTokens { get; set; } = 600;
}

builder.Services.Configure<AiOptions>(builder.Configuration.GetSection("AI"));
```

```plaintext
- Model names and deployment names live in CONFIGURATION, never in code.
- Secrets: dotnet user-secrets in development; Key Vault or your platform's secret store in production;
  on Azure prefer KEYLESS auth (DefaultAzureCredential with a managed identity) so there is nothing to leak or rotate.
- Never put a provider key in a client app — clients call YOUR backend.
```

### Multiple clients with keyed services

```csharp
builder.Services.AddKeyedChatClient("fast",
    new OpenAIClient(apiKey).GetChatClient("gpt-4o-mini").AsIChatClient());

builder.Services.AddKeyedChatClient("smart",
    new OpenAIClient(apiKey).GetChatClient("gpt-4o").AsIChatClient());

// Consumers pick by key
public class TriageService([FromKeyedServices("fast")] IChatClient cheap,
               
