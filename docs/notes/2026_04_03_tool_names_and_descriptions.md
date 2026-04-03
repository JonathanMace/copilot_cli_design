---
name: Tool Names and Descriptions
description: Verbatim list of the currently visible tool names and top-level descriptions in prompt order.
copilot_version: 1.0.17
---

# Tool Names and Descriptions

This note records the tool names and their top-level descriptions exactly as they appear in the currently visible tool registry, in prompt order.

| Tool | Description |
|---|---|
| `functions.powershell` | Runs a PowerShell command in an interactive PowerShell session. |
| `functions.write_powershell` | Sends input to the specified command or PowerShell session. |
| `functions.read_powershell` | Reads output from a PowerShell command. |
| `functions.stop_powershell` | Stops a running PowerShell command. |
| `functions.list_powershell` | Lists all active PowerShell sessions. |
| `functions.store_memory` | Store a fact about the codebase in memory, so that it can be used in future code generation or review tasks. The fact should be a clear and concise statement about the codebase conventions, structure, logic, or usage. It may be based on the code itself, or on information provided by the user. |
| `functions.view` | Tool for viewing files and directories. |
| `functions.web_fetch` | Fetches a URL from the internet and returns the page as either markdown or raw HTML. Use this to safely retrieve up-to-date information from HTML web pages. |
| `functions.report_intent` | Use this tool to update the current intent of the session. This is displayed in the user interface and is important to help the user understand what you're doing. |
| `functions.show_file` | Show file contents to the user. Unlike the view tool (which reads files into your context), this tool is for presenting code to the user in a prominent, always-visible way. |
| `functions.fetch_copilot_cli_documentation` | Fetches documentation about you, the GitHub Copilot CLI, and your capabilities. Use this tool when the user asks how to use you, what you can do, or about specific features of the GitHub Copilot CLI. |
| `functions.skill` | Execute a skill within the main conversation |
| `functions.ask_user` | Ask the user a question and wait for their response. |
| `functions.sql` | Execute SQL queries against SQLite databases. This tool provides two databases: |
| `functions.read_agent` | Retrieves the status and results of a background agent. |
| `functions.list_agents` | Lists all active and completed background agents. |
| `functions.write_agent` | Sends a message to a running or idle background agent, delivered as a new user turn in the agent's conversation. |
| `functions.rg` | Fast and precise code search using ripgrep. Search for patterns in file contents. |
| `functions.glob` | Fast file pattern matching using glob patterns. Find files by name patterns. |
| `functions.task` | Custom agent: Launch specialized agents in separate context windows for specific tasks. |
| `functions.github-mcp-server-actions_get` | Get details about specific GitHub Actions resources. |
| `functions.github-mcp-server-actions_list` | Tools for listing GitHub Actions resources. |
| `functions.github-mcp-server-get_commit` | Get details for a commit from a GitHub repository |
| `functions.github-mcp-server-get_copilot_space` | This tool can be used to provide additional context to the chat from a specific Copilot space. If the user mentions the keyword 'Copilot space' with the name and owner of the space, execute this tool. |
| `functions.github-mcp-server-get_file_contents` | Get the contents of a file or directory from a GitHub repository |
| `functions.github-mcp-server-get_job_logs` | Get logs for GitHub Actions workflow jobs. |
| `functions.github-mcp-server-issue_read` | Get information about a specific issue in a GitHub repository. |
| `functions.github-mcp-server-list_branches` | List branches in a GitHub repository |
| `functions.github-mcp-server-list_commits` | Get list of commits of a branch in a GitHub repository. Returns at least 30 results per page by default, but can return more if specified using the perPage parameter (up to 100). |
| `functions.github-mcp-server-list_copilot_spaces` | Retrieves the list of Copilot Spaces accessible to the user, including their names and owners. |
| `functions.github-mcp-server-list_issues` | List issues in a GitHub repository. For pagination, use the 'endCursor' from the previous response's 'pageInfo' in the 'after' parameter. |
| `functions.github-mcp-server-list_pull_requests` | List pull requests in a GitHub repository. If the user specifies an author, then DO NOT use this tool and use the search_pull_requests tool instead. |
| `functions.github-mcp-server-pull_request_read` | Get information on a specific pull request in GitHub repository. |
| `functions.github-mcp-server-search_code` | Fast and precise code search across ALL GitHub repositories using GitHub's native search engine. Best for finding exact symbols, functions, classes, or specific code patterns. |
| `functions.github-mcp-server-search_issues` | Search for issues in GitHub repositories using issues search syntax already scoped to is:issue |
| `functions.github-mcp-server-search_pull_requests` | Search for pull requests in GitHub repositories using issues search syntax already scoped to is:pr |
| `functions.github-mcp-server-search_repositories` | Find GitHub repositories by name, description, readme, topics, or other metadata. Perfect for discovering projects, finding examples, or locating specific repositories across GitHub. |
| `functions.github-mcp-server-search_users` | Find GitHub users by username, real name, or other profile information. Useful for locating developers, contributors, or team members. |
| `functions.web_search` | This tool performs an AI-powered web search to provide intelligent, contextual answers with citations. |
| `functions.extensions_reload` | Reload all extensions from disk. Call this after creating or modifying extension files in .github/extensions/ or the user extensions directory. New tools are available immediately after reload. |
| `functions.extensions_manage` | Manage the Copilot CLI extension ecosystem. Use "list" to see all loaded extensions, "inspect" to get details about a specific extension, "scaffold" to generate a skeleton extension file, or "guide" for a comprehensive authoring guide with examples. Extensions can live in the project (.github/extensions/) or in the user's personal extensions directory. Always call with operation "guide" first before writing any extension code. |
| `functions.apply_patch` | Use the `apply_patch` tool to edit files. This is a FREEFORM tool, so do not wrap the patch in JSON. |
| `multi_tool_use.parallel` | This tool serves as a wrapper for utilizing multiple developer tools. Only tools defined in developer messages (any developer message in the conversation) are permitted to be called in this tool. Calling system tools will result in errors. Ensure that the parameters provided to each tool are valid according to that tool's specification. |
