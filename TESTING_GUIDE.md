# Testing Guide: Automatic Synchronization System

## Overview

This guide provides step-by-step instructions for testing the automatic synchronization system between PRINT-DRIVER-CSHARP and PRINT-SERVICE-GO.

## Prerequisites

Before testing, ensure:

1. ✅ All repositories have the latest code:
   - PRINT-DIST has `TRIGGER_GO_REBUILD.yml`
   - PRINT-SERVICE-GO has updated `BUILD_AND_SIGN.yml` with `repository_dispatch`

2. ✅ Secrets are configured:
   - `PAT_TOKEN` in PRINT-DIST
   - `PAT_TOKEN` in PRINT-SERVICE-GO
   - `PAT_TOKEN`, `EV`, `EV_ID` in APP_SIGNER

3. ✅ GitHub Actions are enabled in all repositories

4. ✅ You have push access to PRINT-DRIVER-CSHARP

## Test Scenarios

### Test 1: End-to-End Automatic Sync (Recommended First Test)

**Purpose:** Verify the entire automatic synchronization flow works

**Steps:**

1. **Make a small change to C# code:**
   ```bash
   cd C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-DRIVER-CSHARP

   # Add a comment to trigger build
   # Edit any .cs file and add a comment
   echo "// Test sync $(date)" >> LabelDesigner/Program.cs

   git add .
   git commit -m "Test: Trigger automatic sync"
   git push origin main
   ```

2. **Monitor PRINT-DRIVER-CSHARP build:**
   ```bash
   # Open in browser
   start https://github.com/MatCargoEx/PRINT-DRIVER-CSHARP/actions

   # Or use CLI
   gh run watch -R MatCargoEx/PRINT-DRIVER-CSHARP
   ```

   Expected: Build completes successfully and triggers APP_SIGNER

3. **Monitor APP_SIGNER (First Time - Signs C#):**
   ```bash
   gh run watch -R MatCargoEx/APP_SIGNER
   ```

   Expected:
   - Downloads LabelDesigner artifact
   - Signs executable
   - Pushes to PRINT-DIST/emb/

4. **Monitor PRINT-DIST auto-trigger:**
   ```bash
   gh run watch -R MatCargoEx/PRINT-DIST
   ```

   Expected output in logs:
   ```
   ✅ Proceeding: This is a LabelDesigner update
   LabelDesigner version: 1.0.X
   ✅ Go service rebuild triggered successfully!
   ```

5. **Monitor PRINT-SERVICE-GO build (Auto-triggered):**
   ```bash
   gh run watch -R MatCargoEx/PRINT-SERIVCE-GO
   ```

   Expected in logs:
   ```
   ========================================
   BUILD TRIGGERED BY LABELDESIGNER UPDATE
   ========================================
   LabelDesigner Version: 1.0.X
   ```

6. **Monitor APP_SIGNER (Second Time - Signs Go):**
   ```bash
   gh run watch -R MatCargoEx/APP_SIGNER
   ```

   Expected:
   - Downloads cargoex-service artifact
   - Signs executable
   - Pushes to PRINT-DIST/dist/

7. **Verify final state:**
   ```bash
   # Check PRINT-DIST has both updated files
   cd C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-DIST
   git pull

   # Check timestamps
   ls -lh emb/
   ls -lh dist/

   # Verify versions match
   cat emb/built-info.json | jq -r '.version'
   # Note the version

   # The Go service should have been rebuilt after C# update
   cat dist/built-info.json | jq -r '.built-at'
   # Should be AFTER the C# build time
   ```

**Success Criteria:**
- ✅ All workflows completed successfully
- ✅ PRINT-SERVICE-GO was triggered automatically (check "repository_dispatch" event)
- ✅ No manual intervention required
- ✅ Total time: ~9-12 minutes

---

### Test 2: Verify Loop Prevention

**Purpose:** Ensure Go updates don't trigger unnecessary rebuilds

**Steps:**

1. **Manually trigger Go build:**
   ```bash
   gh workflow run BUILD_AND_SIGN.yml -R MatCargoEx/PRINT-SERIVCE-GO
   ```

2. **Wait for completion:**
   ```bash
   gh run watch -R MatCargoEx/PRINT-SERIVCE-GO
   ```

3. **Check PRINT-DIST did NOT trigger:**
   ```bash
   gh run list -R MatCargoEx/PRINT-DIST -w TRIGGER_GO_REBUILD.yml --limit 3
   ```

   Expected:
   - No new run after the Go build completed
   - Last run should be from previous C# update

**Success Criteria:**
- ✅ Go build completes
- ✅ APP_SIGNER signs and pushes to dist/
- ✅ PRINT-DIST does NOT trigger a new Go build
- ✅ No infinite loop

---

### Test 3: Manual Dispatch Test

**Purpose:** Verify manual triggering works

**Steps:**

1. **Trigger manually via GitHub UI:**
   - Go to: https://github.com/MatCargoEx/PRINT-SERIVCE-GO/actions
   - Click "Build and Sign CargoEx Service"
   - Click "Run workflow" dropdown
   - Select "main" branch
   - Click "Run workflow" button

2. **Or via CLI:**
   ```bash
   gh workflow run BUILD_AND_SIGN.yml -R MatCargoEx/PRINT-SERIVCE-GO
   ```

3. **Verify it runs:**
   ```bash
   gh run list -R MatCargoEx/PRINT-SERIVCE-GO --limit 1
   ```

   Expected: Shows "workflow_dispatch" as trigger

**Success Criteria:**
- ✅ Workflow starts
- ✅ Shows as "Manual" or "workflow_dispatch" trigger
- ✅ Completes successfully

---

### Test 4: Verify Commit Message Detection

**Purpose:** Ensure commit message filtering works

**Steps:**

1. **Create a test commit in PRINT-DIST:**
   ```bash
   cd C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-DIST

   # Make a harmless change
   echo "test" > emb/test.txt
   git add emb/test.txt
   git commit -m "Add signed cargoex-service.exe - Version 1.0.0 (test)"
   git push origin main
   ```

2. **Check workflow skipped trigger:**
   ```bash
   gh run view -R MatCargoEx/PRINT-DIST
   ```

   Expected in logs:
   ```
   ❌ Skipping trigger: This is a Go service update
   ```

3. **Clean up:**
   ```bash
   git rm emb/test.txt
   git commit -m "Clean up test file"
   git push origin main
   ```

**Success Criteria:**
- ✅ Workflow runs
- ✅ Detects "cargoex-service.exe" in commit message
- ✅ Skips triggering Go rebuild
- ✅ Logs explain why it was skipped

---

### Test 5: Verify Metadata Propagation

**Purpose:** Ensure version information passes correctly

**Steps:**

1. **Trigger a C# build** (see Test 1)

2. **When Go build runs, check logs:**
   ```bash
   gh run view <run-id> -R MatCargoEx/PRINT-SERIVCE-GO
   ```

   Look for the "Log Trigger Information" step:
   ```
   LabelDesigner Version: 1.0.X
   LabelDesigner Commit: abc123
   Triggered from: PRINT-DIST
   Triggered by: <username>
   ```

3. **Verify all fields are populated:**
   - ✅ `labeldesigner_version` - not empty
   - ✅ `labeldesigner_commit` - valid commit hash
   - ✅ `trigger_source` - says "PRINT-DIST"
   - ✅ `triggered_by` - shows GitHub username

**Success Criteria:**
- ✅ All metadata fields populated
- ✅ Version matches what's in PRINT-DIST/emb/built-info.json
- ✅ Information helps with debugging/auditing

---

## Debugging Failed Tests

### Workflow Didn't Start

**Check 1: Verify workflow file exists**
```bash
# PRINT-DIST
ls -la C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-DIST\.github\workflows\TRIGGER_GO_REBUILD.yml

# PRINT-SERVICE-GO
ls -la C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-SERIVCE-GO\.github\workflows\BUILD_AND_SIGN.yml
```

**Check 2: Verify files are committed and pushed**
```bash
cd C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-DIST
git status
git pull

cd C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-SERIVCE-GO
git status
git pull
```

**Check 3: Check Actions are enabled**
```bash
gh api repos/MatCargoEx/PRINT-DIST/actions/permissions
gh api repos/MatCargoEx/PRINT-SERIVCE-GO/actions/permissions
```

### Trigger Sent but Go Didn't Build

**Check 1: View PRINT-DIST workflow logs**
```bash
gh run view --log -R MatCargoEx/PRINT-DIST
```

Look for:
```
✅ Go service rebuild triggered successfully!
Status: 204 (No Content - Success)
```

**Check 2: Verify event type matches**
```bash
# In PRINT-DIST workflow
grep "event_type" C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-DIST\.github\workflows\TRIGGER_GO_REBUILD.yml

# In PRINT-SERVICE-GO workflow
grep "repository_dispatch:" -A 3 C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-SERIVCE-GO\.github\workflows\BUILD_AND_SIGN.yml
```

Both should show `labeldesigner-updated`

**Check 3: Check PAT_TOKEN**
```bash
gh secret list -R MatCargoEx/PRINT-DIST
```

Expected: `PAT_TOKEN` should be listed

### Infinite Loop Detected

**Check 1: Verify path filter**
```bash
grep "paths:" -A 5 C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-DIST\.github\workflows\TRIGGER_GO_REBUILD.yml
```

Expected:
```yaml
paths:
  - 'emb/LabelDesigner.exe'
  - 'emb/built-info.json'
  - 'emb/LabelDesigner.exe.config'
```

Should NOT include `dist/`

**Check 2: View recent workflow runs**
```bash
gh run list -R MatCargoEx/PRINT-DIST --limit 10
```

If you see many runs in quick succession, there's a loop

**Fix:**
```bash
# Disable workflow temporarily
gh api -X PUT repos/MatCargoEx/PRINT-DIST/actions/workflows/TRIGGER_GO_REBUILD.yml/disable

# Fix the issue
# Then re-enable
gh api -X PUT repos/MatCargoEx/PRINT-DIST/actions/workflows/TRIGGER_GO_REBUILD.yml/enable
```

### Missing Metadata in Go Build

**Check 1: View PRINT-DIST workflow**
```bash
gh run view --log -R MatCargoEx/PRINT-DIST | grep "version"
```

Expected: Should extract version from built-info.json

**Check 2: Verify built-info.json format**
```bash
curl -s https://raw.githubusercontent.com/MatCargoEx/PRINT-DIST/main/emb/built-info.json | jq .
```

Expected fields:
```json
{
  "built-at": "2025-01-17T10:00:00Z",
  "commit": "abc123",
  "version": "1.0.5",
  "filename": "LabelDesigner.exe",
  "repository": "MatCargoEx/PRINT-DRIVER-CSHARP",
  "signed": true,
  "file-size-bytes": 12345678
}
```

## Test Checklist

Before considering testing complete, verify:

- [ ] Test 1 (End-to-End) passed
- [ ] Test 2 (Loop Prevention) passed
- [ ] Test 3 (Manual Dispatch) passed
- [ ] Test 4 (Commit Message) passed
- [ ] Test 5 (Metadata) passed
- [ ] No infinite loops observed
- [ ] All workflows complete in reasonable time (~9-12 min total)
- [ ] Logs are clear and helpful
- [ ] No errors in any workflow
- [ ] Both `emb/` and `dist/` updated correctly

## Rollback Plan

If testing reveals critical issues:

**Option 1: Disable auto-trigger temporarily**
```bash
# Rename workflow to disable it
cd C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-DIST
git mv .github/workflows/TRIGGER_GO_REBUILD.yml .github/workflows/TRIGGER_GO_REBUILD.yml.disabled
git commit -m "Temporarily disable auto-sync"
git push origin main
```

**Option 2: Remove repository_dispatch from Go**
```bash
cd C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-SERIVCE-GO

# Edit BUILD_AND_SIGN.yml
# Remove the repository_dispatch and workflow_dispatch sections
# Keep only push trigger

git commit -m "Rollback: Remove auto-trigger support"
git push origin main
```

**Option 3: Full rollback**
```bash
cd C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-DIST
git revert <commit-hash>
git push origin main

cd C:\Users\admin-murrutia\.dev\ETQ_SYS_REF\PRINT-SERIVCE-GO
git revert <commit-hash>
git push origin main
```

## Success Metrics

After all tests pass, you should observe:

1. **Automation:** C# pushes automatically trigger Go rebuilds
2. **Speed:** Complete sync in ~9-12 minutes
3. **Reliability:** No manual intervention needed
4. **Safety:** No infinite loops
5. **Transparency:** Clear logs explaining what happened
6. **Compatibility:** Manual builds still work

## Next Steps After Testing

Once all tests pass:

1. ✅ Document the system for team (use AUTOMATIC_SYNC.md)
2. ✅ Train developers on the new flow
3. ✅ Set up monitoring/alerts for failed workflows
4. ✅ Create runbook for common issues
5. ✅ Plan regular audits to ensure sync is working

## Support

If you encounter issues during testing:

1. Check workflow logs first
2. Verify secrets are set correctly
3. Review this testing guide
4. Check AUTOMATIC_SYNC.md for architecture details
5. Contact DevOps if problems persist
