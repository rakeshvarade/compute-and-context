# 🎟️🤖 Foundry Agents and Entra ID: A Deep Dive Into the Token Flow

> *Part 2 of 4 in the series: **Securing Microsoft Foundry Agents with Microsoft Entra ID***
> 👈 Missed Part 1? Read *"Why Foundry Agents Need an Identity, Not Just an API Key"* first!

![level](https://img.shields.io/badge/level-beginner%20to%20intermediate-brightgreen) ![topic](https://img.shields.io/badge/topic-AuthN%20%26%20AuthZ-blue) ![read](https://img.shields.io/badge/read%20time-10%20min-orange)

---

## 🎬 The Hook: "But I Logged In! Why Am I Getting a 403?!"

Picture this. Your shiny new Foundry agent is ready. You ask it to read a file from Azure Storage. It thinks for a moment... and replies:

```text
403 Forbidden 🚫
```

You stare at the screen. 😤 *"But the agent has an identity! Entra said it's legit! Why won't Storage let it in?!"*

Congratulations, you've just met the most common beginner confusion in cloud security:

> **Being who you say you are** ≠ **Being allowed to do what you want** 🙃

In this post, we'll untangle both ideas, follow the **token flow step by step**, and finally fix that 403 like a pro. 🛠️

---

## ⚡ TL;DR (For the Busy Humans)

- 🪪 **Authentication (AuthN)** = "Who are you?" Entra ID answers this by issuing tokens.
- 🔑 **Authorization (AuthZ)** = "What may you do?" The target service answers this, usually with **RBAC** roles.
- 🔄 When an agent calls a tool, Foundry runs a **4-stage token exchange** for you automatically.
- 🚫 A brand-new agent identity starts with **zero permissions**. You must grant them.
- 🎯 Always grant **least privilege** at the **smallest scope** that works.

---

## 🧠 Step 1: AuthN vs AuthZ (The Airport Analogy ✈️)

Think about catching a flight:

| Airport moment 🛫 | Security concept 🔐 |
|---|---|
| Showing your **passport** at check-in | **Authentication**: proving who you are |
| Your **boarding pass** says which flight and seat | **Authorization**: what you're allowed to do |
| Passport is valid, but it's the wrong gate | ✅ Authenticated, ❌ not authorized (**403**) |
| Passport expired or fake | ❌ Not authenticated (**401**) |

So when your agent gets a `403`, the good news is that Entra **did** recognize it. The bad news is that nobody gave it a boarding pass for that flight. 😅

---

## 🎭 Step 2: Meet the Cast (Quick Recap)

From Blog 1, you already know the main characters:

| Character | What it is | Analogy |
|---|---|---|
| 📋 **Agent identity blueprint** | The parent object that models the agent and holds its credential setup | The HR template for a job role |
| 🪪 **Agent identity** | A special service principal in Entra ID that represents the agent when it accesses resources | The actual employee badge |
| 🤖 **Foundry Agent Service** | The service that runs your agent and handles the token dance | The friendly concierge |
| 🏢 **Downstream resource** | Storage, Cosmos DB, an MCP server, etc. | The building the agent wants to enter |

Good news: Foundry automatically provisions these for you. When you create a new agent, it gets its own blueprint and an identity for its default instance. You don't have to build any of that by hand. 🎉

---

## 🔄 Step 3: The 4-Stage Token Flow (The Hotel Analogy 🏨)

Here's the part that sounds scary but is actually a polite conversation. When your agent calls a tool that uses agent identity authentication, a multistep OAuth 2.0 token exchange happens **automatically** between Agent Service, Entra ID and the downstream resource.

Let's use a **hotel** analogy. 🏨 Your agent wants to enter the hotel spa (Azure Storage):

### 🥇 Stage 1: Blueprint Authentication
**Agent Service → Entra ID:** *"Here's proof that I'm allowed to act for this blueprint."*

> 🏨 The concierge shows the hotel manager's signed letter: "I'm authorized to speak for this guest."

### 🥈 Stage 2: Agent Identity Token
**Entra ID → Agent Service:** *"Verified! Here's a token for this specific agent identity."*

> 🏨 Reception checks the letter and hands over the guest's **room key card**. This token marks your agent as its own independent actor, separate from humans and managed identities.

### 🥉 Stage 3: Scoped Token Request
**Agent Service → Entra ID:** *"Now I need a token for the **Storage** audience only."*

> 🏨 "Can I get a pass that works for the **spa**? Not the gym. Not the penthouse."

The **audience** is the resource identifier the token is meant for. For example, `https://storage.azure.com` is the audience for Azure Storage.

### 🏅 Stage 4: Authenticated Tool Call
**Agent Service → Tool/Resource:** *"Here's my scoped token."*
**Resource:** *"Token valid. Let me check my guest list (RBAC)... you're on it. Come in!"* ✅ (or `403` if you're not 😬)

### 🗺️ The Whole Flow in One Picture

```text
 ┌────────────────┐  1️⃣ blueprint credentials   ┌──────────────┐
 │ 🤖 Agent       │ ───────────────────────────▶ │ 🛡️ Entra ID  │
 │    Service     │ ◀─────────────────────────── │              │
 │ (the concierge)│  2️⃣ agent identity token     │              │
 │                │ ───────────────────────────▶ │              │
 │                │ ◀─────────────────────────── │              │
 └───────┬────────┘  3️⃣ scoped token (audience)  └──────────────┘
         │
         │ 4️⃣ call tool with scoped token
         ▼
 ┌────────────────┐
 │ 🏢 Downstream  │  ✅ validates token
 │ Resource       │  ✅ checks RBAC for the agent identity
 └────────────────┘
```

> 🔥 **Notice what's missing?** No password, no API key in your code, no secret in a config file. The service does the exchange for you.

---

## 🔑 Step 4: What Credentials Does the Blueprint Use?

The blueprint needs *some* way to prove itself to Entra. Entra supports a few options, and they are not equally safe:

| Credential type | Analogy | Verdict |
|---|---|---|
| 🔤 **Client secret** | A password on a sticky note | 😬 Works, but it can leak and expire |
| 📜 **Certificate** | A fancy signed ID card | 👍 Better, but you must manage and rotate it |
| 🤝 **Federated identity credential** | "I trust that guy, he vouches for me" | 🏆 **Best.** No secret to store at all |

With Foundry, you get the best option out of the box. Foundry provisions a **federated identity credential** on the blueprint for service-managed authentication, so you don't need to configure a client secret or certificate for this flow. 🙌

---

## 🎯 Step 5: Two Ways an Agent Can Act

An agent can use its identity in two different modes. Understanding this early saves a lot of confusion later:

| Mode | Meaning | Analogy |
|---|---|---|
| 🤖 **Application-only** | The agent acts under **its own** authority, with no signed-in user | An employee doing a job using their own badge |
| 👤 **Delegated (on behalf of user)** | The agent acts for a signed-in user, and both the agent and the user need access | A personal assistant acting with your permission and only inside your access |

Delegated access for agent identities is currently in **preview** and has extra requirements (such as agent blueprint endpoint access and consent), so check the docs before betting your production plans on it. We'll dig into this in **Blog 3**. 🔮

---

## 🛂 Step 6: Authorization, Where the 403 Gets Fixed

Remember: a **new agent identity starts with no resource permissions**, and Foundry doesn't automatically assign RBAC roles or other downstream permissions to it. You (or your admin) have to grant what it needs.

Different services use different "guest lists":

| Mechanism | Example |
|---|---|
| 🏷️ **Azure RBAC** | Storage Blob Data Reader on a storage account |
| 🗄️ **Service-specific data-plane roles** | Cosmos DB has its own data-plane roles, separate from Azure RBAC |
| 📱 **Application permissions / app roles** | APIs such as Microsoft Graph, where supported |
| 📋 **Resource-specific policies** | The target service's own access rules |

> 🧠 **Gotcha alert:** Azure RBAC and Cosmos DB data-plane roles are separate systems. Granting one does not grant the other!

### 💻 Code Sample 1: Fix the 403 With a Role Assignment

Use the **object ID of the agent identity** as the assignee:

```bash
# 🪪 The agent identity's object ID (find it in the Entra admin center)
AGENT_OBJECT_ID="<agent-identity-object-id>"

# 🎯 Smallest scope that works: ONE storage account, not the whole subscription
SCOPE="/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account>"

az role assignment create \
  --assignee-object-id $AGENT_OBJECT_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" \
  --scope $SCOPE
```

Read-only role, one storage account, one agent. That's **least privilege** in three lines. 🎯

> ⏳ **Heads up:** Role assignments can take a few minutes to propagate. If you still get a 403 right after running this, grab a coffee ☕ and retry before panicking.

---

## 🔍 Step 7: Peek Inside a Token (Because It's Fun 🕵️)

Tokens are just JSON Web Tokens (JWTs). You can decode one to see who it's for. This is a great debugging trick when you see unexpected errors.

### 💻 Code Sample 2: Decode a Token (for learning only!)

```python
import base64, json

def peek_at_token(token: str) -> dict:
    """Decode the payload of a JWT. ⚠️ For debugging only. Does NOT verify the signature!"""
    payload = token.split(".")[1]
    payload += "=" * (-len(payload) % 4)          # fix base64 padding
    return json.loads(base64.urlsafe_b64decode(payload))

claims = peek_at_token("<paste-a-token-here>")
print("🎯 aud (who is this token for?):", claims.get("aud"))
print("🪪 oid (which identity?):       ", claims.get("oid"))
print("⏰ exp (when does it expire?):  ", claims.get("exp"))
```

What to look for:

- **`aud`**: Is the audience the service you're calling? A token for the wrong audience gets rejected. (Wrong hotel, wrong key card. 🏨)
- **`oid`**: Is this the identity you gave the role to?
- **`exp`**: Tokens are short-lived on purpose. That's a feature, not a bug. ✨

> 🚨 **Safety tip:** Treat tokens like passwords. Never paste real tokens into public chats, issues or blog posts (yes, including this one 😉).

---

## 🪤 Common Gotchas (a.k.a. "Things That Will Waste Your Afternoon")

1. 🚫 **Authenticated but forbidden (403)**: You forgot the role assignment. Fix it per Step 6.
2. 🔀 **Wrong identity got the role**: Each new agent gets its **own** identity. Roles you gave to a different identity don't apply.
3. 🧳 **Migrating from legacy agents**: Moving from a legacy shared project identity or Agent Application to the current model changes the agent identity, and **existing role assignments don't transfer**. Re-grant them!
4. 🗄️ **Mixing up RBAC types**: Azure RBAC ≠ Cosmos DB data-plane roles.
5. 🌍 **Over-broad scope**: Subscription-wide roles are tempting. Resist! Prefer a resource or resource group scope.
6. 🏷️ **Confusing role names**: The Foundry roles were renamed (for example, *Azure AI User* is now **Foundry User**). Role IDs and permissions are unchanged, but you may see both names in docs and portals during the rollout.

---

## 🏆 Best Practices Checklist

- ✅ Understand **AuthN (who)** vs **AuthZ (what)** before debugging anything
- ✅ Let Foundry manage the blueprint's **federated credential**. No secrets needed
- ✅ Grant roles to the **agent identity**, using its object ID
- ✅ Choose the **smallest scope** and the **least-privileged role**
- ✅ Know whether the target uses **Azure RBAC, data-plane roles or app permissions**
- ✅ Decode the token (`aud`, `oid`) when troubleshooting 🕵️
- ✅ Re-grant permissions after migrating agents

---

## 🧠 Key Takeaways

1. 🪪 **Authentication** proves who the agent is, and **authorization** decides what it can do.
2. 🔄 Foundry runs a **4-stage token exchange** automatically: blueprint auth → agent identity token → scoped token → authenticated call.
3. 🤝 The blueprint's **federated credential** means no client secrets to store, leak or rotate.
4. 🚪 A new agent identity starts with **zero permissions**. A 403 usually means a missing role assignment.
5. 🎯 **Least privilege + smallest scope** is your best friend.

---

## 🔮 What's Next?

In **Blog 3**, we'll tackle **acting on behalf of users** (delegated access), how to keep agents from becoming over-sharing "confused deputies" 🤡, and how **Conditional Access, lifecycle governance and Identity Protection** keep your agents in line. 🚔

Until then, may all your tokens be short-lived and all your scopes be small! 🪙✨

---

## 📚 References (Microsoft Docs)

- 🪪 [Agent identity concepts in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity)
- 🧭 [Microsoft Entra Agent ID documentation](https://learn.microsoft.com/en-us/entra/agent-id/)
- 📋 [Agent identity blueprints (Entra)](https://learn.microsoft.com/en-us/entra/agent-id/agent-blueprint)
- 🤖 [Agent identities (Entra)](https://learn.microsoft.com/en-us/entra/agent-id/agent-identities)
- 🛂 [Azure role-based access control in Foundry](https://learn.microsoft.com/en-us/azure/foundry/concepts/rbac-foundry)
- 🏷️ [What is Azure RBAC?](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- 💻 [Assign Azure roles using the Azure CLI](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-cli)
- 🔌 [MCP server authentication in Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/mcp-authentication)
- 🤝 [Agent-to-agent (A2A) authentication](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-to-agent-authentication)
- 🎫 [Hosted agent token service reference](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/hosted-agent-token-service)
- 🔐 [Permissions and consent in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)

> 🔎 *Parts of Entra Agent ID and Foundry (such as delegated access) are in preview and the docs are updated often. Always verify the current status in the links above.*

---

💬 **Enjoyed this post?** Drop a ⭐ on the repo, share it with the teammate who keeps getting 403s, and see you in Blog 3! 🚀