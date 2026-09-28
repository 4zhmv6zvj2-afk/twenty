# Twenty eng List-fit study

**Study notes only. Do not merge. Do not ship. Do not copy this repository into 1231FS.**

This fork (`4zhmv6zvj2-afk/twenty`) is the standing study surface for every lane when a lane needs it (UX, visual, security, eng). This note is the **eng** pass. AI / agents / chat / MCP is the primary subject (sections A–E). The later section is a short skim of other eng patterns that help keep List, not replace it. It is not a UX, visual, or security review.

Written for Reed to tip Avery on the 1231FS List roadmap. It records patterns observed in this fork, not a design for Twenty, and not vendor code for List.

Licence constraint, from this tree: `packages/twenty-server/package.json` is `AGPL-3.0`. The root `LICENSE` states the project is mostly AGPLv3, with an application exception for works that only call Twenty’s published HTTP APIs, and with MIT on listed packages including `twenty-shared` and `twenty-ui`. The AI chat module, tool registry, and MCP server live under `twenty-server`. Copy the patterns below. Do not copy those server files into 1231FS.

List internals (Clerk, Templates, Confirm, desk-tools) were **not** inspected in 1231FS source for this note. A library search did not return List product documents. Risks in section C are mapped from Twenty’s code onto the framing in the task. Where that framing and Twenty disagree, Twenty’s code wins for the pattern, and the List consequence is called out as a risk to verify on List.

---

## A. Recommended approach for List

**Confirm the direction: one tool spine on the existing List API, MCP first, in-app chat second.**

Twenty already works this way. Chat and MCP do not each own a write path. Both call the same factories — `createLearnToolsTool`, `createExecuteToolTool`, `createLoadSkillTool` — and those factories call `ToolRegistryService.resolveAndExecute`. Record writes then go through `record-crud` services into the same `Common*QueryRunner` layer as the product API (`packages/twenty-server/src/engine/core-modules/tool-provider/services/tool-executor.service.ts`).

**Revise three parts of the framing before Avery builds it.**

1. **MCP should expose a short meta-tool surface, not one tool per List endpoint.** `POST /mcp` lists about seven tools (`search_help_center`, `get_tool_catalog`, `execute_tool`, `load_skills`, `list_object_metadata_names`, `list_skills`, `learn_tools`). The CRM catalogue (hundreds of `find_many_*` / `create_one_*` tools) is not on `tools/list`. The model calls `learn_tools`, then `execute_tool`. See `buildMcpToolSet` in `packages/twenty-server/src/engine/api/mcp/services/mcp-protocol.service.ts`. For List: a few MCP tools whose `execute` is the existing desk-tools List API. Do not dump every desk endpoint into the client’s tool list.

2. **Engagement id is a server argument, not page context.** Twenty’s browsing context is advisory. It is appended to the last user message inside a tag whose note says not to call tools from that context alone (`injectBrowsingContextIntoLastUserMessage` in `packages/twenty-server/src/engine/metadata-modules/ai/ai-chat/services/chat-execution.service.ts`). The record-page string does include the record id and a workspace URL, and it tells the model it may fetch details if needed — so the wrapper and the body pull in different directions. List’s engagement id is the system-of-record key. Put it on the tool input and enforce it in the List API. Do not rely on “the user is looking at this engagement” as the scope.

3. **Confirm stays a hard gate on the List API.** Twenty does not hard-stop deletes. The MCP instructions tell the model to wait for confirmation (`build-mcp-server-instructions.util.ts`). The in-app chat system prompt lists `delete_one_*` / `delete_many_*` and does not require confirmation (`chat-system-prompts.const.ts`). `execute_tool` is annotated `destructiveHint: false` on purpose, so MCP clients do not prompt on every call (`mcp-execute-tool-annotations.const.ts`). The executor then soft-deletes immediately. If List copies that, Confirm gates and template ticks can be skipped whenever the model decides the user already agreed. The gate has to live in the handler the tool calls.

Chat comes second, on the same spine, plus three chat-only behaviours Twenty keeps off the shared write path: an advisory page context, a real pause tool (`ask_questions`), and a front-end handler that navigates when a tool result says so. Agents updating records and opening pages is already how Twenty chat behaves. The update is a registry tool. The page change is a client effect.

---

## B. Concrete patterns to copy

### 1. Shared spine, two surfaces

| Surface | Entry | What the model can call directly |
| --- | --- | --- |
| MCP | `POST /mcp` (`ApiPath.Mcp` in `packages/twenty-shared/src/types/ApiPath.ts`) | Meta-tools in `buildMcpToolSet` |
| In-app chat | GraphQL `sendChatMessage` on `AgentChatResolver` | Preloaded tools, native model tools, `ask_questions`, and the same meta-tools |

`createExecuteToolTool` states that registry calls have no fast path (`packages/twenty-server/src/engine/core-modules/tool-provider/tools/execute-tool.tool.ts`). Native tools (web search) are bound on the chat `ToolSet` and are not routed through `execute_tool`.

Chat and MCP pass different `isToolAllowed` sets into the same factories:

- Chat excludes `create_file_upload` and `complete_file_upload` (`ai-chat-excluded-tool-names.const.ts`).
- MCP excludes `code_interpreter`, `http_request`, `extract_json_paths`, and `search_output` (`mcp-excluded-tool-names.const.ts`).
- Workflow agents also drop `navigate_app` (`workflow-agent-excluded-tool-names.const.ts`).

**List pattern:** one registry whose handlers are desk-tools functions. MCP and the later chatbox both call it. UI-only tools are registered only on the chat surface.

### 2. Discover, then execute

Chat prompt order for non-trivial work is plan → `load_skills` → `learn_tools` → `execute_tool` (`chat-system-prompts.const.ts`). Simple record CRUD skips the skill and still requires `learn_tools` before `execute_tool`. `get_tool_catalog` is the fallback browse, exposed on MCP, not bound in the chat `ToolSet`.

**List pattern:** keep schemas out of the system prompt. Publish names in a short index. Return the schema from `learn_tools` for the tools the caller may use. Reject a direct call the model was not given (`guide-uncallable-tool-calls-to-meta-tool.util.ts` rewrites failed direct calls into that guidance).

### 3. CRUD grammar over the existing API

Operations are a fixed list in `packages/twenty-shared/src/ai/constants/database-crud-operation.const.ts`:

`find_many`, `find_one`, `group_by`, `create_one`, `create_many`, `update_one`, `update_many`, `upsert_many`, `delete_one`, `delete_many`.

Names are `{operation}_{snake_case object}` from `DatabaseToolProvider`. Descriptors carry `executionRef.kind: 'database_crud'`. `ToolExecutorService.dispatchDatabaseCrud` calls `FindRecordsService`, `CreateRecordService`, `UpdateRecordService`, `DeleteRecordService`, and the many-record variants. Those build a context with `rolePermissionConfig` and call common query runners.

`delete_one` from the tool path always passes `soft: true`. `DeleteRecordService` can hard-destroy when `soft` is false; the tool executor does not take that branch. `DeleteManyRecordsService` rejects an empty filter (`Filter must not be empty — deleting without a filter is not allowed`).

**List pattern:** name tools after the desk operation and the engagement resource (`update_engagement`, `tick_template`, not a free-form HTTP tool). The handler is the existing List function. Refuse an unscoped write in code (the empty-filter check is the shape to copy). Do not add a second persistence path inside the agent module.

### 4. Role scope before the model sees the tool, and again at dispatch

`DatabaseToolProvider.generateDescriptors` emits read tools only when `canReadObjectRecords`, write tools only when `canUpdateObjectRecords` and `canObjectBeManagedByAutomation`, and delete tools only when `canSoftDeleteObjectRecords`. Field schemas are narrowed with `restrictedFields`. `DeleteRecordService` and `DeleteManyRecordsService` also refuse objects automation may not manage.

`ToolExecutorService` re-checks `provider.isAvailable` at static dispatch (“defence in depth” comment at the dispatch site).

OAuth user plus application: `resolveRoleIdsForUser` intersects the person’s role with the application’s declared role. Comment in that file: an application stays inside both. If the user role is missing, the function returns no role ids — the application role is not a substitute (`packages/twenty-server/src/engine/twenty-orm/utils/resolve-role-ids-for-user.util.ts`). MCP uses that intersection for user callers and `{ unionOf: [apiKeyRoleId] }` for API keys (`McpProtocolService.resolveCallerRoles`).

Chat additionally requires `PermissionFlagType.AI` via `SettingsPermissionGuard` on `AgentChatResolver`. The MCP controller uses `NoPermissionGuard`, so the route does not apply that settings flag. Tool permissions still apply after auth.

Workflow agents can set `requireExplicitObjectGrants: true` so only objects with an explicit grant appear. In-app chat does not set that flag.

**List pattern:** build the catalogue from the Clerk principal’s existing List permissions. A delegated agent token must be the intersection of the human’s grants and the agent’s declared grants. Re-check at execution. Do not treat a service key as a union of every desk role.

### 5. Human-in-the-loop that is actually a pause

`ask_questions` is not in the registry. Chat adds it to the `ToolSet` (`chat-execution.service.ts`). `stopWhen` ends the turn on that tool call. The stream job sets `pendingQuestionMessageId` on the thread. `AgentChatStreamingService` will not claim a new stream while that id is set. Resume is `answerAgentChatQuestion`. The tool schema allows 1–4 questions, 2–4 options, at most one recommended option (`ask-questions.tool.ts`). The prompt tells the model not to use it for lookups or trivial defaults.

This is separate from delete confirmation, which is not a pause.

Workspace setup has a stronger prompt rule (“never create, update, or delete anything before approval”) in `workspace-setup-system-prompt.constant.ts`. That rule is prompt text on a special thread. It is not a write interceptor.

**List pattern:** use a pause tool for ambiguous choices (which template, which engagement). Use Confirm on the API for ticks and destructive writes. Do not merge those two gates into one prompt sentence.

### 6. Update the record, then navigate by id

Two mechanisms, both id-based:

**Move the page.** `navigate_app` input is a discriminated union: `navigateToObject`, `navigateToView`, `navigateToRecord` (object name + record **name**), or `wait` up to 30 seconds (`navigate-app-tool.schema.ts`). The tool returns structured JSON (`action`, `recordId`, `objectNameSingular`, `viewId`), not a URL the model is told to open. The front detects `execute_tool` parts whose `toolName` is `navigate_app` and switches on `action` (`useProcessUIToolCallMessage.ts`). `useChatTargetNavigation` opens `AppPath.RecordShowPage` with `objectRecordId`, or the side panel when the chat is an artefact surface. An MCP client that calls the same tool receives JSON only. The Twenty UI does not move.

**Link inside the reply.** The chat prompt requires `[[record:objectName:recordId:displayName]]` copied from tool `recordReferences`, and forbids invented ids (`CHAT_SYSTEM_PROMPTS.RESPONSE_FORMAT`). The front parses those tokens (`packages/twenty-front/src/modules/ai/utils/`).

Record-page browsing context also builds a real record URL with `getAppPath(AppPath.RecordShowPage, …)` (`buildRecordPageContext` in `chat-execution.service.ts`). That URL is context for the model, not the navigation side effect.

Actor on chat writes: `AgentActorContextService` stamps `FieldActorSource.AGENT` and the workspace member id (`agent-actor-context.service.ts`). MCP does the same source, using the API key name or the member name (`buildActorContext` in `mcp-protocol.service.ts`).

**List pattern:** tools return the engagement id. The chat client routes to the engagement page. Assistant text may only link ids the tool returned. Stamp the write as the Clerk user the agent acted for, with an agent source, so the audit trail is the same row the desk would have written.

### 7. Auth on `/mcp`

`McpCoreController` guards: `McpAuthGuard` → `JwtAuthGuard`, then workspace, not-suspended, `NoPermissionGuard`. `McpAuthGuard` on 401 sets `WWW-Authenticate` to the protected-resource metadata URL `/.well-known/oauth-protected-resource/mcp` with OAuth scopes (`mcp-auth.guard.ts`). Discovery routes live under `/.well-known/` (OAuth authorisation server, protected resource, `mcp/server-card.json`). Protocol version advertised in `initialize` is `2025-06-18` (`mcp-protocol-version.const.ts`). Non-POST gets 405 (`mcp-method-guard.middleware.ts`). There is no server-side MCP session store in this module; each POST is authenticated on its own.

`AccessTokenService` rejects session tokens presented as Bearer. Comment: session tokens are cookie-only, because accepting them as Bearer would reopen the XSS-exfiltration surface.

Docs (`packages/twenty-docs/user-guide/ai/capabilities/mcp.mdx`) describe OAuth (preferred) or `Authorization: Bearer` API key against `{workspace}/mcp`. That matches the guard. Client token refresh described in the doc is client behaviour, not a session table in `engine/api/mcp/`.

**List pattern:** MCP authenticates as the same principal desk-tools already trust (Clerk). 401 points at discovery metadata. Do not accept the Clerk session cookie as a Bearer token.

### 8. Rails that are code, not prompts

Verified in code:

- Empty `delete_many` filter rejected (`delete-many-records.service.ts`).
- Catalogue and dispatch both consult permissions (section B.4).
- Tool output spill in chat when output exceeds the inline cap, with `extract_json_paths` / `search_output` to read it back (`tool-output-spill.service.ts`). MCP does not enable spill and excludes those two tools.
- `http_request` is permission-flagged and excluded from MCP. The chat prompt says it is only for third-party APIs, never Twenty’s own data.
- `send_email` and `draft_email` are separate tools.
- Integration tests assert delegated MCP `execute_tool` honour role intersection, including a refused `delete_one_blocklist` (`packages/twenty-server/test/integration/graphql/suites/application-role-intersection/`).

Verified as prompt-only:

- “Wait for explicit confirmation” before `delete_one` / `delete_many` (MCP instructions).
- Workspace-setup “never write before approval”.
- Browsing-context “do not call tools based on this context” (contradicted in part by the record-page body, which invites a fetch).

Not found: a confirmation modal on agent delete/update. The chat-thread delete modal (`AiChatThreadDeleteConfirmationModal`) deletes the conversation, not a CRM record.

---

## C. Eng risks for List, Clerk, and Templates

These assume the task framing: List is the engagement system of record; Clerk is identity; Templates tick by engagement id; Confirm gates writes; desk-tools are the List API. Verify each against List before building.

1. **A second write path.** If chat or MCP updates engagements through anything other than desk-tools, template ticks and Confirm will diverge from the desk. Twenty avoids that by calling common query runners. List should call the same functions the desk calls, including the Confirm check inside them.

2. **Confirm implemented as a model instruction.** Twenty shows the failure mode: instructions say confirm; annotations set `destructiveHint: false`; the executor deletes. A template tick is closer to a controlled write than to Twenty’s reversible soft delete (`delete_one_*` description says the record is hidden and the delete is reversible). Do not copy soft-delete as the safety model for a tick.

3. **Engagement id taken from the page.** Twenty’s list-view context includes `viewId` and filter text and tells the model to call `get_view_query_parameters`. That is enough for a CRM. For List it is enough to tick the wrong engagement if the model treats “current page” as authority. Require `engagementId` on every mutating tool and ignore page context for authorisation.

4. **Clerk session cookie as MCP bearer.** Twenty refuses session tokens on `Authorization: Bearer` (`access-token.service.ts`). An MCP client that can read a cookie and replay it as Bearer is an XSS exfiltration path. Issue a dedicated access token, or reuse the credential desk-tools already accept, and keep the session cookie on the browser.

5. **API key wider than the user.** MCP API keys resolve to `{ unionOf: [apiKeyRoleId] }` — one role, and it is the whole permission. A shared “List agent” key that can tick every engagement will ignore the Clerk user’s grants. Prefer user-bound tokens intersected with an agent role, matching `resolveRoleIdsForUser`. If the user role is absent, Twenty grants nothing. Copy that.

6. **MCP broader than chat.** Chat requires the AI settings flag. `/mcp` does not (`NoPermissionGuard`). Anyone who can call the List API could call MCP unless List adds the same authorisation desk-tools use. Tool filtering is not a substitute for “may this principal use an agent at all”.

7. **Fuzzy navigation with permission bypass.** `NavigateAppTool` loads every record of the object to fuzzy-match a name, with `shouldBypassPermissionChecks: true` and `buildSystemAuthContext` (`navigate-app-tool.ts`, around the repository `find`). The tool can resolve a record the caller cannot read, then hand the id to the client. List navigation should take an engagement id the caller already retrieved through a permissioned read.

8. **`navigate_app` on the shared registry.** It is not in `MCP_EXCLUDED_TOOL_NAMES`, so an external MCP client can call it. Nothing in the MCP adapter navigates a browser. The `wait` arm sleeps up to 30 seconds inside the tool. Keep navigation on the chat client. Do not put a sleep in the List API.

9. **Actor stamped as a generic agent.** Both surfaces set `FieldActorSource.AGENT`. Chat also stores the member id. If List writes “agent” without the Clerk user id, template history will not show who approved the tick.

10. **Unscoped bulk tools.** `update_many` applies one payload to every match; `delete_many` refuses an empty filter but still deletes every match of a non-empty filter with no confirm step. A List `tick_many_templates` without an engagement id, or with a filter the model invented, is the same shape. Start with single-engagement operations.

11. **Skills and metadata tools.** Twenty’s catalogue includes object, field, view, role, workflow, and webhook providers behind settings flags (`DATA_MODEL`, `VIEWS`, `ROLES`, …). Pointing a List agent at schema-editing tools would let it change the product, not the engagement. Do not generate those categories.

12. **General HTTP and code interpreter.** Both exist in the registry and are stripped from MCP. The chat prompt still has to tell the model not to use `http_request` against Twenty itself. A List agent with a generic HTTP tool can bypass Confirm by calling the API raw. Do not ship that tool.

---

## D. What not to copy

- **The AGPL server implementation.** No copies of `packages/twenty-server/src/engine/api/mcp/`, `tool-provider/`, `record-crud/`, or `metadata-modules/ai/` into 1231FS. Patterns and names only.
- **One MCP tool per generated object.** Twenty’s `tools/list` is the meta-tool set. The generated CRUD names stay behind `learn_tools`.
- **`destructiveHint: false` plus a prompt that says “please confirm”.** That pair is how Twenty avoids client prompts while still telling the model to ask. List Confirm has to be server-side.
- **Prompt-only delete confirmation, and the workspace-setup approval script**, as if they were gates.
- **`shouldBypassPermissionChecks` on navigate-by-name**, and the 30 second `wait` action.
- **`NoPermissionGuard` on the agent route.**
- **Session cookies accepted as Bearer.**
- **`code_interpreter`, `http_request`, email, calendar, Exa web search, file-upload tools, output spill.** They are Twenty product surface, and the first two are explicitly kept off MCP.
- **Metadata, workflow, dashboard, and role tool providers.** List is not a user-defined CRM schema.
- **A skill corpus as a prerequisite for v1.** Twenty uses skills as long instructions the model must load before specialised tools. Record CRUD is documented as not needing one. List v1 can ship tools with schemas and skip the skill loader until a playbook is actually required.
- **Browsing context as authorisation.** Copy it later, as a hint the server does not trust.
- **Hard destroy, or soft delete, as the template-tick semantic.**

---

## E. Key paths

### MCP

- `packages/twenty-server/src/engine/api/mcp/controllers/mcp-core.controller.ts`
- `packages/twenty-server/src/engine/api/mcp/guards/mcp-auth.guard.ts`
- `packages/twenty-server/src/engine/api/mcp/middlewares/mcp-method-guard.middleware.ts`
- `packages/twenty-server/src/engine/api/mcp/services/mcp-protocol.service.ts`
- `packages/twenty-server/src/engine/api/mcp/services/mcp-tool-executor.service.ts`
- `packages/twenty-server/src/engine/api/mcp/constants/mcp-excluded-tool-names.const.ts`
- `packages/twenty-server/src/engine/api/mcp/constants/mcp-execute-tool-annotations.const.ts`
- `packages/twenty-server/src/engine/api/mcp/utils/build-mcp-server-instructions.util.ts`
- `packages/twenty-server/src/engine/core-modules/well-known/utils/build-mcp-server-card.util.ts`
- `packages/twenty-docs/user-guide/ai/capabilities/mcp.mdx` (English user guide; behaviour above is from code)

### Shared tool spine

- `packages/twenty-server/src/engine/core-modules/tool-provider/services/tool-registry.service.ts`
- `packages/twenty-server/src/engine/core-modules/tool-provider/services/tool-executor.service.ts`
- `packages/twenty-server/src/engine/core-modules/tool-provider/tools/execute-tool.tool.ts`
- `packages/twenty-server/src/engine/core-modules/tool-provider/tools/learn-tools.tool.ts`
- `packages/twenty-server/src/engine/core-modules/tool-provider/tools/load-skill.tool.ts`
- `packages/twenty-server/src/engine/core-modules/tool-provider/providers/database-tool.provider.ts`
- `packages/twenty-server/src/engine/core-modules/tool-provider/tool-provider.module.ts`
- `packages/twenty-shared/src/ai/constants/database-crud-operation.const.ts`
- `packages/twenty-server/src/engine/core-modules/record-crud/services/delete-record.service.ts`
- `packages/twenty-server/src/engine/core-modules/record-crud/services/delete-many-records.service.ts`
- `packages/twenty-server/src/engine/twenty-orm/utils/resolve-role-ids-for-user.util.ts`

### In-app chat

- `packages/twenty-server/src/engine/metadata-modules/ai/ai-chat/resolvers/agent-chat.resolver.ts`
- `packages/twenty-server/src/engine/metadata-modules/ai/ai-chat/services/chat-execution.service.ts`
- `packages/twenty-server/src/engine/metadata-modules/ai/ai-chat/services/agent-chat-streaming.service.ts`
- `packages/twenty-server/src/engine/metadata-modules/ai/ai-chat/constants/chat-system-prompts.const.ts`
- `packages/twenty-server/src/engine/metadata-modules/ai/ai-chat/constants/ai-chat-excluded-tool-names.const.ts`
- `packages/twenty-server/src/engine/metadata-modules/ai/ai-chat/tools/ask-questions.tool.ts`
- `packages/twenty-server/src/engine/metadata-modules/ai/ai-agent/types/browsing-context.type.ts`
- `packages/twenty-front/src/modules/ai/types/BrowsingContext.ts`
- `packages/twenty-front/src/modules/ai/hooks/useBrowsingContext.ts`
- `packages/twenty-server/src/engine/metadata-modules/ai/ai-agent-execution/services/agent-actor-context.service.ts`

### Navigation

- `packages/twenty-server/src/engine/core-modules/tool/tools/navigate-tool/navigate-app-tool.ts`
- `packages/twenty-server/src/engine/core-modules/tool/tools/navigate-tool/navigate-app-tool.schema.ts`
- `packages/twenty-front/src/modules/ai/hooks/useProcessUIToolCallMessage.ts`
- `packages/twenty-front/src/modules/ai/hooks/useChatTargetNavigation.ts`
- `packages/twenty-server/src/engine/metadata-modules/ai/ai-agent-execution/constants/workflow-agent-excluded-tool-names.const.ts`

### Auth detail

- `packages/twenty-server/src/engine/core-modules/auth/token/services/access-token.service.ts` (session token refused as Bearer)

### Tests that lock the permission story

- `packages/twenty-server/test/integration/ai/suites/mcp-tool-catalog.integration-spec.ts`
- `packages/twenty-server/test/integration/ai/suites/mcp.controller.integration-spec.ts`
- `packages/twenty-server/test/integration/graphql/suites/application-role-intersection/` (MCP catalogue narrowing and refused delete)

---

## Other eng List-fit from code

Skim only. The aim is to keep List’s engagement model (Clerk, Templates ticked by engagement id, Confirm on the desk-tools write). These are not features to port.

### Board / kanban

`ViewType.KANBAN` is a view layout, next to `TABLE`, `LIST`, and `CALENDAR` (`packages/twenty-shared/src/types/ViewType.ts`). Creating a kanban view stores `mainGroupByFieldMetadataId` only when the type is `KANBAN` (`packages/twenty-front/src/modules/views/view-picker/hooks/useCreateViewFromCurrentState.ts`). Changing that field rebuilds view groups (`handleFlatViewUpdateSideEffect` in `packages/twenty-server/src/engine/metadata-modules/flat-view/utils/handle-flat-view-update-side-effect.util.ts`). Columns are view groups over the same records, not a second object.

**List:** a board of engagements is a grouping of the engagement rows List already stores (status or stage on the engagement). Template ticks and Confirm stay on the engagement id. Do not add a board table.

### Detail related lists

A record page can embed a relation as a table widget. `getFieldWidgetRelationTraversal` points the embedded view at the far object and scopes it back through the inverse field to the current record id (`packages/twenty-front/src/modules/page-layout/widgets/field/utils/getFieldWidgetRelationTraversal.ts`). `FieldWidgetRelationTable` renders that view for the open `recordId`. A one-to-many hop is what carries the join column back to the current record. Nested hops and junction widgets exist (`RecordTableWidgetContext.ts`); they are extra, not the List shape.

**List:** related rows on an engagement (template lines, child work) are a query filtered by engagement id, written through the same desk-tools path. The widget does not own a private store.

### Same-object parent → child

`RelationType` is `MANY_TO_ONE` or `ONE_TO_MANY` (`packages/twenty-shared/src/types/RelationType.ts`). Creating a relation writes a pair: the many-to-one side holds `joinColumnName` and `onDelete: SET_NULL`; the other side is the one-to-many inverse with no join column (`generateMorphOrRelationFlatFieldMetadataPair` in `packages/twenty-server/src/engine/metadata-modules/flat-field-metadata/utils/generate-morph-or-relation-flat-field-metadata-pair.util.ts`). Source and target are separate object-metadata ids. `validateRelationCreationPayload` checks that the target exists; it does not reject a target id equal to the source object (`packages/twenty-server/src/engine/metadata-modules/flat-field-metadata/validators/utils/validate-relation-creation-payload.util.ts`). This skim did not find a standard object (task, company, person) that ships a parent field to itself.

**List:** if an engagement has a parent engagement, the child row holds the parent id and the parent’s children are the inverse query — the same mechanism as the related list. Do not import Twenty’s metadata relation builder. Do not invent a second hierarchy store.

### Permissions / roles

A user workspace maps to one role id (`UserRoleService.getRoleIdForUserWorkspace` reads `userWorkspaceRoleMap`). A role has global record flags (`canReadAllObjectRecords`, `canUpdateAllObjectRecords`, `canSoftDeleteAllObjectRecords`, `canDestroyAllObjectRecords`) and per-object overrides for the same four verbs (`role-permissions.schema.ts`, `upsert-object-permissions.tool.ts`). Omitting an object from the override list drops that override and falls back to the global flags. Assignability is separate for users, agents, and API keys (`canBeAssignedToUsers`, `canBeAssignedToAgents`, `canBeAssignedToApiKeys`).

**List:** the Clerk user is one role. An agent or MCP key is assigned only if that role allows it, and its grants stay the intersection described in section B.4. “May tick a template” belongs on that role (or on Confirm inside the write), not in a chat-only allow-list. Read and destroy are different flags; do not treat read as permission to tick.

### API write shape

Core REST is `RestApiCoreController` at `ApiPath.Rest` (`/rest`), guarded by JWT, workspace, and `CustomPermissionGuard`:

| Verb | Effect |
| --- | --- |
| `POST /rest/{object}` | create one, HTTP 201 |
| `POST /rest/batch/{object}` | create many |
| `GET` | find one or many |
| `PATCH` and `PUT` | both call `update` (PUT kept so old clients do not break; comment in the controller) |
| `DELETE` | see below |
| `PATCH …/restore` | restore |
| `PATCH …/merge` | merge many |

`RestApiCoreService.delete` reads `soft_delete`. `parseSoftDeleteRestRequest` returns false when the query param is absent, so a bare `DELETE` is destroy (`rest-api-destroy-one.handler.ts` → `CommonDestroyOneQueryRunnerService`). `soft_delete=true` soft-deletes (`CommonDeleteOneQueryRunnerService`). The AI tool path does the opposite for `delete_one`: `ToolExecutorService` always passes `soft: true` and does not expose restore.

REST handlers and the AI record-crud services both call those common query runners. GraphQL record operations use the same runner family. The write shape is one runner layer, several fronts.

**List:** desk-tools, MCP, and the later chatbox must call that one write. Match the verb List already uses for a template tick. Do not let an agent `DELETE` mean “tick” or “archive” because Twenty’s REST default and Twenty’s agent delete disagree. Confirm sits in the runner List already trusts, so every front hits it.

## What this note did not verify

- Runtime behaviour against a running Twenty workspace. Findings are from source and the tests named above, not from an executed chat, MCP session, or board.
- 1231FS List, Clerk, Templates, or Confirm implementation. Section C and the List lines above are risks against the stated framing.
- A UX, visual, or security review. Other seats own those. This note does not judge layout, colour, or threat models.
- Whether Twenty’s commercial `@license Enterprise` headers appear on any AI file. The root `LICENSE` describes that marker; this note did not inventory every AI file for it. The server package licence field is still `AGPL-3.0`.
- A shipped same-object parent field on a standard CRM object. The relation pair allows it; this skim did not find one.
