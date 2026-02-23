# Lessons Learned (Global)

This file tracks patterns, corrections, and insights from development sessions across all projects to prevent repeating mistakes.

---

## 2026-02-23: Deployment & Caching Issues (weather-backend)

### Service Worker Caching Blocks Deployments
**Problem:** Added water station status feature to toolbar. Code was committed and deployed successfully, but feature didn't appear in browser.

**Root Cause:** Service Worker was aggressively caching the old HTML, preventing new version from loading even after hard refresh.

**Solution:**
```javascript
navigator.serviceWorker.getRegistrations().then(registrations => {
    registrations.forEach(reg => reg.unregister());
}).then(() => location.reload());
```

**Lesson:** When features don't appear after deployment:
1. Check browser DevTools console for Service Worker
2. Unregister Service Worker before debugging further
3. Consider versioning Service Worker cache keys to auto-invalidate

---

### Syntax Errors Silently Break Deployments
**Problem:** Water station feature deployed but API endpoint returned 404. Server appeared healthy in Azure logs.

**Root Cause:** Incomplete debug statement in server.js:
```javascript
if (device.id === 7) {
    // Missing closing brace
if (!reading) {
```
This caused "Unexpected token 'catch'" syntax error at line 3210.

**Solution:** Check deployment logs for startup errors:
```bash
gh run view <run-id> --log | grep -i error
```

**Lesson:**
- Syntax errors prevent server from starting entirely
- Azure may show "healthy" status even when app crashes on startup
- Always check deployment logs after pushing
- Remove or complete debug statements before committing

---

### Public vs Authenticated Endpoints
**Problem:** After security audit, added `authenticateToken` middleware to several endpoints including `/api/devices`. Frontend failed with 401 errors.

**Root Cause:** Over-secured public endpoints that need to be accessible without login.

**Solution:** Removed authentication from:
- `/api/devices` - Lists weather stations
- `/api/device/:id/data` - Gets device readings
- `/api/device/:id/data-with-forecast` - Combined endpoint

**Lesson:**
- Public dashboards need public read endpoints
- Authentication should protect: admin, user preferences, device management
- Mark endpoints clearly in comments: `// (public)` or `// (authenticated)`

---

### Expired Auth Tokens Spam Console
**Problem:** Browser console filled with 403 errors on `/api/user/preferences` even when not logged in.

**Root Cause:** Old auth tokens in localStorage from previous sessions. Frontend tried to load preferences with expired token.

**Solution:** Handle 401/403 gracefully in preference functions:
```javascript
if (response.status === 403 || response.status === 401) {
    localStorage.removeItem('authToken');
    localStorage.removeItem('userEmail');
    localStorage.removeItem('userRole');
    // Continue without auth, don't spam console
}
```

**Lesson:**
- Always clear invalid tokens silently
- Don't let failed preference loads break user experience
- Preferences are optional - fail gracefully

---

### CSS Layout: justify-content Confusion
**Problem:** Toolbar items weren't visible (water station status). Changed `justify-content: center` to `space-between` to fix, but then everything spread out too much.

**Solution:** Use centered layout with flex-wrap:
```css
.toolbar {
    display: flex;
    justify-content: center;  /* Center as a group */
    flex-wrap: wrap;          /* Allow wrapping */
    gap: 20px;                /* Space between items */
}
```

**Lesson:**
- `justify-content: space-between` pushes items to edges
- `justify-content: center` centers items together as a group
- Use `flex-wrap` and `gap` for responsive, centered layouts

---

## Patterns to Follow

### 1. Deployment Verification Checklist
After every deployment:
- [ ] Check GitHub Actions for green build
- [ ] Review deployment logs for errors: `gh run view --log`
- [ ] Test API endpoints directly: `curl https://domain.com/api/endpoint`
- [ ] Hard refresh browser (Ctrl+Shift+R)
- [ ] Unregister Service Worker if needed
- [ ] Check browser console for errors

### 2. Authentication Middleware Rules
```javascript
// Public endpoints - NO auth required
app.get('/api/devices')
app.get('/api/device/:id/data')
app.get('/api/public/*')

// Protected endpoints - authenticateToken required
app.get('/api/admin/*')
app.put('/api/user/preferences')
app.post('/api/devices')
app.put('/api/devices/:id')
app.delete('/api/devices/:id')
```

### 3. Error Handling for Optional Features
When implementing optional features (preferences, etc.):
- Check for auth token before calling
- Handle 401/403 by clearing invalid tokens
- Fail silently - don't break UX
- Log errors to console.error but don't spam

### 4. Debug Code Hygiene
Before committing:
- Remove or complete all debug `console.log()` statements
- Remove incomplete `if` blocks
- Search for `TODO`, `FIXME`, `DEBUG` comments
- Verify syntax: `node --check server.js`

### 5. Plan Mode Usage
From workflow orchestration guidelines:
- Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions)
- If something goes sideways, STOP and re-plan immediately
- Use plan mode for verification steps, not just building
- Write detailed specs upfront to reduce ambiguity

### 6. Subagent Strategy
- Use subagents liberally to keep main context window clean
- Offload research, exploration, and parallel analysis to subagents
- For complex problems, throw more compute at it via subagents
- One task per subagent for focused execution

### 7. Security Audit Cadence
**Every 10 commits:** Run OWASP Top 10 security checks and implement fixes

**Process:**
1. Check commit count: `git log --oneline | wc -l`
2. If divisible by 10, trigger security audit
3. Run comprehensive checks:
   - SQL Injection vulnerabilities
   - XSS (Cross-Site Scripting)
   - Authentication/Authorization flaws
   - Sensitive data exposure
   - Security misconfiguration
   - npm audit vulnerabilities
   - Rate limiting effectiveness
   - Input validation
4. Create issues/tasks for all findings
5. Fix Critical and High severity issues immediately
6. Schedule Medium/Low issues for next sprint

**Tools:**
```bash
npm audit
npm audit fix
# Manual code review for OWASP Top 10
# Check for hardcoded secrets/credentials
# Verify authentication middleware coverage
```

---

## Next Session Review
- Did any new patterns emerge?
- Were any lessons from this file violated?
- What mistakes were made that should be documented?
- Review this file at the start of each session for relevant context
