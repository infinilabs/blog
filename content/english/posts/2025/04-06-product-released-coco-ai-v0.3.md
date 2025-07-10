---
title: "Coco AI v0.3 Launches with Powerful Widget Integration for External Platforms"
meta_title: "Coco AI v0.3 Released"
description: "Coco AI v0.3 is now available! A powerful, open-source AI-powered search and productivity tool for seamless access to personal and enterprise knowledge."
date: "2025-04-06T08:00:00+08:00"
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

# Coco AI v0.3 Released

We’re excited to announce the official release of **Coco AI v0.3** – the **third** preview version of our powerful, open-source, cross-platform unified AI search and productivity tool!

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

## Coco AI Client v0.3.0

![Client chat-rag](/images/posts/2025/coco-v0.2/chat-rag.gif)

### Functional Updates

- Added keyboard shortcuts settings
  ![Keyboard shortcuts settings](/images/posts/2025/coco-v0.3/1.png)
- Added support for multiple chat sessions

### Bug Fixes

- Fixed abnormal candidate list with missing icons in application search

### Optimization and Improvements

- Refactored code to provide Web Widget external integration by reusing frontend components

## Coco AI Server v0.3.0

### Functional Updates

- Added connector UI management support
  ![Connector UI management 1](/images/posts/2025/coco-v0.3/2.png)
  ![Connector UI management 2](/images/posts/2025/coco-v0.3/3.png)
- Added document searchability control based on data source status
  ![Document searchability control](/images/posts/2025/coco-v0.3/4.png)
- Added support for passing WebSocket session ID through request headers
- Added integration component management
  ![Integration component 1](/images/posts/2025/coco-v0.3/5.png)
  ![Integration component 2](/images/posts/2025/coco-v0.3/6.png)
  ![Integration component 3](/images/posts/2025/coco-v0.3/7.png)
  ![Integration component 4](/images/posts/2025/coco-v0.3/8.png)
  ![Integration component 5](/images/posts/2025/coco-v0.3/9.png)
- Added search box widget for easy website embedding
- Added integration CRUD management and CORS configuration support
- Added attachment deletion API
- Added dynamic JS wrapper for widgets
- Added server-side document icon parsing
- Added recommended themes for widget integration
- Added sensitive field filtering support

### Bug Fixes

- Fixed provider information version issue
- Fixed data source keyword search filtering not working as expected
- Fixed unselected data source conditions not being removed in required conditions

## Download & Get Started

Coco AI v0.3 is now available for **MacOS 12 and above** and **Windows platform**. You can start using it today by downloading from the following links:

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
