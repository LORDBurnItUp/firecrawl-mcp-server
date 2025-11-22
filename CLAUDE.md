# CLAUDE.md - AI Assistant Guide for Firecrawl MCP Server

This document provides comprehensive guidance for AI assistants working with the Firecrawl MCP Server codebase.

## Repository Overview

**Project**: Firecrawl MCP Server
**Type**: Model Context Protocol (MCP) server implementation
**Language**: TypeScript (ES2022, ESM)
**Framework**: FastMCP (firecrawl-fastmcp v1.0.4)
**Primary SDK**: @mendable/firecrawl-js v4.3.6
**Node Version**: >=18.0.0
**Current Version**: 3.6.1
**License**: MIT

### Purpose
Provides MCP tools for web scraping, crawling, search, and content extraction via the Firecrawl API. Supports both cloud-hosted (firecrawl.dev) and self-hosted Firecrawl instances.

### Key Features
- 6 core MCP tools: scrape, batch_scrape, map, crawl, search, extract
- Dual transport support: stdio (CLI) and httpStream (cloud service)
- Safe mode for cloud environments (disables unsafe browser actions)
- Comprehensive error handling with exponential backoff retries
- Credit usage monitoring and rate limiting
- Authentication via API key (header-based for cloud, env-based for self-hosted)

---

## Codebase Structure

```
/home/user/firecrawl-mcp-server/
├── src/
│   ├── index.ts                    # Main MCP server (646 lines) - CORE FILE
│   ├── legacy/
│   │   └── index.md                # Legacy implementation reference
│   └── types/
│       └── fastmcp.d.ts            # FastMCP TypeScript type definitions
├── .github/workflows/              # CI/CD pipelines
│   ├── ci.yml                      # Build and test
│   ├── publish.yml                 # NPM & MCP registry publishing
│   ├── image.yml                   # Docker image deployment
│   └── ...
├── docker/
│   ├── entrypoint.sh               # Docker startup script
│   └── nginx.conf                  # NGINX reverse proxy config
├── Configuration Files
│   ├── package.json                # Dependencies & scripts
│   ├── tsconfig.json               # TypeScript compiler config
│   ├── jest.config.js              # Jest testing framework
│   ├── .eslintrc.json              # ESLint rules
│   ├── .prettierrc                 # Code formatting
│   └── server.json                 # MCP server manifest
├── Dockerfiles
│   ├── Dockerfile                  # Multi-stage build
│   └── Dockerfile.service          # NGINX + Node service
└── Documentation
    ├── README.md                   # Installation & usage guide
    ├── CHANGELOG.md                # Version history
    ├── VERSIONING.md               # API versioning docs
    └── this file (CLAUDE.md)
```

### Critical Files to Understand

1. **src/index.ts** (646 lines) - The entire server implementation
   - FastMCP server initialization
   - All 6 tool implementations
   - Authentication logic
   - Logging system
   - Transport configuration

2. **package.json** - Dependencies and scripts
   - Build: `tsc && chmod +x dist/index.js`
   - Test: `jest` with ESM support
   - Scripts: build, test, start, start:cloud, lint, format, publish

3. **tsconfig.json** - TypeScript configuration
   - Target: ES2022
   - Module: NodeNext (ESM)
   - Strict mode enabled
   - Output: ./dist

---

## Key Conventions and Patterns

### Code Style

**Formatting** (enforced by Prettier):
- **Indentation**: 2 spaces
- **Line Width**: 80 characters
- **Quotes**: Single quotes
- **Semicolons**: Required
- **Trailing Commas**: ES5 style (multi-line objects/arrays)

**Naming Conventions**:
- **Tool Names**: `snake_case` with `firecrawl_` prefix (e.g., `firecrawl_scrape`)
- **Variables**: `camelCase` (e.g., `apiKey`, `sessionData`)
- **Constants**: `UPPER_SNAKE_CASE` (e.g., `SAFE_MODE`, `PORT`, `HOST`)
- **Classes**: `PascalCase` (e.g., `ConsoleLogger`, `SessionData`)
- **Interfaces**: `PascalCase` (e.g., `SessionData`, `ToolContext`)

### TypeScript Patterns

**Strict Mode**: Enabled - all code must be type-safe

**Type Validation Strategy**:
```typescript
// Runtime validation with Zod (preferred for user inputs)
const scrapeParamsSchema = z.object({
  url: z.string(),
  formats: z.array(z.union([z.string(), z.object(...)])).optional(),
  // ...
});

// TypeScript interfaces for internal types
interface SessionData {
  firecrawlApiKey?: string;
  [key: string]: unknown;
}
```

**Common Patterns**:
- Use `z.infer<typeof schema>` to derive TypeScript types from Zod schemas
- Use `@ts-expect-error` with TODO comments for known issues
- Prefer union types over complex inheritance
- Use `unknown` instead of `any` where possible

### Error Handling

**Pattern**:
```typescript
try {
  // Operation
} catch (error) {
  log.error('Tool execution failed:', {
    tool: 'tool_name',
    error: error instanceof Error ? error.message : String(error),
    args: JSON.stringify(arguments),
  });
  throw error; // Re-throw for FastMCP to handle
}
```

**Error Types**:
- **Validation Errors**: Caught by Zod, fail fast
- **API Errors**: From Firecrawl SDK, include retry logic
- **Network Errors**: Exponential backoff (configurable)
- **Authentication Errors**: Clear messaging for missing/invalid API keys

### Logging

**Logger Class** (`ConsoleLogger`):
```typescript
class ConsoleLogger implements Logger {
  debug(...args: unknown[]): void;
  error(...args: unknown[]): void;
  info(...args: unknown[]): void;
  log(...args: unknown[]): void;
  warn(...args: unknown[]): void;
}
```

**Usage**:
```typescript
log.info('Operation started', { url, options });
log.error('Operation failed', { error: err.message });
```

**Conditional Logging**:
- Suppressed in cloud service mode (CLOUD_SERVICE=true)
- Suppressed in SSE mode (SSE_LOCAL=true)
- Suppressed in HTTP streamable mode (HTTP_STREAMABLE_SERVER=true)
- Full logging in stdio mode (default)

---

## Architecture Patterns

### Safe Mode

**When Activated**: `CLOUD_SERVICE=true`

**Purpose**: Comply with ChatGPT/Claude safety requirements by restricting browser automation actions

**Restricted Actions**:
- `click` - Clicking elements
- `write` - Form input
- `press` - Keyboard events
- `executeJavascript` - JS execution
- `generatePDF` - PDF generation

**Allowed Actions** (always safe):
- `wait` - Waiting for elements/time
- `screenshot` - Taking screenshots
- `scroll` - Scrolling pages
- `scrape` - Content extraction

**Implementation**:
```typescript
const SAFE_MODE = process.env.CLOUD_SERVICE === 'true';
const safeActionTypes = ['wait', 'screenshot', 'scroll', 'scrape'] as const;
const otherActions = ['click', 'write', 'press', 'executeJavascript', 'generatePDF'] as const;
const allowedActionTypes = SAFE_MODE ? safeActionTypes : [...safeActionTypes, ...otherActions];
```

### Authentication

**Cloud Mode** (header-based):
```typescript
// Extracted from HTTP headers
function extractApiKey(headers: http.IncomingHttpHeaders): string | undefined {
  return headers['x-firecrawl-api-key']
    || headers['x-api-key']
    || headers['authorization']?.replace(/^Bearer /i, '');
}
```

**Self-Hosted Mode** (environment-based):
```typescript
const apiKey = process.env.FIRECRAWL_API_KEY;
const apiUrl = process.env.FIRECRAWL_API_URL; // Required for self-hosted
```

### Transport Modes

**stdio** (Default):
- Standard input/output communication
- Used by Claude Desktop, Cursor, VS Code
- Command: `npx -y firecrawl-mcp`

**httpStream** (Cloud Service):
- HTTP-based streaming transport
- Stateless mode for scalability
- Health endpoint: `/health`
- Port: 3000 (default), Host: 0.0.0.0 (cloud) or localhost (local)
- Triggered by: `CLOUD_SERVICE=true` or `HTTP_STREAMABLE_SERVER=true`

---

## Development Workflows

### Local Development

```bash
# 1. Install dependencies
npm install

# 2. Build TypeScript
npm run build

# 3. Run locally (stdio mode)
npm start

# 4. Run in cloud mode (httpStream)
npm run start:cloud

# 5. Run tests
npm test

# 6. Lint code
npm run lint

# 7. Format code
npm run format
```

### Making Changes

**Step-by-Step**:

1. **Modify src/index.ts** (primary file for all changes)
   - Tool implementations
   - Zod schemas
   - Authentication logic
   - Logging

2. **Update types if needed** (src/types/fastmcp.d.ts)
   - Only if FastMCP library changes

3. **Run linter and formatter**
   ```bash
   npm run lint:fix
   npm run format
   ```

4. **Build and test**
   ```bash
   npm run build
   npm test
   ```

5. **Manual testing** (if needed)
   ```bash
   # Test with MCP client or direct execution
   echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | npm start
   ```

### Adding a New Tool

**Template**:
```typescript
server.addTool({
  name: 'firecrawl_new_tool',
  description: `
    Brief description of what this tool does.

    **Best for:** Use case guidance
    **Not recommended for:** Anti-patterns
    **Common mistakes:** Pitfalls to avoid
    **Prompt Example:** "Sample user request"
  `,
  parameters: z.object({
    requiredParam: z.string().describe('Parameter description'),
    optionalParam: z.number().optional().describe('Optional parameter'),
  }),
  execute: async (args, context: ToolContext<SessionData>) => {
    const { log, session } = context;

    try {
      log.info('Starting new tool', { args });

      const client = await getClient(session);

      // Tool implementation
      const result = await client.someMethod(args);

      return asText(result);
    } catch (error) {
      log.error('Tool execution failed:', {
        tool: 'firecrawl_new_tool',
        error: error instanceof Error ? error.message : String(error),
      });
      throw error;
    }
  },
});
```

**Best Practices**:
- Use descriptive parameter names and descriptions
- Include examples in the description
- Validate inputs with Zod schemas
- Log operations for debugging
- Handle errors gracefully
- Return results as text (use `asText()` helper)

### Modifying Existing Tools

**Location**: All tools are in `src/index.ts` (lines ~200-600)

**Checklist**:
1. ✅ Update Zod schema if parameters change
2. ✅ Update tool description with new parameters/behavior
3. ✅ Update examples in description
4. ✅ Test with various inputs
5. ✅ Update README.md if user-facing changes
6. ✅ Update CHANGELOG.md with changes

---

## Testing Approach

### Test Configuration

**Framework**: Jest with ts-jest preset for ESM

**Test Files**: Match pattern `**/*.test.ts`

**Mock Strategy**:
- Firecrawl SDK is fully mocked in `jest.setup.ts`
- No real API calls in tests
- Mock responses for all SDK methods

**Current Mock Setup** (`jest.setup.ts`):
```typescript
jest.mock('@mendable/firecrawl-js', () => ({
  __esModule: true,
  default: jest.fn().mockImplementation(() => ({
    search: jest.fn().mockResolvedValue(mockSearchResponse),
    asyncBatchScrapeUrls: jest.fn().mockResolvedValue(mockBatchScrapeResponse),
    checkBatchScrapeStatus: jest.fn().mockResolvedValue(mockBatchScrapeStatusResponse),
  })),
}));
```

### Running Tests

```bash
# Run all tests
npm test

# Run specific test file
npm test -- path/to/test.test.ts

# Run with coverage
npm test -- --coverage

# Test endpoints (manual validation)
npm run test:endpoints
```

### Writing Tests

**Pattern**:
```typescript
import { describe, it, expect } from '@jest/globals';

describe('Tool Name', () => {
  it('should handle valid input', async () => {
    // Arrange
    const input = { param: 'value' };

    // Act
    const result = await tool.execute(input);

    // Assert
    expect(result).toMatchObject({ expected: 'output' });
  });

  it('should reject invalid input', async () => {
    // Arrange
    const invalidInput = { invalid: 'data' };

    // Act & Assert
    await expect(tool.execute(invalidInput)).rejects.toThrow();
  });
});
```

---

## Build and Deployment

### Build Process

**Command**: `npm run build`

**Steps**:
1. TypeScript compilation: `tsc`
   - Source: `./src/**/*.ts`
   - Output: `./dist/**/*.js`
   - Target: ES2022
   - Module: NodeNext (ESM)
2. Make executable: `chmod 755 dist/index.js`

**Output Structure**:
```
dist/
├── index.js           # Main entry point (executable)
├── index.js.map       # Source map
└── types/
    └── fastmcp.d.ts   # Type definitions
```

### Publishing to NPM

**Automated** (via GitHub Actions):
- Trigger: Create a GitHub Release
- Workflow: `.github/workflows/publish.yml`
- Steps: Install → Build → Publish to NPM → Publish to MCP registry

**Manual**:
```bash
# Standard release
npm run publish

# Beta release
npm run publish-beta
```

**Pre-publish Hook**: `npm run prepare` (auto-runs build)

### Docker Deployment

**Local Container** (Dockerfile):
```bash
docker build -t firecrawl-mcp .
docker run -e FIRECRAWL_API_KEY=your-key firecrawl-mcp
```

**Service Container** (Dockerfile.service - NGINX + Node):
```bash
docker build -f Dockerfile.service -t firecrawl-mcp-service .
docker run -p 8080:8080 -e FIRECRAWL_API_KEY=your-key firecrawl-mcp-service
```

**Production Image**:
- Registry: `ghcr.io/firecrawl/firecrawl-mcp-server:latest`
- Auto-built on push to main via `.github/workflows/image.yml`
- Multi-arch support (check workflow for details)

---

## Environment Variables

### Required

**Cloud API**:
- `FIRECRAWL_API_KEY` - Your Firecrawl API key from https://firecrawl.dev/app/api-keys

**Self-Hosted**:
- `FIRECRAWL_API_URL` - Custom API endpoint (e.g., https://firecrawl.your-domain.com)
- `FIRECRAWL_API_KEY` - Optional, if your self-hosted instance requires auth

### Optional Configuration

**Retry Configuration**:
- `FIRECRAWL_RETRY_MAX_ATTEMPTS` (default: 3) - Max retry attempts
- `FIRECRAWL_RETRY_INITIAL_DELAY` (default: 1000) - Initial delay in ms
- `FIRECRAWL_RETRY_MAX_DELAY` (default: 10000) - Max delay in ms
- `FIRECRAWL_RETRY_BACKOFF_FACTOR` (default: 2) - Exponential backoff multiplier

**Credit Monitoring**:
- `FIRECRAWL_CREDIT_WARNING_THRESHOLD` (default: 1000) - Warning threshold
- `FIRECRAWL_CREDIT_CRITICAL_THRESHOLD` (default: 100) - Critical threshold

**Server Configuration**:
- `CLOUD_SERVICE` - Set to "true" for cloud mode (enables httpStream, safe mode)
- `SSE_LOCAL` - Set to "true" for SSE transport
- `HTTP_STREAMABLE_SERVER` - Set to "true" for HTTP streamable mode
- `PORT` (default: 3000) - HTTP server port
- `HOST` (default: localhost or 0.0.0.0 for cloud) - HTTP server host

**Example .env File**:
```bash
# Required for cloud
FIRECRAWL_API_KEY=fc-your-api-key-here

# Optional: Retry configuration
FIRECRAWL_RETRY_MAX_ATTEMPTS=5
FIRECRAWL_RETRY_INITIAL_DELAY=2000
FIRECRAWL_RETRY_MAX_DELAY=30000
FIRECRAWL_RETRY_BACKOFF_FACTOR=3

# Optional: Credit monitoring
FIRECRAWL_CREDIT_WARNING_THRESHOLD=2000
FIRECRAWL_CREDIT_CRITICAL_THRESHOLD=500

# Optional: Server mode
CLOUD_SERVICE=false
PORT=3000
```

---

## Common Tasks

### Task 1: Add Support for a New Firecrawl Feature

**Scenario**: Firecrawl SDK adds a new method/parameter

**Steps**:
1. Update `@mendable/firecrawl-js` in package.json
2. Run `npm install`
3. Locate the relevant tool in `src/index.ts`
4. Update the Zod schema with new parameters
5. Update the tool description
6. Implement the new feature in the execute function
7. Test locally: `npm run build && npm start`
8. Update README.md if user-facing
9. Update CHANGELOG.md
10. Commit and push

**Example** (adding a new parameter to scrape):
```typescript
// Before
parameters: z.object({
  url: z.string(),
  formats: z.array(...).optional(),
}),

// After
parameters: z.object({
  url: z.string(),
  formats: z.array(...).optional(),
  newFeature: z.boolean().optional().describe('New feature description'),
}),

// In execute function
const result = await client.scrape({
  url: args.url,
  formats: args.formats,
  newFeature: args.newFeature, // Add new parameter
});
```

### Task 2: Fix a Bug in Tool Logic

**Steps**:
1. **Identify the tool**: Check which tool is affected (src/index.ts)
2. **Locate the bug**: Read the execute function (~50-100 lines per tool)
3. **Write a test**: Add test case in jest.setup.ts or create new test file
4. **Fix the bug**: Modify the tool logic
5. **Verify**: Run `npm test` and `npm run build`
6. **Update CHANGELOG.md**: Document the fix
7. **Commit with descriptive message**: "fix(tool_name): description"

### Task 3: Update Dependencies

**Safe Updates** (patch/minor):
```bash
npm update
npm run build
npm test
```

**Major Updates** (breaking changes):
```bash
# Update one at a time
npm install @mendable/firecrawl-js@latest
npm run build
npm test

# Check for breaking changes in:
# 1. Firecrawl SDK: Check their changelog
# 2. FastMCP: Check type definitions
# 3. Zod: Check schema validation behavior
```

**Critical Dependencies**:
- `@mendable/firecrawl-js` - Core functionality, test thoroughly
- `firecrawl-fastmcp` - Server framework, check type compatibility
- `zod` - Validation library, verify all schemas still work
- `typescript` - Language, check tsconfig.json compatibility

### Task 4: Improve Error Messages

**Pattern**:
```typescript
// Before
throw new Error('Invalid input');

// After
throw new Error(
  `Invalid input for ${toolName}: ` +
  `Expected ${expectedType}, got ${typeof actualValue}. ` +
  `Please provide ${exampleValue}.`
);
```

**Best Practices**:
- Include tool name in error
- Describe what was expected
- Describe what was received
- Provide an example if possible
- Log errors before throwing

### Task 5: Add Performance Monitoring

**Pattern**:
```typescript
execute: async (args, context: ToolContext<SessionData>) => {
  const { log } = context;
  const startTime = Date.now();

  try {
    log.info('Tool started', { args });

    // Tool implementation
    const result = await client.someMethod(args);

    const duration = Date.now() - startTime;
    log.info('Tool completed', { duration, resultSize: result.length });

    return asText(result);
  } catch (error) {
    const duration = Date.now() - startTime;
    log.error('Tool failed', { duration, error });
    throw error;
  }
}
```

---

## Important Gotchas

### 1. ESM vs CommonJS

**This Project Uses ESM (ES Modules)**:
- `package.json` has `"type": "module"`
- Use `import`/`export`, not `require()`
- File extensions in imports: Use `.js` even for TypeScript (compiled output)
- `__dirname` not available: Use `import.meta.url` instead

**Example**:
```typescript
// ✅ Correct (ESM)
import { FastMCP } from 'firecrawl-fastmcp';

// ❌ Wrong (CommonJS)
const { FastMCP } = require('firecrawl-fastmcp');
```

### 2. Zod Schema Validation

**Complex Unions**: Scrape formats parameter accepts strings OR objects
```typescript
z.array(
  z.union([
    z.enum(['markdown', 'html', ...]),
    z.object({
      type: z.enum(['extract', 'llm-extraction']),
      schema: z.record(z.string(), z.any()).optional(),
      prompt: z.string().optional(),
    }),
  ])
).optional()
```

**Important**: Test complex schemas with various input combinations

### 3. Safe Mode Restrictions

**When CLOUD_SERVICE=true**:
- Browser actions are limited
- Users cannot click, type, or execute JavaScript
- This is by design for security
- Document clearly in error messages if user tries unsafe actions

### 4. Rate Limiting

**Firecrawl API has rate limits**:
- Automatic retry with exponential backoff (configurable)
- Default: 3 attempts, 1s initial delay, 2x backoff
- Can be overwhelming for large batch operations
- Recommend users to use `maxConcurrency` parameter for crawls

### 5. Token Limits

**Large Crawls Can Exceed Context Windows**:
- Warn users to limit crawl depth and page count
- Suggest `map` + `batch_scrape` workflow instead of unlimited crawls
- Default maxDepth should be low (2-3)
- Document this in tool descriptions

### 6. TypeScript Strict Mode

**All Code Must Be Type-Safe**:
- No `any` types (use `unknown` if necessary)
- All functions must have return types
- Null checks required
- Use `@ts-expect-error` with TODO comment for known issues

### 7. Async Operations

**Crawl and Batch Operations Are Async**:
- Return operation ID immediately
- Users must call status check tools separately
- Status checks may need multiple attempts
- Document expected workflow in descriptions

---

## Tool Reference Guide

### Quick Decision Tree for Users

```
Do you know the exact URL(s)?
├─ YES: One URL?
│  ├─ YES: Use firecrawl_scrape
│  └─ NO: Use firecrawl_batch_scrape
└─ NO: Do you need to discover URLs?
   ├─ YES: Use firecrawl_map (then scrape discovered URLs)
   └─ NO: Searching the web?
      ├─ YES: Use firecrawl_search
      └─ NO: Need structured data extraction?
         ├─ YES: Use firecrawl_extract
         └─ NO: Want comprehensive site coverage?
            └─ Use firecrawl_crawl (with limits!)
```

### Tool Implementations

**All tools are in src/index.ts**:

1. **firecrawl_scrape** (~200-250 lines)
   - Location: ~line 200
   - Zod schema: scrapeParamsSchema (~100 lines)
   - Supports: formats, parsers, actions, waitFor, mobile, etc.

2. **firecrawl_map** (~30 lines)
   - Location: ~line 350
   - Simple URL discovery
   - Parameters: url, search, sitemap, includeSubdomains, limit

3. **firecrawl_search** (~50 lines)
   - Location: ~line 400
   - Web search with optional scraping
   - Parameters: query, limit, tbs, filter, location, sources

4. **firecrawl_crawl** (~70 lines)
   - Location: ~line 470
   - Async crawling operation
   - Returns: operation ID
   - Warning: Can exceed token limits

5. **firecrawl_check_crawl_status** (~30 lines)
   - Location: ~line 550
   - Status checker for crawl operations
   - Parameters: id

6. **firecrawl_extract** (~40 lines)
   - Location: ~line 590
   - LLM-powered extraction
   - Parameters: urls[], prompt, schema, enableWebSearch

---

## Contributing Guidelines

### Before Making Changes

1. ✅ Read this CLAUDE.md file
2. ✅ Read the README.md for user-facing documentation
3. ✅ Understand the tool you're modifying
4. ✅ Check CHANGELOG.md for recent changes
5. ✅ Review existing issues/PRs for context

### Making Changes

1. **Write clear commit messages**
   - Format: `type(scope): description`
   - Types: feat, fix, docs, style, refactor, test, chore
   - Example: `feat(scrape): add support for PDF parsing`

2. **Update documentation**
   - Code comments for complex logic
   - Tool descriptions for user-facing changes
   - README.md for installation/configuration changes
   - CHANGELOG.md for all changes
   - This file (CLAUDE.md) for codebase structure changes

3. **Test thoroughly**
   - Run `npm test`
   - Run `npm run build`
   - Manual testing with real API if needed
   - Test both cloud and self-hosted modes if applicable

4. **Follow code style**
   - Run `npm run lint:fix`
   - Run `npm run format`
   - Check that `npm run build` succeeds

### Pull Request Checklist

- [ ] Code builds successfully (`npm run build`)
- [ ] Tests pass (`npm test`)
- [ ] Linting passes (`npm run lint`)
- [ ] Code is formatted (`npm run format`)
- [ ] CHANGELOG.md updated
- [ ] README.md updated (if user-facing changes)
- [ ] Commit messages follow convention
- [ ] No sensitive data (API keys, tokens) in code

---

## Debugging Tips

### Local Debugging

**Enable Full Logging**:
```bash
# Don't set CLOUD_SERVICE or other logging suppressors
unset CLOUD_SERVICE SSE_LOCAL HTTP_STREAMABLE_SERVER

# Run server
FIRECRAWL_API_KEY=your-key npm start
```

**Test Specific Tool**:
```bash
# Send MCP request via stdin
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | npm start

# For more complex debugging, use MCP inspector
npx @modelcontextprotocol/inspector npm start
```

**Check Environment**:
```typescript
// In src/index.ts, add logging
console.error('DEBUG:', {
  CLOUD_SERVICE: process.env.CLOUD_SERVICE,
  SAFE_MODE,
  apiUrl: process.env.FIRECRAWL_API_URL,
  hasApiKey: !!process.env.FIRECRAWL_API_KEY,
});
```

### Common Issues

**Issue**: "API key required"
- **Cause**: Missing FIRECRAWL_API_KEY in cloud mode
- **Fix**: Set environment variable or pass via header

**Issue**: Build fails with module errors
- **Cause**: ESM/CommonJS mismatch
- **Fix**: Check package.json has `"type": "module"`, use import/export syntax

**Issue**: Tests fail after dependency update
- **Cause**: Mock in jest.setup.ts doesn't match new SDK
- **Fix**: Update mock to match new SDK interface

**Issue**: TypeScript errors after changes
- **Cause**: Strict mode catches type issues
- **Fix**: Add proper types, avoid `any`, use Zod inference

**Issue**: Tool not appearing in MCP client
- **Cause**: Build not run or syntax error in tool definition
- **Fix**: Run `npm run build`, check console for errors

---

## Version History

**Current**: 3.6.1 (as of this CLAUDE.md creation)

**Recent Major Changes** (see CHANGELOG.md for full history):
- 3.6.x: FastMCP upgrade, branding format support
- 3.5.x: Bug fixes and improvements
- 3.0.x: FastMCP migration, modern architecture
- 2.x: V2 API support
- 1.x: Original MCP SDK implementation

---

## Additional Resources

### External Documentation

- **Firecrawl API Docs**: https://docs.firecrawl.dev
- **MCP Specification**: https://spec.modelcontextprotocol.io
- **FastMCP Library**: https://github.com/firecrawl/firecrawl-mcp-server (same repo)
- **TypeScript Handbook**: https://www.typescriptlang.org/docs/
- **Zod Documentation**: https://zod.dev

### Internal Documentation

- **README.md**: User installation and usage guide
- **VERSIONING.md**: API version migration guide
- **CHANGELOG.md**: Version history and changes
- **src/legacy/index.md**: Legacy implementation reference

### Getting Help

- **Issues**: https://github.com/firecrawl/firecrawl-mcp-server/issues
- **Discussions**: GitHub Discussions (if enabled)
- **Firecrawl Discord**: Check README for invite link

---

## Summary for AI Assistants

**When working with this codebase, remember**:

1. ✅ **Single Source File**: Almost all code is in `src/index.ts`
2. ✅ **ESM Only**: Use import/export, not require()
3. ✅ **Type Safety**: Strict TypeScript, Zod validation
4. ✅ **Tool Descriptions Matter**: They guide users, be comprehensive
5. ✅ **Safe Mode**: Cloud service restricts browser actions
6. ✅ **Async Operations**: Crawl/batch return IDs, not results
7. ✅ **Error Handling**: Log everything, provide context
8. ✅ **Documentation**: Update README, CHANGELOG, tool descriptions
9. ✅ **Testing**: Mock SDK, test edge cases
10. ✅ **Build Before Test**: `npm run build && npm test`

**Most Common Tasks**:
- Adding/modifying tools → Edit src/index.ts
- Updating dependencies → npm install, test thoroughly
- Fixing bugs → Locate tool, add test, fix, verify
- Documentation → Update tool description + README.md + CHANGELOG.md

**Key Principle**: This is an MCP server that wraps the Firecrawl API. Focus on providing a great developer experience through clear documentation, robust error handling, and sensible defaults.

---

**Last Updated**: 2025-11-22
**Version**: 3.6.1
**Maintainers**: Firecrawl team (@firecrawl)
