# @neoxr/telegram

A high-performance TypeScript/JavaScript framework for building production-grade Telegram bots with [Telegraf](https://telegraf.js.org/). Designed for scalability, it features an automated long-polling manager, rich message serialization, instant hot-reloading plugin architecture, multi-tenant child bot virtualization, and reactive database proxy adapters.

---

## Table of Contents

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Client Configuration](#client-configuration)
- [Message Object (Serialized `m`)](#message-object-serialized-m)
- [Messaging & Bot Helpers (`client.poll.*`)](#messaging--bot-helpers-clientpoll)
  - [Chat Presence & Actions](#chat-presence--actions)
  - [Text & Interactive Messaging](#text--interactive-messaging)
  - [Media & File Dispatchers](#media--file-dispatchers)
  - [User Profile & Verification](#user-profile--verification)
  - [Group & Channel Administration](#group--channel-administration)
  - [Forum & Topic Management](#forum--topic-management)
  - [Bot Commands & Menu](#bot-commands--menu)
- [Plugin Architecture & Hot Reloading](#plugin-architecture--hot-reloading)
- [Multi-Bot / Child Bot System](#multi-bot--child-bot-system)
  - [Subbot Virtualization](#subbot-virtualization)
  - [Isolated Owner Access](#isolated-owner-access)
  - [Lifecycle Management](#lifecycle-management)
- [Database Proxy Adapters](#database-proxy-adapters)
  - [Reactive In-Memory Architecture](#reactive-in-memory-architecture)
  - [Available Proxies](#available-proxies)
  - [Primary Key & `@username` Resolution](#primary-key--username-resolution)
- [Production Handler Architecture](#production-handler-architecture)
- [Graceful Shutdown & Data Persistence](#graceful-shutdown--data-persistence)
- [Utilities & Bundled Modules](#utilities--bundled-modules)

---

## Features

- **Robust Telegram Connection:** Long-polling engine with automatic backoff reconnection and TDLib/local Bot API server support.
- **Normalized Message Object (`m`):** WhatsApp/Baileys-like normalized message interface with unified helpers (`m.reply`, `m.edit`, `m.react`, `m.download`, `m.getProfilePhoto`).
- **Interactive Messaging:** Native support for inline buttons, chunked message delivery, and automatic Markdown/HTML fallbacks.
- **High-Speed Plugin Loader:** Scans directories in parallel with high concurrency and instant hot-reloading via file system watchers.
- **Virtual Multi-Bot Architecture:** Spin up dozens of child bots (*subbots*) within a single Node.js runtime process, each with its own isolated database scope and independent owners.
- **Unified Database Proxies:** Reactive ORM-like proxies for JSON, SQLite (WAL mode), PostgreSQL, MySQL, MongoDB, and Redis with memory caching and background synchronization.
- **Extensive Moderation Suite:** One-line helpers for bans, kicks, mutes, unpins, permissions, and group/channel metadata management.
- **Zero-Crash Safety:** Graceful error handling, auto-fallback on malformed entity parsing, and safe shutdown handlers.

---

## Requirements

- Node.js 18.0.0 or higher (ESM support enabled)
- A Telegram bot token obtained from [@BotFather](https://t.me/BotFather)
- Yarn, npm, or pnpm

---

## Installation

Install the package:

```bash
yarn add @neoxr/telegram
# or
npm install @neoxr/telegram
```

### Optional Database Drivers

Install the driver corresponding to the database proxy adapter you choose:

```bash
# For SQLite (Recommended for local low-latency)
yarn add better-sqlite3

# For PostgreSQL
yarn add pg

# For MySQL
yarn add mysql2

# For MongoDB
yarn add mongodb

# For Redis
yarn add redis
```

---

## Quick Start

### 1. Configure Environment Variables

Create a `.env` file in the root directory:

```env
BOT_TOKEN=123456789:AAHxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DATABASE_URL=postgres://user:password@localhost:5432/bot_db
```

### 2. Create the Bot Entry Point (`client.js`)

```javascript
import { Client } from '@neoxr/telegram'

const client = new Client({
  plugsdir: './plugins',
  debug: true
})

client.on('connect', (botInfo) => {
  console.log(`Connected to Telegram as @${botInfo.username}`)
})

client.on('ready', () => {
  console.log('Bot is ready to process updates')
})

client.on('message', async (ctx) => {
  const { m } = ctx
  if (m.text === '/ping') {
    await m.reply('Pong!')
  }
})

client.on('error', (err) => console.error('Bot Error:', err))
```

### 3. Run the Application

```bash
node client.js
```

---

## Client Configuration

The `Client` class accepts configuration options to tune polling, plugin directories, multi-bot routing, and default prefixes:

```javascript
import { Client } from '@neoxr/telegram'

const client = new Client({
  plugsdir: ['./plugins', './plugins/events'],
  create: {
    token: process.env.BOT_TOKEN
  },
  multiple: true,
  debug: false,
  prefix_data: 'db',
  setting: {
    multiprefix: true,
    prefix: ['.', '#', '!', '/'],
    onlyprefix: '/',
    hidden: [],
    pluginDisable: []
  },
  apiRoot: 'http://127.0.0.1:8081'
}, {
  telegraf: {
    handlerTimeout: 30000
  }
})
```

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `plugsdir` | `string \| string[]` | `''` | Directory path(s) scanned by the plugin loader. |
| `create.token` | `string` | `process.env.BOT_TOKEN` | Overrides the bot token used for initialization. |
| `multiple` | `boolean` | `false` | Enables virtual child bot creation and multi-instance orchestration. |
| `debug` | `boolean` | `false` | Outputs connection diagnostics and update logs to standard output. |
| `prefix_data` | `string` | `'db'` | The global variable key (`global.db`) used for data persistence. |
| `setting` | `object` | `{}` | Configures default command prefixes, disabled plugins, and hidden categories. |
| `apiRoot` / `tdlib` | `string \| object` | `undefined` | Custom Telegram Bot API server endpoint for uploading files larger than 50MB. |

---

## Message Object (Serialized `m`)

When a message arrives, the framework serializes it into a unified, clean `m` object:

```javascript
client.on('message', async ({ m }) => {
  console.log({
    id: m.id,
    chatId: m.chat,
    senderId: m.sender.id,
    username: m.sender.username,
    text: m.text,
    body: m.body,
    fromMe: m.fromMe,
    isMedia: m.isMedia,
    mentionedJid: m.mentionedJid
  })
})
```

### Properties of `m`

| Property | Type | Description |
| --- | --- | --- |
| `m.id` | `number` | Unique Telegram message identifier. |
| `m.chat` | `number \| string` | Chat identifier where the message was sent. |
| `m.isGroup` | `boolean` | `true` if the message originated in a group or supergroup. |
| `m.isPrivate` | `boolean` | `true` if the message originated in a direct message conversation. |
| `m.isChannel` | `boolean` | `true` if the update originated from a broadcast channel. |
| `m.fromMe` | `boolean` | `true` if the message was dispatched by this bot instance. |
| `m.isMedia` | `boolean` | `true` if the message contains an image, video, document, audio, or sticker. |
| `m.sender` | `object` | Information about the sender: `{ id, jid, name, username, isBot }`. |
| `m.text` / `m.body` | `string` | The text content, caption, or callback query data payload. |
| `m.mentionedJid` | `string[]` | Array of user IDs and usernames mentioned inside the message entities. |
| `m.file` | `object \| null` | File metadata object if media is attached. |
| `m.quoted` | `object \| null` | Serialized message object of the replied/quoted message. |

### Instance Methods on `m`

- **`await m.reply(text, options)`**: Replies directly to this message. Defaults to `Markdown` formatting with automatic text chunking and error fallback.
- **`await m.edit(text, options)`**: Edits the text or caption of this message in-place.
- **`await m.delete()`**: Deletes this message from the chat.
- **`await m.react(emoji)`**: Applies an emoji reaction to this message (Bot API 7.0+).
- **`await m.download()`**: Downloads the attached media file into a memory `Buffer`.
- **`await m.getProfilePhoto(options)`**: Retrieves the direct URL or `file_id` of the message sender avatar.

### Quoted Message Helpers (`m.quoted`)

When `m.quoted` is present, it contains identical helper functions scoped to the quoted message:

```javascript
if (m.quoted) {
  await m.quoted.reply('Replying directly to your referenced message')
  await m.quoted.react('❤️')
  const buffer = await m.quoted.download()
  const avatarUrl = await m.quoted.getProfilePhoto()
}
```

---

## Messaging & Bot Helpers (`client.poll.*`)

The `poll` property on `Client` is an extended Telegraf instance injected with convenience methods for rich content dispatch and chat moderation.

### Chat Presence & Actions

```javascript
await client.poll.presence(chatId, 'typing')
await client.poll.presence(chatId, 'upload_photo')
await client.poll.presence(chatId, 'record_voice')
await client.poll.presence(chatId, 'upload_document')
```

### Text & Interactive Messaging

#### `reply(chatId, text, quoted, options)`
Dispatches a text message. Text exceeding 4096 characters is safely divided into multiple sequential chunks without breaking sentences.

```javascript
await client.poll.reply(m.chat, '*Hello world!*', m)
```

#### `replyButton(chatId, text, buttons, quoted, options)`
Sends an interactive message with inline keyboard buttons.

```javascript
const buttons = [
  [
    { text: 'Documentation', url: 'https://telegram.org' },
    { text: 'Profile', callback_data: '.profile' }
  ],
  [
    { text: 'Confirm Action', callback_data: 'confirm_yes' }
  ]
]

await client.poll.replyButton(m.chat, 'Select an option below:', buttons, m)
```

#### `edit(chatId, text, msgId, quoted, options)`
Edits an existing text message or media caption.

```javascript
await client.poll.edit(m.chat, 'Updated text', messageId)
```

#### `delete(chatId, msgId)` and `deleteMessages(chatId, messageIds)`
Removes single or multiple messages in one operation.

```javascript
await client.poll.delete(m.chat, m.id)
await client.poll.deleteMessages(m.chat, [m.id, m.id - 1, m.id - 2])
```

#### `react(chatId, msgId, emoji)`
Applies an emoji reaction to a specific message.

```javascript
await client.poll.react(m.chat, m.id, '🔥')
```

#### `answerCallbackQuery(callbackQueryId, text, showAlert, options)`
Acknowledges an inline keyboard button press to dismiss the loading animation.

```javascript
await client.poll.answerCallbackQuery(ctx.callbackQuery.id, 'Action completed!', true)
```

#### `sendProgress(chatId, finalText, quoted, options)`
Renders an animated in-place updating percentage bar.

```javascript
await client.poll.sendProgress(m.chat, 'Download complete 100%!', m)
```

---

### Media & File Dispatchers

All media helpers accept URLs, local file paths, `Buffer` objects, or existing Telegram `file_id` strings. Captions longer than 1024 characters are automatically split, with the remainder delivered as follow-up text messages.

#### `sendPhoto(chatId, source, caption, quoted, options)`
```javascript
await client.poll.sendPhoto(m.chat, 'https://example.com/banner.jpg', 'Project Overview', m, {
  reply_markup: {
    inline_keyboard: [[{ text: 'Details', callback_data: 'view_details' }]]
  }
})
```

#### `sendVideo(chatId, source, caption, quoted, options)`
```javascript
await client.poll.sendVideo(m.chat, './media/clip.mp4', 'Sample video', m)
```

#### `sendAudio(chatId, source, caption, quoted, options)`
```javascript
await client.poll.sendAudio(m.chat, buffer, 'Track 01 - Intro', m)
```

#### `sendVoice(chatId, source, quoted, options)`
```javascript
await client.poll.sendVoice(m.chat, './media/voice.ogg', m)
```

#### `sendDocument(chatId, source, filename, caption, quoted, options)`
```javascript
await client.poll.sendDocument(m.chat, './data/report.pdf', 'Annual_Report.pdf', 'Confidential', m)
```

#### `sendSticker(chatId, source, quoted, options)`
```javascript
await client.poll.sendSticker(m.chat, './stickers/animated.tgs', m)
```

#### `sendMediaGroup(chatId, mediaArray, quoted, options)`
Sends an album of up to 10 photos or videos.

```javascript
await client.poll.sendMediaGroup(m.chat, [
  { type: 'photo', media: 'https://example.com/1.jpg', caption: 'Image 1' },
  { type: 'photo', media: 'https://example.com/2.jpg' }
], m)
```

#### `sendFile(chatId, source, filename, caption, quoted, options)`
Universal media detector. Automatically detects the MIME type of the given buffer or URL and routes it to `sendPhoto`, `sendVideo`, `sendAudio`, or `sendDocument`.

```javascript
await client.poll.sendFile(m.chat, 'https://example.com/file.zip', 'Archive.zip', 'Backup files', m)
```

---

### User Profile & Verification

#### `getProfilePhoto(userId, options)`
Retrieves the direct download URL of a user highest-resolution avatar. Returns a default placeholder URL if private or unavailable.

```javascript
const photoUrl = await client.poll.getProfilePhoto(m.sender.id)
if (photoUrl) {
  await client.poll.sendPhoto(m.chat, photoUrl, 'Profile Photo', m)
}
```

#### `isBlockedBy(userId)`
Performs a silent check using `sendChatAction` to verify if a user has blocked the bot or deleted their account, catching `403 Forbidden` without sending visible messages.

```javascript
const isBlocked = await client.poll.isBlockedBy(targetUserId)
if (isBlocked) {
  console.log('User has blocked the bot')
}
```

---

### Group & Channel Administration

```javascript
await client.poll.banMember(m.chat, targetUserId)
await client.poll.unbanMember(m.chat, targetUserId)
await client.poll.kickMember(m.chat, targetUserId)

await client.poll.muteMember(m.chat, targetUserId, 3600)
await client.poll.unmuteMember(m.chat, targetUserId)

await client.poll.setChatPermissions(m.chat, {
  can_send_messages: true,
  can_send_photos: false
})

await client.poll.promoteMember(m.chat, targetUserId, {
  can_manage_chat: true,
  can_delete_messages: true
})
await client.poll.demoteMember(m.chat, targetUserId)

await client.poll.pinMessage(m.chat, m.id)
await client.poll.unpinMessage(m.chat, m.id)
await client.poll.unpinAllMessages(m.chat)

await client.poll.setChatTitle(m.chat, 'Official Community Group')
await client.poll.setChatDescription(m.chat, 'Community guidelines and rules')
await client.poll.setChatPhoto(m.chat, './logo.jpg')
await client.poll.deleteChatPhoto(m.chat)

const link = await client.poll.exportInviteLink(m.chat)
const customLink = await client.poll.createInviteLink(m.chat, { member_limit: 10 })
await client.poll.revokeInviteLink(m.chat, customLink.invite_link)
```

---

### Forum & Topic Management

Full support for supergroup forum topics (Telegram Bot API 6.3+):

```javascript
const topic = await client.poll.createForumTopic(m.chat, 'General Discussion', {
  icon_color: 7322096
})

await client.poll.editForumTopic(m.chat, topic.message_thread_id, {
  name: 'Updated Topic Name'
})

await client.poll.closeForumTopic(m.chat, topic.message_thread_id)
await client.poll.reopenForumTopic(m.chat, topic.message_thread_id)
await client.poll.deleteForumTopic(m.chat, topic.message_thread_id)
```

---

### Bot Commands & Menu

Set or read the command menu displayed next to the chat bar in Telegram:

```javascript
await client.poll.setMyCommands([
  { command: 'start', description: 'Start the bot' },
  { command: 'menu', description: 'Show command list' },
  { command: 'profile', description: 'View user profile' }
])

const commands = await client.poll.getMyCommands()
```

---

## Plugin Architecture & Hot Reloading

Plugins reside in the folder specified by `plugsdir`. Each plugin is an ES module exporting a `run` object. Changes are automatically detected and hot-reloaded within milliseconds without restarting the process.

### Basic Command Plugin (`plugins/ping.js`)

```javascript
export const run = {
  usage: ['ping'],
  category: 'general',
  async: async (m, { client }) => {
    await client.poll.reply(m.chat, 'Pong!', m)
  },
  error: false
}
```

### Parameterized Plugin with Target Resolution (`plugins/profile.js`)

```javascript
export const run = {
  usage: ['profile', 'me'],
  category: 'user',
  async: async (m, { client, args, db, command }) => {
    let targetId = m.quoted?.sender?.id 
      || (m.mentionedJid && m.mentionedJid[0]) 
      || (args[0] && args[0].replace(/^@/, '')) 
      || m.sender.id

    const user = db.users.get(targetId)
    if (!user) {
      return client.poll.reply(m.chat, 'User not found in database.', m)
    }

    const caption = [
      'User Profile:',
      `• ID: ${user.id}`,
      `• Name: ${user.name || '-'}`,
      `• Limit: ${user.limit ?? 0}`,
      `• Premium: ${user.premium ? 'Yes' : 'No'}`
    ].join('\n')

    const avatar = await client.poll.getProfilePhoto(user.id)
    if (avatar) {
      return client.poll.sendPhoto(m.chat, avatar, caption, m)
    }

    return client.poll.reply(m.chat, caption, m)
  },
  error: false
}
```

---

## Multi-Bot / Child Bot System

Enable `multiple: true` in your client options to turn your bot into a parent instance capable of orchestrating independent child bots inside the same process.

```
       [Main Bot Instance]
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
[Child Bot 1] [Child Bot 2] [Child Bot 3]
  (@bot_a)      (@bot_b)      (@bot_c)
    │           │           │
    ▼           ▼           ▼
[DB Scope 1]  [DB Scope 2]  [DB Scope 3]
```

### Subbot Virtualization

Child bots inherit plugins and loaders from the main instance without duplicate disk scans, while maintaining independent long-polling connections and isolated databases.

### Isolated Owner Access

When creating a child bot, pass the `owner` parameter. The child bot will automatically inject this owner into its database `setting.owners`:

```javascript
const child = await client.create({
  create: { token: userSubmittedToken },
  owner: m.sender.id
})
```

### Lifecycle Management

```javascript
await client.start('bailysbot')
await client.stop('bailysbot')
await client.restart('bailysbot')
await client.autoReconnectAllBots()
```

---

## Database Proxy Adapters

The framework exports `Proxy` containing 6 database adapters that wrap data structures in reactive ES6 Proxies. Any mutation to an object immediately writes to storage or disk with debounce and transaction protection.

```javascript
import { Proxy } from '@neoxr/telegram'

const { proxyJson, proxySqlite, proxyPgSql, proxyMySql, proxyMongo, proxyRedis } = Proxy
```

### Reactive In-Memory Architecture

All collection rows are cached in memory and indexed by Primary Key. When modifying properties:

```javascript
const user = db.users.get(m.sender.id)

user.limit += 5
user.premium = true
```

### Available Proxies

| Adapter | Backend | Characteristics |
| --- | --- | --- |
| `proxyJson` | Local JSON files | Zero setup. Includes atomic temporary writes (`.tmp`), debouncing, and WeakMap proxy caching. |
| `proxySqlite` | `better-sqlite3` | In-memory cache backed by SQLite WAL mode. Fast, low memory footprint, crash-proof. |
| `proxyPgSql` | `pg` (PostgreSQL) | Native connection pooling, background hydration, transactional bulk writes. |
| `proxyMySql` | `mysql2/promise` | Connection pooling with keep-alive, transactional bulk writes, and optional zlib Gzip payload compression. |
| `proxyMongo` | `mongodb` | Native document mapping, bulk write operations, isolated collection prefix routing. |
| `proxyRedis` | `redis` | In-memory Redis Hashes (`HSET`/`HGETALL`) with optional Gzip compression for large values. |

### Primary Key & `@username` Resolution

All adapters use an intelligent primary key resolution engine. Data can be retrieved interchangeably by numerical ID, `@username`, or plain username:

```javascript
const userById = db.users.get('7865157900')
const userByTag = db.users.get('@johndoe')
const userByName = db.users.get('johndoe')
```

---

## Production Handler Architecture

Here is the battle-tested dispatcher architecture integrating `Client`, database routing, dynamic child-bot scoping, and command execution:

### `client.js`

```javascript
import { Client } from '@neoxr/telegram'
import system from './lib/adapter.js'
import { models } from './lib/models.js'
import handler from './handler.js'

let clientInstance = null

const connect = async () => {
  try {
    const client = new Client({
      plugsdir: './plugins',
      multiple: true,
      debug: false
    })

    clientInstance = client

    await system.init(models, models.structure)

    const bindListener = (bot) => {
      bot.on('message', async (ctx) => {
        handler(bot.poll, { ...ctx, system, connector: bot })
      })
    }

    bindListener(client)

    client._bind((child) => {
      bindListener(child)
    })
  } catch (err) {
    console.error('Connection failed:', err)
    setTimeout(connect, 5000)
  }
}

connect()

const shutdown = async (signal) => {
  console.log(`\nShutting down (${signal})...`)
  if (clientInstance?.close) await clientInstance.close('SHUTDOWN')
  if (system.closeAll) await system.closeAll()
  if (system.flushAll) await system.flushAll()
  process.exit(0)
}

process.on('SIGINT', () => shutdown('SIGINT'))
process.on('SIGTERM', () => shutdown('SIGTERM'))
```

---

## Graceful Shutdown & Data Persistence

Ensure no transactions or queued file writes are lost during container redeployments or server restarts by hooking into termination signals:

```javascript
const gracefulShutdown = async () => {
  await client.close('MAIN_SHUTDOWN')

  if (typeof system.flushAll === 'function') {
    await system.flushAll()
  }

  if (typeof system.closeAll === 'function') {
    await system.closeAll()
  }

  process.exit(0)
}

process.on('SIGINT', gracefulShutdown)
process.on('SIGTERM', gracefulShutdown)
```

---

## Utilities & Bundled Modules

```javascript
import { Utils, Cooldown, Spam, Config, NeoxrApi } from '@neoxr/telegram'
```

- **`Utils`**: Common helpers for string formatting, greeting detectors, size formatting, and delay timers (`Utils.delay`).
- **`Cooldown`**: In-memory token bucket rate-limiter for command throttling.
- **`Spam`**: Message frequency monitor with warning, mute, and temporary ban thresholds.
- **`Config`**: Auto-loaded configuration object from `config.json` in the root folder.
- **`NeoxrApi`**: API client wrapper for Neoxr network services.

---