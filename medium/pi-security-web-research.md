# Securing Pi credentials while retaining multi-provider access

Research snapshot: 14-09-2026, U.S. Central Time. Primary sources only. No local secrets or configuration inspected.

## Requested Anthropic report

The exact requested URL **exists and is currently verifiable**. Anthropic's live page is titled “Detecting and countering misuse of AI: September 2026,” links its PDF and IOC file, covers December 2025–August 2026, and contains 09–10-09-2026 creation/update metadata. It is not unavailable or future-dated as of this research date. No unavailable content was inferred. [Anthropic report](https://www.anthropic.com/threat-intelligence-report-september-2026)

## Pi credential surface

- Pi retains cross-provider model access through many API-key providers plus subscription login for Anthropic, OpenAI Codex, GitHub Copilot, xAI, OpenRouter, and Radius. [Pi README](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md#providers--models)
- Pi stores login API keys and OAuth credentials in ~/.pi/agent/auth.json. OAuth tokens auto-refresh, except OpenRouter login creates a user-controlled key that does not auto-expire. Pi says auth.json is created mode 0600. Credential precedence is CLI flag, auth.json, process environment, then custom-provider key. [Pi provider docs](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md)
- Pi can execute a !command and consume stdout as a key. auth.json command results are process-cached; models.json commands resolve per request unless wrapped. Pi supports proxy base URLs and OpenAI-, Anthropic-, and Google-compatible custom providers. [Pi provider docs](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md#key-resolution) [Pi model docs](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md#value-resolution)
- Pi packages are privileged: extensions execute arbitrary code and skills can direct command execution. Pi recommends source review; its core npm example uses --ignore-scripts. [Pi package warning](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md#pi-packages) [Pi quick start](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md#quick-start)

## Recommended Windows design

### Preferred: OS vault plus resolver

Store each static provider key separately in Windows Credential Manager or user-scoped DPAPI. CredWrite/CredRead operate on the current logon session's credential set. User-scoped DPAPI normally requires the same user and machine; avoid CRYPTPROTECT_LOCAL_MACHINE because Microsoft says any machine user can decrypt it. [CredWrite](https://learn.microsoft.com/en-us/windows/win32/api/wincred/nf-wincred-credwritew) [CredRead](https://learn.microsoft.com/en-us/windows/win32/api/wincred/nf-wincred-credreadw) [CryptProtectData](https://learn.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptprotectdata)

Point Pi's !command resolver at a small allowlisted helper that retrieves only one named credential and writes only its value to stdout. Keep secrets out of arguments, source, logs, and Pi JSON. This preserves direct multi-provider access and improves at-rest security, but the value still briefly exists in helper output and Pi memory. [Pi provider docs](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md#key-resolution)

### Stronger: localhost broker/proxy

Keep upstream keys in a user-scoped service backed by Credential Manager/DPAPI. Route Pi provider baseUrl overrides/custom providers to loopback; let the broker inject upstream authorization and enforce destinations, provider/model allowlists, budgets, and sanitized audit records. Pi supports proxies while retaining built-in model catalogs. [Pi custom models](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md#custom-models) [Pi overrides](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/models.md#overriding-built-in-providers)

Bind only to loopback, require broker-local authentication, reject arbitrary upstream URLs, and never log authorization headers or bodies. This keeps upstream keys away from Pi/extensions; broker compromise still exposes its allowed authority.

### Environment/process inheritance

Windows child processes inherit the parent's environment by default. Persistent user/system variables or keys set in Pi's terminal can reach tools, shells, package scripts, and extensions. Prefer the vault resolver or broker; otherwise use a dedicated short-lived launcher and sanitized child environments. [Microsoft environment variables](https://learn.microsoft.com/en-us/windows/win32/procthread/environment-variables)

Avoid Pi's --api-key option: it has highest precedence and places the key on a command line. Prefer login, a vault resolver, or broker-local token. [Pi resolution order](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md#resolution-order)

## Provider controls

### Anthropic

Use personal keys for one developer and service-account keys for shared automation; scope to one workspace. Anthropic supports expirations, recommends secret-manager storage and periodic rotation, and permits disabling/deleting leaked keys. Set workspace spend/rate limits and alerts. [Anthropic authentication](https://platform.claude.com/docs/en/manage-claude/authentication) [Anthropic workspaces](https://platform.claude.com/docs/en/manage-claude/workspaces) [Anthropic limits](https://platform.claude.com/docs/en/api/rate-limits)

For production/CI, Anthropic WIF exchanges an IdP JWT for a short-lived service-account token; configured lifetime is 60–86,400 seconds and SDKs refresh it. Pi docs do not claim native Anthropic WIF, so use a broker/custom integration if needed. [Anthropic WIF](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation)

### OpenAI

Create a Pi-specific project/key and grant only required model-request permissions. Set expiration, rotate with brief old/new overlap, then revoke old. OpenAI documents maximum key lifetimes, usage monitoring, separate projects, RBAC, service accounts, IP allowlists, mTLS, and project rate/spend limits. [OpenAI production practices](https://platform.openai.com/docs/guides/production-best-practices) [OpenAI RBAC](https://platform.openai.com/docs/guides/rbac)

Pi's OpenAI subscription login avoids a user-managed API key but stores refreshable OAuth credentials in auth.json; protect it as credential material. [Pi subscriptions](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md#subscriptions)

### Google Gemini / Vertex AI

Prefer Gemini authorization keys: Google says they are service-account-bound, Gemini-restricted by default, and support faster leaked-key enforcement. Apply API and origin/IP restrictions, separate projects, billing alerts, and replacement-before-revocation rotation. Google says never expose production keys client-side and recommends a backend proxy. [Gemini keys](https://ai.google.dev/gemini-api/docs/api-key)

For Vertex AI, Pi supports Application Default Credentials. Prefer ambient identity or WIF over service-account key files; Google documents short-lived token exchange and dedicated least-privileged service accounts. [Pi Vertex](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md#google-vertex-ai) [Google ADC](https://cloud.google.com/docs/authentication/provide-credentials-adc) [Google WIF](https://cloud.google.com/iam/docs/best-practices-for-using-workload-identity-federation)

### OpenRouter

Create a named Pi-only key with a credit limit and replace exposed keys immediately. Pi's PKCE login still stores a non-expiring user-controlled key. [OpenRouter authentication](https://openrouter.ai/docs/api_reference/authentication) [OpenRouter OAuth](https://openrouter.ai/docs/guides/overview/auth/oauth)

Assign guardrails—not merely create them—for budgets, model/provider allowlists, ZDR, regions, and sensitive-information controls. Enterprise WIF exchanges an IdP JWT for an inference-only token valid at most 15 minutes; use it behind a broker because Pi's built-in login yields a persistent key. [OpenRouter guardrails](https://openrouter.ai/docs/guides/features/guardrails) [OpenRouter WIF](https://openrouter.ai/docs/guides/overview/auth/workload-identity-federation)

## Rotation and incident response

1. Inventory keys by provider/owner without copying values; name each Pi credential distinctly.
2. Create a least-privileged replacement with workspace/project scope, spend limits, and expiration.
3. Atomically update the vault/broker; restart Pi because auth.json command results may be process-cached. [Pi key resolution](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/providers.md#key-resolution)
4. Verify required models/fallbacks, then revoke old. [Anthropic rotation](https://platform.claude.com/docs/en/manage-claude/authentication) [OpenAI rotation](https://platform.openai.com/docs/guides/production-best-practices)
5. On suspected leakage, revoke first, then inspect provider usage, billing, and audit records. Never paste the value into chat, tickets, commands, or logs. [Gemini response](https://ai.google.dev/gemini-api/docs/api-key) [OpenRouter response](https://openrouter.ai/docs/api_reference/authentication)

## npm/package supply chain

Untrusted extensions or lifecycle scripts running as the same user can defeat secure storage. Install only reviewed Pi packages, pin versions/commits where practical, and use Pi's documented --ignore-scripts core installation. [Pi README](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md#quick-start) [Pi warning](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/README.md#pi-packages)

For publishing Pi packages, npm recommends OIDC trusted publishing with short-lived workflow-specific credentials. If tokens remain necessary, use granular tokens restricted by package/scope, access level, IP range, and expiration; avoid bypass-2FA. [npm trusted publishers](https://docs.npmjs.com/trusted-publishers) [npm access tokens](https://docs.npmjs.com/about-access-tokens) [npm 2FA](https://docs.npmjs.com/requiring-2fa-for-package-publishing-and-settings-modification)

## Bottom line

Retain Pi's provider catalog. Keep OAuth where useful; place unavoidable static keys in Credential Manager/user-scoped DPAPI and resolve through Pi's command hook. Use a loopback broker when Pi/extensions should never receive upstream keys. Enforce provider-side least privilege, expiration, budgets, alerts, and tested rotation. Treat environment variables as compatibility plumbing, not a long-term vault.
