# AgentAuth Launchpad

Deploy OAuth for AI Agents in one click. Give your AI agents secure, scoped, time-limited permissions.

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/new?stackName=agentauth&templateURL=https://agentauth-launchpad.s3.amazonaws.com/template.yaml)

## The Problem

AI agents need access to your services (calendar, email, databases), but:
- **Full API keys are dangerous** - Agent could do anything
- **No audit trail** - No idea what agent actually did
- **No revocation** - Can't stop a rogue agent quickly
- **No budget limits** - Agent could rack up huge bills

## The Solution: AgentAuth

Like OAuth, but for AI agents:

```
┌─────────────────────────────────────────────────────────────┐
│                        AgentAuth                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   Scoped    │    │    Time     │    │   Budget    │     │
│  │ Permissions │    │   Limits    │    │   Controls  │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   Audit     │    │   Instant   │    │    Rate     │     │
│  │   Logging   │    │  Revocation │    │   Limiting  │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

## Features

- **Scoped Permissions** - Grant only what's needed (`calendar:read`, `email:send`)
- **Time Limits** - Auto-expire permissions after hours/days
- **Budget Controls** - Set spending limits per agent
- **Audit Logging** - Track every agent action
- **Instant Revocation** - One-click to kill access
- **Rate Limiting** - Control API calls per day
- **Dashboard** - Manage everything visually

## Quick Start (5 minutes)

### Prerequisites

1. **AWS Account** - [Create one here](https://aws.amazon.com/free/)
2. **Database Password** - You'll create one during setup

### Step 1: Launch the Stack

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/new?stackName=agentauth&templateURL=https://agentauth-launchpad.s3.amazonaws.com/template.yaml)

### Step 2: Configure

| Parameter | What to Enter |
|-----------|---------------|
| Stack name | `agentauth` |
| Instance Type | `t3.small` |
| Database Password | Create a strong password (min 8 chars) |
| Admin Email | Your email |

### Step 3: Deploy

1. Click **Next** through options
2. Check "I acknowledge IAM resources"
3. Click **Create stack**
4. Wait 5-10 minutes

### Step 4: Access Dashboard

1. Go to **Outputs** tab
2. Click **DashboardURL**
3. Log in with your admin email

## How It Works

### For Users (Granting Permissions)

```
1. User logs into AgentAuth Dashboard
2. User creates a "Grant" for an agent:
   - Select agent (e.g., "Calendar Bot")
   - Choose scopes (e.g., "calendar:read", "calendar:write")
   - Set time limit (e.g., "24 hours")
   - Set budget (e.g., "$10 max")
3. AgentAuth generates a token
4. User gives token to the agent
5. User monitors usage in dashboard
6. User revokes if needed
```

### For Agents (Using Permissions)

```javascript
// Agent requests permission grant
const response = await fetch('https://your-agentauth.com/api/validate', {
  headers: {
    'Authorization': `Bearer ${agentToken}`
  },
  body: JSON.stringify({
    scope: 'calendar:read',
    action: 'list_events'
  })
});

// AgentAuth validates and logs
if (response.ok) {
  // Permission granted - proceed
  const events = await calendarAPI.listEvents();
}
```

### For Services (Validating Permissions)

```javascript
// Service validates agent token with AgentAuth
app.use('/api', async (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');

  const validation = await fetch('https://your-agentauth.com/api/validate', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      token,
      scope: req.requiredScope,
      action: req.path
    })
  });

  if (!validation.ok) {
    return res.status(403).json({ error: 'Agent not authorized' });
  }

  next();
});
```

## API Reference

### Create Permission Grant

```bash
POST /api/grants
{
  "agent_id": "calendar-bot-123",
  "scopes": ["calendar:read", "calendar:write"],
  "expires_in": "24h",
  "budget_limit": 10.00,
  "rate_limit": 100
}

Response:
{
  "grant_id": "grant_abc123",
  "token": "eyJhbGc...",
  "expires_at": "2025-01-28T00:00:00Z"
}
```

### Validate Token

```bash
POST /api/validate
{
  "token": "eyJhbGc...",
  "scope": "calendar:read",
  "action": "list_events"
}

Response:
{
  "valid": true,
  "grant_id": "grant_abc123",
  "remaining_budget": 8.50,
  "calls_remaining": 95
}
```

### Revoke Grant

```bash
DELETE /api/grants/{grant_id}

Response:
{
  "revoked": true,
  "revoked_at": "2025-01-27T12:00:00Z"
}
```

### Get Audit Logs

```bash
GET /api/grants/{grant_id}/logs

Response:
{
  "logs": [
    {
      "timestamp": "2025-01-27T11:30:00Z",
      "action": "calendar:read",
      "status": "allowed",
      "cost": 0.01
    }
  ]
}
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Your AWS Account                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                 EC2 Instance (t3.small)                 │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │
│  │  │   Frontend   │  │   Backend    │  │  PostgreSQL  │  │ │
│  │  │  (Next.js)   │  │  (Express)   │  │   Database   │  │ │
│  │  │   :3000      │  │    :4000     │  │    :5432     │  │ │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────────┘  │ │
│  │         │                 │                             │ │
│  │  ┌──────▼─────────────────▼───────┐                    │ │
│  │  │          Caddy (SSL)           │                    │ │
│  │  │            :80/:443            │                    │ │
│  │  └────────────────────────────────┘                    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              Secrets Manager (API Keys)                 │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Monthly Costs

| Resource | Cost |
|----------|------|
| EC2 t3.small | ~$15/month |
| Elastic IP | ~$3.65/month |
| Data transfer | ~$1-5/month |
| **Total** | **~$20-25/month** |

## Security

- **Encrypted secrets** in AWS Secrets Manager
- **HTTPS by default** via Caddy auto-SSL
- **JWT tokens** with short expiry
- **VPC isolation** - database not publicly accessible
- **Audit logging** - every action recorded
- **Token blacklisting** - instant revocation works

## Pricing Tiers

### Free (This Repo)
- Deploy yourself
- Self-managed
- Community support

### Support Tier ($9/month)
- Priority email support
- Setup assistance
- [Contact us](mailto:support@agentauth-launchpad.com)

### Setup Service ($99 one-time)
- We deploy for you
- 30-minute walkthrough
- Custom configuration
- [Book setup](mailto:setup@agentauth-launchpad.com)

## Use Cases

### 1. Personal AI Assistant
Grant your Claude/GPT agent limited access to your calendar and email.

### 2. Agentic Workflows
Let autonomous agents interact with your services safely.

### 3. Multi-Agent Systems
Manage permissions across many agents with centralized control.

### 4. Enterprise AI
Compliance-ready audit trails for regulated industries.

## Roadmap

- [ ] OAuth provider integration (Google, GitHub)
- [ ] Web3 wallet authentication
- [ ] Multi-tenant support
- [ ] Webhooks for events
- [ ] Agent marketplace
- [ ] Compliance certifications (SOC2, ISO27001)

## Contributing

PRs welcome! See [CONTRIBUTING.md](docs/CONTRIBUTING.md).

## License

MIT License - See [LICENSE](LICENSE)

---

**Questions?** Open an issue or email support@agentauth-launchpad.com
