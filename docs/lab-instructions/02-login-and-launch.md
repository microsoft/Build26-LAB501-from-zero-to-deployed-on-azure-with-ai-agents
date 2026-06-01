# Before You Begin — Login & Launch

## 1. Log in to Azure

Open a terminal — use the shortcut (Ctrl + Shift + 4) to open PowerShell — and complete these steps.

```bash
az login
```

When the sign-in pop-up shows up, select **Work or school account** and select **Continue**. Input the username found in the **Resources** tab of your Skillable VM by clicking on the keyboard icon and select **Next**. Then, input the TAP found in the same tab by clicking on the keyboard icon to complete sign-in. On the **Sign in to all apps and websites on this device?** dialog, click **Yes**.

When the terminal prompts you for subscription selection, hit **Enter** for no changes.

> ⚠️ **Do NOT select "Microsoft account" (personal/consumer).** The login page may show multiple options — always select **Work or school account**. Selecting the wrong option will result in access-denied errors.

## 2. Log in to Azure Developer CLI

```bash
azd auth login
```

Select the Azure account from the previous step and complete authentication.

## 3. Log in to GitHub

Open this link in the browser: <https://github.com/enterprises/skillable-events/sso>. Select **Continue** when prompted to single sign-on to Skillable Events. Select the Azure account you just authenticated to. Follow the prompts to complete authentication.

## 4. Log in to GitHub Copilot CLI

For this lab exercise, we will be using a specific version of the Copilot CLI. 

```bash
winget uninstall github.copilot
```
```bash
winget install github.copilot --version 1.0.48
```

Once Copilot CLI is updated to the latest version, enter the following command:

```bash
copilot
```

This opens the interactive Copilot CLI session. All "Say to Copilot" prompts in this lab are typed here. **Keep this session open for the rest of the lab** — this is where you'll interact with AI skills.

> 💡 **Terminal vs. Copilot:** Throughout this lab, you'll run commands in two places. **Copilot CLI** is for AI-driven prompts (e.g., "Deploy my app to Azure"). **Terminal commands** (prefixed with `!` in Copilot) are for shell operations like `curl`, `az`, and `git`. When in doubt, you can run any terminal command inside Copilot by prefixing it with `!`.

```bash
/login
```

When prompted what account do you want to log into, select GitHub.com. Copilot will prompt you to enter any key to open a browser to complete login. Follow the instructions in Copilot to complete authorization using the signed-in account.

## 5. Install the Azure Skills Plugin

1. Add the Microsoft marketplace:
   ```
   /plugin marketplace add microsoft/azure-skills
   ```

2. Install the Azure plugin:
   ```
   /plugin install azure@azure-skills
   ```

3. Reload Azure MCP:
   ```
   /mcp reload
   ```

4. To update later:
   ```
   /plugin update azure@azure-skills
   ```

> 💡 **MCP tools vs. Azure skills:** The Azure MCP server provides **MCP tools** — low-level operations like listing resources, querying logs, and managing deployments. Azure **skills** are higher-level prompt instructions that chain these tools together with domain knowledge (e.g., `azure-diagnostics` knows how to follow a triage reasoning chain). This lab uses both: skills drive the workflow, MCP tools execute the Azure operations.

✅ **Checkpoint:** You're logged into GitHub and Azure, Copilot CLI is running, Azure skills and Azure MCP Server are installed.

---

**Next:** [Set Up the Starter App →](03-getting-started.md)
