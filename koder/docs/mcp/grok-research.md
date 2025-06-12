# Model Context Protocol (MCP) Documentation

## Overview
The Model Context Protocol (MCP), introduced by Anthropic in November 2024, is an open standard designed to standardize integration between large language model (LLM) applications and external data sources or tools. Often compared to a "USB-C for AI apps," MCP enables seamless connections for AI-powered IDEs, chat interfaces, and custom workflows. It uses a client-server architecture to provide LLMs with context, enhancing their relevance and functionality.

## Components
MCP comprises several key components that facilitate its operation:

| Component          | Description                                                                 |
|--------------------|-----------------------------------------------------------------------------|
| **MCP Hosts**      | Applications (e.g., IDEs, chatbots) that leverage MCP for data/tool access.  |
| **MCP Clients**    | Connectors within hosts maintaining 1:1 connections with MCP servers.        |
| **MCP Servers**    | Lightweight programs exposing resources, prompts, and tools via MCP.         |
| **Local Data Sources** | Files, databases, or services on the user's computer accessed by servers. |
| **Remote Services** | External systems (e.g., APIs) that MCP servers connect to.                  |

## Concepts
MCP introduces several core concepts to standardize interactions:

| Concept       | Description                                                                 |
|---------------|-----------------------------------------------------------------------------|
| **Resources** | Data/context provided to users or LLMs (e.g., files, database entries).     |
| **Prompts**   | Templated messages/workflows for guiding LLM responses.                     |
| **Tools**     | Functions executable by LLMs for specific tasks (e.g., calculations, APIs). |
| **Sampling**  | Server-initiated agentic behaviors for recursive LLM interactions.          |

## Technical Description
MCP uses JSON-RPC 2.0 for communication between hosts, clients, and servers, ensuring standardized message formats. Key technical features include:

- **Capability Negotiation**: Servers and clients negotiate available features.
- **Standardized Methods**: For listing/accessing resources, prompts, and tools.
- **Security Principles**: Emphasize user consent, data privacy, and tool safety, requiring explicit user authorization for actions.

The protocol draws inspiration from the Language Server Protocol, providing a model-agnostic interface for AI applications.

## Building an MCP Client
To extend an application like the OpenAI Codex CLI to support MCP servers, developers can use the MCP TypeScript SDK. Below are the steps to build an MCP client:

1. **Install the SDK**:
   ```bash
   npm install @modelcontextprotocol/sdk
   ```

2. **Import Modules**:
   ```typescript
   import { Client } from "@modelcontextprotocol/sdk/client/index.js";
   import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
   ```

3. **Create a Transport**:
   For stdio-based servers:
   ```typescript
   const transport = new StdioClientTransport({
     command: "node",
     args: ["path/to/server.js"],
   });
   ```
   For HTTP-based servers, use `StreamableHTTPClientTransport`.

4. **Initialize the Client**:
   ```typescript
   const client = new Client({
     name: "codex-mcp-client",
     version: "1.0.0",
   });
   ```

5. **Connect to the Server**:
   ```typescript
   await client.connect(transport);
   ```

6. **Interact with Server**:
   - List resources:
     ```typescript
     const resources = await client.listResources();
     ```
   - Read a resource:
     ```typescript
     const resourceContent = await client.readResource({ uri: "file:///example.txt" });
     ```
   - Call a tool:
     ```typescript
     const toolResult = await client.callTool({
       name: "example-tool",
       arguments: { arg1: "value" },
     });
     ```

These steps enable the OpenAI Codex CLI to connect to MCP servers, retrieve context, or execute tools, enhancing its functionality.

## TypeScript SDK Documentation
The MCP TypeScript SDK, available at [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk), implements the full MCP specification. Key features include:

- **High-Level Interfaces**: Simplifies client and server development.
- **Transport Support**: Includes stdio and Streamable HTTP transports.
- **Protocol Handling**: Manages MCP messages and lifecycle events.

The SDK is published on npm as `@modelcontextprotocol/sdk` (version 1.12.1 as of May 2025). Example usage for client operations is provided above, with additional server-side examples available in the repository.

## Feasibility of Using MCP SDK
The MCP TypeScript SDK is highly suitable for building an MCP client to extend the OpenAI Codex CLI. It abstracts protocol complexities, offering straightforward methods for connecting to servers and accessing resources, prompts, and tools. Integrating this into the Codex CLI involves adding the client logic to interface with MCP servers, which the SDK simplifies significantly. The SDK’s comprehensive support for MCP features ensures that developers can efficiently enhance the CLI to leverage external data and functionalities, making the task feasible and efficient.

## Key Citations
- [Model Context Protocol GitHub Repository](https://github.com/modelcontextprotocol)
- [Model Context Protocol Specification](https://spec.modelcontextprotocol.io)
- [TypeScript SDK for Model Context Protocol](https://github.com/modelcontextprotocol/typescript-sdk)
- [Introduction to Model Context Protocol](https://modelcontextprotocol.io/introduction)
- [Anthropic's Model Context Protocol Announcement](https://www.anthropic.com/news/model-context-protocol)