# Project Templates

Tina uses these templates during preflight to create missing `/docs/` files. Create the relevant file under `/docs/`, then ask the user to fill in the blanks.

---

## /docs/OSS.md

For projects that ship to end customers.

```markdown
# Open Source Policy

## Choosing Dependencies

- Prefer well-maintained libraries with active communities
- Fewer dependencies is better — don't add a library for something trivial
- Check the library's license BEFORE adding it

## License Rules

**ALLOWED:** MIT, BSD (2-clause, 3-clause), Apache 2.0, ISC, MPL-2.0
**BLOCKED:** GPL, LGPL, AGPL, SSPL, or any copyleft license
**FLAG:** No license found, unknown license, dual-licensed with copyleft option
```

---

## /docs/DEVOPS.md

Pick the template that matches the project type.

### IoT / Embedded Device

```markdown
### Target: Embedded device via SSH

### Connection
- **Host**: <device IP>
- **User**: <SSH user>
- **Auth**: <key path>

### Build
- **Command**: <cross-compile command>
- **Target arch**: <e.g. linux/arm64>
- **Output**: <path to binary>

### Deploy
- **Stop**: <stop command>
- **Copy**: <scp/rsync command>
- **Start**: <start command>
- **Known-good backup**: <path to rollback binary>

### Verify
- **Health check**: <command to verify running>
- **Success criteria**: <what indicates healthy>
- **Wait time**: <seconds before checking>

### Rollback
- **Procedure**: Stop → restore known-good → start → verify
- **Max downtime**: <acceptable seconds>

### Critical Notes
<Project-specific warnings, e.g. "Heartbeat must not stop">
```

### API Server (VPS / Cloud)

```markdown
### Target: Staging VPS

### Connection
- **Host**: <server IP or hostname>
- **User**: <SSH user>
- **Auth**: <key path>

### Build
- **Command**: <build command>
- **Output**: <path to binary or container image>

### Deploy
- **Stop**: <e.g. `systemctl stop api` or `docker stop api`>
- **Deploy**: <e.g. `scp`, `docker pull && docker run`, or `kubectl apply`>
- **Start**: <e.g. `systemctl start api`>
- **Migrations**: <e.g. `./migrate up` — run BEFORE starting new version>

### Verify
- **Health endpoint**: <e.g. `curl https://staging.example.com/health`>
- **Smoke tests**: <e.g. `curl -X POST .../api/test-endpoint`>
- **Success criteria**: <e.g. "HTTP 200 on /health, all smoke tests pass">
- **Wait time**: <seconds before checking>

### Rollback
- **Procedure**: <e.g. deploy previous Docker image, revert migration>
- **Previous version**: <how to identify, e.g. Docker tag, git tag>
```

### Web UI (Browser Testing)

```markdown
### Target: Browser-based verification

### Build
- **Command**: <e.g. `npm run build`>
- **Output**: <e.g. `dist/`>

### Deploy
- **Dev server**: <e.g. `npm run preview` or `npx serve dist/`>
- **URL**: <e.g. `http://localhost:4173`>

### Verify
- **Tool**: <e.g. Playwright, Puppeteer, Cypress>
- **Test command**: <e.g. `npx playwright test`>
- **Success criteria**: All browser tests pass

### Rollback
- **Procedure**: Kill dev server, no persistent state to revert
```

### Mobile App

```markdown
### Target: Test device / emulator

### Build
- **Command**: <e.g. `flutter build apk --debug` or `npx expo build`>
- **Output**: <path to APK/IPA>

### Deploy
- **Install**: <e.g. `adb install -r app.apk`>
- **Launch**: <e.g. `adb shell am start -n com.app/.MainActivity`>

### Verify
- **Tool**: <e.g. Appium, Detox, XCTest>
- **Test command**: <e.g. `npx detox test`>
- **Success criteria**: App launches, critical flows pass

### Rollback
- **Procedure**: Install previous APK/IPA
```

### Release / Production (CI/CD-triggered)

Every `/docs/DEVOPS.md` needs this section too: the test-target blocks above cover
daisy's pre-merge deploy check, this one covers the release at STEP 12.

```markdown
### Target: Production, deployed by CI/CD

### Trigger
- **Release event**: <e.g. push to `main`, or push of tag `vX.Y.Z`>
- **Tag format**: <e.g. `v1.2.3`, or "none — main pushes deploy">

### Pipeline
- **Watch command**: <e.g. `gh run watch --exit-status`, `gh run list --branch main`>
- **Expected duration**: <minutes — how long before a stall is suspicious>
- **Logs**: <where to read failures>

### Verify
- **Health endpoint**: <e.g. `curl https://prod.example.com/health`>
- **Smoke tests**: <commands proving the deployed version actually serves>
- **Version check**: <how to confirm the deployed version is the pushed commit/tag>
- **Wait time**: <seconds after the pipeline reports success>

### Rollback
- **Preferred**: <redeploy previous known-good release, e.g. previous tag or image>
- **Git fallback**: `git revert <sha> && git push` — never force-push published history
- **Migrations**: <how to reverse, or "forward-only — do not roll back the DB">
- **Who to tell**: <channel or person, if a human must know>
```
