---
title: "Coco AI v0.5 Unlocks Local Plugins, Smarter App Search, AI Overview and Instant AI Command"
meta_title: "Coco AI v0.5 Released"
description: "Coco AI v0.5 is now available! A powerful, open-source AI-powered search and productivity tool for seamless access to personal and enterprise knowledge."
date: "2025-06-03T18:00:00+08:00"
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

# Coco AI v0.5 Released

We’re excited to announce the official release of **Coco AI v0.5** – the latest stable version of our powerful, open-source, cross-platform unified AI search and productivity tool!

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
- **Local Plugin Extension**: Enhance functionality through local plugins, enabling customization and extension of Coco AI's capabilities.
- **Pizza-based Application Search**: Advanced application search powered by Pizza technology for more accurate and context-aware results.
- **AI Command Quick Access**: Streamlined access to AI capabilities through quick commands, improving workflow efficiency and productivity.

## Coco App v0.5

### 🚀 Features

- Check or enter to close the list of assistants #469
- Add dimness settings for pinned window #470
- Supports Shift + Enter input box line feeds #472
- Support for snapshot version updates #480
- History list add put away button #482
- The chat input box supports multi-line input #490
- Add `~/Applications` to the search path #493
- The chat content has added a button to return to the bottom #495
- The search input box supports multi-line input #501
- Websocket support self-signed TLS #504
- Add option to allow self-signed certificates #509
- Add AI summary component #518
- Dynamic log level via env var COCO_LOG #535
- Add quick AI access to search mode #556
- Rerank search results #561

### 🐛 Bug fix

- Solve the problem of modifying the assistant in the chat #476
- Several issues around search #502
- Fixed the newly created session has no title when it is deleted #511
- Loading chat history for potential empty attachments
- Datasource & MCP list synchronization update #521
- App icon & category icon #529
- Show only enabled datasource & MCP list
- Server image loading failure #534
- Panic when fetching app metadata on Windows #538
- Service switching error #539
- Switch server assistant and session unchanged #540
- History list height #550
- Secondary page cannot be searched #551
- The scroll button is not displayed by default #552
- Suggestion list position #553
- Independent chat window has no data #554
- Resolved navigation error on continue chat action #558
- Make extension search source respect parameter datasource #576

### ✈️ Improvements

- Adjust list error message #475
- Refine wording on search failure
- Search and MCP show hidden logic #494
- Greetings show hidden logic #496
- Fetch app list in settings in real time #498
- UpdateApp component loading location #499
- Add clear monitoring & cache calculation to optimize performance #500
- Optimizing the code #505
- Optimized the modification operation of the numeric input box #508
- Modify the style of the search input box #513
- Chat input icons show #515
- Refactoring icon component #514
- Optimizing list styles in markdown content #520
- Add a component for text reading aloud #522
- History component styles #528
- Search error styles #533
- Skip register server that not logged in #536
- Service info related components #537
- Chat content can be copied #539
- Refactoring search error #541
- Add assistant count #542
- Add global login judgment #544
- Mark server offline on user logout #546
- Logout update server profile #549
- Assistant keyboard events and mouse events #559
- Web component start page config #560
- Assistant chat placeholder & refactor input box components #566
- Input box related components #568
- Mark unavailable server to offline on refresh info #569
- Only show available servers in chat #570
- Search result related components #571

### 📸 Screenshots

![search-1](/images/posts/2025/coco-v0.5/1.jpg)

![search-2](/images/posts/2025/coco-v0.5/2.jpg)

![search-3](/images/posts/2025/coco-v0.5/3.png)

![search-4](/images/posts/2025/coco-v0.5/4.jpg)

![chat-20250428](/images/posts/2025/coco-v0.5/ai-abstract.gif)

![search-coco](/images/posts/2025/coco-v0.5/ai-overview.gif)

![search-menu](/images/posts/2025/coco-v0.5/ai-quick.gif)

## Coco Server v0.5

### 🚀 Features

- Allow converting icon to base64 #261
- Implement ask api for assistant
- Add placeholder to chat settings
- Return number of assistants in provider info API
- Add assistant to search results #274
- Add built-in AI assistant `AI Overview`
- Add throttle filter wth request fingerprint #294
- Multi-user login support

### 🐛 Bug fix

- Add missing cors feature flags to settings api
- Incorrect datasource icon #265
- Handle empty URL values in HugoSite-type datasource
- Datasource & MCP selection problem #267
- Resolve compatibility issue with crypto.randomUUID when creating model provider
- Start page configuration of integration is not working

### ✈️ Improvements

- Clean up unused LLM settings code
- Sort chat history by created
- Add enabled by default params to assistant edit
- Password supports more special characters
- Refactoring chat api #273
- Add placeholder, category and tags to AI Assistant
- Ignore empty chunk_message #288

### 📸 Screenshots

![assistant](/images/posts/2025/coco-v0.4/assistant1.png)

![assistant-edit](/images/posts/2025/coco-v0.4/assistant-edit.png)

![mcp](/images/posts/2025/coco-v0.4/mcp.png)

![mcp-edit](/images/posts/2025/coco-v0.4/mcp-edit.png)

![model](/images/posts/2025/coco-v0.4/model.png)

![model-edit](/images/posts/2025/coco-v0.4/model-edit.png)

![start](/images/posts/2025/coco-v0.4/start.png)

## Download & Get Started

Coco AI v0.5 is now available for **MacOS 12 and above**, **Windows 10 and above**, and **Linux Ubuntu**. You can start using it today by downloading from the following links:

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
