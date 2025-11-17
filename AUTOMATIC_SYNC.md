# Automatic Synchronization System

## Overview

This document describes the automatic synchronization system that ensures the Go service (cargoex-service.exe) always embeds the latest signed version of the C# application (LabelDesigner.exe).

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. PRINT-DRIVER-CSHARP                                          │
│    Developer pushes to main                                     │
│    ↓                                                             │
│    Builds LabelDesigner.exe (unsigned)                          │
│    ↓                                                             │
│    Uploads artifact & triggers APP_SIGNER                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. APP_SIGNER                                                   │
│    Downloads artifact from PRINT-DRIVER-CSHARP                  │
│    ↓                                                             │
│    Signs with Certum certificate (self-hosted runner)           │
│    ↓                                                             │
│    Pushes to PRINT-DIST/emb/LabelDesigner.exe                   │
│    Creates PRINT-DIST/emb/built-info.json                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. PRINT-DIST (AUTOMATIC TRIGGER) ⭐ NEW                        │
│    Detects changes in emb/ directory                            │
│    ↓                                                             │
│    Verifies this is NOT a cargoex-service update                │
│    ↓                                                             │
│    Reads emb/built-info.json for version info                   │
│    ↓                                                             │
│    Triggers PRINT-SERVICE-GO rebuild via repository_dispatch    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. PRINT-SERVICE-GO                                             │
│    Receives repository_dispatch event                           │
│    ↓                                                             │
│    fetch_signed_label.ps1 downloads latest from PRINT-DIST/emb/ │
│    ↓                                                             │
│    Embeds updated LabelDesigner.exe                             │
│    ↓                                                             │
│    Builds cargoex-service.exe                                   │
│    ↓                                                             │
│    Uploads artifact & triggers APP_SIGNER                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. APP_SIGNER (Second Time)                                     │
│    Downloads artifact from PRINT-SERVICE-GO                     │
│    ↓                                                             │
│    Signs cargoex-service.exe                                    │
│    ↓                                                             │
│    Pushes to PRINT-DIST/dist/cargoex-service.exe                │
│    Creates PRINT-DIST/dist/built-info.json                      │
└─────────────────────────────────────────────────────────────────┘
                         │
                         ▼
                    ✅ COMPLETE
    Both executables are signed and synchronized
```

## Key Components

### 1. TRIGGER_GO_REBUILD.yml (PRINT-DIST)

**Location:** `.github/workflows/TRIGGER_GO_REBUILD.yml`

**Purpose:** Automatically triggers Go service rebuild when LabelDesigner.exe updates

**Trigger Conditions:**
- Push to `main` branch
- Changes in:
  - `emb/LabelDesigner.exe`
  - `emb/built-info.json`
  - `emb/LabelDesigner.exe.config`

**Safety Mechanisms:**

1. **Path Filter:** Only monitors `emb/` directory, NOT `dist/`
   - Prevents infinite loops when cargoex-service.exe updates

2. **Commit Message Check:** Verifies the commit doesn't mention "cargoex-service.exe"
   - Double safety to prevent accidental triggers

3. **Version Extraction:** Reads metadata from `emb/built-info.json`
   - Passes version info to PRINT-SERVICE-GO

**Jobs:**

#### a. check-trigger-source
- Verifies this is a LabelDesigner update
- Extracts version information from built-info.json
- Outputs: `should_trigger`, `labeldesigner_version`, `labeldesigner_commit`

#### b. trigger-go-rebuild
- Runs only if `should_trigger == true`
- Sends `repository_dispatch` to PRINT-SERVICE-GO
- Payload includes:
  - `labeldesigner_version`
  - `labeldesigner_commit`
  - `trigger_source`
  - `triggered_at`
  - `triggered_by`

### 2. BUILD_AND_SIGN.yml (PRINT-SERVICE-GO)

**Location:** `.github/workflows/BUILD_AND_SIGN.yml`

**Changes Made:**

#### a. Additional Triggers
```yaml
on:
  push:
    branches:
      - main
  repository_dispatch:
    types:
      - labeldesigner-updated  # NEW: Auto-trigger from PRINT-DIST
  workflow_dispatch:  # NEW: Manual triggering
```

#### b. Trigger Logging
New step added after checkout to log when triggered automatically:
- Shows LabelDesigner version being embedded
- Displays trigger metadata
- Helps with debugging and auditing

## Workflow Execution Flow

### Scenario: C# Developer Updates LabelDesigner

1. **Developer Action:**
   ```bash
   cd PRINT-DRIVER-CSHARP
   # Make changes to C# code
   git commit -m "Update LabelDesigner: Add new feature"
   git push origin main
   ```

2. **PRINT-DRIVER-CSHARP Pipeline:**
   - Builds `LabelDesigner.exe`
   - Uploads as artifact
   - Triggers APP_SIGNER

3. **APP_SIGNER (First Time):**
   - Downloads artifact
   - Signs executable
   - Commits to PRINT-DIST:
     ```
     PRINT-DIST/
       emb/
         LabelDesigner.exe        ← Updated & signed
         built-info.json          ← Version metadata
         LabelDesigner.exe.config
     ```

4. **PRINT-DIST Auto-Trigger:**
   - Detects change in `emb/LabelDesigner.exe`
   - Reads version: `1.0.8`
   - Sends dispatch to PRINT-SERVICE-GO

5. **PRINT-SERVICE-GO Pipeline:**
   - Receives dispatch event
   - Logs: "Building with LabelDesigner v1.0.8"
   - Downloads signed LabelDesigner.exe from PRINT-DIST
   - Embeds in Go binary
   - Builds `cargoex-service.exe`
   - Triggers APP_SIGNER

6. **APP_SIGNER (Second Time):**
   - Signs `cargoex-service.exe`
   - Commits to PRINT-DIST:
     ```
     PRINT-DIST/
       dist/
         cargoex-service.exe  ← Updated with new embedded version
         built-info.json      ← Go service metadata
     ```

7. **Result:**
   - ✅ LabelDesigner.exe v1.0.8 signed in `emb/`
   - ✅ cargoex-service.exe with embedded v1.0.8 signed in `dist/`
   - ✅ Both synchronized automatically

### Timeline Example

| Time  | Event |
|-------|-------|
| 10:00 | Developer pushes C# changes |
| 10:02 | C# build completes |
| 10:03 | APP_SIGNER signs LabelDesigner.exe |
| 10:04 | PRINT-DIST receives signed file → Auto-trigger |
| 10:05 | Go build starts (triggered automatically) |
| 10:07 | Go build completes |
| 10:08 | APP_SIGNER signs cargoex-service.exe |
| 10:09 | ✅ Both files synchronized |

**Total time:** ~9 minutes (fully automatic)

## Loop Prevention

### Problem Avoided

Without path filtering, this would create an infinite loop:

```
❌ WRONG (would loop):
PRINT-DIST detects ANY change → Trigger Go
  ↓
Go builds → APP_SIGNER → Push to PRINT-DIST/dist/
  ↓
PRINT-DIST detects change → Trigger Go again
  ↓
🔁 INFINITE LOOP
```

### Solutions Implemented

1. **Path-Based Filtering:**
   ```yaml
   paths:
     - 'emb/LabelDesigner.exe'  # ✅ Only emb/, NOT dist/
   ```

2. **Commit Message Check:**
   ```bash
   if echo "$COMMIT_MSG" | grep -q "cargoex-service.exe"; then
     skip_trigger  # Don't trigger on Go updates
   ```

3. **Separate Directories:**
   - C# artifacts → `emb/`
   - Go artifacts → `dist/`
   - Trigger only watches `emb/`

## Manual Triggering

### Trigger Go Rebuild Manually

If you need to manually trigger a Go rebuild:

```bash
# Using GitHub CLI
gh workflow run BUILD_AND_SIGN.yml -R MatCargoEx/PRINT-SERIVCE-GO

# Or via GitHub UI
# 1. Go to: https://github.com/MatCargoEx/PRINT-SERIVCE-GO/actions
# 2. Select "Build and Sign CargoEx Service"
# 3. Click "Run workflow" → "Run workflow"
```

## Monitoring

### Check Workflow Status

**PRINT-DIST Trigger:**
```bash
# View recent triggers
gh run list -R MatCargoEx/PRINT-DIST -w TRIGGER_GO_REBUILD.yml --limit 5
```

**PRINT-SERVICE-GO Builds:**
```bash
# View recent builds
gh run list -R MatCargoEx/PRINT-SERIVCE-GO -w BUILD_AND_SIGN.yml --limit 5

# Check specific run
gh run view <run-id> -R MatCargoEx/PRINT-SERIVCE-GO
```

### Verify Synchronization

Check that both executables have matching LabelDesigner versions:

```bash
# Check C# version
curl -s https://raw.githubusercontent.com/MatCargoEx/PRINT-DIST/main/emb/built-info.json | jq -r '.version'

# Check what Go embedded (from Go's build-info.json)
# This would show the metadata of the Go build, not the embedded version
# To truly verify, download and inspect the binary
```

## Troubleshooting

### Trigger Didn't Fire

**Check 1: Verify paths changed**
```bash
git log -1 --stat PRINT-DIST
# Should show changes in emb/
```

**Check 2: Review workflow run**
```bash
gh run list -R MatCargoEx/PRINT-DIST -w TRIGGER_GO_REBUILD.yml --limit 1
```

**Check 3: Verify PAT_TOKEN**
- Ensure `PAT_TOKEN` secret is set in PRINT-DIST
- Token needs `repo` and `workflow` scopes

### Go Build Didn't Start

**Check 1: Verify dispatch was sent**
Look at PRINT-DIST workflow logs for:
```
✅ Go service rebuild triggered successfully!
```

**Check 2: Check PRINT-SERVICE-GO received it**
```bash
gh run list -R MatCargoEx/PRINT-SERIVCE-GO --event repository_dispatch
```

**Check 3: Verify event type matches**
In TRIGGER_GO_REBUILD.yml:
```yaml
"event_type": "labeldesigner-updated"  # Must match
```

In BUILD_AND_SIGN.yml:
```yaml
repository_dispatch:
  types:
    - labeldesigner-updated  # Must match
```

### Loop Detected

If you see Go rebuilding repeatedly:

**Check 1: Verify path filter**
```yaml
# Should NOT include dist/
paths:
  - 'emb/LabelDesigner.exe'
```

**Check 2: Check commit messages**
Ensure APP_SIGNER uses different messages for C# vs Go:
- C# commit: "Add signed LabelDesigner.exe..."
- Go commit: "Add signed cargoex-service.exe..."

## Configuration

### Required Secrets

**PRINT-DIST:**
- `PAT_TOKEN` - GitHub Personal Access Token with:
  - `repo` scope
  - `workflow` scope

**PRINT-SERVICE-GO:**
- `PAT_TOKEN` - Same token (for fetching from PRINT-DIST)

**APP_SIGNER:**
- `PAT_TOKEN` - Same token
- `EV` - Certum signing credentials
- `EV_ID` - Certum signing ID

### Repository Settings

**PRINT-DIST:**
- Actions: Enabled
- Workflow permissions: Read and write

**PRINT-SERVICE-GO:**
- Actions: Enabled
- Allow `repository_dispatch` events

## Benefits

1. ✅ **Fully Automatic:** No manual intervention needed
2. ✅ **Always Synchronized:** Go always embeds latest C# version
3. ✅ **Loop-Free:** Multiple safety mechanisms prevent infinite loops
4. ✅ **Auditable:** Full logs of what triggered what and when
5. ✅ **Fast:** ~9 minutes from C# push to synchronized binaries
6. ✅ **Safe:** No changes to APP_SIGNER logic
7. ✅ **Backwards Compatible:** Manual pushes still work

## Maintenance

### Updating the Sync Logic

**To modify trigger conditions:**
Edit `PRINT-DIST/.github/workflows/TRIGGER_GO_REBUILD.yml`

**To change event name:**
1. Update `event_type` in TRIGGER_GO_REBUILD.yml
2. Update `types` in PRINT-SERVICE-GO BUILD_AND_SIGN.yml
3. Ensure both match exactly

**To disable auto-sync temporarily:**
```yaml
# In TRIGGER_GO_REBUILD.yml
jobs:
  trigger-go-rebuild:
    if: false  # Disable temporarily
```

## References

- PRINT-DRIVER-CSHARP: `.github/workflows/DEPLOY.yml`
- PRINT-SERVICE-GO: `.github/workflows/BUILD_AND_SIGN.yml`
- APP_SIGNER: `.github/workflows/SIGN.yml`
- PRINT-DIST: `.github/workflows/TRIGGER_GO_REBUILD.yml` (NEW)

## Support

For issues or questions:
1. Check workflow logs in GitHub Actions
2. Verify secrets are properly configured
3. Ensure all repositories have Actions enabled
4. Review this documentation for troubleshooting steps
