# ChittyXL Session Protocol - Checkpoint Algorithm

**Purpose**: Define the exact checkpoint execution process for consistent, efficient state persistence.

---

## Checkpoint Trigger Detection

### Token Counting
Monitor context usage continuously (implementation varies by Claude interface):
- **Claude Code CLI**: Track via internal token counter if available
- **Estimation**: ~4 chars = 1 token (rough approximation)
- **Explicit**: User can request `status` for current count

### Thresholds (190k budget)
```
Level 1: 38,000 tokens  (20%)  → First checkpoint
Level 2: 76,000 tokens  (40%)  → Second checkpoint
Level 3: 114,000 tokens (60%)  → Third checkpoint
Level 4: 152,000 tokens (80%)  → Fourth checkpoint
Level 5: 171,000 tokens (90%)  → HARD LIMIT → Force compact + fork alert
```

### Auto vs Manual
- **Auto**: Triggered when threshold crossed during response generation
- **Manual**: User command `checkpoint` forces execution immediately
- **Behavior**: Identical process for both triggers

---

## Checkpoint Execution Phases

### Phase 1: State Extraction (2-5s)

**Scan conversation** for:
1. **Projects**: Named initiatives, features, systems
   - Look for: "project", "initiative", "building", "working on"
   - Extract: Name, current status, key decisions, blockers

2. **Actions**: Specific tasks, TODOs, pending work
   - Look for: "need to", "TODO", "next step", "action item"
   - Extract: Task description, status, related project, technical notes

3. **Decisions**: Key choices made during session
   - Look for: "decided", "choosing", "going with", "will use"
   - Extract: What was decided, rationale, timestamp/context

4. **Blockers**: Impediments, questions, waiting-for items
   - Look for: "blocked", "waiting", "need clarification", "unclear"
   - Extract: What's blocking, impact, resolution needed

5. **Technical Context**: Schemas, APIs, entities, architecture
   - Look for: Data models, service definitions, API endpoints
   - Extract: Entity names, relationships, key technical references

**Example Extraction**:
```json
{
  "projects": [
    {
      "name": "Payment Service v2",
      "status": "Active",
      "context": "Migrating from Stripe to multi-provider",
      "decisions": ["Using adapter pattern", "PostgreSQL for transaction log"],
      "blockers": ["Awaiting vendor API keys"]
    }
  ],
  "actions": [
    {
      "task": "Implement PaymentAdapter interface",
      "status": "In Progress",
      "project": "Payment Service v2",
      "notes": "Base interface with process(), refund(), validate() methods"
    },
    {
      "task": "Set up PostgreSQL transaction_log table",
      "status": "Not Started",
      "project": "Payment Service v2",
      "notes": "Schema: id, provider, amount, status, metadata, created_at"
    }
  ],
  "technical_context": {
    "entities": ["PaymentAdapter", "TransactionLog", "ProviderConfig"],
    "apis": ["POST /payments/process", "POST /payments/refund"],
    "schemas": ["transaction_log table definition"]
  }
}
```

---

### Phase 2: Deduplication (1-2s)

**Remove redundancy**:
- **Duplicate projects**: Keep most recent status, merge decisions
- **Superseded actions**: If "implemented X" appears later, mark earlier "implement X" as Done
- **Repetitive decisions**: Keep final decision only, discard deliberation
- **Resolved blockers**: If "blocker resolved" mentioned, remove from blockers list

**Example**:
```
BEFORE DEDUP:
- Action: "Design payment adapter interface" (early conversation)
- Action: "Implement PaymentAdapter interface" (later conversation)
- Decision: "Considering adapter pattern" (early)
- Decision: "Using adapter pattern" (later)

AFTER DEDUP:
- Action: "Implement PaymentAdapter interface" | Status: In Progress
- Decision: "Using adapter pattern for multi-provider support"
```

**Deduplication Rules**:
1. **Actions**: If task implies completion of earlier task, mark earlier as Done
2. **Projects**: One entry per unique project name, merged status
3. **Decisions**: Final decision wins, remove exploratory discussion
4. **Blockers**: Remove if resolution mentioned in later conversation

---

### Phase 3: Notion Sync (3-7s)

**A. Update Projects Database**

**Query**: Check if project exists by name
- **If exists**: Update Status, append to Context Notes, add decisions
- **If new**: Create new project entry

**Context Notes Format**:
```
[SESSION:2025-01-16-14:32] Brief: Implementing multi-provider payment system

[ENTITIES]
- PaymentAdapter (interface)
- TransactionLog (model)
- ProviderConfig (config)

[DECISIONS]
2025-01-16 14:15 - Using adapter pattern for provider abstraction
2025-01-16 14:28 - PostgreSQL for transaction logging (not MongoDB)

[CONTINUATION]
Next: Complete PaymentAdapter implementation, set up transaction_log table
Blockers: Awaiting vendor API keys for testing
```

**API Call** (conceptual):
```javascript
notion.pages.update({
  page_id: "<project-page-id>",
  properties: {
    "Status": { select: { name: "Active" } },
    "Context Notes": {
      rich_text: [{ text: { content: contextNotesText } }]
    },
    "Decision Log": {
      rich_text: [{ text: { content: decisionsText } }]
    },
    "Blockers": {
      rich_text: [{ text: { content: blockersText } }]
    },
    "Last Updated": { date: { start: new Date().toISOString() } }
  }
})
```

**B. Sync Actions Database**

For each action extracted:
1. **Query**: Check if action exists (match by description + parent project)
2. **If exists**: Update Status, append to Notes
3. **If new**: Create new action entry with Parent Project relation

**API Call** (conceptual):
```javascript
notion.pages.create({
  parent: { database_id: "<actions-db-id>" },
  properties: {
    "Action": {
      title: [{ text: { content: "Implement PaymentAdapter interface" } }]
    },
    "Status": { select: { name: "In Progress" } },
    "Notes": {
      rich_text: [{ text: { content: "Base interface with process(), refund(), validate()" } }]
    },
    "Parent Project": {
      relation: [{ id: "<payment-service-v2-project-id>" }]
    },
    "Tags": {
      multi_select: [
        { name: "backend" },
        { name: "payments" }
      ]
    }
  }
})
```

**Error Handling**:
- **Notion API failure**: Retry once after 2s delay
- **Still failing**: Generate CSV export artifact as fallback
- **Rate limits**: Queue updates, process batch on next checkpoint

---

### Phase 4: Summary Generation (1-2s)

**A. Create Checkpoint Summary Artifact**

**Format**: Markdown, concise bullet points

**Template**:
```markdown
# Checkpoint #3 Summary
**Tokens**: 114k/190k (60%)
**Timestamp**: 2025-01-16 14:32
**Next Checkpoint**: 152k (80%)

## Projects Updated
- **Payment Service v2** | Active
  - Decisions: Adapter pattern, PostgreSQL transaction log
  - Blockers: Awaiting vendor API keys

## Actions Modified
- ✓ Design payment adapter interface → Done
- ⚙️ Implement PaymentAdapter interface → In Progress
- ◯ Set up transaction_log table → Not Started
- ◯ Integrate Stripe adapter → Not Started
- ◯ Integrate PayPal adapter → Not Started

## Technical Context
**Entities**: PaymentAdapter, TransactionLog, ProviderConfig
**APIs**: POST /payments/process, POST /payments/refund
**Schema**: transaction_log (id, provider, amount, status, metadata, created_at)

## Notion Sync
✓ Projects: 1 updated
✓ Actions: 5 synced (1 updated, 4 created)
✓ Tracker: https://www.notion.so/83e8d8f77e5a45bb96f7188c6fe092d3
```

**B. Optional: CSV Export Artifact**

If Notion sync failed, or user prefers manual import:

**projects_export.csv**:
```csv
Name,Status,Context Notes,Decision Log,Blockers,Next Actions
"Payment Service v2","Active","[SESSION:2025-01-16-14:32] Multi-provider payment system...","2025-01-16 14:15 - Adapter pattern|2025-01-16 14:28 - PostgreSQL","Awaiting vendor API keys","Complete adapter implementation"
```

**actions_export.csv**:
```csv
Action,Status,Notes,Parent Project,Tags
"Implement PaymentAdapter interface","In Progress","Base interface with process(), refund(), validate()","Payment Service v2","backend,payments"
"Set up transaction_log table","Not Started","Schema: id, provider, amount, status, metadata, created_at","Payment Service v2","backend,database"
```

---

### Phase 5: User Notification (≤3 lines in chat)

**Success Message**:
```
Checkpoint #3 complete | 114k/190k (60%) | Next: 152k
Synced: 1 project, 5 actions → Notion
```

**With Failures**:
```
Checkpoint #3 complete | 114k/190k (60%) | Next: 152k
Notion sync failed → CSV export in artifact ↑
```

**Hard Limit Warning** (at 171k):
```
⚠️ Checkpoint #5 | 171k/190k (90%) - Session capacity reached
State saved → Consider `fork` for fresh session
```

---

## Special Case: Session Continuation

**Trigger**: User command `continue` in new session

### Process

**Step 1: Query Notion Projects** (2-3s)
```
Filter: Status != "Completed" AND Status != "Archived"
Sort: Last Updated DESC
Limit: 10 most recent
```

**Step 2: Parse Context Notes** (1s)
Extract session metadata:
```
[SESSION:timestamp] → Last active timestamp
[ENTITIES] → Technical context to restore
[DECISIONS] → Decision history
[CONTINUATION] → What to do next
```

**Step 3: Load Related Actions** (2-3s)
```
For each active project:
  Query Actions where:
    - Parent Project = <project>
    - Status != "Done"
  Group by project
```

**Step 4: Generate Briefing Artifact** (1-2s)

**Template**:
```markdown
# Session Continuation Briefing
**Last Active**: 2 hours ago (2025-01-16 14:32)
**Tracker**: https://www.notion.so/83e8d8f77e5a45bb96f7188c6fe092d3

---

## Active Projects (3)

### 1. Payment Service v2
**Status**: Active
**Last Decision**: Using adapter pattern for multi-provider support
**Blockers**: Awaiting vendor API keys
**Pending Actions** (5):
- ⚙️ Implement PaymentAdapter interface (In Progress)
- ◯ Set up transaction_log table
- ◯ Integrate Stripe adapter
- ◯ Integrate PayPal adapter
- ◯ Write adapter integration tests

### 2. User Dashboard Redesign
**Status**: Active
**Last Decision**: Moving to component-based architecture
**Blockers**: None
**Pending Actions** (3):
- ⚙️ Create DashboardLayout component (In Progress)
- ◯ Build MetricsCard widget
- ◯ Add responsive breakpoints

### 3. API Rate Limiting
**Status**: Paused
**Blocker**: Waiting for Redis cluster provisioning
**Pending Actions** (1):
- ◯ Implement sliding window rate limiter

---

## Technical Context to Restore

**Entities**:
- PaymentAdapter, TransactionLog, ProviderConfig (Payment Service)
- DashboardLayout, MetricsCard (User Dashboard)
- RateLimiter (API)

**APIs**:
- POST /payments/process
- POST /payments/refund
- GET /dashboard/metrics

**Schemas**:
- transaction_log: id, provider, amount, status, metadata, created_at
- dashboard_widgets: id, type, config, position, user_id

---

## Suggested Next Steps
1. Complete PaymentAdapter implementation (methods: process, refund, validate)
2. Finish DashboardLayout component (add widget grid)
3. Follow up on Redis provisioning for rate limiter
```

**Step 5: Chat Confirmation** (≤150 words)
```
Session restored from Notion ✓

Active: 3 projects, 9 pending actions
Last checkpoint: 2h ago (Checkpoint #3 at 114k tokens)

Priority: Complete PaymentAdapter + DashboardLayout
Blocked: API Rate Limiting (waiting on Redis)

Ready to continue - see briefing artifact ↑ for full context.
```

---

## Performance Optimization

### Reduce Latency
1. **Parallel API calls**: Query projects + actions concurrently
2. **Batch updates**: Group Notion writes where possible
3. **Incremental state**: Track changes since last checkpoint (don't re-extract everything)
4. **Caching**: Store project IDs to avoid repeated lookups

### Reduce Token Usage
1. **Compact summaries**: Bullet points, abbreviations, no fluff
2. **Artifacts for details**: Keep chat minimal, put technical content in artifacts
3. **Dedup aggressively**: Remove all redundancy, keep only actionable state

### Reliability
1. **Idempotent syncs**: Safe to run checkpoint twice (upsert pattern)
2. **Fallback to CSV**: If Notion fails, still preserve state in artifact
3. **Retry logic**: One retry on transient failures
4. **Logging**: Store checkpoint metadata in Notion Context Notes for debugging

---

## Checkpoint Algorithm Pseudocode

```python
def execute_checkpoint(token_count, checkpoint_number):
    """Execute full checkpoint process"""

    # Phase 1: Extract (2-5s)
    state = extract_conversation_state()
    # Returns: {projects, actions, decisions, blockers, technical_context}

    # Phase 2: Deduplicate (1-2s)
    state = deduplicate_state(state)
    # Removes redundancy, merges duplicates

    # Phase 3: Sync to Notion (3-7s)
    try:
        sync_result = sync_to_notion(state)
        notion_success = True
    except NotionAPIError as e:
        # Retry once
        try:
            sync_result = sync_to_notion(state)
            notion_success = True
        except:
            # Fallback to CSV export
            csv_artifact = generate_csv_export(state)
            notion_success = False

    # Phase 4: Generate summary (1-2s)
    summary_artifact = generate_checkpoint_summary(
        checkpoint_number=checkpoint_number,
        token_count=token_count,
        state=state,
        sync_result=sync_result if notion_success else None
    )

    # Phase 5: Notify user (≤3 lines)
    next_threshold = get_next_threshold(token_count)

    if notion_success:
        print(f"Checkpoint #{checkpoint_number} complete | {token_count}/190k | Next: {next_threshold}")
        print(f"Synced: {len(state.projects)} projects, {len(state.actions)} actions → Notion")
    else:
        print(f"Checkpoint #{checkpoint_number} complete | {token_count}/190k | Next: {next_threshold}")
        print(f"Notion sync failed → CSV export in artifact ↑")

    # Log to Notion (for debug)
    log_checkpoint_metadata(checkpoint_number, token_count, latency, sync_result)

    # Update internal state
    update_next_checkpoint_threshold(next_threshold)

    # Hard limit warning
    if token_count >= 171000:  # 90%
        print("⚠️ Session capacity: 90% - Consider `fork` for fresh session")

def get_next_threshold(current_tokens):
    """Calculate next checkpoint threshold"""
    thresholds = [38000, 76000, 114000, 152000, 171000]
    for t in thresholds:
        if current_tokens < t:
            return t
    return None  # Past hard limit
```

---

## Metrics & Monitoring

**Track per session**:
- Total checkpoints executed
- Average checkpoint latency (target: <20s)
- Notion sync success rate (target: >95%)
- Token efficiency (checkpoints per 190k tokens, target: 8-10)
- State persistence rate (successful continuation %, target: 100%)

**Log format** (store in Notion Context Notes):
```
[LOG:2025-01-16-14:32] CP#3 | 114k tok | Lat: 12s | Notion: ✓ | Projects: 1 updated, Actions: 5 synced
```

**Alert conditions**:
- Checkpoint latency >20s → Investigate Notion API lag
- Notion sync failure rate >10% → Check API credentials/rate limits
- Token usage >90% without fork → Warn user of capacity issue

---

## Version History

**v2.0.0** (2025-01-16)
- Initial implementation
- 20% interval checkpoints (38k, 76k, 114k, 152k, 171k)
- Notion auto-sync with CSV fallback
- Session continuation via `continue` command

**Future Enhancements**:
- Adaptive checkpoint intervals (more frequent if high complexity)
- Multi-session threading (link related sessions)
- Automatic project archival (completed >30 days ago)
