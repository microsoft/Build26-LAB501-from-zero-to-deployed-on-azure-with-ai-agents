# Scenario 3 — Break It & Triage It (~10 min)

It's 2 AM. Your app returns 503. You open a terminal. Pay attention to the AI's diagnostic reasoning chain, not just the answer.

## Introduce the Failure

Replace `<app>` and `<rg>` with your actual Container App name and resource group from Scenario 1A (run `azd env get-values` if you need to find them). Then, in a new powershell tab, run the following command:

```powershell
az containerapp ingress update --name <app> -g <rg> --target-port 9999
```

> ⏱️ **This command takes ~30 seconds to 2 minutes** while the new Container Apps revision activates. This is expected — don't Ctrl+C.

Hit the endpoint — you'll get `503 Service Unavailable`.

---

## Diagnose with AI

**Say to Copilot:**

```
My Container App is returning 503. What's wrong?
```

### 6️⃣ `azure-diagnostics` activates

Watch the triage chain:

1. **Hypothesis formation** — the skill considers multiple failure modes: app crash? ingress misconfiguration? bad image? unhealthy environment?
2. **Log retrieval** — pulls Container App system logs using `az containerapp logs show --type system`
3. **Log correlation** — finds `Reason: ProbeFailed` — *"Probe of StartUp failed with status code: 1"* (the startup probe fails because the container isn't listening on port 9999)
4. **Config verification** — cross-references ingress config (port 9999) against the container's listening port (8000, as set by gunicorn in the Dockerfile)
5. **Root cause + fix** — delivers the exact CLI command to restore the correct port

> 💡 **Skill spotlight:** `azure-diagnostics` doesn't just search logs for errors — it follows a diagnostic reasoning chain. It starts broad (what could cause 503?), narrows via evidence (system logs show ProbeFailed), and confirms with config data. This is the same triage pattern a senior SRE would follow.

---

## Apply the Fix

> Skip this step if Copilot already recovered the endpoint in the previous attempt to diagnose the issue.

Run the suggested fix command. It will be something like:

```powershell
az containerapp ingress update --name <app> -g <rg> --target-port 8000
```

Verify recovery → `200 OK`.

✅ **Checkpoint:** `curl <your-endpoint-url>` returns the LEGO set browser HTML again.

**Takeaway:** One natural language question → `azure-diagnostics` activated → root cause + fix in ~30 seconds. The skill did the log correlation you'd normally do manually in the portal.

---

**Next:** [Scenario 4 — Investigate & Operationalize →](07-scenario-4-investigate-and-operationalize.md)
