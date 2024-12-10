---
title: "Adding Searchbox to Hugo!"
meta_title: "Adding Searchbox to Hugo Blog!"
description: "How to add a searchbox to your hugoblog site."
date: 2024-12-10T05:00:00Z
image: "/images/posts/2024/adding-searchbox-to-blog-site.png"
categories: ["Pizza", "Search"]
author: "Medcl"
tags: ["search", "searchbox", "hugo", "pizza","pizza-searchbox", "pizza-wasm", "wasm", "hugo"]
draft: false
---

As a static website, having a search function significantly enhances the user experience, allowing readers to locate content quickly and efficiently. Whether you’re managing a personal blog or a comprehensive documentation site, a search box is an essential feature to improve navigation and engagement.  

In this article, we’ll guide you through the step-by-step process of integrating a search box into your Hugo-powered site. We’ll be using **Pizza**, a blazing-fast, WASM-based search engine that outperforms traditional JavaScript-based solutions. Pizza supports the complete Lucene query syntax, offering robust and versatile search capabilities for your site. From setup to customization, we’ll cover everything you need to create a powerful and seamless search experience.  

## What is INFINI Pizza?

INFINI Pizza is an upcoming search engine developed by INFINI Labs, written in Rust (soon to be fully open-sourced). The basic search capabilities have already been completed, and based on the core engine of INFINI Pizza, a WASM version of the ultra-lightweight kernel is provided. This can be easily embedded into various application systems, such as websites, especially static sites or small blog systems.

Currently, The website of INFINI Pizza have integrated INFINI Pizza for WebAssembly. The specific search results are shown in the image below:

![INFINI Pizza Search](/images/posts/2024/pizza-search-box-1.png)

## How to use INFINI Pizza for WebAssembly?
Visit the website ([http://pizza.rs](http://pizza.rs/)), and by pressing the shortcut key 's', you can bring up the search box and experience the search capabilities provided by INFINI Pizza. Notably, during the search process, all of your actions are executed locally in the browser. Unlike traditional search implementations, where each query requires an interaction with a backend search server, INFINI Pizza for WebAssembly operates entirely offline. Even if you're disconnected from the internet, you can still enjoy a seamless search experience.

Without further ado, let's dive into how you can use INFINI Pizza for WebAssembly on your own site.

First, INFINI Pizza for WebAssembly is open source. You can find the GitHub repository here: https://github.com/infinilabs/pizza-wasm. The compiled WASM package is available for direct download here: https://github.com/infinilabs/pizza-wasm/tree/main/pkg.

```shell
    ➜  wasm git:(main) ✗ du -sh pkg/*
    4.0K    pkg/README.md
    4.0K    pkg/package.json
    4.0K    pkg/pizza_wasm.d.ts
    4.0K    pkg/pizza_wasm.js
     12K    pkg/pizza_wasm_bg.js
    580K    pkg/pizza_wasm_bg.wasm
    4.0K    pkg/pizza_wasm_bg.wasm.d.ts
    256K    pkg/pizza_wasm_bg.wasm.gz
```

You'll notice that the WASM package is just over 500 KB, and after Gzip compression, it’s reduced to just over 200 KB, making it quite lightweight. You may have thinking there are some JavaScript based static search solution maybe more smaller, but please note they are not full featured search engine, and I will explain this when INFINI Pizza officially reveal sometime later.

Pizza-WASM is a WebAssembly interface wrapper for the core engine of INFINI Pizza, exposing only a few simple access interfaces. These are sufficient for current frontend search applications. You can find a very basic example of how to call the WASM methods in the following directory: https://github.com/infinilabs/pizza-wasm/tree/main/web.

However, just having the Pizza WASM isn't enough. If we want to add a search box to an existing static site, we also need to consider where the data comes from and how the results are displayed. To address this, we've wrapped up a project called Pizza-searchbox, which further simplifies usage. This project is also open source, and you can find it on GitHub here: https://github.com/infinilabs/pizza-searchbox.

Since the example project has already uploaded the compiled code and samples, we can directly download the source code and preview the functionality locally.

```shell
    ➜  /tmp git clone https://github.com/infinilabs/pizza-searchbox.git
    Cloning into 'pizza-searchbox'...
    remote: Enumerating objects: 174, done.
    remote: Counting objects: 100% (174/174), done.
    remote: Compressing objects: 100% (112/112), done.
    remote: Total 174 (delta 86), reused 147 (delta 59), pack-reused 0 (from 0)
    Receiving objects: 100% (174/174), 941.94 KiB | 1.20 MiB/s, done.
    Resolving deltas: 100% (86/86), done.
    ➜  /tmp cd pizza-searchbox/example/dist 
    ➜  dist git:(main) python3 -m http.server 8083    
    
    Serving HTTP on :: port 8083 (http://[::]:8083/) ...
```

Open your browser and visit: [http://localhost:8083](http://localhost:8083/), as shown below:

![INFINI Pizza Search](/images/posts/2024/pizza-search-box-2.gif)

Observe the network requests in the browser, and you'll see that it loads the sample index.json data:

![INFINI Pizza Search](/images/posts/2024/pizza-search-box-3.png)

In practice, if it's our own static website or blog, simply ensuring that this file (index.json) with the appropriate format exists in the root directory of the site is enough to quickly integrate the search functionality you see into your own website. OK, with the functionality verified, let's start integrating it into our site.

The official website of Pizza/INFINI Labs is statically generated using Hugo, and the index.json file does not need to be manually created. First, we need to enable Hugo to generate content in JSON format, which is a built-in capability of Hugo. We'll need to modify the configuration of the Hugo project:

![INFINI Pizza Search](/images/posts/2024/pizza-search-box-4.png)


Add a JSON output option to the outputs parameter, and then define the JSON output format template in the theme's template files:

![INFINI Pizza Search](/images/posts/2024/pizza-search-box-5.png)


The text content is as follows for easy copying and pasting. Save the file with the name index.json:

```shell
    {{- $index := slice -}}
    {{- range where .Site.RegularPages.ByDate.Reverse "Type" "not in" (slice "page" "json") -}}
        {{- $index = $index | append (dict "title" (.Title | plainify) "url" .Permalink "tags" .Params.tags "category" .Params.category "subcategory" .Params.subcategory "summary" (.Params.Summary | markdownify | plainify) "content" (.Content | markdownify | plainify)) -}}
    {{- end -}}
    {{- $index | jsonify -}}
```
OK, next, we need to add the tags we’ve used above to the metadata of each article or blog post on the site:

![INFINI Pizza Search](/images/posts/2024/pizza-search-box-6.png)


OK, start the Hugo site:

```shell
    hugo server --forceSyncStatic   --minify --theme book
    
    
                       | EN   
    -------------------+------
      Pages            | 181  
      Paginator pages  |   5  
      Non-page files   |   0  
      Static files     | 110  
      Processed images |   0  
      Aliases          |  52  
      Sitemaps         |   1  
      Cleaned          |   0  
    
    Built in 323 ms
    Watching for changes in /Users/medcl/Documents/rust/pizza/website/{assets,content.en,static,themes}
    Watching for config changes in /Users/medcl/Documents/rust/pizza/website/config.yaml
    Environment: "development"
    Serving pages from memory
    Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
    Web Server is available at //localhost:1313/ (bind address 127.0.0.1)
    Press Ctrl+C to stop
```

Open the Hugo site address and try accessing http://localhost:1313/index.json. You should be able to access the JSON file:

![INFINI Pizza Search](/images/posts/2024/pizza-search-box-7.png)

## Integrate the search widget to Hugo

At this point, the data preparation is complete. Next, we'll integrate the frontend search widget.

Do you remember the resource files we downloaded from Pizza-searchbox earlier? We mainly use the three files in the assets folder:

```shell
    /tmp/pizza-searchbox/example/dist
    ➜  dist git:(main) tree    
    .
    ├── assets
      ├── index-C1z1vz3D.css
      ├── index-D_gOo737.js
      └── pizza_wasm_bg-BRCuviY_.wasm
    ├── index.html
    └── index.json
    
    1 directory, 5 files
    ➜  dist git:(main) 
```

Open the index.html file, and you will see the following content:
![INFINI Pizza Search](/images/posts/2024/pizza-search-box-8.png)

In order to integrate with the searchbox, we just need to add the three highlighted sections of code into our own static website.

Copy the files from the assets directory to our Hugo site, in the following location:
![INFINI Pizza Search](/images/posts/2024/pizza-search-box-9.png)

Then, modify the Hugo theme templates by adding a segment of code to the html-head.html file in the header template of all pages to load our CSS stylesheet:
![INFINI Pizza Search](/images/posts/2024/pizza-search-box-10.png)

Next, continue modifying the Hugo theme template files by adding a segment of code to the footer template of all pages to load the JS script files:
![INFINI Pizza Search](/images/posts/2024/pizza-search-box-11.png)

Next, insert a `docsearch` tag in the appropriate place within the page template to place the search box, as shown in the image:
![INFINI Pizza Search](/images/posts/2024/pizza-search-box-12.png)

> the tag `docsearch` may be renamed  to `searchbox` in the future

With that, the task is complete!

Open your browser to see the final result:
![INFINI Pizza Search](/images/posts/2024/pizza-search-box-13.png)

Finally, to summarize, with the help of the 3 lines of code and copy 3 files from INFINI Pizza searchbox, you can add a lightweight offline search functionality to your static site in just 5 minutes. Give it a try!


## References

* [https://pizza.rs](https://pizza.rs/)
* [https://github.com/infinilabs/pizza-wasm](https://github.com/infinilabs/pizza-wasm)
* [https://github.com/infinilabs/pizza-searchbox](https://github.com/infinilabs/pizza-searchbox)