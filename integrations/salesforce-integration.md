# Salesforce CRM Integration

## Overview
The AI Agent captures patient details during conversation (name, phone, email, patient type, 
preferred location, concern, preferred appointment time) and creates a corresponding Lead in 
Salesforce, along with a silently-calculated lead temperature (HOT / WARM / COLD) used by the 
sales team to prioritize follow-up.

## Attempt 1: TARS Native Salesforce Tool
The first approach was to use TARS's built-in Salesforce connector — the right first move, since 
using a vendor's native integration before building something custom is generally the correct 
instinct.

- Connected a Salesforce Developer org
- Mapped Lead fields (name, phone, email, patient type, etc.)
- Ran a test — it failed

**Root cause:** The Event Log showed the tool authenticated successfully and attempted the API 
call, but its field-mapping UI didn't support variable interpolation from the conversation, so 
required fields were sent empty.

## Attempt 2: Custom Two-Step API Integration
Since the native tool couldn't support the required field mapping, a custom integration was built 
using two chained API Call nodes:

1. **Get OAuth Token** — Authenticate with Salesforce and retrieve an access token
2. **Create Lead** — Use that token to create a Lead record with the captured patient data

### Issue: `invalid_grant` error
The first authentication attempt failed with an `invalid_grant` error. Debugging steps taken:

- Verified the password was correct by logging into Salesforce directly
- Regenerated the security token
- Checked IP restrictions on the org
- Rebuilt the request body from scratch

**Root cause:** The Salesforce org used the newer **External Client App** framework, which does 
not support the OAuth flow originally used (Username-Password flow) — it isn't even offered as 
an option for that app type.

### Fix: Client Credentials Flow
Switched to the **OAuth 2.0 Client Credentials flow**, which the External Client App framework 
does support:

- Set up a Run-As user for the Client Credentials flow
- Rebuilt the token request against the org's specific My Domain
- Re-tested the two-step flow end to end

## Result
A real Lead is now created in Salesforce directly from a chatbot conversation, with the patient's 
name, contact details, and lead temperature populated in the Rating field — ready for the sales 
team to act on.

## Key Takeaway
When a native/no-code integration fails, checking the platform's execution/event logs first 
(rather than assuming the integration itself is broken) is critical — it revealed the real issue 
was a field-mapping limitation, not an authentication problem. The subsequent OAuth failure 
required the same debugging discipline: isolating variables one at a time (credentials, tokens, 
network) before identifying the actual root cause (an unsupported flow for the app type).