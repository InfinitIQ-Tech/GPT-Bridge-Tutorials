# Migrating From GPTBridge Assistants to Chat Completions

## Introduction

GPTBridge originally focused on OpenAI's Assistants API: your app selected an
assistant, created a thread, added messages to that thread, and created runs to
let the assistant respond. GPTBridge now also includes a
[Chat Completions](https://developers.openai.com/api/reference/resources/chat)
interface for apps that want a simpler request/response model.

This tutorial shows how to adapt an existing GPTBridge Assistants app to the new
Chat Completions surface. You will keep the same SwiftUI chat experience, but
you will move conversation state into your app instead of relying on OpenAI
threads and runs.

By the end, you will know how to:

- Replace assistant IDs with model IDs and local instructions.
- Replace OpenAI threads with a local `[ChatCompletionMessage]` array.
- Replace assistant run streaming events with completion streaming events.
- Move assistant dashboard tools into Swift-defined `ChatCompletionTool` values.
- Send tool results back through the next Chat Completions request.

> `Note`: OpenAI's API reference currently marks
> [Assistants](https://developers.openai.com/api/reference/resources/beta/subresources/assistants/methods/create)
> endpoints as deprecated. This tutorial focuses on GPTBridge's Chat Completions
> API for new or migrated app code. Keep the old Assistants flow only when you
> need to maintain an existing integration.

---

## What Changes?

### Assistants API Flow

In the Assistants API version, GPTBridge keeps your app close to OpenAI's
thread/run model:

1. List assistants with `GPTBridge.listAssistants()`.
2. Create or reuse a thread.
3. Add a user message to the thread.
4. Create a run for the selected assistant.
5. Stream or poll run events.
6. Submit tool outputs if the run requires action.

That flow stores important state server-side: assistant instructions live on
the assistant, and conversation history lives in the thread.

### Chat Completions Flow

With Chat Completions, your app sends the conversation messages on each request:

1. Keep a local `[ChatCompletionMessage]`.
2. Add a developer or system instruction message at the beginning.
3. Append each user message.
4. Call `GPTBridge.createChatCompletion` or `GPTBridge.streamChatCompletion`.
5. Append the assistant response to your local message array.
6. If the model requests a tool call, execute the tool, append a tool-output
message, and call Chat Completions again.

The important shift is ownership: your app now owns message history,
instructions, and tool definitions.

---

## Step 1: Replace Assistant State With Completion State

An Assistants view model usually has state like this:

```swift
@Published var activeAssistant: Assistant
@Published var messages: [ChatMessage] = []
@Published var streamingText: String = ""

private var threadId: String?
```

For Chat Completions, keep a display transcript for your UI and a separate
message array for the API:

```swift
@Published var messages: [ChatMessage] = []
@Published var streamingText: String = ""

private let model = "your-model-id"
private var completionMessages: [ChatCompletionMessage] = [
    ChatCompletionMessage(
        role: .developer,
        content: "You are a concise assistant inside my Swift app."
    )
]
```

Use a current model ID from OpenAI's model docs or your own configuration. Avoid
hardcoding a model across your app; keep it in one place so you can update it
without changing every call site.

---

## Step 2: Replace `createAndStreamThreadRun`

In the Assistants API version, the first user message often looked like this:

```swift
let stream = try await GPTBridge.createAndStreamThreadRun(
    text: message,
    assistantId: activeAssistant.id
)
```

In Chat Completions, append the user message locally and stream the completion:

```swift
@MainActor
func sendMessageFromUser(_ text: String) async {
    let userDisplayMessage = ChatMessage(content: text, role: .user)
    messages.append(userDisplayMessage)

    completionMessages.append(
        ChatCompletionMessage(role: .user, content: text)
    )

    do {
        let stream = try await GPTBridge.streamChatCompletion(
            model: model,
            messages: completionMessages
        )
        try await handleCompletionStream(stream)
    } catch {
        let errorMessage = ChatMessage(
            content: "Something went wrong. Please try again.",
            role: .assistant
        )
        messages.append(errorMessage)
        print(error)
    }
}
```

Notice that there is no `threadId` and no `assistantId`. The conversation is the
`completionMessages` array you send with each request.

---

## Step 3: Handle Chat Completion Streaming

Assistants streaming uses `RunStatusEvent` values such as `.threadCreated`,
`.messageDelta`, and `.messageCompleted`.

Chat Completions streaming uses `ChatCompletionStreamEvent` values. For a basic
chat UI, handle `.contentDelta` and append the final assistant message when the
stream finishes:

```swift
private func handleCompletionStream(
    _ stream: AsyncThrowingStream<ChatCompletionStreamEvent, any Error>
) async throws {
    var finalText = ""
    var didAppendFinalMessage = false

    for try await event in stream {
        switch event {
        case .contentDelta(let text):
            finalText += text
            await MainActor.run {
                self.streamingText += text
            }

        case .done:
            await appendCompletedAssistantMessage(finalText)
            didAppendFinalMessage = true

        case .errorOccurred(let error):
            await MainActor.run {
                let message = ChatMessage(content: error.message, role: .assistant)
                self.messages.append(message)
            }

        default:
            break
        }
    }

    if !finalText.isEmpty,
       !didAppendFinalMessage {
        await appendCompletedAssistantMessage(finalText)
    }
}

@MainActor
private func appendCompletedAssistantMessage(_ text: String) {
    guard !text.isEmpty else { return }

    streamingText = ""
    messages.append(ChatMessage(content: text, role: .assistant))
    completionMessages.append(
        ChatCompletionMessage(role: .assistant, content: text)
    )
}
```

The last fallback handles streams that finish without an explicit `.done` event
being observed by your loop.

---

## Step 4: Replace Existing-Thread Logic

In an Assistants app, you probably branch on `threadId`:

```swift
if threadId == nil {
    try await createThreadAndRun(withUserMessage: message)
} else {
    try await addMessageToThread(message)
}
```

Delete that branch for Chat Completions. Your app always appends to the local
message array and sends the full conversation:

```swift
@MainActor
func send(_ text: String) async {
    messages.append(ChatMessage(content: text, role: .user))
    completionMessages.append(ChatCompletionMessage(role: .user, content: text))

    do {
        let response = try await GPTBridge.createChatCompletion(
            model: model,
            messages: completionMessages
        )

        guard let reply = response.choices.first?.message else { return }
        completionMessages.append(reply)

        if let content = reply.content {
            messages.append(ChatMessage(content: content, role: .assistant))
        }
    } catch {
        print(error)
    }
}
```

This non-streaming version is useful when you want the simplest migration first.
Once it works, switch back to `streamChatCompletion` for a better chat
experience.

---

## Step 5: Move Assistant Instructions Into the Message List

Assistant instructions used to live in the OpenAI dashboard. With Chat
Completions, make them explicit in your request:

```swift
private var completionMessages: [ChatCompletionMessage] = [
    ChatCompletionMessage(
        role: .developer,
        content: """
        You are a helpful assistant inside a SwiftUI app.
        Keep answers short unless the user asks for detail.
        """
    )
]
```

Use this message for behavior that applies to the whole session. User messages
come after it, and assistant responses are appended as they arrive.

---

## Step 6: Move Assistant Tools Into Swift

In the Assistants API version, tools were often configured on the assistant in
the OpenAI dashboard. With Chat Completions, pass tools with the request.

For example, define a weather tool:

```swift
let weatherTool = ChatCompletionTool(
    function: ChatCompletionFunctionDefinition(
        name: "get_current_weather",
        description: "Get the current weather in a given location.",
        parameters: [
            "type": FunctionArgument("object"),
            "properties": FunctionArgument([
                "location": FunctionArgument([
                    "type": FunctionArgument("string"),
                    "description": FunctionArgument("City and state, such as San Francisco, CA")
                ])
            ]),
            "required": FunctionArgument(["location"])
        ]
    )
)
```

Then send it with the request:

```swift
let response = try await GPTBridge.createChatCompletion(
    model: model,
    messages: completionMessages,
    tools: [weatherTool],
    toolChoice: .auto
)
```

If the model requests a tool call, GPTBridge decodes it into `toolCalls` on the
assistant message:

```swift
guard let assistantMessage = response.choices.first?.message else { return }
completionMessages.append(assistantMessage)

if let toolCall = assistantMessage.toolCalls?.first {
    let location = toolCall.function.arguments["location"]?.asString ?? ""
    let weatherSummary = try await getWeatherSummary(for: location)

    completionMessages.append(
        ChatCompletionMessage.toolOutput(
            toolCallId: toolCall.id,
            output: weatherSummary
        )
    )

    let followUp = try await GPTBridge.createChatCompletion(
        model: model,
        messages: completionMessages,
        tools: [weatherTool],
        toolChoice: .auto
    )

    if let finalMessage = followUp.choices.first?.message {
        completionMessages.append(finalMessage)
        messages.append(
            ChatMessage(content: finalMessage.content ?? "", role: .assistant)
        )
    }
}
```

Your app is still responsible for validating arguments before using them. Treat
model-generated tool arguments as input, not trusted internal state.

---

## Step 7: Migration Checklist

Use this checklist while converting an existing GPTBridge app:

- Replace `Assistant` selection with model selection or a model configuration.
- Replace assistant dashboard instructions with an initial `.developer` or
`.system` message.
- Replace `threadId` with a local `[ChatCompletionMessage]`.
- Replace `ChatMessage` API payloads with `ChatCompletionMessage` payloads.
- Keep `ChatMessage` only as a UI display type if your views already use it.
- Replace `createAndStreamThreadRun` with `streamChatCompletion`.
- Replace `RunStatusEvent.messageDelta` with `ChatCompletionStreamEvent.contentDelta`.
- Replace `RunStatusEvent.messageCompleted` with appending the final streamed
text after `.done` or after the stream loop exits.
- Replace `submitToolOutputs` with `ChatCompletionMessage.toolOutput` followed
by another `createChatCompletion` request.
- Remove assistant listing UI if your app no longer lets users pick assistants.

---

## Before and After

### Assistants API

```swift
let stream = try await GPTBridge.addMessageAndStreamThreadRun(
    text: text,
    threadId: threadId,
    assistantId: assistantId
)

for try await event in stream {
    if case .messageDelta(let token) = event {
        streamingText += token
    }
}
```

### Chat Completions API

```swift
completionMessages.append(ChatCompletionMessage(role: .user, content: text))

let stream = try await GPTBridge.streamChatCompletion(
    model: model,
    messages: completionMessages
)

var reply = ""
for try await event in stream {
    if case .contentDelta(let token) = event {
        reply += token
        streamingText += token
    }
}

completionMessages.append(ChatCompletionMessage(role: .assistant, content: reply))
messages.append(ChatMessage(content: reply, role: .assistant))
```

---

## Conclusion

Migrating to Chat Completions mostly changes where state lives. Assistants gave
you server-side assistants, threads, and runs. Chat Completions gives you a
direct model call where your app sends the current conversation each time.

For many Swift apps, that is easier to reason about:

- The prompt is visible in code.
- The transcript is local app state.
- Tools are versioned with the app.
- Streaming is token-focused instead of run-status-focused.

Keep the old Assistants tutorial nearby if you are maintaining an existing app,
but prefer this flow for new GPTBridge work.
