# ChittyXL v2.0 - Web Deployment Guide

**Target**: Claude.ai web interface (claude.ai)
**Method**: Project custom instructions
**Scope**: Single project (repeat for multiple projects)

---

## Prerequisites

- Active Claude.ai account (Pro recommended for higher token limits)
- Existing project OR create new project for ChittyXL deployment
- Notion tracker setup (see Notion Setup section below)

---

## Deployment Steps

### Step 1: Navigate to Project Settings

1. Open Claude.ai in browser
2. Click **project dropdown** in top navigation (e.g., "ChittyOS", "My Project")
3. Select **"Project settings"** from dropdown menu
4. Navigate to **"Custom instructions"** tab

### Step 2: Prepare Custom Instructions

**IMPORTANT**: Do NOT replace existing custom instructions. Append ChittyXL block to existing content.

**Locate** the `<userPreferences>` section (if it exists).
**Append** the ChittyXL block below to the END of userPreferences.
**If no userPreferences**: Create the section and add ChittyXL block inside it.

### Step 3: Add ChittyXL Configuration Block

**Copy this entire XML block** and paste into your custom instructions:

```xml
<!-- ChittyXL v2.0: Auto-Compacting Session Persistence -->
<chittyxl_protocol>
## ChittyXL v2.0 (AUTO-ACTIVE)

**Status**: ENABLED | **Priority**: CRITICAL | **Version**: 2.0.0

### Auto-Load Behavior
• Initialize silently on every session start
• Monitor token usage continuously: current/budget
• Trigger checkpoints at: 38k, 76k, 114k, 152k tokens (20% intervals)
• Hard limit: 171k tokens (90%) - force compact + session fork alert
• Persist state to Notion tracker on every checkpoint

### Artifact-First Protocol (ENFORCED)
**Chat window limited to:**
• Confirmation messages: ≤3 lines
• Blocking questions only
• Checkpoint summaries: ≤150 words, bullet format

**Always use artifacts for:**
• Technical specifications, schemas, APIs
• Code implementations, scripts, functions
• Documentation, guides, protocols
• Analysis results, data outputs, reports
• CSV exports (Notion import format)

**Always use Notion tracker for:**
• Project state: Context Notes, Decision Log, Blockers
• Actions: Status, Notes, Due, Parent Project relations
• Session metadata: Continuation context, technical references
• Cross-references: Entities, services, relationships

### Checkpoint Execution (Automatic)
**At each threshold (38k, 76k, 114k, 152k, 171k):**
1. Extract conversation state: projects, actions, decisions, blockers
2. Deduplicate: remove repetition, keep final decisions only
3. Sync to Notion: update projects, create/update actions
4. Generate compact summary: ≤150 words + CSV artifact
5. Update next checkpoint threshold
6. Confirm to user: ≤3 lines

### Notion Integration
• Tracker: https://www.notion.so/83e8d8f77e5a45bb96f7188c6fe092d3
• Projects: collection://999c414c-06c5-4064-a51b-921193830968
• Actions: collection://6b52d580-f810-4009-964d-478039c144e1
• Auto-sync: Enabled on checkpoints + session end
• State persistence: Context Notes field (session metadata)

### User Commands
• `checkpoint` → immediate compact + sync
• `continue` → load last session from Notion
• `status` → show token usage + next checkpoint
• `fork` → save state + offer fresh session
• `history` → show last 5 checkpoints

### Anti-Patterns (FORBIDDEN)
❌ Long chat explanations (use artifacts)
❌ Inline code blocks >10 lines (use artifacts)
❌ Multiple questions per message (max 1)
❌ Redundant confirmations ("I've done X, Y, Z...")
❌ Apologetic preambles ("Sorry for...")
❌ Over-explaining process ("First I'll..., then...")

### Performance Targets
✓ <150 words per chat message average
✓ >80% content in artifacts/Notion (not chat)
✓ <20 seconds checkpoint latency
✓ Zero state loss across sessions
✓ Seamless continuation within 1 exchange

### Session Continuation
**On "continue" command:**
1. Query Notion: active projects (Status != Completed/Archived)
2. Parse Context Notes: extract session metadata + continuation brief
3. Load pending actions: Status != Done, linked to active projects
4. Generate briefing artifact: projects, actions, blockers, technical context
5. Present in chat: ≤150 word summary with artifact link

### Debug & Monitoring
• Log checkpoints: timestamp, token count, state changes
• Log Notion syncs: success/failure, latency
• Alert at 90%: "Session capacity reached - compacting now"
• Track metrics: checkpoint count, avg latency, state persistence rate
</chittyxl_protocol>
```

---

### Step 4: Save Configuration

1. Click **"Save"** button in Project Settings (bottom right)
2. Verify no syntax errors displayed (red error messages)
3. Close settings modal
4. Return to project chat interface

---

### Step 5: Verification

**Test 1: Initialize New Session**

1. Start fresh chat in the project (click "New chat" if needed)
2. ChittyXL should load silently (no "loaded" message - this is intentional)
3. Type: `status`
4. **Expected response**:
   ```
   ChittyXL: Active ✓
   Token Usage: ~2k/190k (1%)
   Next Checkpoint: 38k (20%)
   Active Projects: 0
   ```

**Test 2: Artifact Enforcement**

1. Request technical content: "Write a TypeScript interface for a payment adapter"
2. **Expected**: Response in artifact block (not inline in chat)
3. **Chat portion**: ≤150 words, summary only
4. **Artifact**: Full TypeScript interface code

**Test 3: Checkpoint Trigger** (optional - requires generating ~40k tokens)

1. Have extended conversation (code implementations, documentation, etc.)
2. When approaching 38k tokens, ChittyXL auto-triggers checkpoint
3. **Expected**: Chat message ≤150 words confirming checkpoint
4. **Artifact**: Checkpoint summary with projects/actions synced
5. **Notion**: Project Last Updated timestamp updated

**Test 4: Session Continuation**

1. Close current session (close browser tab or click "New chat")
2. Start new chat session in same project
3. Type: `continue`
4. **Expected**: Briefing artifact with last session state
5. **Chat**: ≤150 word summary of active projects/actions

---

## Notion Setup (One-Time)

**Required**: Notion workspace with ChittyXL tracker databases.

### Option 1: Use Existing Tracker (Recommended)

**Tracker URL**: https://www.notion.so/83e8d8f77e5a45bb96f7188c6fe092d3

**Steps**:
1. Open tracker URL in browser
2. Verify two databases exist: **Projects** and **Actions**
3. Check Claude integration connected (Settings → Integrations)
4. Test write access: Create test project manually
5. If successful, delete test project → Ready to use

### Option 2: Create New Tracker (Advanced)

**If you need a separate tracker**:

1. **Create Notion page**: "ChittyXL Tracker"

2. **Create Projects database** (table):
   - Name (title)
   - Status (select): Active | Paused | Completed | Archived
   - Context Notes (text)
   - Decision Log (text)
   - Blockers (text)
   - Next Actions (text)
   - Last Updated (date)
   - Tags (multi-select)

3. **Create Actions database** (table):
   - Action (title)
   - Status (select): Not Started | In Progress | Blocked | Done
   - Notes (text)
   - Parent Project (relation to Projects DB)
   - Due (date)
   - Tags (multi-select)
   - Created (date)

4. **Connect Claude integration**:
   - Notion Settings → Integrations
   - Add Claude AI integration
   - Share tracker page with Claude integration

5. **Get database IDs**:
   - Open Projects database in browser
   - Copy URL: `https://notion.so/<workspace>/<database-id>`
   - Extract database ID (32-char hex after last slash)
   - Repeat for Actions database

6. **Update custom instructions**:
   - Replace tracker URL in ChittyXL block
   - Replace Projects collection ID
   - Replace Actions collection ID

---

## Multi-Project Deployment

**To deploy ChittyXL to multiple Claude.ai projects**:

1. Complete deployment for first project (Steps 1-5 above)
2. Copy the `<chittyxl_protocol>` block from custom instructions
3. Navigate to second project → Project settings → Custom instructions
4. Paste the same block
5. Save settings
6. Repeat for additional projects

**All projects share same Notion tracker** → Cross-project continuity.

**Note**: Each project's conversations sync to same Projects/Actions databases in Notion. You can filter by project name if needed.

---

## Updating ChittyXL (Future Versions)

**When new version released**:

1. Open this deployment guide for new version
2. Copy updated `<chittyxl_protocol>` block
3. Navigate to each project using ChittyXL
4. Project Settings → Custom Instructions
5. **Replace** old ChittyXL block with new block (keep other custom instructions)
6. Save settings
7. Start new chat to activate updated version

**Version check**: Type `status` → version shown in response.

---

## Troubleshooting

### ChittyXL not initializing
**Symptom**: `status` command not recognized in new chat

**Solutions**:
- Verify XML block added correctly (no syntax errors on save)
- Check custom instructions saved (re-open settings to confirm)
- Start **new chat** (ChittyXL doesn't retroactively activate in existing chats)
- Verify inside correct project (check project dropdown)
- Try manual activation: paste "Load ChittyXL v2.0 protocol" + brief summary

### Checkpoint not triggering
**Symptom**: Conversation exceeds 38k tokens, no checkpoint message

**Solutions**:
- Force checkpoint: type `checkpoint`
- Check token count estimate: type `status`
- Verify Notion integration connected (test manual write to tracker)
- May be estimation issue (checkpoints based on approximate token count)

### Notion sync failing
**Symptom**: "Notion sync failed → CSV export in artifact" message

**Solutions**:
- Check Notion integration: Settings → Integrations → Claude AI connected
- Verify tracker databases shared with Claude integration
- Test manual write: Create project/action in Notion web interface
- Check database IDs match in custom instructions
- Use CSV export fallback (import manually to Notion)

### Artifacts not generating
**Symptom**: Technical content appears inline in chat, not in artifact

**Solutions**:
- Explicitly request: "Generate as artifact"
- Verify content type qualifies (code, specs, docs >10 lines)
- May be intentional: Simple confirmations don't need artifacts
- Check artifact-first enforcement in custom instructions block

### Session continuation not loading state
**Symptom**: Type `continue`, doesn't restore previous session context

**Solutions**:
- Verify Notion tracker has active projects (Status != Completed/Archived)
- Check Context Notes field populated in Notion project
- Verify Last Updated timestamp is recent
- Manually open Notion tracker, confirm project exists
- Try explicit: "Load projects from Notion tracker: [URL]"

### Multiple questions blocking progress
**Symptom**: ChittyXL keeps asking questions instead of proceeding

**Issue**: This is expected per protocol (1 blocking question per message)

**Workaround**:
- Answer blocking question
- Add: "For future decisions, use reasonable defaults (Stripe, PostgreSQL, REST, etc.)"
- Or: "Proceed with standard best practices, ask only if critical"

---

## Customization

**Want to modify ChittyXL behavior?** Edit the `<chittyxl_protocol>` block in custom instructions.

### Common Customizations

**Change checkpoint thresholds**:
```xml
• Trigger checkpoints at: 30k, 60k, 90k, 120k, 150k tokens (custom intervals)
```

**Adjust chat word limit**:
```xml
• Checkpoint summaries: ≤100 words, bullet format (stricter)
```

**Different Notion tracker**:
```xml
• Tracker: https://www.notion.so/<your-tracker-id>
• Projects: collection://<your-projects-db-id>
• Actions: collection://<your-actions-db-id>
```

**Add custom commands**:
```xml
### User Commands
• `checkpoint` → immediate compact + sync
• `continue` → load last session from Notion
• `status` → show token usage + next checkpoint
• `export` → generate CSV export without Notion sync (NEW)
• `summary` → generate project summary report (NEW)
```

**Modify anti-patterns**:
```xml
❌ No emojis in responses (add custom rule)
❌ No apologizing ever (add custom rule)
```

---

## Performance Tips

**Maximize session longevity**:
- Use artifacts aggressively (keeps chat concise)
- Request checkpoints manually before long coding sessions
- Use `fork` command before hitting 90% (171k tokens)

**Optimize Notion sync speed**:
- Keep project/action counts manageable (<20 active projects)
- Archive completed projects regularly
- Use descriptive names (easier to match/deduplicate)

**Improve continuation quality**:
- Add rich context to Context Notes (entities, schemas, decisions)
- Use Decision Log field (timestamped decisions)
- Fill Blockers field (helps resume context)
- Link actions to parent projects (maintains relationships)

---

## Uninstalling

**To remove ChittyXL from a project**:

1. Project Settings → Custom Instructions
2. Locate `<chittyxl_protocol>` block
3. Delete entire block (from `<!-- ChittyXL...` to `</chittyxl_protocol>`)
4. Save settings
5. Start new chat (ChittyXL no longer active)

**To preserve Notion data**:
- Leave Projects/Actions in Notion tracker (history preserved)
- Archive ChittyXL-related projects (Status → Archived)
- Keep tracker for reference/future reactivation

**To completely remove**:
- Delete ChittyXL projects from Notion tracker
- Delete Actions linked to ChittyXL projects
- Remove tracker page (if dedicated to ChittyXL)

---

## FAQ

**Q: Can I use ChittyXL in multiple projects simultaneously?**
A: Yes. Each project can have ChittyXL enabled. All sync to same Notion tracker (filter by project name).

**Q: Does ChittyXL work in non-Pro Claude.ai accounts?**
A: Yes, but token limits are lower (~100k vs 190k). Adjust checkpoint thresholds accordingly.

**Q: Can I use a different storage backend (not Notion)?**
A: Protocol supports it, but requires custom implementation. Modify Notion Integration section to use Airtable, Google Sheets, etc.

**Q: Does ChittyXL slow down Claude responses?**
A: Checkpoints take <20s, occur only 5 times per session. Normal conversation unaffected.

**Q: Can I disable ChittyXL temporarily?**
A: Yes. In chat, type: "Disable ChittyXL for this session" (lasts until new chat). Or comment out custom instructions block.

**Q: How do I check which version I'm running?**
A: Type `status` → version number shown in response.

**Q: Can ChittyXL work without Notion integration?**
A: Partially. Artifact-first enforcement works without Notion. Session continuation requires external storage (Notion or alternative).

---

## Example Session Flow

**Day 1 - Session Start**:
```
User: [Opens project, starts new chat]
User: Build a payment service with multi-provider support

Claude: [ChittyXL auto-loads silently]
Claude: [Generates PaymentAdapter interface in artifact]
Claude: Interface in artifact ↑ | Methods: process, refund, validate

[Conversation continues, implementing payment service]

[At 38k tokens]
Claude: Checkpoint #1 complete | 38k/190k (20%) | Next: 76k
Claude: Synced: 1 project, 3 actions → Notion

[Conversation continues]

[At 76k tokens]
Claude: Checkpoint #2 complete | 76k/190k (40%) | Next: 114k
Claude: Synced: 1 project, 7 actions → Notion

[User closes browser]
```

**Day 2 - Session Resume**:
```
User: [Opens project, starts new chat]
User: continue

Claude: Session restored from Notion ✓

Claude: Active: 1 project, 7 pending actions
Claude: Last checkpoint: 16h ago (Checkpoint #2 at 76k tokens)

Claude: Priority: Complete Stripe adapter implementation
Claude: Blocked: None

Claude: Ready to continue - see briefing artifact ↑

[Artifact shows:
- Payment Service v2 (Active)
  - 7 pending actions (Implement StripeAdapter, Add tests, etc.)
  - Last decisions (adapter pattern, PostgreSQL logging)
  - Technical context (PaymentAdapter interface, transaction_log schema)
]

User: Let's finish the Stripe adapter

Claude: [Continues exactly where left off]
Claude: [Generates StripeAdapter implementation in artifact]
```

---

## Support & Resources

**Documentation**: See README.md, SKILL.md, SESSION_PROTOCOL.md in repo
**Notion Tracker**: https://www.notion.so/83e8d8f77e5a45bb96f7188c6fe092d3
**Issues**: Report bugs/feature requests on GitHub
**Updates**: Check repo for new versions

**Version**: 2.0.0
**Released**: 2025-01-16
**Status**: Production Ready ✓

---

## Next Steps

After successful deployment:

1. ✓ Verified ChittyXL active (`status` command works)
2. ✓ Tested artifact generation (technical content in artifacts)
3. ✓ Confirmed Notion sync (test project created)
4. → **Use normally** - ChittyXL auto-manages session persistence
5. → **Test continuation** - Close session, reopen, type `continue`
6. → **Monitor performance** - Check checkpoint latency, Notion sync success
7. → **Deploy to other projects** (if needed) - Copy custom instructions block

**Enjoy extended, persistent Claude sessions with zero context loss!** 🚀
