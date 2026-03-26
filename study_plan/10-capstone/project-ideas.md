# Capstone Project Ideas

Now that you've completed the study plan, it's time to apply your knowledge. This document provides project ideas at different difficulty levels to help you solidify your understanding and potentially contribute to the opencode ecosystem.

## How to Choose a Project

Consider these factors when selecting a project:

1. **Your interests** - What aspects of opencode excited you most?
2. **Your goals** - Learning vs. contributing vs. building something useful
3. **Time available** - Beginner projects take days, advanced ones take weeks
4. **Existing skills** - Build on strengths or challenge yourself in new areas

## Beginner Projects

These projects require understanding of 1-2 modules and involve modifying existing patterns.

---

### Project B1: Calculator Tool

**Description:** Add a built-in calculator tool that can evaluate mathematical expressions. The LLM can use this for precise calculations instead of attempting math in its head.

**Skills Practiced:**
- Tool abstraction and registration
- Zod schema definition
- Permission system basics
- TypeScript patterns used in the codebase

**Key Files to Modify:**
- Create: `packages/opencode/src/tool/calculator.ts`
- Create: `packages/opencode/src/tool/calculator.txt` (description)
- Modify: `packages/opencode/src/tool/registry.ts` (register the tool)

**Suggested Approach:**
1. Study `packages/opencode/src/tool/webfetch.ts` as a template
2. Define the tool schema with parameters like `expression: string`
3. Use a safe math evaluation library (e.g., `mathjs`) or implement basic parsing
4. Handle edge cases: division by zero, invalid expressions, overflow
5. Write the tool description in a `.txt` file
6. Register in the tool registry

**Success Criteria:**
- [ ] Tool appears in available tools list
- [ ] LLM can invoke the tool with expressions like `2 + 2 * 3`
- [ ] Returns accurate results for basic arithmetic
- [ ] Handles errors gracefully with helpful messages
- [ ] Follows the codebase style guide (single-word names, no destructuring)

---

### Project B2: Custom TUI Theme

**Description:** Create a custom color theme for the terminal UI. This teaches you about the Solid-based TUI rendering system.

**Skills Practiced:**
- OpenTUI framework
- Solid.js reactive patterns
- Terminal color handling
- Configuration system

**Key Files to Study:**
- `packages/opencode/src/cli/cmd/tui/` - TUI components
- `packages/opencode/src/cli/cmd/tui/theme.ts` (if exists) or color definitions
- `packages/opencode/src/config/config.ts` - Configuration loading

**Suggested Approach:**
1. Identify where colors are defined in the TUI
2. Create a theme configuration option in the config schema
3. Define 2-3 themes (dark, light, high-contrast)
4. Wire up theme selection to the TUI components
5. Test in different terminal emulators

**Success Criteria:**
- [ ] Theme can be selected via configuration
- [ ] All TUI elements respect the theme
- [ ] Colors are accessible (sufficient contrast)
- [ ] Theme persists across sessions

---

### Project B3: Simple Plugin

**Description:** Create a plugin that adds a custom greeting or status message to sessions.

**Skills Practiced:**
- Plugin system architecture
- Event bus subscription
- Effect patterns
- Package structure

**Key Files to Study:**
- `packages/plugin/` - Plugin package structure
- `packages/opencode/src/plugin/` - Plugin loading system
- Session event types

**Suggested Approach:**
1. Study the plugin package structure
2. Create a new plugin that listens to session events
3. Add custom behavior (e.g., log session starts, add a footer message)
4. Test loading the plugin via configuration

**Success Criteria:**
- [ ] Plugin loads without errors
- [ ] Custom behavior executes at the right time
- [ ] Plugin can be enabled/disabled via config
- [ ] No performance impact on normal operation

---

## Intermediate Projects

These projects require understanding of 3-4 modules and involve creating new subsystems.

---

### Project I1: New LLM Provider

**Description:** Add support for a new LLM provider (e.g., Mistral, Cohere, or a local model server like Ollama).

**Skills Practiced:**
- Provider abstraction layer
- Vercel AI SDK integration
- Model transforms
- Authentication handling
- Streaming responses

**Key Files to Modify:**
- `packages/opencode/src/provider/provider.ts` - Provider definitions
- `packages/opencode/src/provider/transform.ts` - Model transforms
- `packages/opencode/src/provider/auth.ts` - Authentication
- May need to add to `models.dev` repository first

**Suggested Approach:**
1. Check if the provider is already in [models.dev](https://github.com/anomalyco/models.dev)
2. If not, contribute the provider definition there first
3. Study how existing providers are configured
4. Implement any provider-specific transforms
5. Add authentication flow if needed
6. Test with various model sizes

**Success Criteria:**
- [ ] Provider appears in `opencode providers` list
- [ ] Can authenticate with the provider
- [ ] Chat works with streaming responses
- [ ] Tool calls work correctly
- [ ] Error messages are helpful

---

### Project I2: New CLI Command

**Description:** Add a new CLI command, such as `opencode history` to browse past sessions or `opencode config` to manage settings.

**Skills Practiced:**
- Yargs command definition
- CLI argument parsing
- Database queries
- Terminal output formatting

**Key Files to Modify:**
- Create: `packages/opencode/src/cli/cmd/history.ts` (or your command)
- Modify: `packages/opencode/src/cli/cmd/cmd.ts` - Register the command

**Suggested Approach:**
1. Study existing commands like `packages/opencode/src/cli/cmd/session.ts`
2. Define the command with yargs builder pattern
3. Implement the handler function
4. Add appropriate subcommands if needed
5. Format output nicely for terminal display

**Example: `opencode history` command:**
```typescript
// Subcommands: list, show <id>, search <query>, delete <id>
// Options: --limit, --since, --format (json|table)
```

**Success Criteria:**
- [ ] Command appears in `opencode --help`
- [ ] All subcommands work correctly
- [ ] Help text is clear and complete
- [ ] Follows existing command patterns
- [ ] Handles errors gracefully

---

### Project I3: MCP Server Integration

**Description:** Create an MCP (Model Context Protocol) server that provides opencode with access to external data or services.

**Skills Practiced:**
- MCP protocol understanding
- Server implementation
- Tool exposure via MCP
- Configuration for MCP servers

**Key Files to Study:**
- `packages/opencode/src/mcp/` - MCP client implementation
- MCP specification documentation

**Suggested Approach:**
1. Choose a data source (e.g., Jira, GitHub Issues, local database)
2. Implement an MCP server using the MCP SDK
3. Define tools that expose the data source
4. Configure opencode to connect to your server
5. Test tool invocations through the LLM

**Success Criteria:**
- [ ] MCP server starts and accepts connections
- [ ] Tools appear in opencode's available tools
- [ ] LLM can successfully invoke the tools
- [ ] Data is returned in a useful format
- [ ] Server handles errors gracefully

---

### Project I4: New Message Part Type

**Description:** Add a new message part type for specialized content (e.g., diagrams, charts, or structured data).

**Skills Practiced:**
- Message schema design
- Zod discriminated unions
- Rendering in TUI and web
- Serialization/deserialization

**Key Files to Modify:**
- `packages/opencode/src/session/message.ts` - Part type definitions
- TUI rendering components
- Web app rendering components
- Storage serialization

**Suggested Approach:**
1. Study existing part types in `message.ts` (TextPart, ReasoningPart, etc.)
2. Define your new part type with Zod schema
3. Add it to the Part discriminated union
4. Implement rendering in TUI
5. Implement rendering in web app
6. Ensure it serializes correctly to storage

**Example: DiagramPart**
```typescript
export const DiagramPart = z.object({
  type: z.literal("diagram"),
  format: z.enum(["mermaid", "ascii", "svg"]),
  content: z.string(),
  title: z.string().optional(),
})
```

**Success Criteria:**
- [ ] Part type validates correctly
- [ ] Renders appropriately in TUI
- [ ] Renders appropriately in web app
- [ ] Persists and loads from storage
- [ ] LLM can generate this part type (if applicable)

---

## Advanced Projects

These projects require deep understanding of the entire system and involve significant new functionality.

---

### Project A1: New Agent Type

**Description:** Implement a specialized agent type with unique capabilities (e.g., a "researcher" agent that prioritizes web search, or a "reviewer" agent focused on code review).

**Skills Practiced:**
- Agent architecture
- Prompt engineering
- Permission system
- Model selection
- Tool filtering

**Key Files to Modify:**
- `packages/opencode/src/agent/agent.ts` - Agent definitions
- Agent prompt files in `packages/opencode/src/agent/prompt/`
- Configuration schema

**Suggested Approach:**
1. Study the Agent namespace and Info schema
2. Define your agent's characteristics:
   - Name and description
   - Default model preferences
   - Permission ruleset
   - Custom system prompt
   - Tool restrictions/preferences
3. Create the agent prompt file
4. Register the agent
5. Test in various scenarios

**Example: Researcher Agent**
- Prioritizes `websearch` and `webfetch` tools
- Has a prompt focused on finding and synthesizing information
- May have higher token limits for comprehensive responses
- Could include citation formatting in its prompt

**Success Criteria:**
- [ ] Agent appears in agent selection
- [ ] Behaves according to its specialization
- [ ] Uses appropriate tools for its purpose
- [ ] Prompt produces desired behavior
- [ ] Can be selected via configuration or command

---

### Project A2: New Storage Backend

**Description:** Implement an alternative storage backend (e.g., PostgreSQL, cloud storage, or encrypted local storage).

**Skills Practiced:**
- Drizzle ORM
- Database abstraction
- Migration system
- Effect patterns for services
- Configuration system

**Key Files to Study:**
- `packages/opencode/src/storage/` - Storage layer
- `packages/opencode/src/session/session.sql.ts` - Session schema
- Drizzle configuration

**Suggested Approach:**
1. Study the existing SQLite implementation
2. Create a new Drizzle dialect configuration
3. Adapt schema definitions for your backend
4. Implement connection management
5. Create migration strategy
6. Add configuration options for backend selection

**Considerations:**
- How will migrations work across backends?
- What about existing data migration?
- Performance implications?
- Connection pooling?

**Success Criteria:**
- [ ] Backend connects successfully
- [ ] All CRUD operations work
- [ ] Migrations apply correctly
- [ ] Performance is acceptable
- [ ] Can switch backends via configuration
- [ ] Existing tests pass with new backend

---

### Project A3: Web Dashboard for Session Analytics

**Description:** Create a web dashboard that visualizes session data: token usage, tool invocations, session duration, model performance, etc.

**Skills Practiced:**
- Solid.js web development
- Hono API routes
- Database queries
- Data visualization
- Real-time updates via WebSocket

**Key Files to Study:**
- `packages/app/` - Web application
- `packages/opencode/src/server/` - API routes
- Session and message schemas

**Suggested Approach:**
1. Design the dashboard layout and metrics to display
2. Create API endpoints for analytics data:
   - Token usage over time
   - Tool invocation frequency
   - Session duration distribution
   - Model comparison metrics
3. Build the frontend components
4. Add real-time updates for active sessions
5. Implement filtering and date ranges

**Dashboard Ideas:**
- Token usage chart (input vs output)
- Tool usage heatmap
- Session timeline
- Cost estimation
- Model performance comparison
- Error rate tracking

**Success Criteria:**
- [ ] Dashboard loads and displays data
- [ ] Charts are interactive and informative
- [ ] Real-time updates work for active sessions
- [ ] Filtering and date ranges function correctly
- [ ] Performance is good with large datasets
- [ ] Responsive design works on different screens

---

### Project A4: Contribute a Significant Feature

**Description:** Identify a feature request from the GitHub issues and implement it end-to-end.

**Skills Practiced:**
- All modules combined
- Open source contribution workflow
- Code review process
- Documentation

**Suggested Approach:**
1. Browse [GitHub issues](https://github.com/anomalyco/opencode/issues) with these labels:
   - `help wanted`
   - `good first issue`
   - `enhancement`
2. Comment on the issue to express interest
3. Wait for maintainer feedback before starting
4. Create a design document if the feature is complex
5. Implement incrementally with tests
6. Submit a PR following the contribution guidelines

**Contribution Checklist:**
- [ ] Issue exists and is assigned to you
- [ ] Design reviewed (for complex features)
- [ ] Implementation follows style guide
- [ ] Tests added where appropriate
- [ ] Documentation updated
- [ ] PR description is clear and concise
- [ ] Screenshots/videos for UI changes

**Success Criteria:**
- [ ] PR is merged into the main repository
- [ ] Feature works as specified
- [ ] No regressions introduced
- [ ] Code review feedback addressed

---

## Project Planning Template

Use this template when starting any project:

```markdown
# Project: [Name]

## Goal
[One sentence describing what you're building]

## Motivation
[Why this project? What will you learn?]

## Scope
- In scope: [What you will do]
- Out of scope: [What you won't do]

## Technical Approach
1. [Step 1]
2. [Step 2]
3. [Step 3]

## Files to Modify
- [ ] `path/to/file1.ts` - [What changes]
- [ ] `path/to/file2.ts` - [What changes]

## Testing Plan
- [ ] Manual testing: [How]
- [ ] Automated tests: [What to add]

## Success Metrics
- [ ] [Metric 1]
- [ ] [Metric 2]

## Timeline
- Day 1: [Tasks]
- Day 2: [Tasks]
- Day 3: [Tasks]

## Notes
[Any additional thoughts, questions, or concerns]
```

## Tips for Success

1. **Start small** - Get something working, then iterate
2. **Read existing code** - The codebase is the best documentation
3. **Follow the style guide** - Consistency matters for contributions
4. **Test incrementally** - Don't wait until the end to test
5. **Ask for help** - The community is there to support you
6. **Document as you go** - Future you will thank present you

## Next Steps

After completing a project:

1. **Reflect** - What did you learn? What was challenging?
2. **Share** - Blog about it, share on social media, or present to others
3. **Contribute** - Consider submitting your work as a PR
4. **Level up** - Try a more challenging project

Good luck with your capstone project!
