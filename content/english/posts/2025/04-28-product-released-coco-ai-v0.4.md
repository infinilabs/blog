---
title: "Coco AI v0.4 released, improved assistant settings, added MCP support"
meta_title: "Coco AI v0.4 Released"
description: "Coco AI v0.4 is now available! A powerful, open-source AI-powered search and productivity tool for seamless access to personal and enterprise knowledge."
date: "2025-04-28T08:00:00+08:00"
categories: ["Release", "News"]
image: "/images/posts/2025/coco-v0.4/coco-ai-v0.4-release.png"
author: "Rain9"
tags:
  [
    "Coco AI",
    "Release",
    "AI",
    "Search",
    "Productivity",
    "RAG",
    "MCP",
    "Assistant",
    "Knowledge Management",
    "Open Source",
    "Cross-Platform",
  ]
lang: "en"
category: "Product"
subcategory: "Released"
draft: false
---

# Coco AI v0.4 Released

We’re excited to announce the official release of **Coco AI v0.4** – the **third** preview version of our powerful, open-source, cross-platform unified AI search and productivity tool!

Coco AI allows you to seamlessly search through various data sources such as local applications, cloud storage (Google Drive, Notion, Yuque, Hugo), and enterprise knowledge bases. By leveraging advanced models like **DeepSeek**, Coco AI transforms personal knowledge management, enabling users to efficiently access information with enhanced privacy and security.

With **Coco AI**, users can:

- Search multiple data sources in real-time, including both local and cloud data.
- Integrate AI-powered personal knowledge management for smarter information access.
- Chat with an AI assistant based on your personalized knowledge base, ensuring that information is always at your fingertips.
- Integrate their websites with Coco AI through the Widget system for enhanced functionality.

You can create your own personal AI Assistant with Coco AI~

## Key Features:

- **Unified Search & Productivity Tool**: Connect and search across a wide range of data sources including desktop apps, cloud data, and internal knowledge bases.
- **AI Assistant Mode**: Chat with the AI assistant based on your knowledge base. The assistant understands your documents, personal data, and can offer intelligent summaries and responses.
- **Privacy & Security**: Coco AI allows for **private deployment**, ensuring your data stays secure, with all personal knowledge hosted locally or on your own server.
- **Scalable & Open**: Easily integrate your own data into Coco AI via APIs and extend functionality with custom data connectors.
- **Widget Integration**: Support for external site integration through widgets, allowing seamless embedding of Coco AI functionality into third-party websites. 

## Coco App Client 0.4.0

### Breaking changes

- None

### Features

- History support for searching, renaming and deleting #322
- Linux support for application search #330
- Add shortcuts to most icon buttons #334
- Add font icon for search list #342
- Add a border to the main window in Windows 10 #343
- Mobile terminal adaptation about style #348
- Service list popup box supports keyboard-only operation #359
- Networked search data sources support search and keyboard-only operation #367
- Add application management to the plugin #374
- Add keyboard-only operation to history list #385
- Add error notification #386
- Add support for AI assistant #394
- Add support for calculator function #399
- Auto selects the first item after searching #411
- Web components assistant #422
- Right-click menu support for search #423
- Add chat mode launch page #424
- Add MCP & call LLM tools #430
- AI assistant supports search and paging #431
- Data sources support displaying customized icons #432
- Add shortcut key conflict hint and reset function #442

### Bug fix

- Fixed the problem of not being able to search in secondary directories #338
- Active shadow setting #354
- Chat history was not show up #377
- Get attachments in chat sessions
- Filter http query_args and convert only supported values
- Fixed several search & chat bugs #412

### Improvements

- Refactoring Web components #331
- Refactoring login callback, receive access_token from coco-server
- Adjust web component styles #362
- Modify the style #370
- Search list details display #378
- Refactoring api error handling #382
- Update assistant icon & think mode #397
- Build web components and publish #404

### Screenshots

![search-20250428](/images/posts/2025/coco-v0.4/search-20250428.gif)

![chat-20250428](/images/posts/2025/coco-v0.4/chat-20250428.gif)

![search-coco](/images/posts/2025/coco-v0.4/search-coco.png)

![search-menu](/images/posts/2025/coco-v0.4/search-menu.png)

![calculator-1](/images/posts/2025/coco-v0.4/calculator-1.png)

![calculator-2](/images/posts/2025/coco-v0.4/calculator-2.png)

![start-app](/images/posts/2025/coco-v0.4/start-app.png)

## Coco Server Client 0.4.0

### Breaking changes  

- None

### Features

- Add chat session management API
- Add support for font icons (#183)
- Add support for AI assistant CURD management
- Add support for model provider CURD management
- Add version and license

### Bug fix 

- Fix personal token was not well supported for Yuque connector
- Fix incorrect content-type header for wrapper
- Fix default login url can't be changed afterwards

### Improvements

- Set built-in connector icons as read-only
- Support setting icon and placeholder of integration
- Enhance UI for searchbox
- Refactoring security plugin #199
- Make searchbox's theme styles follows the system if searchbox's theme is set to `auto`
- Support setting suggested topics of integration
- Skip handle wrapper for disabled widget
- When creating a new Google Drive datasource, guide the user to configure the required settings if they are missing
- Default to use go modules
- Support user-provided icon URL in icon component
- Update default query template

### Screenshots

![assistant](/images/posts/2025/coco-v0.4/assistant1.png)

![assistant-edit](/images/posts/2025/coco-v0.4/assistant-edit.png)

![mcp](/images/posts/2025/coco-v0.4/mcp.png)

![mcp-edit](/images/posts/2025/coco-v0.4/mcp-edit.png)

![model](/images/posts/2025/coco-v0.4/model.png)

![model-edit](/images/posts/2025/coco-v0.4/model-edit.png)

![start](/images/posts/2025/coco-v0.4/start.png)

## Download & Get Started

Coco AI v0.4 is now available for **MacOS 12 and above**, **Windows 10 and above**, and **Linux Ubuntu**. You can start using it today by downloading from the following links:

- **Download Coco AI**: [https://coco.rs/](https://coco.rs/)
- **Desktop App**: [Coco AI App's Github](https://github.com/infinilabs/coco-app/)
- **Coco Server**: [Coco AI Server's Github](https://github.com/infinilabs/coco-server)
- **Docs for Desktop App**: [App documentation](https://docs.infinilabs.com/coco-app/main/)
- **API for Coco Server**: [API documentation](https://docs.infinilabs.com/coco-server/main/)

## Looking Ahead

We’re committed to improving Coco AI, adding more features, and enhancing the integration with both personal and enterprise systems. Stay tuned for future updates and features as we continue to develop Coco AI into a powerful AI-driven knowledge management tool.

---

**Happy searching with Coco AI!**  
We look forward to seeing how Coco AI can help streamline your information access and improve productivity.
