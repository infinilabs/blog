---
title: "Coco AI v0.7: A Brand New File Search Experience and Fullscreen Integrations"
meta_title: "Coco AI v0.7 Released"
description: "Coco AI v0.7 is now available! A powerful, open-source AI-powered search and productivity tool for seamless access to personal and enterprise knowledge."
date: "2025-07-28T00:00:00+08:00"
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

# Coco AI v0.7 Released

We’re excited to announce the official release of **Coco AI v0.7** – the latest stable version of our powerful, open-source, cross-platform unified AI search and productivity tool!

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

## Coco App v0.7

### 🚀 Features

- Introduced file search functionality using Spotlight for enhanced performance.
- Enabled voice input for both search and chat, allowing for a hands-free experience.
- Upgraded the text-to-speech engine with a powerful Large Language Model (LLM) for more natural voice output.
- Added file search capabilities for Windows users.

### 🐛 Bug Fixes

- Corrected the file search logic to apply filters before pagination parameters, ensuring accurate results.
- Resolved an issue where searching by both name and content failed to include the file name in the search criteria.
- Fixed a bug that caused the application window to unexpectedly hide when moved on Windows.
- Ensured that hotkeys are properly unregistered when an extension is deleted.
- Application indexing now correctly respects the defined search scope configurations.
- Restored missing category titles on subpages for better navigation.
- Addressed a display issue with the assistant during quick AI access.
- Fixed minor playback issues in the voice feature.
- Corrected the taskbar icon display on Linux systems.
- Resolved a data inconsistency problem that occurred on secondary pages.
- Fixed an issue where an incorrect status was shown during extension installation.
- Increased the `read_timeout` for HTTP streaming to improve stability.
- Addressed issues related to the 'Enter' key functionality.
- Resolved a selection problem that occurred after renaming items.
- Fixed a shortcut issue within the Windows context menu.
- Prevented a panic that occurred from calling `state()` before `manage()` was initialized.
- Resolved an issue with handling multiline input.
- Fixed the `Ctrl+K` keyboard shortcut.
- Ensured that the update window configuration is synchronized correctly.
- Corrected the 'Enter' key behavior on subpages.
- Fixed a panic that occurred on Ubuntu (GNOME) when launching applications.
- Further improved the 'Enter' key behavior for consistency.
- Resolved an issue that caused update checks to fail.

### ✈️ Improvements

- Optimized file type detection by prioritizing `stat(2)` when checking for directories.
- Standardized the File Search extension type to 'extension' for consistency.
- Refactored the chat functionality by creating a dedicated `send chat` API.
- Expanded icon support to include a wider variety of file types.
- Replaced `meval-rs` with an internal fork to resolve dependency warnings.
- Adjusted interface parameters for the assistant, data source, and MCP server.
- Restructured the extension code hierarchy for better organization.
- Updated the `applications-rs` dependency to the latest version.
- Standardized the naming of 'Quicklink' for consistency across the codebase.
- Improved assistant parameters and styling.
- Ensured that optional fields in our data models are now correctly treated as optional.
- Enhanced search-chat components to include `formatUrl`, thinking data, and icon URLs.
- Added custom HTTP request headers for the Coco app.
- Improved robustness by checking the HTTP status code before attempting to deserialize the response.
- Made the splash screen responsive to adapt to mobile device widths.
- Added language and `formatUrl` parameters to the search-chat feature.
- Prevented unnecessary API requests when the user is not logged in.
- Improved Windows Search by cleaning up unsupported characters from the query string.
- Enhanced debugging by displaying a backtrace in panic logs.
- Added a new notification component to the web interface.
- Refined window collection behavior to default to `MoveToActiveSpace`, using `CanJoinAllSpaces` only when a window is pinned.

### 📸 Screenshots

![coco-app](/images/posts/2025/coco-v0.7/coco-app.gif)

## Coco Server v0.7

### ❌ Breaking Changes

- Refactored data mappings for improved consistency and performance.

### 🚀 Features

- Introduced a new chat API based on HTTP streaming for real-time communication.
- Added configuration options for file uploads.
- Enabled support for sending attachments in chat messages.
- Implemented logging for LLM requests to facilitate easier debugging.
- Introduced a new RSS connector to integrate content from RSS feeds.
- Added support for configuring default model reasoning settings upon initialization.
- Introduced a new connector for the local file system.
- Introduced a new connector for Amazon S3.

### 🐛 Bug Fixes

- Resolved an issue where the "filter" query parameter was not functioning correctly.
- Corrected a bug that prevented pagination from working in lists.
- Fixed an issue causing local icons to fail to display when offline.
- Corrected the status display for providers in the LLM provider list.
- Resolved issues with the chat API when handling attachments.
- Prevented a nil pointer exception that occurred during LLM intent parsing when an error was encountered.
- Fixed a bug that prevented the deletion of multiple URL input boxes.
- Ensured that local model providers are correctly updated after being enabled.
- Guaranteed that the correct data source is utilized during Retrieval-Augmented Generation (RAG) processing.
- Resolved an issue that caused the wrong prompt template to be selected.
- Prevented the loss of a reply message when a user cancels an ongoing reply.
- Made the initial chat message in a conversation cancelable.

### ✈️ Improvements

- Refactored the user ID system for better scalability and management.
- Improved data processing by skipping empty response chunks.
- Refactored the query system for enhanced performance and flexibility.
- Enhanced security by masking more sensitive information in search results.
- Refactored the attachment API for improved reliability and functionality.
- Added upload settings to the assistant configuration.
- Refactored the Object-Relational Mapping (ORM) and security interfaces.
- Removed the session_id check from the attachment upload API for simplified integration.
- Added a `formatUrl` option to the search box for better URL handling.
- Added a fullscreen mode for integrations.
- Improved system stability by ignoring invalid connectors.
- Improved system stability by skipping invalid MCP servers.
- Improved the user interface by hiding the delete button for built-in assistants and providers.
- Ensured that a default value is correctly handled for prompt templates.
- The button preview is now automatically disabled if the corresponding integration is disabled.
- Manually flushed the first line of output to ensure immediate display.

### 📸 Screenshots

![coco-server-3](/images/posts/2025/coco-v0.7/coco-server-3.png)

![coco-server-1](/images/posts/2025/coco-v0.7/coco-server-1.jpeg)

![coco-server-2](/images/posts/2025/coco-v0.7/coco-server-2.jpeg)

## Download & Get Started

Coco AI v0.7 is now available for **MacOS 12 and above**, **Windows 10 and above**, and **Linux Ubuntu**. You can start using it today by downloading from the following links:

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
