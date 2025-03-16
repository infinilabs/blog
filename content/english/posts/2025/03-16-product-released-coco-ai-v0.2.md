---
title: "Coco AI v0.2 Unleashed: The Future of Unified AI Search & Productivity is Here!"
meta_title: "Coco AI v0.2 Released"
description: "Coco AI v0.2 is now available! A powerful, open-source AI-powered search and productivity tool for seamless access to personal and enterprise knowledge."
date: "2025-03-16T08:00:00+08:00"
categories: ["Release", "News"]
image: "/images/posts/2025/coco-ai-v0.1.0-release.png"
author: "Rain9"
tags:
  [
    "Coco AI",
    "Release",
    "AI",
    "Search",
    "Productivity",
    "RAG",
    "Knowledge Management",
    "Open Source",
    "Cross-Platform",
  ]
lang: "en"
category: "Product"
subcategory: "Released"
draft: false
---

# Coco AI v0.2 Released

We’re excited to announce the official release of **Coco AI v0.2** – the **second** preview version of our powerful, open-source, cross-platform unified AI search and productivity tool!

Coco AI allows you to seamlessly search through various data sources such as local applications, cloud storage (Google Drive, Notion, Yuque, Hugo), and enterprise knowledge bases. By leveraging advanced models like **DeepSeek**, Coco AI transforms personal knowledge management, enabling users to efficiently access information with enhanced privacy and security.

With **Coco AI**, users can:

- Search multiple data sources in real-time, including both local and cloud data.
- Integrate AI-powered personal knowledge management for smarter information access.
- Chat with an AI assistant based on your personalized knowledge base, ensuring that information is always at your fingertips.

You can now create your own personal AI Assistant with Coco AI now~

## Key Features:

- **Unified Search & Productivity Tool**: Connect and search across a wide range of data sources including desktop apps, cloud data, and internal knowledge bases.
- **AI Assistant Mode**: Chat with the AI assistant based on your knowledge base. The assistant understands your documents, personal data, and can offer intelligent summaries and responses.
- **Privacy & Security**: Coco AI allows for **private deployment**, ensuring your data stays secure, with all personal knowledge hosted locally or on your own server.
- **Scalable & Open**: Easily integrate your own data into Coco AI via APIs and extend functionality with custom data connectors.

## Coco AI Client v0.2.1

![Client chat-rag](/images/posts/2025/coco-v0.2/chat-rag.gif)

Functional updates

- Added richer RAG capabilities
- Supports in-app update prompts and automatic updates #274

Bug fixes

- Fixed the problem that the fusion search includes disabled servers
- Fixed incorrect version type: should be a string instead of u32
- Fixed the inaccurate judgment type of the end of the chat push #280

Optimization and improvement

- Refactored the chat component #273
- Added service link display #282
- Optimized the chat scrolling effect and chat data rendering effect #282
- Set the minimum width of the chat window & remove the input box background #284
- Removed the abandoned selection function & added the function of selecting and hiding APP #286
- Increased the Websocket timeout to 2 minutes #289

![Client v0.2.1](/images/posts/2025/coco-v0.2/0.2.1.png)

![Search Local Desktop Applications](/images/posts/2025/coco-search-local-apps.png)

![Chat with Your AI Assistant](/images/posts/2025/fusion-search-across-datasources.png)

## Coco AI Server v0.2.2

![server-1](/images/posts/2025/coco-v0.2/server-1.png)

Functional updates

- Added a graphical management interface
- Added a document creation API under the data source
- Added file upload related API
- Added API TOKEN management related API
- Data source synchronization supports dynamic configuration time interval
- Supports dynamic update of server settings
- Supports dynamic update of large model related settings
- Added RAG chat session processing
- Added online search capabilities
- Supports docking with DeepSeek large models
- Added document preprocessing Processor

Bug fixes

- Fixed Google Drive Connector missing file error

Optimization and improvement

- Optimize chat session function
- Optimize Websocket session management
- Optimize login and logout interface
- Save Notion other content to Payload field
- Improve the background task exit mechanism
- Optimize the default index template and query template

![server-2](/images/posts/2025/coco-v0.2/server-2.png)

![server-3](/images/posts/2025/coco-v0.2/server-3.png)

![server-4](/images/posts/2025/coco-v0.2/server-4.png)

![server-5](/images/posts/2025/coco-v0.2/server-5.png)

## Download & Get Started

Coco AI v0.2 is now available for **MacOS 12 and above**. You can start using it today by downloading from the following links:

- **Download Coco AI**: - [https://coco.rs/](https://coco.rs/)
- **Desktop App**: [Coco AI App's Github](https://github.com/infinilabs/coco-app/)
- **Coco Server**: [Coco AI Server's Github](https://github.com/infinilabs/coco-server)
- **Docs for Desktop App**: [App documentation](https://docs.infinilabs.com/coco-app/main/)
- **API for Coco Server**: [API documentation](https://docs.infinilabs.com/coco-server/main/)

## Looking Ahead

We’re committed to improving Coco AI, adding more features, and enhancing the integration with both personal and enterprise systems. Stay tuned for future updates and features as we continue to develop Coco AI into a powerful AI-driven knowledge management tool.

---

**Happy searching with Coco AI!**  
We look forward to seeing how Coco AI can help streamline your information access and improve productivity.
