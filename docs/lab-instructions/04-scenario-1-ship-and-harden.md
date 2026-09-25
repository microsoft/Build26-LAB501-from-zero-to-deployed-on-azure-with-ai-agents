# Scenario 1 — Ship It & Harden It (~35 min)

AI can scaffold your Azure deployment in minutes. But would you push AI-generated Bicep to production without reviewing it?

## Part A — Ship It (~25 min)

> 💡 **Make sure your app is NOT currently running.** If you started `python app.py` during the checkpoint, stop it with **Ctrl+C** before proceeding.

If you're not already in the **lego-set-browser** directory, cd into it, then use the prompt below to start a Copilot session in **yolo mode**.

```
copilot --yolo
```
The `--yolo` flag auto-approves commands and skips confirmation prompts — safe here because the lab runs in a sandboxed environment, and it can save you several minutes over the course of the lab. 

To ensure all participants of the lab to have a consistent lab experience, disable **azure-app-onboard-prereq** and **azure-app-onboard** skills. Say to Copilot:

``` 
/skills
```
1. Use the arrow keys to navigate to **azure-app-onboard-prereq** and **azure-app-onboard**.

2. Press **Enter** on each skill and select **Disable**.

3. Verify that a **gray dot** appears next to each skill, indicating it is disabled.

4. Press **Esc** to exit the Skills Manager.

Then say to Copilot:

```
  Create and deploy 2 Azure services 
   	
   **Environment:**
   - Subscription: Current subscription
   - Create a new resource group: rg-lego-set-browser-dev
   - Region: West US 3
   
   **Existing Cosmos DB (do NOT create a new one):**
   - Look for the existing cosmos DB in the current subscription
   - Database: LegoDatabase / Container: legoSets
   
   **1. Python Azure Function App** — HTTP POST trigger on Flex Consumption (FC1):
   - Accepts JSON array of LEGO sets; batch-upserts to Cosmos DB above
   - Fields: set_number (→ id), name, theme_name, year_released, number_of_parts, type, image_url
   - User-assigned managed identity for Cosmos DB (Built-in Data Contributor)
   
   **2. Flask app in this folder → Azure Container Apps:**
   - Name: ca-web-lego-<XXXX>
   - Already uses DefaultAzureCredential + env vars COSMOS_ENDPOINT, COSMOS_DATABASE, COSMOS_CONTAINER
   - System-assigned managed identity for Cosmos DB (Built-in Data Reader)
```

This single prompt triggers a **three-skill chain** — watch Copilot invoke each one:

### 1️⃣ `azure-prepare` activates first

Watch how it handles **two very different starting points** in a single pass — this is the key insight of this step:

- **Flask app → Container Apps (starting point: existing source code in your workspace):**
  - Scans your workspace — finds `requirements.txt` and `app.py`, classifies it as a Python Flask web app
  - Chooses Container Apps as the hosting target (do you agree with that choice over App Service or Functions?)
  - Reviews the existing `Dockerfile` (the app already has one — watch whether the skill uses it as-is or regenerates it)
  - Generates infrastructure *around* code that already exists
- **Python Function App → Azure Functions (starting point: just your prompt — no source code yet):**
  - There is no function code in your workspace. The skill reads your prompt (HTTP POST trigger, JSON array of LEGO sets, batch-upsert to Cosmos DB) and **fetches a suitable Azure Functions Python template**
  - Modifies that template to fit your scenario: rewrites the handler to accept the LEGO JSON shape, maps `set_number → id`, wires it to the existing Cosmos DB database/container, and adds the user-assigned managed identity binding
  - Picks **Flex Consumption (FC1)** as the hosting plan and adds a Storage account for the deployment package
  - Generates infrastructure *and* the source code, from a template
- **Shared across both services:**
  - Produces a single `azure.yaml` that declares both services and Bicep templates in `infra/` (typically `main.bicep` plus per-service modules)
  - Creates an AZD environment and sets your subscription + region

> 💡 **Skill spotlight:** `azure-prepare` doesn't just generate files — it reads skill references for your language runtime, Bicep patterns, AZD conventions, and the Azure Functions Flex Consumption template. Open the generated files: the Bicep in `infra/` and the new `function_app.py` came from skill reference templates, not generic boilerplate.

### 2️⃣ `azure-validate` activates next

It runs pre-flight checks across both services:
- Compiles Bicep (`az bicep build`) for the Container App and the Function App modules — catches syntax errors before deployment
- Verifies Docker is running — the Container App image build will fail without it
- Verifies the Python runtime version and that Flex Consumption (FC1) is available in your selected region — Functions deployment fails fast otherwise
- Confirms your subscription access and that the resource group name isn't taken

### 3️⃣ `azure-deploy` activates last

It runs `azd up --no-prompt`, which provisions and deploys both services in one orchestrated run:
- **Container App side:** Provisions ACR, Container Apps Environment, Log Analytics, and the Container App; builds your Docker image and pushes it to ACR
- **Function App side:** Provisions the Storage account, Flex Consumption (FC1) plan, user-assigned managed identity, Application Insights, and the Function App; packages and deploys the Python function code
- Wires Cosmos DB environment variables into both services and returns live HTTPS endpoints (a Container Apps URL for the Flask UI, and a Functions URL for the ingest endpoint)

### Verify it works

```bash
curl <your-endpoint-url>
```

> 💡 **Finding your endpoint URL:** If the URL scrolled off screen, run `azd env get-values` or ask Copilot: "What's the URL for my Container App?" The URL looks like `https://<app-name>.<region>.azurecontainerapps.io`.

> 💡 **First request may be slow:** The first request after deploy can take 10-15 seconds while the new revision activates. This is normal — retry after a moment.

> ⚠️ **Cosmos DB access from Container App:** After deployment, the Container App needs permission to access Cosmos DB. If you see errors in the app related to database access, this is expected — you'll address this in Part B (Harden It) by configuring managed identity with the appropriate Cosmos DB RBAC role.

**End state:** A live HTTPS endpoint serving the LEGO set browser. Three skills, one prompt. But it's deployed, not production-ready.

![LEGO Vault app running in the browser](images/workingApp.png)

> 📁 **What files were created?** After Part A, your `lego-app` directory should now contain new files generated by `azure-prepare`: typically `azure.yaml` and an `infra/` folder with Bicep templates (e.g., `main.bicep`, `main.parameters.json`, and supporting modules). The existing `Dockerfile` may have been kept as-is or updated. The exact file structure may vary slightly depending on how the AI scaffolded your project.

---

## Part B — Harden It (~10 min)

> ⚠️ **Permissions note:** Some hardening operations (e.g., creating role assignments for managed identity) may require **Owner** or **User Access Administrator** role on the subscription. If you encounter permission errors during this step, that's expected in some lab environments — focus on understanding the pattern and reviewing the AI's recommendations rather than executing every command.

Review the generated Bicep files in your `infra/` directory. Depending on how `azure-prepare` ran, it may have already applied some security hardening during generation. Your job is to **audit what the AI did and didn't do**.

**Say to Copilot:**

```
Review my deployed Container App infrastructure for production readiness gaps. Check for managed identity, Cosmos DB RBAC access (instead of keys), VNet integration, diagnostic settings, and health probes.
```

### What to look for

| Gap | Why It Matters | Severity |
|---|---|---|
| **No managed identity for ACR pull** | Container App uses admin credentials to pull images. Security finding. | High |
| **No managed identity for Cosmos DB** | App uses connection keys instead of RBAC. Keys can leak. | High |
| **No VNet integration** | Container Apps Environment is on a public network. No isolation. | Medium |
| **No diagnostic settings** | Platform metrics aren't forwarded. You'd miss CPU/memory alerts. | Medium |
| **No health probe configured** | Defaults to TCP probes. Your app has a home route — it should use it. | Low |

> 💡 **The AI may have already hardened some of these.** The `azure-prepare` skill includes a security hardening phase that can set up managed identity and RBAC during initial generation. If you find managed identity and AcrPull are already configured — great, that's the skill working as designed. Focus your review on what's still missing, especially Cosmos DB access via managed identity.

✅ **Checkpoint:** You've audited the generated infrastructure and either confirmed the AI hardened it or found the next steps. The Container App should now have managed identity configured for both ACR image pulls and Cosmos DB data access.

**Takeaway:** The AI may build a secure deployment out of the box — or it may not. The skill's security hardening phase is non-deterministic, which is exactly why human review matters. Whether the AI did the hardening or you prompted it, the critical skill is knowing what "production-ready" looks like and verifying it.

---

**Next:** [Scenario 2 — See It & Evaluate It →](05-scenario-2-see-and-evaluate.md)
