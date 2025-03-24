---
title: "Protect Elasticsearch with INFINI Gateway: Block Unreasonable Queries"
meta_title: "Protect Elasticsearch with INFINI Gateway: Block Unreasonable Queries"
description: "Protect Elasticsearch with INFINI Gateway: Block Unreasonable Queries."
date: 2025-03-22T10:00:00+08:00
image: "/images/posts/2025/block-unreasonable-queries-with-infini-gateway/cover.jpg"
categories: ["Elasticsearch", "Gateway"]
author: "Frank"
tags: ["Elasticsearch", "Gateway", "Search"]
lang: "en"
category: "Technology"
subcategory: "Gateway"
draft: false
---

This article explores how to use INFINI Gateway to block unreasonable queries from being sent to Elasticsearch. The same method also applies to Opensearch and INFINI Easysearch.  

In past experiences dealing with Elasticsearch OOM (Out of Memory) issues, we found that many cases were caused by query operations leading to node OOM. After investigation, these cases mainly fall into two categories:  

1. **Excessive Query Throughput**: When the frequency or complexity of query requests exceeds the cluster's processing capacity, it can cause node memory exhaustion, triggering OOM.  
2. **Unreasonable Queries**: Certain types of queries (e.g., deeply nested queries, deep pagination, or complex aggregations) may require significant memory resources, potentially causing OOM during execution.

By identifying and optimizing these query patterns, OOM incidents can be effectively reduced. For managing excessive query throughput, refer to previous articles. Below, we focus on blocking unreasonable queries to protect cluster stability.  

## **Unreasonable Queries**
Unreasonable queries are those that consume excessive system resources (e.g., CPU, Memory), are overly complex, take too long to execute, or require extensive computational resources. These queries can lead to high load, resource exhaustion, and negatively impact cluster stability, response speed, and user experience.  

Examples of unreasonable queries include:  

+ Nested aggregation queries  
+ Complex regex-based fuzzy matching  
+ Deep pagination queries (e.g., from: 10000, size: 10)  
+ Script queries  
+ Large-scale nested aggregations

To prevent these queries from affecting Elasticsearch clusters, INFINI Gateway can be used to block them.  

## **Request Context**
INFINI Gateway provides access to a wealth of information through the request context, which serves as the entry point to access details such as request source and body. The **_ctx** keyword is used as a prefix to access this context.  
![block-unreasonable-queries-with-infini-gateway](/images/posts/2025/block-unreasonable-queries-with-infini-gateway/1.png)

For more details on available context information, refer to the [documentation](https://docs.infinilabs.com/gateway/main/docs/references/context/).  

## **Context Filter**
Context Filter is an online filter provided by INFINI Gateway that allows traffic filtering based on request context. By defining a set of matching rules, traffic can be flexibly screened. The filter supports multiple matching modes, including:  

+ Prefix matching  
+ Suffix matching  
+ Fuzzy matching  
+ Regex matching

For matched requests, they can be directly blocked (denied) with a custom response message. The key is to identify the critical keywords or characteristics of unreasonable requests.  

## **Steps to Implement**
1. **Identify Keywords**: Determine the key features or keywords in problematic query requests.  
2. **Configure Matching Rules**: Define matching rules in **context_filter**, selecting the appropriate matching mode (e.g., prefix, suffix, fuzzy, or regex).  
3. **Block Requests**: Once keywords are matched, INFINI Gateway will automatically block the request and return the specified message.

For more details, refer to the [documentation](https://docs.infinilabs.com/gateway/main/docs/references/filters/context_filter/).  

## **Example: Blocking Wildcard Queries**
Consider a wildcard query example:  

```plain
GET yf-test-1shard/_search
{
  "query": {
    "wildcard": {
      "path.keyword": {
        "value": "/a*"
      }
    }
  }
}
```
This query searches for documents where the **path** field starts with **/a**.  
![block-unreasonable-queries-with-infini-gateway](/images/posts/2025/block-unreasonable-queries-with-infini-gateway/2.png)


**Step 1**: Identify the keyword as **wildcard":** to specifically target wildcard queries. For sometimes the query URL may contain the term expand_wildcards.

**Step 2**: Edit the INFINI Gateway configuration file to add a **context_filter** rule:  

```yaml
- name: default_flow
  filter:
    - context_filter:
        context: _ctx.request.to_string
        message: 'Request blocked. Reason: Forbidden. Please contact the administrator at 010-111111.'
        status: 403
        action: deny
        must_not:
          contain:
            - "wildcard\":"
```

This rule blocks requests containing the keyword **wildcard":** and returns a custom message："Request blocked. Reason: Forbidden. Please contact the administrator at 010–111111."

**Step 3**: Test to ensure wildcard queries are blocked.  
![block-unreasonable-queries-with-infini-gateway](/images/posts/2025/block-unreasonable-queries-with-infini-gateway/3.png)

INFINI Gateway successfully blocks wildcard queries and returns the defined message. This method prevents high-resource-consuming queries from being sent to the Elasticsearch cluster, avoiding performance issues. 

