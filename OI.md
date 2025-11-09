# OI OS Integration Guide for OI-context7

This guide provides complete instructions for AI agents to install, configure, and use the OI-context7 MCP server in OI OS (Brain Trust 4).

## 🚀 Installation

### Prerequisites

| Requirement | Version        |
| ----------- | -------------- |
| **Node.js** | 18.x or higher |
| **npm**     | Latest         |
| **Git**     | Any            |
| **Context7 API Key** | Optional (recommended for higher rate limits) |

### Getting a Context7 API Key (Optional)

1. Go to [context7.com/dashboard](https://context7.com/dashboard)
2. Create an account
3. Generate an API key
4. Copy the key for use in configuration

**Note:** The API key is optional but recommended for:
- Higher rate limits
- Access to private repositories
- Better performance

### Installation Steps

1. **Clone the repository:**
   ```bash
   git clone https://github.com/OI-OS/OI-context7.git
   ```

2. **Navigate to the server directory:**
   ```bash
   cd MCP-servers/OI-context7
   ```

3. **Install dependencies:**
   ```bash
   npm install
   ```

4. **Build the project:**
   ```bash
   npm run build
   ```

5. **Connect the server to OI OS:**
   ```bash
   cd ../../ # Go back to the OI OS root directory
   ./brain-trust4 connect OI-context7 node -- "$(pwd)/MCP-servers/OI-context7/dist/index.js" --transport stdio --api-key=YOUR_API_KEY
   ```

   **Or set environment variable:**
   ```bash
   export CONTEXT7_API_KEY=YOUR_API_KEY
   ./brain-trust4 connect OI-context7 node -- "$(pwd)/MCP-servers/OI-context7/dist/index.js" --transport stdio
   ```

## 🔧 Configuration

### Environment Variables

Create a `.env` file in the server directory (optional):

```bash
cd MCP-servers/OI-context7
echo "CONTEXT7_API_KEY=YOUR_API_KEY" > .env
```

**Or set globally:**
```bash
export CONTEXT7_API_KEY=YOUR_API_KEY
```

### Authentication

The server supports API key authentication via:
- `--api-key` CLI argument (takes precedence)
- `CONTEXT7_API_KEY` environment variable

**Note:** If no API key is provided, the server will work with basic rate limits.

## 📋 Creating Intent Mappings

Intent mappings connect natural language keywords to specific MCP server tools.

**SQL to create intent mappings:**

```sql
BEGIN TRANSACTION;

-- Intent mappings for OI-context7
INSERT OR REPLACE INTO intent_mappings (keyword, server_name, tool_name, priority) VALUES
('resolve library id', 'OI-context7', 'resolve-library-id', 10),
('find library id', 'OI-context7', 'resolve-library-id', 10),
('search library', 'OI-context7', 'resolve-library-id', 10),
('get library id', 'OI-context7', 'resolve-library-id', 10),
('lookup library', 'OI-context7', 'resolve-library-id', 10),
('get library docs', 'OI-context7', 'get-library-docs', 10),
('fetch library docs', 'OI-context7', 'get-library-docs', 10),
('get library documentation', 'OI-context7', 'get-library-docs', 10),
('fetch library documentation', 'OI-context7', 'get-library-docs', 10),
('get docs for library', 'OI-context7', 'get-library-docs', 10),
('context7', 'OI-context7', 'resolve-library-id', 5),
('use context7', 'OI-context7', 'resolve-library-id', 5);

COMMIT;
```

## 📝 Creating Parameter Rules

Parameter rules define which fields are required and how to extract them from natural language queries.

**SQL to create parameter rules:**

```sql
BEGIN TRANSACTION;

-- Parameter rules for OI-context7
INSERT OR REPLACE INTO parameter_rules (server_name, tool_name, tool_signature, required_fields, field_generators, patterns) VALUES
('OI-context7', 'resolve-library-id', 'OI-context7::resolve-library-id', '["libraryName"]',
 '{"libraryName": {"FromQuery": "OI-context7::resolve-library-id.libraryName"}}', '[]'),
('OI-context7', 'get-library-docs', 'OI-context7::get-library-docs', '["context7CompatibleLibraryID"]',
 '{"context7CompatibleLibraryID": {"FromQuery": "OI-context7::get-library-docs.context7CompatibleLibraryID"}, "topic": {"FromQuery": "OI-context7::get-library-docs.topic"}, "tokens": {"FromQuery": "OI-context7::get-library-docs.tokens"}}', '[]');

COMMIT;
```

## 🔍 Parameter Extractors

Add these patterns to `parameter_extractors.toml.default`:

```toml
# ============================================================================
# OI-context7
# ============================================================================

# resolve-library-id
"OI-context7::resolve-library-id.libraryName" = "conditional:if_contains:/|then:regex:(/[a-zA-Z0-9._-]+(?:/[a-zA-Z0-9._-]+)?(?:/[a-zA-Z0-9._-]+)?)|else:remove:resolve,library,id,find,search,lookup,get,for,with,the,a,an"

# get-library-docs
"OI-context7::get-library-docs.context7CompatibleLibraryID" = "conditional:if_contains:/|then:regex:(/[a-zA-Z0-9._-]+(?:/[a-zA-Z0-9._-]+)?(?:/[a-zA-Z0-9._-]+)?)|else:remove:get,fetch,library,docs,documentation,for,with,the,a,an"
"OI-context7::get-library-docs.topic" = "regex:(?:topic|focus|about|on)[\\s:]+([^\\s]+(?:\\s+[^\\s]+)*)"
"OI-context7::get-library-docs.tokens" = "regex:(?:tokens?|max[\\s_-]?tokens?)[\\s:]+(\\d+)"
```

## 🛠️ Available Tools

### 1. `resolve-library-id`

Resolves a package/product name to a Context7-compatible library ID and returns a list of matching libraries.

**Parameters:**
- `libraryName` (required, string): The name of the library to search for

**Example:**
```bash
./oi "resolve library id for react"
./oi "find library id for next.js"
./oi "search library mongodb"
```

**Direct call:**
```bash
./brain-trust4 call OI-context7 resolve-library-id '{"libraryName": "react"}'
```

### 2. `get-library-docs`

Fetches up-to-date documentation for a library using a Context7-compatible library ID.

**Parameters:**
- `context7CompatibleLibraryID` (required, string): Exact Context7-compatible library ID (e.g., `/mongodb/docs`, `/vercel/next.js`, `/supabase/supabase`)
- `topic` (optional, string): Focus the docs on a specific topic (e.g., "routing", "hooks")
- `tokens` (optional, number, default: 5000): Maximum number of tokens to return (minimum: 1000)

**Example:**
```bash
./oi "get library docs for /vercel/next.js"
./oi "fetch library documentation for /supabase/supabase topic authentication"
./oi "get docs for /mongodb/docs tokens 10000"
```

**Direct call:**
```bash
./brain-trust4 call OI-context7 get-library-docs '{
  "context7CompatibleLibraryID": "/vercel/next.js",
  "topic": "routing",
  "tokens": 5000
}'
```

## 💡 Usage Workflow

### Typical Workflow

1. **Resolve Library ID:**
   ```bash
   ./oi "resolve library id for react"
   ```
   This returns a list of matching libraries with their Context7-compatible IDs.

2. **Get Documentation:**
   ```bash
   ./oi "get library docs for /facebook/react"
   ```
   This fetches the documentation for the resolved library ID.

### Using Library ID Directly

If you already know the Context7-compatible library ID (format: `/org/project` or `/org/project/version`), you can skip the resolve step:

```bash
./oi "get library docs for /vercel/next.js topic routing"
```

### Using Context7 in Prompts

You can add `use context7` to your prompts to automatically fetch library documentation:

```bash
./oi "Create a Next.js middleware that checks for a valid JWT in cookies. use context7"
```

## 📚 Library ID Format

Context7 uses a specific library ID format:
- **Basic format:** `/org/project` (e.g., `/vercel/next.js`)
- **With version:** `/org/project/version` (e.g., `/vercel/next.js/v14.3.0-canary.87`)

**Examples:**
- `/mongodb/docs` - MongoDB documentation
- `/vercel/next.js` - Next.js framework
- `/supabase/supabase` - Supabase client
- `/facebook/react` - React library

## ⚠️ Important Notes

1. **Library ID Resolution:** You must call `resolve-library-id` before `get-library-docs` unless the user explicitly provides a library ID in the format `/org/project` or `/org/project/version`.

2. **Token Limits:** The `tokens` parameter has a minimum of 1000. Values less than 1000 are automatically increased to 1000.

3. **API Key:** While optional, an API key is recommended for:
   - Higher rate limits
   - Access to private repositories
   - Better performance

4. **Selection Process:** When resolving library IDs, the tool prioritizes:
   - Name similarity (exact matches first)
   - Description relevance
   - Documentation coverage (higher Code Snippet counts)
   - Trust score (libraries with scores 7-10 are more authoritative)

## 🚨 Troubleshooting

### Server Not Connecting

**Error:** "Not connected" during installation

**Solution:** This is a known issue with `oi install` for Node.js servers. Manually build and connect:
```bash
cd MCP-servers/OI-context7
npm install
npm run build
cd ../../
./brain-trust4 connect OI-context7 node -- "$(pwd)/MCP-servers/OI-context7/dist/index.js" --transport stdio
```

### Library Not Found

**Error:** "Documentation not found or not finalized for this library"

**Solution:** 
1. Verify the library ID is correct (use `resolve-library-id` first)
2. Check that the library ID uses the format `/org/project` or `/org/project/version`
3. Ensure the library exists in Context7's database

### Rate Limit Errors

**Error:** Rate limit exceeded

**Solution:**
1. Get a Context7 API key from [context7.com/dashboard](https://context7.com/dashboard)
2. Set it via `--api-key` flag or `CONTEXT7_API_KEY` environment variable
3. API keys provide higher rate limits

### Parameter Extraction Issues

**Error:** Natural language queries don't extract parameters correctly

**Solution:** 
- Use direct calls for explicit control:
  ```bash
  ./brain-trust4 call OI-context7 resolve-library-id '{"libraryName": "react"}'
  ```
- Or provide library IDs directly in the format `/org/project`:
  ```bash
  ./oi "get library docs for /vercel/next.js"
  ```

## 📖 Additional Resources

- **Context7 Website:** https://context7.com
- **Context7 Dashboard:** https://context7.com/dashboard
- **GitHub Repository:** https://github.com/OI-OS/OI-context7
- **Adding Projects Guide:** See `docs/adding-projects.md` in the repository

## ✅ Verification

After installation, verify the server is working:

```bash
# List tools
./brain-trust4 tools OI-context7

# Test resolve-library-id
./oi "resolve library id for react"

# Test get-library-docs
./oi "get library docs for /facebook/react"
```

If all commands work, the server is successfully installed and configured!

