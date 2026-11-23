# Custom MCP Servers

Model Context Protocol (MCP) servers are how Almanac connects to external data sources. Whether you're using built-in integrations (Slack, GitHub, Notion) or building your own, this section explains how to create and configure data sources.

## What is an MCP Server?

An MCP server is a standardized interface that exposes data from any API or service. Think of it as a translator that speaks your API's language and converts it to a format Almanac understands.

```
Your API → MCP Server → Almanac
```

**Examples:**

- **Slack MCP Server**: Exposes Slack's API (channels, messages, users)
- **GitHub MCP Server**: Exposes GitHub's API (repos, issues, PRs)
- **Custom MCP Server**: Your proprietary system, database, or API

## Why MCP?

**Standardization**: Write once, works everywhere

- Same protocol for all data sources
- No custom integration code per source
- Works with any MCP-compatible tool (Claude Desktop, Cline, Zed, etc.)

**Flexibility**: Any data source

- REST APIs
- GraphQL endpoints
- Databases
- File systems
- Custom protocols

**Security**: Built-in OAuth support

- Standard OAuth 2.0 and 2.1 flows
- Secure credential storage
- Token refresh handling

### OAuth Flow Notes

**OAuth 2.0**:

- Requires both **client ID** and **client secret** to be manually entered
- Client secret must be kept confidential
- Suitable for server-to-server authentication

**OAuth 2.1**:

- Streamlined authentication flow
- Uses PKCE (Proof Key for Code Exchange) by default
- Does not require client secret for public clients
- Enhanced security for mobile and single-page applications
- Recommended for new implementations

Almanac supports both OAuth 2.0 and 2.1, allowing you to choose the appropriate flow based on your security requirements and application type.

## Two Ways to Configure

Almanac offers two methods to create indexing configurations:

### 1. Assisted UI Method (Recommended)

Let Almanac automatically analyze your MCP server and generate the configuration:

- ✅ **Zero Code**: Point and click interface
- ✅ **Automatic**: Discovers tools, classifies them, generates config
- ✅ **Validated**: Tests config before saving
- ✅ **Iterative**: Auto-fixes common errors

**Best for**: Most use cases, especially when starting

[Learn More →](auto-config.md)

### 2. Manual JSON Method

Write the configuration file yourself:

- ✅ **Full Control**: Specify exact behavior
- ✅ **Advanced Features**: Custom transformations, grouping, relationships
- ✅ **Version Control**: Track changes in Git
- ✅ **Reusable**: Share configs across teams

**Best for**: Complex requirements, advanced users

Read the manual JSON example below for the current configuration structure and validation limits.

## Quick Example: Connecting a Data Source

### Using the UI (Assisted Method)

```
1. Go to Data Sources page
2. Click "Add Source"
3. Select your MCP server (or add custom)
4. Click "Generate Config"
5. Review and approve
6. Start syncing!
```

Total time: **~2 minutes**

### Using JSON (Manual Method)

An indexing configuration tells Almanac which MCP tools to call, how to select
records from their responses, and how to map those records into stored documents.
It is separate from the data source's connection and OAuth settings.

This example follows the current [`SyncConfig` type](https://github.com/tryprotege/almanac/blob/849d8e0ae7fe2b9fcb75994094f1b211b96864c5/packages/indexing-engine/src/types/config.ts).
Use it when you need a small, explicit configuration rather than a generated one.
The example reads selected product insights from [ClosedLoop AI](https://closedloop.sh/).
It is a manual configuration example, not a claim that this connection has been
tested against a live workspace.

#### Choose a bounded input

ClosedLoop AI exposes `get_insight`, a read tool that accepts one full insight ID
and returns its title, content, and supporting context. That makes it suitable for
an operator-selected collection: for example, the product insights chosen for one
planning review. The example fetches those IDs again when the configuration runs.

It does not discover every insight in a workspace. A ranked search result is not
a complete inventory, and Almanac normally skips tools classified as `search`
during indexing. Keep `get_insight` classified as `read`; do not change a search
tool's category to make it run.

Before using real IDs, connect an authorized ClosedLoop AI workspace through the
normal data source setup and review which data Almanac will retain. Connection,
OAuth consent, and a live tool call are outside this example. Do not put tokens
or other connection credentials in the indexing configuration.

#### Write the configuration

The following JSON is the `SyncConfig` object, not the enclosing connection
configuration. Use the configured MCP data source's name consistently for
`source`; `closedloop-ai` is the example name here.

```json
{
  "version": "1.0",
  "source": "closedloop-ai",
  "displayName": "Selected ClosedLoop AI product insights",
  "startingPoints": [
    {
      "name": "insightIds",
      "description": "Full IDs of the product insights selected for this collection",
      "required": true,
      "userProvided": true
    }
  ],
  "fetchers": {
    "selected_insights": {
      "tool": "get_insight",
      "seedFrom": {
        "startingPoint": "insightIds",
        "paramMapping": {
          "id": "$"
        },
        "concurrency": 1,
        "continueOnError": false
      },
      "resultPath": "$.content[0].text",
      "arrayPath": "$"
    }
  },
  "recordTypes": {
    "insight": {
      "name": "insight",
      "fetcher": "selected_insights",
      "idField": "$.id",
      "detection": {
        "condition": "typeof record.id === 'string' && typeof record.title === 'string' && typeof record.content === 'string'"
      },
      "fields": {
        "title": {
          "type": "path",
          "path": "$.title"
        },
        "content": {
          "type": "path",
          "path": "$.content"
        }
      }
    }
  },
  "toolClassifications": {
    "get_insight": {
      "toolName": "get_insight",
      "category": "read"
    }
  },
  "syncOrder": ["selected_insights"]
}
```

Starting point values are stored separately from that object. These two full IDs
are synthetic fixtures, not existing product insights:

```json
{
  "insightIds": [
    "11111111-1111-4111-8111-111111111111",
    "22222222-2222-4222-8222-222222222222"
  ]
}
```

Supply the `insightIds` key with your selected values through the normal starting
point input. Do not paste the synthetic IDs into a live sync and expect records.
The stored configuration model keeps these values in `startingPointValues`.

#### Follow one record through the configuration

1. `seedFrom.startingPoint` names the input collection. For a string starting
   value, the fetcher treats `$` as the whole value and supplies it as the tool's
   `id` argument. `concurrency: 1` processes the selected values sequentially.
2. The MCP tool result contains a text block holding JSON. `resultPath` records
   that expected envelope. The current fetcher parses the text block directly;
   `resultPath` is not an arbitrary response transformation mechanism.
3. `arrayPath: "$"` selects the whole parsed object as one record. A tool returning
   an object containing an array would need a path to the array's individual
   entries instead.
4. `recordTypes.insight.fetcher` associates the record type with the fetcher key.
   The detection expression admits objects with the three expected string fields.
   Detection expressions are executable configuration: review them as code.
5. `idField` selects the source ID. `fields.title` and `fields.content` are mapping
   objects, not bare JSONPath strings. No `fields.sourceId` mapping is needed.

For example, an MCP text block could contain this synthetic payload:

```json
{
  "id": "11111111-1111-4111-8111-111111111111",
  "title": "Explain a saved filter before sharing it",
  "content": "The planning review selected a product insight about making saved filter conditions visible before a view is shared."
}
```

With data source name `closedloop-ai`, transformation produces the identity
`closedloop-ai_insight_11111111-1111-4111-8111-111111111111`, the same source ID,
and the payload's title and content. Re-fetching the same source ID and record
type under the same data source name targets the same stored `_id`.

#### Understand storage and updates

The transformer builds `_id` from the actual data source name, record type name,
and source ID. Keep those names stable if subsequent runs should update the same
records. The sync storage path upserts by `_id`, replaces the stored fields, and
increments the stored version. Renaming the source or record type changes the
identity; this example does not provide a migration between names.

This example omits incremental sync and source timestamps. The insight's source
date is not an update cursor. Selected-ID reads do not establish a complete
workspace inventory or a deletion policy. Removing an ID from the input list is
not an instruction to delete its previously stored record.

Field selection is not redaction. The transformer also retains the original
record in `rawData`, and real insight responses can contain evidence, customer
details, or conversation references beyond the mapped title and content. Review
workspace access, storage, and logging before using real data. Do not use a real
response as a public test fixture.

#### Validate in layers

- Check the object against `SyncConfig`, including required `resultPath`,
  `recordTypes.*.name`, `fetcher`, `detection`, and the mapping object shapes.
- Exercise response parsing and transformation with synthetic MCP text results.
  Check the expected ID, title, content, and retained raw record. Include a missing
  ID, a changed title with the same ID, and an empty selected-ID list.
- Treat the import screen's JSON/shape check as an initial check. The current dry
  run fetches a page directly; it does not exercise this `seedFrom` loop and its
  supplied starting points. A green import message alone does not validate this
  selected-ID workflow.
- After connection and data access are authorized, separately test a small
  selected set through the real connection. Inspect failures and the resulting
  records before expanding it. No live connection check is reported here.

`continueOnError: false` makes a rejected selected-value call stop that fetcher's
collection. It does not make a complete sync transactional, disable retries in
the MCP client, or turn a returned error payload into a thrown exception.

For the implementation behind these details, see the
[`seedFrom` fetcher](https://github.com/tryprotege/almanac/blob/849d8e0ae7fe2b9fcb75994094f1b211b96864c5/packages/server/src/services/indexing/config/paginated-fetcher.ts),
[record transformer](https://github.com/tryprotege/almanac/blob/849d8e0ae7fe2b9fcb75994094f1b211b96864c5/packages/indexing-engine/src/executor/transformer.ts),
[configuration validator](https://github.com/tryprotege/almanac/blob/849d8e0ae7fe2b9fcb75994094f1b211b96864c5/packages/server/src/services/indexing/config/config-validator.service.ts),
and [sync storage path](https://github.com/tryprotege/almanac/blob/849d8e0ae7fe2b9fcb75994094f1b211b96864c5/packages/server/src/services/sync/sync.service.ts).

Contributed by ClosedLoop Labs LLC, the company behind ClosedLoop AI.

## Built-In MCP Servers

Almanac includes several pre-built MCP servers:

### Slack

**What it indexes:**

- Channels
- Messages
- Threads
- Users

**OAuth**: ✅ Supported  
**Config**: Auto-generated

### GitHub

**What it indexes:**

- Repositories
- Issues
- Pull Requests
- Commits
- Discussions

**OAuth**: ✅ Supported  
**Config**: Auto-generated

### Notion

**What it indexes:**

- Databases
- Pages
- Blocks

**OAuth**: ✅ Supported  
**Config**: Auto-generated

### Fathom

**What it indexes:**

- Website analytics
- Page views
- Events

**OAuth**: ✅ Supported  
**Config**: Auto-generated

## Creating Your Own MCP Server

Building a custom MCP server is straightforward:

### Option 1: Use MCP SDK

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';

const server = new Server({
  name: 'my-api',
  version: '1.0.0',
});

// Define a tool
server.tool(
  'list_items',
  'Get all items',
  {
    limit: { type: 'number', description: 'Max items to return' },
  },
  async (args) => {
    const items = await fetchFromYourAPI(args.limit);
    return { items };
  },
);
```

### Option 2: Use HTTP Transport

Expose tools via HTTP endpoints:

```typescript
app.post('/mcp/tools/list_items', async (req, res) => {
  const { limit } = req.body;
  const items = await fetchFromYourAPI(limit);
  res.json({ items });
});
```

[Full Guide →](creating-servers.md)

## Configuration Workflow

Here's the complete flow from MCP server to searchable data:

```
1. MCP Server Created/Connected
         ↓
2. Almanac Discovers Tools
   - list_channels
   - get_messages
   - search_users
         ↓
3. Almanac Classifies Tools
   - Read: list_channels, get_messages
   - Write: send_message
   - Search: search_users
         ↓
4. Almanac Generates Config
   - Creates fetchers
   - Maps fields
   - Defines record types
         ↓
5. Almanac Tests Config
   - Dry run with sample data
   - Auto-fixes errors
   - Validates output
         ↓
6. Config Saved
         ↓
7. Data Synced
   - Fetches records
   - Transforms data
   - Stores in MongoDB
         ↓
8. Data Indexed
   - Vector embeddings (Qdrant)
   - Knowledge graph (Memgraph)
         ↓
9. Ready to Query! 🎉
```

## Common Use Cases

### Internal APIs

**Scenario**: Index your company's internal tools

```typescript
// Customer support tickets
server.tool("list_tickets", ...);
server.tool("get_ticket_details", ...);

// Product analytics
server.tool("list_features", ...);
server.tool("get_usage_stats", ...);
```

### Databases

**Scenario**: Index PostgreSQL tables

```typescript
server.tool("query_users", ...);
server.tool("query_orders", ...);
server.tool("query_products", ...);
```

### File Systems

**Scenario**: Index local documentation

```typescript
server.tool("list_markdown_files", ...);
server.tool("read_file", ...);
```

### Third-Party APIs

**Scenario**: Index external services

```typescript
// Zendesk
server.tool("list_zendesk_tickets", ...);

// Jira
server.tool("list_jira_issues", ...);

// Salesforce
server.tool("list_opportunities", ...);
```

## Best Practices

### Tool Design

✅ **DO**: Make tools focused and specific

```typescript
// Good
server.tool("list_open_issues", ...);
server.tool("list_closed_issues", ...);

// Bad
server.tool("list_all_issues_with_filters", ...);
```

✅ **DO**: Use pagination for large datasets

```typescript
server.tool("list_items", {
  limit: { type: "number" },
  offset: { type: "number" },
}, ...);
```

✅ **DO**: Return structured data

```typescript
// Good
return {
  items: [
    { id: 1, title: 'Item 1', description: '...' },
    { id: 2, title: 'Item 2', description: '...' },
  ],
};

// Bad
return 'Item 1: description\nItem 2: description';
```

### Security

✅ **DO**: Use OAuth for authentication  
✅ **DO**: Validate all inputs  
✅ **DO**: Rate limit your endpoints  
✅ **DO**: Encrypt sensitive data

❌ **DON'T**: Store plaintext credentials  
❌ **DON'T**: Expose internal endpoints publicly  
❌ **DON'T**: Return sensitive data in tool responses

## Testing Your MCP Server

Before connecting to Almanac, test your MCP server:

### 1. Manual Testing

```bash
# Call a tool directly
curl http://localhost:3000/mcp/tools/list_items \
  -H "Content-Type: application/json" \
  -d '{"limit": 10}'
```

### 2. MCP Inspector

Use the official MCP inspector tool:

```bash
npx @modelcontextprotocol/inspector my-server
```

### 3. Integration Testing

Connect to Almanac in development mode:

```bash
# Almanac will show detailed logs
LOG_LEVEL=debug pnpm start
```

## Troubleshooting

### Common Issues

**Issue**: Tools not appearing in Almanac

```
Solution: Check your server is exposing the tools correctly
- Verify tool definitions
- Check server is running
- Review logs for errors
```

**Issue**: Config generation fails

```
Solution: Ensure tools return sample data
- Add test data to your API
- Verify tool schemas match actual output
- Check for missing required fields
```

**Issue**: OAuth connection fails

```
Solution: Verify OAuth configuration
- Check redirect URI matches
- Ensure client ID/secret are correct
- Verify scopes are requested
```

## Next Steps

- **[Auto-Configuration Guide](auto-config.md)** - Let Almanac generate configs
- **Manual JSON example on this page** - Understand the configuration format and validation limits
- **[Creating Servers](creating-servers.md)** - Build your own MCP server
- **[Testing & Validation](testing.md)** - Ensure your config works

## Resources

- **[MCP Specification](https://spec.modelcontextprotocol.io/)** - Official protocol docs
- **[MCP SDK](https://github.com/modelcontextprotocol/sdk)** - TypeScript/Python SDKs
- **[Example Servers](https://github.com/tryprotege/almanac/tree/main/packages)** - Reference implementations
