---
title: "Coco AI v0.8: Window Management, View Extensions, Linux File Search and 10x more connectors"
meta_title: "Coco AI v0.8 Released"
description: "Coco AI v0.8 is now available!"
date: "2025-09-28T00:00:00+08:00"
categories: ["Release", "News"]
image: "/images/posts/2025/coco-v0.8/cover.png"
author: "SteveLauC"
tags:
  [
    "Coco AI",
    "Release",
    "AI",
    "Search",
    "Assistant",
    "Knowledge Management",
    "Open Source",
    "",
    "Cross-Platform",
  ]
lang: "en"
category: "Product"
subcategory: "Released"
draft: false
---


# Coco AI

Coco AI is an [open-source][app-github], cross-platform search and assistant system designed 
for both individuals and enterprises. Today, we are excited to announce its latest
release 0.8.0.

## Release Highlights

### Window Management Extension for macOS

We implemented a Window Management extension for macOS that allows users to move 
and resize application windows with it.

![wm](/images/posts/2025/coco-v0.8/wm_extension.gif)

Supported Commands

* Move Up/Down/Left/Right: Move the focused window to the edge of the screen 
  in any direction using keyboard shortcuts.
* Maximize: Maximize the focused window to fit the screen or maximize only width or height.
* Center: Center the focused window in the screen.
* Bottom/Left/Center/Right/Top Half: Resize the focused window to the bottom, 
left, center, right, or top half of the screen.
* First/First Two/Center/Last Two/Last Third: Resize the focused window to the 
first, first two, center, last two, or last third of the screen.
* Toggle Fullscreen: Toggle the fullscreen mode of the focused window.
* Top Left/Top Right/Bottom Left/Bottom Right Quarter: Resize the focused window
to the top left, top right, bottom left, or bottom right quarter of the screen.
* Previous/Next Display: Move the focused window to the previous or next display.
* Restore: Restore the focused window to its last position.

### View Extension

View extension is a new extension type we introduced in Coco. As its name implies,
it has a "view". For example, below is a screenshot of [2048] running within Coco:

![view](/images/posts/2025/coco-v0.8/view_extension.png)

As an user, you should expect more view extensions to come to our extension store.
As a developer, you can build your own View extensions and submit them to our 
extension store. We will have tutorials on how to do this, stay tuned.

### File Search for Linux

macOS/Windows Coco users love our file search extension, Linux users want that
as well! So in this release, we have brought it to Linux. Both GNOME and KDE desktop
environments are supported.

> Coco File Search relies on desktop environments' search functionality, if you
> do not use a DE, you probably could install the desktop search components and
> start them to make Coco File Search work.
>
> On GNOME, it is Tracker. And On KDE, it is Baloo.
>
> NOTE: We have not verified this.

### Search for apps using your own language (for macOS and Linux)

App search has been one of our most core features! In this release, we make it 
even better! Previously, Coco app only indexed the English app names, in this release,
we index the apps in:

* English
* Chinese
* Your system language

And the search results now respect your Coco app's language:

![app_search](/images/posts/2025/coco-v0.8/app_search_lang.png)

### Integrated Search Portal in Coco server

With Coco AI, setting up a search portal is simple, so users can quickly access 
the information they need, whether internally or externally.

![app_search](/images/posts/2025/coco-v0.8/coco_server_integrated_search_portal.gif)

### 10x more connectors

![app_search](/images/posts/2025/coco-v0.8/more_connectors.png)

We always want to bring more connectors to Coco AI, making searching more platforms
and datasources easier. In this release, we've unleashed a tsunami of connectivity 
with 10x more connectors:

* Postgres
* MySQL
* GitHub
* GitLab
* Gitea
* Microsoft SQL Server
* Oracle Database
* Salesforce
* Feishu/Lark

## Detailed technical release notes

### Coco App v0.8.0

#### Features

- feat: enhance ui for skipped version #834
- feat: support installing local extensions #749
- feat: support sending files in chat messages #764
- feat: sub extension can set 'platforms' now #847
- feat: add extension uninstall option in settings #855
- feat: impl extension settings 'hide_before_open' #862
- feat: index both en/zh_CN app names and show app name in chosen language #875
- feat: support context menu in debug mode #882
- feat: file search for Linux/GNOME #884
- feat: file search for Linux/KDE #886
- feat: extension Window Management for macOS #892
- feat: new extension type View #894
- feat: support opening file in its containing folder #900

#### Bug fix

- fix: fix issue with update check failure #833
- fix: web component login state #857
- fix: shortcut key not opening extension store #877
- fix: set up hotkey on main thread or Windows will complain #879
- fix: resolve deeplink login issue #881
- fix: use kill_on_drop() to avoid zombie proc in error case #887
- fix: settings window rendering/loading issue 889
- fix: ensure search paths are indexed #896
- fix: bump applications-rs to fix empty app name issue #898

#### Improvements

- refactor: calling service related interfaces #831
- refactor: split query_coco_fusion() #836
- chore: web component loading font icon #838
- chore: delete unused code files and dependencies #841
- chore: ignore tauri::AppHandle's generic argument R #845
- refactor: check Extension/plugin.json from all sources #846
- refactor: pinning window won't set CanJoinAllSpaces on macOS #854
- build: web component build error #858
- refactor: coordinate third-party extension operations using lock #867
- refactor: index iOS apps and macOS apps that store icon in Assets.car #872
- refactor: accept both '-' and '\_' as locale str separator #876
- refactor: relax the file search conditions on macOS #883
- refactor: ensure Coco won't take focus #891
- chore: skip login check for web widget #895
- chore: convertFileSrc() "link[href]" and "img[src]" #901

### Coco Server v0.8.0

#### Features

- chore: support access docs via path hierarchy manner in datasource #484
- feat: handle path_hierarchy config for document search #486
- feat: confluence wiki connector #31
- feat: extract content for notion connector #70
- feat: network storage connector #461
- feat: postgresql connector #476
- feat: mysql connector #489
- feat: github connector #492
- feat: Feishu/Lark connector #493
- feat: gitlab connector #494
- feat: salesforce connector #505
- feat: gitea connector #509
- feat: mssql connector #511
- feat: oracle connector #522

#### Bug fix

- fix: correct assistant update logic
- fix: generate unique icon key to prevent accidental deletion of all icons
- fix: modify the access_token URL during coco server login (#480) 
- fix: fix permission issue for web widget #512
- fix: extra height because of importing the icon in Searchbox #519
- fix: extra height because of importing the icon in Searchbox #519
- fix: page scrolling not working in Fullscreen #520
- fix: resolve API token list pagination issue #523
- fix: mssql paging bug #522
- fix: fix S3 connector icon #533

#### Improvements

- chore: remove unused websocket api #443
- chore: add missing root folders for gdrive #483
- chore: update default connector settings form on create/modify connector page #502
- chore: adjust title of data source detail #485
- refactor: refactoring summary processor #487
- chore: add missing docs for google drive #488
- docs: update the easysearch initial admin password to complex rule #501
- chore: unify license header #499
- chore: update default datasource edit page #506
- refactor: refactoring oauth connect component #507
- chore: set default size for datasource list to 12 #508
- chore: add search settings to settings
- chore: support page mode in integration fullscreen
- chore: add icon to list items #524
- refactor: refactoring security api for non-managed mode #527
- refactor: support access documents via path hierarchy manner for local_fs connector #530
- refactor: support access documents via path hierarchy manner for S3, Network Driver, GitHub, GitLab and Gitee connectors #532


# Getting Started

Coco AI is available on macOS, Windows, and Ubuntu. Get started by downloading it from our [website][download].


[app-github]: https://github.com/infinilabs/coco-app
[download]: https://coco.rs/download
[2048]: https://en.wikipedia.org/wiki/2048_(video_game)