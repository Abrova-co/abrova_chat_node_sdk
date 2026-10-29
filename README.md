# AbrovaChat Node.js SDK

Server-side SDK for AbrovaChat. Manage chat rooms, messages, reactions, pins, webhooks, and push notification templates from your Node.js backend.

> **Server-side only.** This SDK requires a server secret and must never be used in client-side applications. The server secret can be extracted from client apps.

## Features

- Room management (create, find, list, add participants)
- Message operations (send, edit, delete, search, mentions)
- Read receipts and unread counts
- Emoji reactions
- Message pinning
- Webhook configuration
- Push notification templates (via Abrova Push)
- Realtime connection token generation
- Zero runtime dependencies — uses native Node.js `http`/`https`
- TypeScript declarations included

## Installation

The package is published on [Abrova Center](https://center.abrova.ir). Add this line to your project's `.npmrc`:

```ini
@abrova:registry=https://center.abrova.ir/api/packages/abrova/npm/
```

Then install:

```bash
npm install @abrova/chat
```

## Quick Start

```typescript
import { AbrovaChat } from '@abrova/chat';

const abrovaChat = new AbrovaChat({
  apiKey: 'YOUR_API_KEY',
  serverSecret: 'YOUR_SERVER_SECRET',
});
```

### Create a Room

```typescript
const { room, created } = await abrovaChat.rooms.findOrCreate({
  externalId: 'order-123',
  participants: [
    { externalUserId: 'user-1', displayName: 'Alice', role: 'member' },
    { externalUserId: 'user-2', displayName: 'Bob', role: 'member' },
  ],
});

if (created) {
  // first time this room existed — send a welcome, seed state, and so on
}
```

### Send a Message

```typescript
const message = await abrovaChat.messages.send(room.id, {
  senderUserId: 'user-1',
  content: 'Hello from the server!',
});
```

### Get Message History

```typescript
const { messages, hasMore } = await abrovaChat.messages.getHistory(room.id, {
  userId: 'user-1',
  limit: 50,
});
```

### Mark as Read

```typescript
await abrovaChat.messages.markAsRead(room.id, 'user-1');
```

### Unread Counts

```typescript
const summary = await abrovaChat.rooms.getUnreadSummary('user-1');
// { totalUnread: 5, rooms: [{ roomId, externalId, unreadCount }] }
```

## API Reference

### Users — verified identity (recommended)

Your API key ships inside every app, so it cannot prove who a user is. Mint a
short-lived token here and hand it to the client SDK's `tokenProvider`:

```typescript
// Express example — your own login protects the route
app.get('/chat-token', requireLogin, async (req, res) => {
  const { token } = await abrovaChat.users.createToken({ userId: req.user.id });
  res.json({ token });
});
```

```typescript
abrovaChat.users.createToken({ userId, info?, ttl? })  // 1 h default, 24 h max
abrovaChat.users.revokeTokens(userId)                  // on logout, password change, ban
abrovaChat.users.getNotificationSettings(userId)                // App-wide push settings
abrovaChat.users.updateNotificationSettings(userId, options)    // Turn chat pushes off, mentions only, mute
```

`revokeTokens` invalidates every token issued to that user so far; it takes
effect within about 15 seconds across servers. Client SDKs then ask your
endpoint for a new token, so a signed-out user gets nothing.

### Rooms

```typescript
abrovaChat.rooms.create(options)                    // Create a room
abrovaChat.rooms.findOrCreate(options)               // -> { room, created }
abrovaChat.rooms.get(roomId)                         // Get room by ID
abrovaChat.rooms.getByExternalId(externalId)          // Get room by external ID
abrovaChat.rooms.list(userId)                        // List rooms for a user
abrovaChat.rooms.addParticipant(roomId, options)      // Add participant to room
abrovaChat.rooms.removeParticipant(roomId, userId)    // Remove participant (idempotent)
abrovaChat.rooms.delete(roomId)                       // Permanently delete a room
abrovaChat.rooms.getUnreadSummary(userId)             // Get unread counts
abrovaChat.rooms.getNotificationSettings(roomId, userId)             // A user's push settings for a room
abrovaChat.rooms.updateNotificationSettings(roomId, userId, options) // Mute a room or mentions only
```

### Messages

```typescript
abrovaChat.messages.send(roomId, options)             // Send a message
abrovaChat.messages.getHistory(roomId, options)        // Get paginated history
abrovaChat.messages.markAsRead(roomId, userId)         // Mark messages as read
abrovaChat.messages.edit(messageId, userId, content)   // Edit a message
abrovaChat.messages.delete(messageId, userId)          // Delete a message
abrovaChat.messages.search(roomId, options)            // Search messages
abrovaChat.messages.getMentions(roomId, options)        // Get @mentions
```

### Reactions

```typescript
abrovaChat.reactions.add(messageId, userId, emoji)     // Add reaction
abrovaChat.reactions.remove(messageId, userId, emoji)   // Remove reaction
abrovaChat.reactions.list(messageId)                    // List reactions
```

### Pins

```typescript
abrovaChat.pins.pin(messageId, userId)                 // Pin a message
abrovaChat.pins.unpin(messageId, userId)                // Unpin a message
abrovaChat.pins.list(roomId)                           // List pinned messages
```

### Webhooks

```typescript
abrovaChat.webhooks.get()                              // Get webhook config
abrovaChat.webhooks.setWebhook({ url, secret? })        // Set webhook URL
abrovaChat.webhooks.removeWebhook()                     // Remove webhook
abrovaChat.webhooks.setPushTemplate(templateId)          // Set push template
abrovaChat.webhooks.removePushTemplate()                 // Remove push template
abrovaChat.webhooks.setPushTemplates(templates)           // Per-event push templates
abrovaChat.webhooks.removePushTemplates()                 // Remove per-event templates
```

### Tokens

```typescript
abrovaChat.tokens.getConnectionToken({ userId, info? })  // Get a realtime connection token
```

## Push Notifications

Configure push notifications for offline users via [Abrova Push](https://abrova.ir/cloud/abrova-push):

```typescript
// Use a Abrova Push template
await abrovaChat.webhooks.setPushTemplate('your-template-id');

// Or use per-event inline templates
await abrovaChat.webhooks.setPushTemplates({
  new_message: {
    title: '{{senderName}}',
    body: '{{messagePreview}}',
  },
  mention: {
    title: '{{senderName}} mentioned you',
    body: '{{messagePreview}}',
  },
});
```

### Notification settings

Let each user decide which chat pushes they get. Nothing changes until you set something.

```typescript
// Turn off chat pushes for a user across your app
await abrovaChat.users.updateNotificationSettings('user-1', { pushLevel: 'none' });

// Only push when mentioned, and no reaction pushes
await abrovaChat.users.updateNotificationSettings('user-1', {
  pushLevel: 'mentions',
  disabledEvents: ['reaction'],
});

// Mute one room for 8 hours, then unmute
await abrovaChat.rooms.updateNotificationSettings(roomId, 'user-1', {
  mutedUntil: new Date(Date.now() + 8 * 3600_000),
});
await abrovaChat.rooms.updateNotificationSettings(roomId, 'user-1', { mutedUntil: null });

await abrovaChat.users.getNotificationSettings('user-1');
```

`pushLevel` is `all` (default), `mentions` or `none`. App-wide and room settings both apply: a push is sent only if neither blocks it.

## Links

- [Abrova Chat](https://abrova.ir/cloud/abrova-chat) - Learn more about Abrova Chat
- [Documentation](https://abrova.ir/cloud/abrova-chat/docs/quick-start) - Full documentation and guides
- [Abrova Console](https://console.abrova.ir) - Manage your chat rooms

## License

MIT
