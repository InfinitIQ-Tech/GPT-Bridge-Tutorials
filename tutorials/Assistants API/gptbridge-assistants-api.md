# Using GPTBridge With the Assistants API

## Introduction

This tutorial shows how to use GPTBridge's original
[Assistants API](https://developers.openai.com/api/reference/resources/beta/subresources/assistants/methods/create)
interface in a Swift app. You will list assistants from your OpenAI account,
send messages to an assistant, stream replies, and handle tool calls.

Use this tutorial when you are maintaining an existing GPTBridge app that already
depends on assistant IDs, threads, runs, or assistant dashboard tools.

> `Note`: OpenAI's API reference currently marks Assistants endpoints as
> deprecated, and GPTBridge now marks its Assistants API methods as deprecated in
> favor of
> [Chat Completions](https://developers.openai.com/api/reference/resources/chat).
> The API surface in this tutorial still works for existing GPTBridge
> integrations, but new app code should consider the Chat Completions tutorial
> first.

By the end, you will know how to:

- Configure GPTBridge with an OpenAI API key.
- List assistants.
- Create a thread.
- Add messages to a thread.
- Create and poll a run.
- Stream assistant responses.
- Submit tool outputs when a run requires action.

---

## Prerequisites

You should have:

1. A SwiftUI app targeting iOS 15 or macOS 12 or later.
2. GPTBridge added as a Swift Package dependency.
3. An OpenAI API key.
4. At least one assistant created in the OpenAI dashboard.

The examples below assume your app imports GPTBridge:

```swift
import GPTBridge
```

---

## Step 1: Configure GPTBridge

Call `GPTBridge.appLaunch(openAIAPIKey:)` before making GPTBridge requests:

```swift
GPTBridge.appLaunch(openAIAPIKey: "sk-your-key")
```

For a tutorial app, you can call this from your app's startup path or from a
SwiftUI `.task` while you are experimenting.

For production apps, do not hardcode API keys in the app bundle. Route requests
through your own backend or use another secure key-management approach.

---

## Step 2: List Assistants

GPTBridge can list assistants attached to your OpenAI account:

```swift
@MainActor
final class AssistantListViewModel: ObservableObject {
    @Published var assistants: [Assistant] = []
    @Published var errorMessage: String?

    func loadAssistants() async {
        do {
            let response = try await GPTBridge.listAssistants()
            assistants = response.data
        } catch {
            errorMessage = String(describing: error)
        }
    }
}
```

A simple SwiftUI list can display the assistants:

```swift
struct AssistantListView: View {
    @StateObject private var viewModel = AssistantListViewModel()

    var body: some View {
        List(viewModel.assistants, id: \.id) { assistant in
            Text(assistant.name ?? assistant.id)
        }
        .task {
            GPTBridge.appLaunch(openAIAPIKey: "sk-your-key")
            await viewModel.loadAssistants()
        }
    }
}
```

If your account has many assistants, use `PaginatedRequestParameters`:

```swift
let response = try await GPTBridge.listAssistants(
    paginatedBy: PaginatedRequestParameters(
        limit: 10,
        order: .descending
    )
)
```

---

## Step 3: Understand Threads, Messages, and Runs

The Assistants API version has three core resources:

- **Thread:** The conversation container.
- **Message:** A user or assistant message inside a thread.
- **Run:** The assistant processing a thread and deciding whether to reply or
call tools.

GPTBridge exposes each part directly:

```swift
let threadId = try await GPTBridge.createThread()

try await GPTBridge.addMessageToThread(
    message: "Give me a one-sentence app idea.",
    threadId: threadId,
    role: .user
)

let runId = try await GPTBridge.createRun(
    threadId: threadId,
    assistantId: assistant.id
)

let result = try await GPTBridge.pollRunStatus(
    threadId: threadId,
    runId: runId
)
```

`pollRunStatus` returns a `RunStepResult`. If the assistant generated a normal
message, `result.message` contains text. If the assistant requested a tool,
`result.functions` contains function calls to handle.

---

## Step 4: Build a Non-Streaming Chat Method

For the simplest implementation, send a message, wait for the run to finish, and
append the result to your UI:

```swift
@MainActor
final class AssistantChatViewModel: ObservableObject {
    @Published var messages: [ChatMessage] = []
    @Published var errorMessage: String?

    let assistant: Assistant
    private var threadId: String?

    init(assistant: Assistant) {
        self.assistant = assistant
    }

    func send(_ text: String) async {
        messages.append(ChatMessage(content: text, role: .user))

        do {
            let threadId = try await getOrCreateThreadId()

            try await GPTBridge.addMessageToThread(
                message: text,
                threadId: threadId,
                role: .user
            )

            let runId = try await GPTBridge.createRun(
                threadId: threadId,
                assistantId: assistant.id
            )

            let result = try await GPTBridge.pollRunStatus(
                threadId: threadId,
                runId: runId
            )

            if let message = result.message {
                messages.append(ChatMessage(content: message, role: .assistant))
            } else if let functions = result.functions {
                try await handle(functions: functions, threadId: threadId, runId: runId)
            }
        } catch {
            errorMessage = String(describing: error)
        }
    }

    private func getOrCreateThreadId() async throws -> String {
        if let threadId {
            return threadId
        }

        let newThreadId = try await GPTBridge.createThread()
        threadId = newThreadId
        return newThreadId
    }

    private func handle(
        functions: [AssistantFunction],
        threadId: String,
        runId: String
    ) async throws {
        guard let function = functions.first else { return }
        print("Assistant requested tool: \(function.name)")
    }
}
```

This version is easy to debug because each API step is explicit. Once it works,
you can move to streaming.

---

## Step 5: Stream the First Assistant Reply

GPTBridge includes a helper that creates a thread, sends the first message, and
streams the run:

```swift
let stream = try await GPTBridge.createAndStreamThreadRun(
    text: text,
    assistantId: assistant.id
)
```

The stream emits `RunStatusEvent` values:

```swift
private func handleRunStream(
    _ stream: AsyncThrowingStream<RunStatusEvent, any Error>
) async throws {
    for try await event in stream {
        switch event {
        case .threadCreated(let threadId):
            self.threadId = threadId

        case .messageDelta(let text):
            streamingText += text

        case .messageCompleted(let message):
            streamingText = ""
            messages.append(message)

        case .errorOccurred(let error):
            errorMessage = error.message

        case .done:
            break

        default:
            break
        }
    }
}
```

Add the required published state:

```swift
@Published var streamingText: String = ""
@Published var errorMessage: String?
```

Now send the first message:

```swift
@MainActor
func sendFirstMessage(_ text: String) async {
    messages.append(ChatMessage(content: text, role: .user))

    do {
        let stream = try await GPTBridge.createAndStreamThreadRun(
            text: text,
            assistantId: assistant.id
        )
        try await handleRunStream(stream)
    } catch {
        errorMessage = String(describing: error)
    }
}
```

---

## Step 6: Stream Follow-Up Messages in the Same Thread

After the first stream emits `.threadCreated`, store the `threadId`. Use that ID
for follow-up messages:

```swift
@MainActor
func sendFollowUpMessage(_ text: String) async {
    guard let threadId else {
        await sendFirstMessage(text)
        return
    }

    messages.append(ChatMessage(content: text, role: .user))

    do {
        let stream = try await GPTBridge.addMessageAndStreamThreadRun(
            text: text,
            threadId: threadId,
            assistantId: assistant.id
        )
        try await handleRunStream(stream)
    } catch {
        errorMessage = String(describing: error)
    }
}
```

This keeps the conversation in one OpenAI thread, so the assistant can respond
with previous messages in context.

---

## Step 7: Display Streaming Text in SwiftUI

If your chat view already displays `[ChatMessage]`, add one extra bubble while
streaming text is non-empty:

```swift
LazyVStack {
    ForEach(viewModel.messages) { message in
        MessageBubbleView(message: message)
    }

    if !viewModel.streamingText.isEmpty {
        MessageBubbleView(
            message: ChatMessage(
                content: viewModel.streamingText,
                role: .assistant
            )
        )
    }
}
```

This gives users immediate feedback while the assistant is generating a reply.

---

## Step 8: Handle Tool Calls

When a run requires action, `pollRunStatus` returns a `FunctionRunStepResult`.
The result includes the requested functions and the tool-call ID.

For non-streaming flows:

```swift
let result = try await GPTBridge.pollRunStatus(
    threadId: threadId,
    runId: runId
)

if let functionResult = result as? FunctionRunStepResult,
   let function = functionResult.functions?.first {
    let output = try await performTool(function)

    let nextResult = try await GPTBridge.submitToolOutputs(
        threadId: threadId,
        runId: runId,
        toolCallOutputs: [
            ToolCallOutput(
                toolCallId: functionResult.toolCallId,
                output: output
            )
        ]
    )

    if let message = nextResult.message {
        messages.append(ChatMessage(content: message, role: .assistant))
    }
}
```

Your app decides how to execute the function:

```swift
private func performTool(_ function: AssistantFunction) async throws -> String {
    switch function.name {
    case "get_current_weather":
        let location = function.arguments["location"]?.asString ?? ""
        return "The weather in \(location) is sunny."

    default:
        return "Unsupported tool: \(function.name)"
    }
}
```

Always validate function arguments before using them. They are generated by the
model and should be treated as external input.

---

## Step 9: Handle Errors

For tutorial apps, logging errors is enough:

```swift
catch {
    print(error)
}
```

For user-facing apps, store an error message and show it in the UI:

```swift
@Published var errorMessage: String?

private func showError(_ error: Error) {
    errorMessage = "Something went wrong. Please try again."
    print(error)
}
```

During streaming, `RunStatusEvent.errorOccurred` exposes OpenAI error payloads:

```swift
case .errorOccurred(let error):
    errorMessage = error.message
```

---

## Full Streaming ViewModel

Here is a compact version of the streaming pattern:

```swift
@MainActor
final class AssistantChatViewModel: ObservableObject {
    @Published var messages: [ChatMessage] = []
    @Published var streamingText: String = ""
    @Published var errorMessage: String?

    let assistant: Assistant
    private var threadId: String?

    init(assistant: Assistant) {
        self.assistant = assistant
    }

    func send(_ text: String) async {
        messages.append(ChatMessage(content: text, role: .user))

        do {
            let stream: AsyncThrowingStream<RunStatusEvent, any Error>

            if let threadId {
                stream = try await GPTBridge.addMessageAndStreamThreadRun(
                    text: text,
                    threadId: threadId,
                    assistantId: assistant.id
                )
            } else {
                stream = try await GPTBridge.createAndStreamThreadRun(
                    text: text,
                    assistantId: assistant.id
                )
            }

            try await handle(stream)
        } catch {
            errorMessage = String(describing: error)
        }
    }

    private func handle(
        _ stream: AsyncThrowingStream<RunStatusEvent, any Error>
    ) async throws {
        for try await event in stream {
            switch event {
            case .threadCreated(let threadId):
                self.threadId = threadId

            case .messageDelta(let text):
                streamingText += text

            case .messageCompleted(let message):
                streamingText = ""
                messages.append(message)

            case .errorOccurred(let error):
                errorMessage = error.message

            default:
                break
            }
        }
    }
}
```

---

## Next Steps

Use this Assistants API version when you need server-side assistant
configuration, existing assistant IDs, or existing thread/run behavior.

For new GPTBridge apps, review the Chat Completions migration tutorial. It shows
how to replace assistant threads with local message history and how to move tool
definitions into Swift code.

---

## Conclusion

The Assistants API version of GPTBridge gives you a structured thread/run model:

- `listAssistants` discovers assistants.
- `createThread` starts a conversation.
- `addMessageToThread` adds user input.
- `createRun` asks an assistant to process the thread.
- `pollRunStatus` waits for a final message or tool call.
- `createAndStreamThreadRun` and `addMessageAndStreamThreadRun` stream replies.
- `submitToolOutputs` sends tool results back to the run.

That model is still useful for maintaining existing apps. For new work, prefer
the Chat Completions tutorial so your app owns its prompt, transcript, and tools
directly.
