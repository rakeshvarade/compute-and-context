# 🏰🤖 Taking Foundry Agents to Production: A Defense-in-Depth Blueprint with Entra ID

> *Part 4 of 5 (the grand finale!) in the series: **Securing Microsoft Foundry Agents with Microsoft Entra ID***
> 👈 Catch up: **Blog 1** (why identity beats API keys), **Blog 2** (the token flow), **Blog 3** (delegation and governance).

![level](https://img.shields.io/badge/level-beginner%20to%20intermediate-brightgreen) ![topic](https://img.shields.io/badge/topic-Production%20Readiness-blue) ![read](https://img.shields.io/badge/read%20time-14%20min-orange)

---
![defense in depth](/assets/04-defense-in-depth.png)
## 🎬 The Hook: "It Works on My Laptop!" 💻🔥

Remember **HR-Buddy**, our overly helpful HR agent from Blog 3? 🤖

It now has:
- ✅ A proper Entra agent identity (Blog 1)
- ✅ Federated credentials and least-privilege roles (Blog 2)
- ✅ Delegated access, a sponsor and Conditional Access (Blog 3)

The CEO announces: *"Ship it to production on Friday!"* 🚀

Cue the sweating. 😅 Because production brings a whole new set of questions:

- 🌐 *Can the internet reach our agent's data?*
- 🏗️ *Is this deployed by hand-clicking in a portal? (Please say no.)*
- 🔑 *How does our pipeline log in to Azure without a stored password?*
- 🔭 *If something weird happens at 3 AM, how will we even know?*
- 🧯 *And if it does, how do we stop it fast?*

This final post answers all of those with one simple idea: **defense in depth**. 🧅

---

## ⚡ TL;DR (For the Busy Humans)

- 🏰 Don't rely on one lock. **Layer** network, identity, data, monitoring and response.
- 🏗️ Deploy with **Infrastructure as Code** (Bicep or Terraform), not clicking.
- 🔐 Let your CI/CD log in with **federated credentials (OIDC)**, not stored secrets.
- 📜 Use **Azure Policy** to make the secure setup the *only* possible setup.
- 🔭 Turn on **logging** (Entra sign-in and audit logs, plus Azure diagnostics) and write a few KQL alerts.
- 🧯 Write the **kill-switch runbook before you need it**.

---

## 🏰 Part 1: The Castle Analogy (a.k.a. Defense in Depth)

A smart castle doesn't depend on a single front door. It has **layers**:

| Castle layer 🏰 | Foundry + Entra equivalent | Series link |
|---|---|---|
| 🌊 **Moat** | Private networking, private endpoints, no public access | Part 2 below |
| 🚪 **Gate guards checking IDs** | Entra authentication and Conditional Access | Blogs 2 and 3 |
| 🗝️ **Keys that open only certain rooms** | RBAC with least privilege | Blog 2 |
| 🏦 **The vault** | Your own Storage, Search and Cosmos DB with data protections | Part 2 below |
| 📹 **CCTV cameras** | Sign-in logs, audit logs, diagnostics, alerts | Part 5 below |
| 🏗️ **Master building plans** | Infrastructure as Code | Part 3 below |
| 👷 **Builders with temporary passes** | CI/CD with federated credentials | Part 4 below |
| 🧯 **Fire alarm and evacuation plan** | Incident-response runbook | Part 6 below |

If an attacker gets past one layer, the next one is waiting. 🧅

---

## 🌊 Part 2: The Moat, Securing the Network and Data

### 🧠 Why bother if identity is solid?
Identity answers *"who is knocking?"* The network answers *"should this door even be visible from the street?"* 🏘️ Both matter. Even a perfect badge system helps less if the building sits on a public sidewalk with the door propped open.

### 🛠️ Foundry's Secure Option: Standard Setup with Private Networking

Foundry Agent Service offers **Standard Setup with private networking**, where you bring your own (BYO) virtual network. Here's what that means in plain English:

| Concept | What it means | Analogy |
|---|---|---|
| 🏠 **BYO resources** | Agent files, vector data and conversation state live in **your** Azure Storage, Azure AI Search and Azure Cosmos DB | Your own safe, not a shared locker |
| 🔌 **Private endpoints** | Services get private IP addresses inside your VNet | Doors only visible from inside the castle |
| 🧩 **Delegated agent subnet** | Agent compute connects to a subnet you provide | The agents live inside the castle walls |
| 🚫 **No public egress** | Agent traffic stays private by default | Drawbridge stays up |

### 🪤 Gotchas to Know Before You Start
Based on the current docs (these evolve, so always recheck):

1. 🧑‍💻 **Programmatic deployment is required.** Network-secured Agent Service is deployed with Bicep or Terraform, not the portal.
2. 🌍 **Same region.** The Foundry resource must sit in the same region as the virtual network.
3. 🔒 **Dedicated agent subnet.** Each Foundry resource needs its own agent subnet, and it can't be shared.
4. 🔢 **Private IP ranges only.** The delegated subnet must use RFC 1918 private ranges.
5. 🔗 **Private endpoints aren't automatic** for Storage, Search and Cosmos DB, so you configure them (plus private DNS zones) yourself unless you use the provided template.
6. 🔌 **Private MCP servers** in the same VNet were listed as not supported, with only publicly accessible MCP servers supported. Check whether that has changed.
7. 🔑 **Keyless inside the moat.** Azure AI Search tool connections over a private VNet use keyless Entra authentication with the project managed identity. *Identity again. See, we told you.* 😉

> 💡 **Tip:** If you don't have a VNet yet, Microsoft's Standard Setup template can provision the networking for you.

---

## 🏗️ Part 3: The Master Plans, Infrastructure as Code

### 😱 The Click-Ops Nightmare

```text
Dev:  "I set it up in the portal last month."
Ops:  "Which settings did you change?"
Dev:  "...Some?" 🤷
```

Hand-clicked environments drift, can't be reviewed and can't be rebuilt. IaC fixes this:

- 📜 **Reviewable:** security settings go through pull requests
- 🔁 **Repeatable:** dev, test and prod come from the same template
- 🕵️ **Auditable:** Git history shows who changed what and why
- 🪦 **Disposable:** rebuild instead of "fixing" a mystery environment

### 🧱 What Your Template Should Include

- ✅ Foundry resource and project (with `disableLocalAuth: true` from Blog 1 ✨)
- ✅ Model deployments
- ✅ BYO Storage, Search and Cosmos DB, with public access disabled
- ✅ Private endpoints and DNS zones
- ✅ **Role assignments** for the agent identity and the project's managed identity
- ✅ Diagnostic settings sending logs to Log Analytics
- ✅ Tags, including an **owner and sponsor** tag (Blog 3 💪)

> 📌 Foundry's agent identity is created by the platform, but you can still script the role assignments *to* it. Keep a separate identity per environment (dev, test and prod), so a dev experiment can never touch production data.

### 📜 Make Security the Default with Azure Policy

IaC covers what *you* deploy. **Azure Policy** covers what *anyone* deploys. Assign policies that audit or deny insecure settings, such as keys left enabled (local authentication) or public network access on AI resources. Microsoft provides built-in policy definitions for these kinds of checks, so search the built-in list before writing your own.

> 🏰 **Analogy:** IaC is the architect's plan. Azure Policy is the building inspector who refuses to let anyone build a door without a lock.

---

## 👷 Part 4: The Builders With Temporary Passes, CI/CD Without Secrets

Your pipeline needs to deploy to Azure. The old way is to store a **client secret** in your pipeline settings. 😬 It leaks, it expires, and nobody remembers who created it.

The better way is **workload identity federation**. Your pipeline proves who it is with a short-lived token from the CI system, and Entra trusts it based on a rule you configured. No password is stored anywhere. 🪄

> 🏨 **Analogy (callback to Blog 2!):** Instead of giving the builder a permanent key, the building manager says: *"Anyone sent by the official contractor on weekdays, from this specific project, gets a pass that expires in an hour."*

### 💻 Code Sample 1: GitHub Actions Deploying With OIDC

```yaml
name: Deploy Foundry infra 🚀

on:
  push:
    branches: [main]

# 🔑 Lets the job request an OIDC token. No stored secrets!
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 🔐 Azure login (OIDC, no password)
        uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: 🏗️ Deploy Bicep
        run: |
          az deployment group create \
            --resource-group rg-hrbuddy-prod \
            --template-file infra/main.bicep \
            --parameters infra/prod.bicepparam
```

The matching trust rule (a **federated credential**) is set up once on the app or managed identity used by the pipeline, and it limits which repo and branch or environment may sign in. Scope it tightly: **production deploys should only trust the `main` branch or a protected environment**. 🔒

> ⚠️ Give the pipeline identity only the roles it needs on the target resource group, not Owner on the subscription. (Yes, we're still preaching least privilege. It's the series theme!) 🎤

---

## 🔭 Part 5: The CCTV, Logging and Monitoring

If an agent does something odd at 3 AM, will you know? Without logs, the answer is *"we'll find out when the CEO emails us."* 📧😱

### 📹 What to Watch

| Source | What it tells you |
|---|---|
| 🪪 **Entra sign-in logs** | When agent identities authenticate, from where, and whether it succeeded or was blocked |
| 📝 **Entra audit logs** | Changes to agent identities and blueprints, and new credentials being added |
| 🏢 **Azure resource diagnostics** | Who accessed Storage, Search, Cosmos DB and the Foundry resource |
| 🧠 **Agent traces** | What the agent actually did (prompts, tool calls), via your tracing setup |

### ✨ Entra Makes Agents Easy to Spot

Entra's logs include **agent-specific details**: sign-in logs have an *agent* filter (you can filter by agent type such as agent identity or blueprint), and audit events carry an `agentType` value and a `blueprintId` that ties an agent identity back to its blueprint. That blueprint link is handy: one query can tell you everything a whole *family* of agents did. 🧬

### 🪤 Gotcha: Logs Aren't Always Streamed by Default
To query service principal and managed identity sign-ins in Log Analytics or Microsoft Sentinel, you need to **enable those categories in Entra diagnostic settings**. Do this *before* you need the data. Logs you didn't collect can't help you. 🕳️

### 💻 Code Sample 2: A Starter KQL Query

This sketch looks for an agent identity failing to sign in repeatedly, which could mean misconfiguration or someone poking around. 🔍

```kusto
// 🕵️ Which agent identities are failing sign-in the most in the last 24h?
AADServicePrincipalSignInLogs
| where TimeGenerated > ago(24h)
| where ResultType != "0"                       // 0 = success, anything else = failure
| summarize
    Failures = count(),
    DistinctIPs = dcount(IPAddress),
    LastSeen = max(TimeGenerated)
    by ServicePrincipalName, ServicePrincipalId, ResultType
| where Failures > 10
| order by Failures desc
```

And a second one to catch a classic persistence trick, where **someone adds a new credential**:

```kusto
// 🚨 New secrets/certificates added to apps or service principals
AuditLogs
| where TimeGenerated > ago(7d)
| where OperationName has_any ("credentials", "Certificates and secrets")
| project TimeGenerated, OperationName, InitiatedBy, TargetResources
```

> 🧪 Table and column names can differ between workspace setups and schema updates, so **test these in your own workspace** and tune the thresholds. Turn the good ones into **alert rules**. 🔔

### 🚨 Don't Forget Identity Protection
Microsoft Entra ID Protection watches agent identities for anomalous behavior, and (as covered in Blog 3) its risk signals can feed a Conditional Access policy that **blocks high-risk agents automatically**. Logs tell you what *happened*. Protection helps stop it *happening*. 🛡️

---

## 🧯 Part 6: The Fire Alarm, an Incident Runbook

Something's wrong: odd sign-ins, strange data access, or HR-Buddy suddenly speaking in pirate. 🏴‍☠️ Panic is not a plan. Write this runbook **today**:

### 🧯 Agent Incident Runbook (Copy and Adapt!)

| Step | Action | Why |
|---|---|---|
| 1️⃣ **Contain** | Disable the **agent identity** (or its **blueprint** to stop every agent built from it) or block it with a **Conditional Access** policy | Stops new tokens being issued (the kill switch from Blog 3 🔴) |
| 2️⃣ **Cut access** | Remove the agent's **role assignments** on resources | Revokes what it could do with any token still in flight |
| 3️⃣ **Rotate** | Rotate or remove any credentials (should be few or none, thanks to federation 🎉) and any keys on connected tools | Closes backdoors |
| 4️⃣ **Investigate** | Query sign-in logs, audit logs, resource logs and agent traces, using the `blueprintId` to find related agents | Understand the blast radius |
| 5️⃣ **Notify** | Contact the agent's **sponsor** and your security team | That's why sponsors exist (Blog 3 🧑‍⚖️) |
| 6️⃣ **Recover** | Fix the root cause, redeploy from IaC and restore least-privilege access | Rebuild from a known-good state |
| 7️⃣ **Learn** | Hold a blameless post-incident review and update policies | Make it harder next time |

### 💻 Code Sample 3: Step 2, Strip the Agent's Roles

```bash
# 🪪 Object ID of the agent identity (from the Entra admin center)
AGENT_OBJECT_ID="<agent-identity-object-id>"

# 👀 See everything this agent can access, across scopes
az role assignment list \
  --assignee $AGENT_OBJECT_ID \
  --all \
  --query "[].{role:roleDefinitionName, scope:scope}" -o table

# 🧹 Remove them (review the list first!)
az role assignment delete --assignee $AGENT_OBJECT_ID --all
```

> ⚠️ **Practice this in a test environment first.** An untested runbook is just a wish. 🌠

---

## 🗓️ Part 7: Your 30-Day Roadmap (a.k.a. "What to Do Monday")

| Week | Focus | Do this |
|---|---|---|
| 📅 **Week 1** | Identity basics | Switch to Entra auth, disable keys, give each agent its own identity and least-privilege roles |
| 📅 **Week 2** | Governance | Assign sponsors, set up lifecycle workflows and enable Conditional Access policies (check licensing!) |
| 📅 **Week 3** | Platform | Move to IaC, add Azure Policy, switch the pipeline to OIDC |
| 📅 **Week 4** | Detect and respond | Enable log categories, add KQL alerts, write and rehearse the runbook |

---

## 🏆 The Go-Live Checklist

**🪪 Identity**
- ✅ Every agent has its own identity, with a named **sponsor**
- ✅ API keys disabled (`disableLocalAuth`), Entra auth only
- ✅ Least-privilege roles at the smallest scope
- ✅ Delegation used where answers depend on the user; security trimming on search

**🌊 Network and Data**
- ✅ Private networking where required, public access disabled on BYO resources
- ✅ Same-region and dedicated-subnet requirements checked

**🏗️ Platform**
- ✅ Everything deployed from IaC, with Azure Policy as a safety net
- ✅ CI/CD uses OIDC federation, with no stored secrets

**🔭 Detection and Response**
- ✅ Entra diagnostic categories enabled and logs flowing
- ✅ Alerts for failed sign-ins and new credentials
- ✅ Conditional Access plus Identity Protection configured
- ✅ Kill-switch runbook written **and rehearsed** 🧯

**📋 Reality Check**
- ✅ Preview features and license requirements re-verified in the docs

---

## 🧠 Series Recap: The Four Posts in One Table

| Blog | Core idea | One-liner |
|---|---|---|
| 1️⃣ | Why identity | *Don't hand your intern the master key.* 🗝️ |
| 2️⃣ | AuthN and AuthZ | *Passport proves who you are, boarding pass says where you can go.* ✈️ |
| 3️⃣ | Delegation and governance | *Use the user's valet key, and give every agent a responsible parent.* 🚗 |
| 4️⃣ | Production | *Layer your defenses and rehearse the fire drill.* 🏰 |

---

## 🎬 Final Thoughts

Agents are powerful because they can *do things*. That's also why they need the same discipline we apply to people: a **badge**, a **manager**, **limited access**, **supervision** and an **off switch**. 🪪

Entra ID is what makes all of that possible without reinventing the wheel. Foundry gives your agents their brains. Entra gives them their **identity, accountability and guardrails**. 🧠🛡️

Now go ship HR-Buddy on Friday, ideally on a day when you're not on call. 😄🚀

---

## 📚 References (Microsoft Docs)

**🌊 Network and Setup**
- 🔌 [Set up private networking for Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/virtual-networks)
- 🧱 [Foundry Agent Service environment setup](https://learn.microsoft.com/azure/foundry/agents/environment-setup)
- 🎓 [Learn module: Agent Service and network access](https://learn.microsoft.com/en-us/training/modules/intro-ai-agent-service-security-controls/4-agent-service-and-network-access)

**🔐 Identity and CI/CD**
- 🪪 [Agent identity concepts in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity)
- 🔁 [Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- 🐙 [Use GitHub Actions to connect to Azure (OpenID Connect)](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure-openid-connect)
- 📜 [Azure Policy overview](https://learn.microsoft.com/en-us/azure/governance/policy/overview)

**🔭 Monitoring and Governance**
- 📹 [Sign-in and audit logs for agents in Microsoft Entra](https://learn.microsoft.com/en-us/entra/agent-id/sign-in-audit-logs-agents)
- 🧑‍💼 [Manage agent identities in your organization](https://learn.microsoft.com/en-us/entra/agent-id/manage-agent-identities-admin)
- ✅ [Best practices for Microsoft Entra Agent ID](https://learn.microsoft.com/en-us/entra/agent-id/best-practices-agent-id)
- 🛃 [Conditional Access for agents](https://learn.microsoft.com/en-us/entra/identity/conditional-access/agent-id)
- 🏛️ [Microsoft Entra ID Governance for agent identities](https://learn.microsoft.com/en-us/entra/id-governance/agent-id-governance-overview)
- 🔎 [Microsoft Sentinel community post: detecting AI agents and non-human identities](https://techcommunity.microsoft.com/t5/microsoft-sentinel/detecting-ai-agents-and-non-human-identities-in-microsoft/m-p/4526787)

> 🔎 *Foundry networking requirements, Entra Agent ID features and licensing change often. Treat the links above as the source of truth and verify before you deploy.*

---

💬 **That's a wrap!** 🎉 If this series helped, drop a ⭐ on the repo, share it with your team, and send it to the person who still has an API key in a config file. You know the one. 😉

**Thanks for reading, and happy (secure) agent building!** 🤖🔐✨