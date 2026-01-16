# Notion Integration Templates

**Purpose**: CSV export formats and field mappings for manual Notion imports when API sync fails.

---

## Database Structure

### Projects Database
**Collection ID**: `999c414c-06c5-4064-a51b-921193830968`
**URL**: https://www.notion.so/83e8d8f77e5a45bb96f7188c6fe092d3

**Properties**:
| Property | Type | Description |
|----------|------|-------------|
| Name | Title | Project name (unique identifier) |
| Status | Select | Active \| Paused \| Completed \| Archived |
| Context Notes | Text | Session metadata + continuation brief |
| Decision Log | Text | Timestamped key decisions |
| Blockers | Text | Current impediments |
| Next Actions | Text | Summary of pending tasks |
| Last Updated | Date | Auto-timestamp on sync |
| Tags | Multi-select | Project categories/labels |

### Actions Database
**Collection ID**: `6b52d580-f810-4009-964d-478039c144e1`

**Properties**:
| Property | Type | Description |
|----------|------|-------------|
| Action | Title | Task description |
| Status | Select | Not Started \| In Progress \| Blocked \| Done |
| Notes | Text | Technical details, context, references |
| Parent Project | Relation | Link to Projects DB (required) |
| Due | Date | Optional deadline |
| Tags | Multi-select | Entity types, service names, etc. |
| Created | Date | Auto-timestamp on creation |

---

## CSV Export Templates

### Projects Export

**Filename**: `chittyxl_projects_[timestamp].csv`

**Headers**:
```csv
Name,Status,Context Notes,Decision Log,Blockers,Next Actions,Tags
```

**Example Row**:
```csv
"Payment Service v2","Active","[SESSION:2025-01-16-14:32] Brief: Implementing multi-provider payment system

[ENTITIES]
- PaymentAdapter (interface)
- TransactionLog (model)
- ProviderConfig (config)

[DECISIONS]
2025-01-16 14:15 - Using adapter pattern for provider abstraction
2025-01-16 14:28 - PostgreSQL for transaction logging (not MongoDB)

[CONTINUATION]
Next: Complete PaymentAdapter implementation, set up transaction_log table
Blockers: Awaiting vendor API keys for testing","2025-01-16 14:15 - Using adapter pattern for provider abstraction
2025-01-16 14:28 - PostgreSQL for transaction logging","Awaiting vendor API keys for testing","Complete PaymentAdapter implementation
Set up transaction_log table
Integrate provider adapters","backend,payments,api"
```

**Formatting Rules**:
- **Multiline text**: Wrap in double quotes, use actual newlines (not \n)
- **Commas in content**: Must wrap entire field in double quotes
- **Quotes in content**: Escape with double quotes ("")
- **Empty fields**: Leave blank or use empty quotes ""
- **Arrays (Tags)**: Comma-separated values

---

### Actions Export

**Filename**: `chittyxl_actions_[timestamp].csv`

**Headers**:
```csv
Action,Status,Notes,Parent Project,Due,Tags
```

**Example Rows**:
```csv
"Implement PaymentAdapter interface","In Progress","Base interface with three core methods:
- process(payment_data): Process transaction
- refund(transaction_id): Handle refunds
- validate(config): Verify provider configuration

Reference: See payment_adapter.ts in /src/services/","Payment Service v2","2025-01-20","backend,interface"
"Set up transaction_log table","Not Started","Schema definition:
- id: UUID primary key
- provider: VARCHAR (stripe, paypal, etc.)
- amount: DECIMAL(10,2)
- status: ENUM (pending, completed, failed, refunded)
- metadata: JSONB
- created_at: TIMESTAMP

Migration: Create in PostgreSQL main DB","Payment Service v2","2025-01-22","database,schema"
"Integrate Stripe adapter","Not Started","Implement StripeAdapter extends PaymentAdapter
Use Stripe SDK v12.x
Config: API key from env vars
Test with Stripe test mode","Payment Service v2","2025-01-25","backend,integration,payments"
```

**Formatting Rules** (same as Projects):
- Multiline notes: Wrap in quotes, actual newlines
- Parent Project: Must match exact name from Projects DB
- Due: ISO date format (YYYY-MM-DD) or leave empty
- Tags: Comma-separated, no quotes around individual tags

---

## Import Instructions (Manual)

### Using Notion CSV Import

**Step 1: Prepare CSV File**
- Generate from checkpoint artifact
- Save as `.csv` file (not `.xlsx` or `.txt`)
- Verify formatting (quotes, newlines, commas)

**Step 2: Open Target Database**
- Navigate to Projects or Actions database in Notion
- Click "..." menu in top right
- Select "Merge with CSV"

**Step 3: Upload & Map Fields**
- Upload your CSV file
- Notion will auto-detect headers
- **Map columns** to database properties:
  - CSV "Name" → DB "Name"
  - CSV "Status" → DB "Status"
  - CSV "Context Notes" → DB "Context Notes"
  - etc.

**Step 4: Import Options**
- **Existing items**: "Update existing pages" (match by Name/Action)
- **New items**: "Create new pages"
- **Preview**: Check first few rows
- Click "Import"

**Step 5: Verify Import**
- Check imported projects/actions appear in database
- Verify multiline text preserved correctly
- Check relations (Parent Project links) working
- Fix any formatting issues manually if needed

---

## CSV Generation Code (Reference)

**For Projects**:
```python
import csv
from datetime import datetime

def generate_projects_csv(projects, filename=None):
    """Generate projects CSV export"""
    if not filename:
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"chittyxl_projects_{timestamp}.csv"

    with open(filename, 'w', newline='', encoding='utf-8') as f:
        writer = csv.writer(f, quoting=csv.QUOTE_MINIMAL)

        # Header
        writer.writerow([
            'Name', 'Status', 'Context Notes', 'Decision Log',
            'Blockers', 'Next Actions', 'Tags'
        ])

        # Data rows
        for project in projects:
            writer.writerow([
                project.name,
                project.status,
                project.context_notes,  # Multiline OK
                project.decision_log,   # Newline-separated decisions
                project.blockers,       # Newline-separated blockers
                project.next_actions,   # Newline-separated actions
                ','.join(project.tags)  # Comma-separated tags
            ])

    return filename
```

**For Actions**:
```python
def generate_actions_csv(actions, filename=None):
    """Generate actions CSV export"""
    if not filename:
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"chittyxl_actions_{timestamp}.csv"

    with open(filename, 'w', newline='', encoding='utf-8') as f:
        writer = csv.writer(f, quoting=csv.QUOTE_MINIMAL)

        # Header
        writer.writerow([
            'Action', 'Status', 'Notes', 'Parent Project', 'Due', 'Tags'
        ])

        # Data rows
        for action in actions:
            writer.writerow([
                action.task,
                action.status,
                action.notes,           # Multiline technical details OK
                action.parent_project,  # Must match project name exactly
                action.due.isoformat() if action.due else '',
                ','.join(action.tags)   # Comma-separated tags
            ])

    return filename
```

---

## Context Notes Format Specification

**Purpose**: Store session metadata in Projects DB for continuation.

**Structure**:
```
[SESSION:YYYY-MM-DD-HH:MM] Brief description of session focus

[ENTITIES]
- EntityName (type): description
- AnotherEntity (type): description

[DECISIONS]
YYYY-MM-DD HH:MM - Decision text with rationale
YYYY-MM-DD HH:MM - Another decision

[TECHNICAL]
Key technical details, schemas, API endpoints, references

[CONTINUATION]
What to do next
Current blockers
Resume point for next session
```

**Example**:
```
[SESSION:2025-01-16-14:32] Implementing multi-provider payment system with adapter pattern

[ENTITIES]
- PaymentAdapter (interface): Core abstraction for payment providers
- TransactionLog (model): PostgreSQL table for audit trail
- ProviderConfig (config): Environment-based provider credentials
- StripeAdapter (implementation): Stripe-specific payment adapter
- PayPalAdapter (implementation): PayPal-specific payment adapter

[DECISIONS]
2025-01-16 14:15 - Using adapter pattern for provider abstraction (vs direct integration)
2025-01-16 14:28 - PostgreSQL for transaction logging (vs MongoDB) - better ACID guarantees
2025-01-16 14:45 - Storing provider credentials in environment variables (vs database)

[TECHNICAL]
API Endpoints:
- POST /api/v2/payments/process (body: {provider, amount, metadata})
- POST /api/v2/payments/refund (body: {transaction_id, amount})
- GET /api/v2/payments/{id} (returns transaction details)

Database Schema (transaction_log):
- id: UUID PRIMARY KEY
- provider: VARCHAR(50) NOT NULL
- amount: DECIMAL(10,2) NOT NULL
- status: ENUM('pending', 'completed', 'failed', 'refunded')
- metadata: JSONB
- created_at: TIMESTAMP DEFAULT NOW()
- updated_at: TIMESTAMP DEFAULT NOW()

Interface Definition (PaymentAdapter):
```typescript
interface PaymentAdapter {
  process(data: PaymentData): Promise<TransactionResult>;
  refund(transactionId: string, amount?: number): Promise<RefundResult>;
  validate(config: ProviderConfig): Promise<boolean>;
}
```

[CONTINUATION]
NEXT STEPS:
1. Complete PaymentAdapter interface implementation in payment_adapter.ts
2. Create transaction_log table migration (migrations/002_transaction_log.sql)
3. Implement StripeAdapter class (services/stripe_adapter.ts)
4. Write integration tests for adapter pattern

BLOCKERS:
- Awaiting Stripe API keys from ops team (requested 2025-01-15)
- PayPal merchant account approval pending (ETA: 2025-01-20)

RESUME POINT:
Start with PaymentAdapter interface - base methods defined, need implementation details.
Reference: Stripe SDK docs at https://stripe.com/docs/api, PayPal at https://developer.paypal.com
```

---

## Decision Log Format

**Purpose**: Track key decisions with timestamps for project history.

**Format**: One decision per line, timestamp + description
```
YYYY-MM-DD HH:MM - Decision text with rationale
```

**Example**:
```
2025-01-16 14:15 - Using adapter pattern for provider abstraction (vs direct integration) - better extensibility for future providers
2025-01-16 14:28 - PostgreSQL for transaction logging (vs MongoDB) - need ACID guarantees for financial data
2025-01-16 14:45 - Environment variables for credentials (vs database) - security best practice, easier rotation
2025-01-16 15:02 - Stripe SDK v12.x (vs v11.x) - has better TypeScript support and async/await
```

**Guidelines**:
- **Timestamp**: When decision was made (not when logged)
- **Decision**: What was chosen
- **Context**: Why (alternatives considered, rationale)
- **Concise**: ≤100 words per decision
- **Actionable**: Focus on decisions that affect implementation

---

## Blockers Format

**Purpose**: Track impediments to progress.

**Format**: One blocker per line, optional context
```
Blocking item - Context/impact - Resolution needed
```

**Example**:
```
Awaiting Stripe API keys - Blocking adapter testing - Ops team request sent 2025-01-15
PayPal merchant approval pending - Can't integrate PayPal adapter - ETA: 2025-01-20
Redis cluster not provisioned - Blocking rate limiter implementation - Infrastructure ticket #1234
Unclear requirement on refund policy - Blocking refund() method logic - Need PM clarification
```

**Guidelines**:
- **Active blockers only** - Remove when resolved
- **Clear impact** - What's blocked by this
- **Resolution path** - Who/what/when to unblock
- **Owner** - Who's responsible for unblocking (if not self)

---

## Tags Taxonomy

**Project Tags**:
- **Type**: `feature`, `bugfix`, `refactor`, `infrastructure`, `research`
- **Domain**: `backend`, `frontend`, `database`, `api`, `devops`
- **Tech**: `typescript`, `python`, `postgres`, `redis`, `docker`
- **Priority**: `p0-critical`, `p1-high`, `p2-medium`, `p3-low`
- **Status**: `blocked`, `in-review`, `needs-testing`, `ready-to-deploy`

**Action Tags**:
- **Type**: `coding`, `testing`, `documentation`, `deployment`, `review`
- **Component**: `auth`, `payments`, `dashboard`, `api`, `database`
- **Effort**: `quick-win`, `medium`, `complex`, `research-needed`

**Example**:
```
Project: "Payment Service v2"
Tags: feature,backend,payments,typescript,p1-high

Action: "Implement PaymentAdapter interface"
Tags: coding,payments,backend,medium
```

---

## Fallback CSV Generation (Checkpoint Failure)

When Notion API sync fails during checkpoint, generate CSV artifacts for manual import.

**Artifact Structure**:
```markdown
# Checkpoint #3 - Manual Import Required

Notion API sync failed. Import these CSVs manually:

## Projects Export
[Download: chittyxl_projects_20250116_143200.csv]

```csv
Name,Status,Context Notes,Decision Log,Blockers,Next Actions,Tags
"Payment Service v2","Active","[SESSION:2025-01-16-14:32] ...","...","...","...","backend,payments"
```

## Actions Export
[Download: chittyxl_actions_20250116_143200.csv]

```csv
Action,Status,Notes,Parent Project,Due,Tags
"Implement PaymentAdapter","In Progress","...","Payment Service v2","2025-01-20","backend"
"Set up transaction_log","Not Started","...","Payment Service v2","2025-01-22","database"
```

## Import Instructions
1. Save each CSV section to a file
2. Notion → Database → "..." → "Merge with CSV"
3. Upload file, map columns, import
4. Verify relations (Parent Project) linked correctly
```

---

## Version & Support

**Version**: 2.0.0
**Last Updated**: 2025-01-16
**Notion Tracker**: https://www.notion.so/83e8d8f77e5a45bb96f7188c6fe092d3
**Projects DB**: collection://999c414c-06c5-4064-a51b-921193830968
**Actions DB**: collection://6b52d580-f810-4009-964d-478039c144e1
