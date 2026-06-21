# Handoff to Human for Laravel AI SDK 🤝

![PHP](https://img.shields.io/badge/PHP-8.2+-777BB4?logo=php&logoColor=white)
![Laravel](https://img.shields.io/badge/Laravel-12.x-FF2D20?logo=laravel&logoColor=white)
![Laravel AI SDK](https://img.shields.io/badge/Laravel_AI_SDK-0.x-EE672F?logo=laravel&logoColor=white)

A minimal, copy-paste guide for adding AI-to-human handoff to an app that already uses the [Laravel AI SDK](https://laravel.com/docs/ai-sdk).

The handoff state lives directly on the SDK's `agent_conversations` table. No separate tickets, no duplicate history — the same chat thread keeps going, first with the AI, then with a human, then back to the AI.

## Lifecycle 🔄

```
ai_active → pending_human → human_active → resolved
```

| Status | Meaning |
| --- | --- |
| `ai_active` | The AI replies normally. |
| `pending_human` | A human has been requested; user messages are stored but not sent to the AI. |
| `human_active` | A human is replying in the same thread. |
| `resolved` | Handoff is closed; the AI can take over again. |

## Prerequisites 📦

You have already installed and configured the Laravel AI SDK, published its migrations, and run them:

```bash
composer require laravel/ai
php artisan vendor:publish --provider="Laravel\Ai\AiServiceProvider"
php artisan migrate
```

Your agent uses the `RemembersConversations` trait so it has `forUser()`, `continue()`, and `currentConversation()`.

## Step 1: Migration 🗄️

Add the handoff columns to the existing conversations table.

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        $table = config('ai.conversations.tables.conversations', 'agent_conversations');

        Schema::table($table, function (Blueprint $table) {
            $table->string('handoff_status', 20)->default('ai_active')->index();
            $table->text('handoff_reason')->nullable();
            $table->unsignedBigInteger('assigned_to')->nullable();
            $table->timestamp('handoff_requested_at')->nullable();
            $table->timestamp('assigned_at')->nullable();
            $table->timestamp('resolved_at')->nullable();
        });
    }

    public function down(): void
    {
        $table = config('ai.conversations.tables.conversations', 'agent_conversations');

        Schema::table($table, function (Blueprint $table) {
            $table->dropColumn([
                'handoff_status',
                'handoff_reason',
                'assigned_to',
                'handoff_requested_at',
                'assigned_at',
                'resolved_at',
            ]);
        });
    }
};
```

## Step 2: The handoff tool 🛠️

Create `app/Ai/Tools/HandoffToHuman.php`.

```php
<?php

namespace App\Ai\Tools;

use Illuminate\Contracts\JsonSchema\JsonSchema;
use Laravel\Ai\Contracts\Tool;
use Laravel\Ai\Models\Conversation;
use Laravel\Ai\Tools\Request;
use Stringable;

class HandoffToHuman implements Tool
{
    public function __construct(protected ?string $conversationId) {}

    public function description(): Stringable|string
    {
        return 'Use this when the user needs a human — for example, when they are upset, '
            .'explicitly ask for a person, or the request is outside what you can help with. '
            .'A human will join the same chat thread; do not tell the user to email anyone.';
    }

    public function handle(Request $request): Stringable|string
    {
        if (! $this->conversationId) {
            return 'Unable to hand off: this conversation has not been saved yet. '
                .'Tell the user you are looking into it, then try again once the conversation exists.';
        }

        Conversation::where('id', $this->conversationId)->update([
            'handoff_status' => 'pending_human',
            'handoff_reason' => (string) $request['reason'],
            'handoff_requested_at' => now(),
        ]);

        return 'A human has been notified and will join this chat shortly. '
            .'Tell the user help is on the way. Do not say the issue is resolved.';
    }

    public function schema(JsonSchema $schema): array
    {
        return [
            'reason' => $schema->string()->required(),
        ];
    }
}
```

## Step 3: Wire the tool into your agent

Add the tool to your agent's `tools()` method, passing the current conversation ID from `RemembersConversations`.

```php
use App\Ai\Tools\HandoffToHuman;

public function tools(): iterable
{
    return [
        new HandoffToHuman($this->currentConversation()),
        // ... other tools
    ];
}
```

## Step 4: Route user messages 🚦

In your chat endpoint, check the handoff status before prompting the AI.

```php
use App\Ai\Agents\SupportAgent;
use Illuminate\Http\Request;
use Illuminate\Support\Str;
use Laravel\Ai\Models\Conversation;

Route::post('/chat', function (Request $request) {
    $agent = new SupportAgent;
    $user = $request->user();
    $message = $request->string('message');
    $conversationId = $request->input('conversation_id');

    $handoffStatus = Conversation::where('id', $conversationId)->value('handoff_status');

    if (in_array($handoffStatus, ['pending_human', 'human_active'])) {
        $conversation = Conversation::findOrFail($conversationId);

        $conversation->messages()->create([
            'id' => (string) Str::uuid7(),
            'user_id' => $user->id,
            'agent' => SupportAgent::class,
            'role' => 'user',
            'content' => $message,
            'attachments' => [],
            'tool_calls' => [],
            'tool_results' => [],
            'usage' => [],
            'meta' => [],
        ]);

        $conversation->touch();

        return [
            'conversation_id' => $conversationId,
            'awaiting_human' => true,
        ];
    }

    $response = $conversationId
        ? $agent->continue($conversationId, as: $user)->prompt($message)
        : $agent->forUser($user)->prompt($message);

    return [
        'conversation_id' => $response->conversationId,
        'message' => $response->text,
        'awaiting_human' => false,
    ];
});
```

## Step 5: Human replies

A human picks up the conversation, replies in the same thread, and resolves it when done.

```php
use Illuminate\Support\Str;
use Laravel\Ai\Models\Conversation;

// Pick up
Conversation::where('id', $conversationId)->update([
    'handoff_status' => 'human_active',
    'assigned_to' => $staff->id,
    'assigned_at' => now(),
]);

// Reply in the same thread
$conversation = Conversation::findOrFail($conversationId);

$conversation->messages()->create([
    'id' => (string) Str::uuid7(),
    'user_id' => $conversation->user_id,
    'agent' => 'human-agent',
    'role' => 'assistant',
    'content' => $reply,
    'attachments' => [],
    'tool_calls' => [],
    'tool_results' => [],
    'usage' => [],
    'meta' => [
        'source' => 'human',
        'handled_by' => $staff->id,
    ],
]);

$conversation->touch();

// Hand back to the AI
Conversation::where('id', $conversationId)->update([
    'handoff_status' => 'resolved',
    'resolved_at' => now(),
]);
```

## Step 6: Staff inbox

Query conversations that are waiting for a human.

```php
$pending = Conversation::where('handoff_status', 'pending_human')
    ->latest('updated_at')
    ->get();
```

## Testing

Use the SDK's `Agent::fake()` in a controller test to make sure messages after handoff do not reach the AI.

```php
use App\Ai\Agents\SupportAgent;
use Laravel\Ai\Models\Conversation;
use Laravel\Ai\Models\ConversationMessage;

it('does not prompt the AI while waiting for a human', function () {
    SupportAgent::fake(['I will get a human for you.']);

    $user = User::factory()->create();

    $start = $this->actingAs($user)->postJson('/chat', ['message' => 'I need a human']);
    $conversationId = $start->json('conversation_id');

    Conversation::where('id', $conversationId)->update([
        'handoff_status' => 'pending_human',
    ]);

    $this->actingAs($user)->postJson('/chat', [
        'conversation_id' => $conversationId,
        'message' => 'Are you still there?',
    ])
        ->assertOk()
        ->assertJsonPath('awaiting_human', true);

    // Only the first fake AI response exists
    expect(ConversationMessage::where('role', 'assistant')->count())->toBe(1);
});
```

## Summary ✅

| Piece | Lines of code |
| --- | --- |
| Migration | ~30 |
| `HandoffToHuman` tool | ~55 |
| Chat endpoint guard | ~25 |
| Human reply logic | ~20 |
| **Total** | **~130** |

No package required. The whole feature is a migration, one tool, and two small routing rules inside the same chat thread the Laravel AI SDK already gives you.
