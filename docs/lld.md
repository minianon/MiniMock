# MiniMock: Low-Level Design

State, components and the export pipeline. For the big picture see the [High-Level Design](hld.md).

## 1. Source layout

| Path | Role |
|---|---|
| `src/App.jsx` | All state, tab switching, resizable layout, PNG export |
| `src/components/Navbar.jsx` | Tab bar and app light / dark theme |
| `src/components/Editor.jsx` + `PlatformSelector`, `TypeSelector`, `PeopleSection`, `MessagesSection`, `AppearanceSection` | Chat editor |
| `src/components/ChatPreview.jsx` + `preview/` (`ChatHeader`, `ChatBubble`, `ChatFooter`, `StatusBar`) | Chat preview |
| `src/components/ai/` | AI chat editor and preview |
| `src/components/stories/`, `posts/`, `comments/`, `email/` | Editor + preview pair per tab |
| `src/components/DownloadModal.jsx` | File name prompt before export |
| `src/themes/platformThemes.js`, `src/themes/aiThemes.jsx` | Visual definitions per platform |

## 2. State model

```mermaid
classDiagram
  class AppState {
    activeTab: chat | ai | stories | posts | comments | email
    appTheme: light | dark
    mobileView: edit | preview
    editorWidth: 280..600
  }
  class ChatState {
    platform: whatsapp | instagram | discord | x | teams | slack | snapchat | telegram | reddit | linkedin
    chatType: dm | group
    groupData: name, avatar
    people: Person[]
    messages: Message[]
    appearance: Appearance
  }
  class Person {
    id
    name
    avatar
    role: sender | receiver
    verified: boolean
  }
  class Message {
    id
    senderId
    text
    date
    time
    status: sent | delivered | read
    image
  }
  class Appearance {
    darkMode
    transparentBg
    timeFormat: 12 | 24
    showHeader
    showFooter
    phoneFrame
    statusBar
    statusBarTime
    battery
  }
  class AIState {
    aiPlatform
    aiModel
    aiMessages
    aiPerson
    aiAppearance
  }
  AppState --> ChatState
  AppState --> AIState
  ChatState --> Person
  ChatState --> Message
  ChatState --> Appearance
  Message --> Person : senderId
```

Stories, posts, comments and email follow the same pattern: a platform, the content (slides, author + content + metrics, creator + commenters + comments, participants + thread) and an appearance object.

## 3. Render flow

```mermaid
flowchart LR
  T{activeTab} -->|chat| CE[Editor] & CP[ChatPreview]
  T -->|ai| AE[AIEditor] & AP[AIPreview]
  T -->|stories| SE[StoriesEditor] & SP[StoriesPreview]
  T -->|posts| PE[PostsEditor] & PP[PostsPreview]
  T -->|comments| ME[CommentsEditor] & MP[CommentsPreview]
  T -->|email| EE[EmailEditor] & EP[EmailPreview]
  CP & AP & SP & PP & MP & EP --> R[previewRef]
```

Each preview is rendered into the element held by `previewRef`, which is what gets exported.

## 4. Export pipeline

```mermaid
sequenceDiagram
  actor User
  participant M as DownloadModal
  participant App as handleDownload
  participant DOM
  participant H2I as html-to-image
  User->>M: Download, enter file name
  M->>App: handleDownload(fileName)
  App->>DOM: read current scroll of .preview-messages
  App->>DOM: deep-clone preview, place off-screen inside .app
  App->>DOM: remove phone frame border, shadow, radius on clone
  alt chat or AI chat
    App->>DOM: shift first message by scroll offset (keeps visible area)
  else email
    App->>DOM: expand clone to full thread height
  end
  App->>H2I: toPng(clone, pixelRatio 3, transparent bg if set)
  H2I-->>App: data URL
  App->>DOM: remove clone
  App-->>User: download fileName.png
```

Working on a clone means the live editor never flickers or changes during export, and the clone inherits the theme CSS variables because it is attached inside `.app`.

## 5. Layout behavior

| Behavior | Rule |
|---|---|
| Editor width | Dragging the divider sets width between 280 and 600 px |
| Mobile | At ≤ 768 px wide, a toggle switches between editor and preview |
| Default file name | Based on the active tab and platform |

## 6. Configuration

None. MiniMock needs no environment variables or keys.
