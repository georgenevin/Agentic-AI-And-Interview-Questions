# Direct Line v3 Flow (Web Client → Azure Bot)

A summary of how a web frontend talks to a Microsoft Bot Framework bot through Direct Line v3 with a WebSocket stream.

## Flow at a glance

```
Browser --HTTP POST activity--> Direct Line --POST /api/messages--> Bot
Browser <----WebSocket (streamUrl)---- Direct Line <--reply------- Bot
```

## Step by step

### 1. Get a token (ideally from your backend)

The frontend should not hold the Direct Line secret. Your backend calls:

```
POST https://directline.botframework.com/v3/directline/tokens/generate
Authorization: Bearer <DIRECT_LINE_SECRET>
```

The secret comes from the Direct Line channel of the Azure Bot. The backend returns the short-lived token to the web client.

### 2. Start the conversation

The client calls:

```
POST https://directline.botframework.com/v3/directline/conversations
Authorization: Bearer <token>
```

The response contains:

| Field | Purpose |
|---|---|
| `conversationId` | Identifies the conversation |
| `token` | Token for subsequent Direct Line calls |
| `expires_in` | Token lifetime in seconds |
| `streamUrl` | WebSocket URL for receiving activities |

### 3. Open the WebSocket using `streamUrl`

- The `streamUrl` is already authorized, so the token is **not** sent again over the socket.
- The WebSocket is **receive-only**. It carries activities from the bot to the client (messages, typing indicators, events), not the other way.

### 4. Send messages over HTTP

Each user message is sent with:

```
POST /v3/directline/conversations/{conversationId}/activities
Authorization: Bearer <token>
```

### 5. Direct Line forwards to your bot

Direct Line forwards the activity as an HTTP POST to the bot's **messaging endpoint**, for example:

```
https://yourbot.azurewebsites.net/api/messages
```

- The request carries a JWT signed by the Bot Framework.
- The bot's adapter validates it using the bot's App ID and credentials.
- The client's token is only for talking to Direct Line. The bot never sees it.
- The Azure Bot registration is what links Direct Line to your endpoint.

### 6. Reply path

The bot sends its response back to Direct Line (via the `serviceUrl` in the incoming activity), and Direct Line pushes it to the client over the WebSocket.

## Inside the bot: from HTTP request to TurnContext

When Direct Line POSTs to `/api/messages`, the request goes through these stages:

```
HTTP POST /api/messages
   │
   ▼
Controller / route handler
   │  adapter.ProcessAsync(request, response, bot)
   ▼
CloudAdapter
   │  1. Validate JWT (Authorization header) via BotFrameworkAuthentication
   │  2. Deserialize request body into an Activity
   │  3. Create TurnContext (activity + adapter + turn state)
   ▼
Middleware pipeline (logging, transcript, custom middleware ...)
   ▼
bot.OnTurnAsync(turnContext)
```

### 1. Entry point: the messaging endpoint

**C# (ASP.NET Core)**

```csharp
[Route("api/messages")]
[ApiController]
public class BotController : ControllerBase
{
    private readonly IBotFrameworkHttpAdapter _adapter;
    private readonly IBot _bot;

    public BotController(IBotFrameworkHttpAdapter adapter, IBot bot)
    {
        _adapter = adapter;
        _bot = bot;
    }

    [HttpPost, HttpGet]
    public async Task PostAsync()
    {
        // Hands the raw HTTP request to the adapter
        await _adapter.ProcessAsync(Request, Response, _bot);
    }
}
```

**JavaScript (Restify)**

```javascript
server.post('/api/messages', async (req, res) => {
    await adapter.process(req, res, (context) => bot.run(context));
});
```

### 2. What the adapter does

1. **Authenticates** the request by validating the JWT in the `Authorization` header against the bot's App ID and credentials. An invalid token is rejected with 401.
2. **Deserializes** the JSON body into an `Activity` object (type, text, from, recipient, conversation, channelId, serviceUrl, and so on).
3. **Builds the `TurnContext`**, which wraps:
   - the incoming `Activity` (`turnContext.Activity`)
   - a reference to the adapter
   - a **turn state** collection holding per-turn services such as the `ConnectorClient` (used to send replies to `serviceUrl`), the bot identity, and the user token client
   - hooks for `SendActivityAsync`, `UpdateActivityAsync`, and `DeleteActivityAsync`
4. **Runs the middleware pipeline**, then calls `bot.OnTurnAsync(turnContext, cancellationToken)`.

The `TurnContext` lives for **one turn** (one incoming activity). Anything that must persist between turns has to go into state (see below).

### 3. `ActivityHandler` routes by activity type

`ActivityHandler.OnTurnAsync` inspects `turnContext.Activity.Type` and calls the matching method:

| Activity type | Handler method |
|---|---|
| `message` | `OnMessageActivityAsync` |
| `conversationUpdate` | `OnConversationUpdateActivityAsync` → `OnMembersAddedAsync` / `OnMembersRemovedAsync` |
| `event` | `OnEventActivityAsync` |
| `invoke` | `OnInvokeActivityAsync` |
| `typing`, `endOfConversation`, etc. | their own overrides |

## How the message reaches the main dialog

A common pattern is a `DialogBot` base class that forwards every incoming message to a root ("main") dialog.

### The turn, step by step

```
OnMessageActivityAsync(turnContext)
   │
   ▼
MainDialog.RunAsync(turnContext, dialogStateAccessor)
   │  1. Load DialogState from ConversationState
   │  2. Create a DialogSet and add the dialog(s)
   │  3. Create a DialogContext from the DialogSet + TurnContext
   │  4. ContinueDialogAsync: resume the active dialog if one exists
   │  5. If nothing was active (status Empty), BeginDialogAsync(MainDialog.Id)
   ▼
Waterfall step / prompt runs, uses stepContext.Context (the TurnContext)
   │
   ▼
ConversationState.SaveChangesAsync(turnContext)   ← persists the dialog stack
```

### C# example

```csharp
public class DialogBot<T> : ActivityHandler where T : Dialog
{
    protected readonly Dialog Dialog;
    protected readonly BotState ConversationState;
    protected readonly BotState UserState;

    public DialogBot(ConversationState conversationState,
                     UserState userState, T dialog)
    {
        ConversationState = conversationState;
        UserState = userState;
        Dialog = dialog;
    }

    // Runs at the end of every turn: save state so the dialog stack persists
    public override async Task OnTurnAsync(ITurnContext turnContext,
                                           CancellationToken cancellationToken = default)
    {
        await base.OnTurnAsync(turnContext, cancellationToken);
        await ConversationState.SaveChangesAsync(turnContext, false, cancellationToken);
        await UserState.SaveChangesAsync(turnContext, false, cancellationToken);
    }

    protected override async Task OnMessageActivityAsync(
        ITurnContext<IMessageActivity> turnContext,
        CancellationToken cancellationToken)
    {
        // Hand the turn to the main dialog
        await Dialog.RunAsync(
            turnContext,
            ConversationState.CreateProperty<DialogState>("DialogState"),
            cancellationToken);
    }
}
```

### JavaScript example

```javascript
class DialogBot extends ActivityHandler {
    constructor(conversationState, userState, dialog) {
        super();
        this.conversationState = conversationState;
        this.userState = userState;
        this.dialog = dialog;
        this.dialogState = conversationState.createProperty('DialogState');

        this.onMessage(async (context, next) => {
            // Hand the turn to the main dialog
            await this.dialog.run(context, this.dialogState);
            await next();
        });
    }

    async run(context) {
        await super.run(context);
        // Save state at the end of every turn
        await this.conversationState.saveChanges(context, false);
        await this.userState.saveChanges(context, false);
    }
}
```

### What `RunAsync` does internally

The `RunAsync` extension performs the equivalent of:

```csharp
var dialogSet = new DialogSet(accessor);
dialogSet.Add(mainDialog);

var dialogContext = await dialogSet.CreateContextAsync(turnContext, ct);
var results = await dialogContext.ContinueDialogAsync(ct);

if (results.Status == DialogTurnStatus.Empty)
{
    await dialogContext.BeginDialogAsync(mainDialog.Id, null, ct);
}
```

- **First message:** the dialog stack is empty, so `BeginDialogAsync` starts the main dialog and its first waterfall step runs.
- **Later messages:** the stack was restored from `ConversationState`, so `ContinueDialogAsync` resumes the active step or prompt, which receives the user's reply through `turnContext.Activity.Text`.

### Inside a waterfall step

Each step receives a `WaterfallStepContext`, and the original `TurnContext` is available as `stepContext.Context`:

```csharp
private async Task<DialogTurnResult> AskNameStepAsync(
    WaterfallStepContext stepContext, CancellationToken ct)
{
    // stepContext.Context is the TurnContext for this turn
    var userText = stepContext.Context.Activity.Text;

    return await stepContext.PromptAsync(
        nameof(TextPrompt),
        new PromptOptions { Prompt = MessageFactory.Text("What is your name?") },
        ct);
}

private async Task<DialogTurnResult> GreetStepAsync(
    WaterfallStepContext stepContext, CancellationToken ct)
{
    var name = (string)stepContext.Result;   // the prompt's result
    await stepContext.Context.SendActivityAsync($"Hello {name}!", cancellationToken: ct);
    return await stepContext.EndDialogAsync(null, ct);
}
```

Calling `SendActivityAsync` uses the `ConnectorClient` in the turn state to post the reply to the `serviceUrl`. Direct Line then delivers it to the browser over the WebSocket.

## Advantages of Direct Line over a custom channel

Building your own chat transport (custom REST API + WebSocket server that calls the bot) is possible, but Direct Line gives you a lot for free.

### 1. Security is handled for you
- **Secret/token model:** the secret stays on your backend and the browser only gets a short-lived, conversation-scoped token that can be refreshed.
- **Bot-side authentication:** Direct Line signs the calls to your messaging endpoint with a JWT that the adapter validates. A custom channel would need its own scheme, key management, and validation.
- **Enhanced authentication options** (user ID validation via trusted origins) help prevent user impersonation.

### 2. No transport infrastructure to build or scale
- Direct Line is a hosted, managed service on Azure. You don't run or scale your own WebSocket servers, load balancers, or sticky sessions.
- **Reconnection and message replay** are built in: the `watermark` mechanism lets a client reconnect after a network drop without losing messages.
- Ordering and delivery semantics of activities are already defined.

### 3. Standard Activity protocol
- Messages, typing indicators, events, suggested actions, attachments, and Adaptive Cards all use the standard **Activity schema**, so the same bot code works unchanged across channels.
- With a custom channel you'd have to design and maintain your own message format and translate it to and from `Activity`.

### 4. Ready-made client components
- **Web Chat** (`botframework-webchat`) is a fully featured, customizable, accessible chat UI that speaks Direct Line out of the box, with support for Adaptive Cards, markdown, file uploads, speech, and theming.
- **Direct Line client libraries** (`botframework-directlinejs`) and REST APIs are available for custom UIs and other platforms.
- Building an equivalent UI and protocol client yourself is significant effort.

### 5. Channel independence
- The bot is registered once in Azure Bot Service. Adding Teams, Slack, Telegram, Facebook, and others is configuration, not new code, and Direct Line is just one more channel.
- A custom channel ties you to your own protocol and needs a bespoke adapter for every new client.

### 6. Tooling and ecosystem integration
- Works with the **Bot Framework Emulator** and the Azure portal's **Test in Web Chat**.
- Integrates with **Application Insights** telemetry, transcript logging middleware, and **OAuth / SSO** through the Bot Framework token service (`OAuthPrompt`, sign-in cards).
- **Direct Line Speech** offers a path to voice without building speech plumbing.

### 7. Less code to maintain
- No custom auth, session store, reconnect logic, or protocol versioning.
- Microsoft maintains the service, SDKs, and security patches.

### When a custom channel might still make sense

| Situation | Why |
|---|---|
| Strict data-residency or network isolation needs | Direct Line traffic goes through Microsoft's public service. Private-network options exist, but a custom design may give tighter control. |
| Very specialized protocol needs | Non-chat real-time use cases or custom binary formats. |
| Avoiding a dependency on Azure Bot Service | A fully self-hosted stack, at the cost of building everything above. |
| Extreme latency or throughput requirements | A purpose-built transport may be tuned better, though it is rarely worth the effort. |

For most web chat scenarios, Direct Line is the faster, safer, and more maintainable choice.

## Reconnection and token refresh

- **Dropped WebSocket:** call `GET /v3/directline/conversations/{id}?watermark=...` to get a fresh `streamUrl` and reconnect without losing messages.
- **Token expiry:** tokens last about 30 minutes. Refresh them with `POST /v3/directline/tokens/refresh` before they expire.

## Notes

- This is the standard **Direct Line v3** flow.
- **Direct Line Streaming** and **Direct Line Speech** work differently.
