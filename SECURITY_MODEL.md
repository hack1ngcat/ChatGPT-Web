# ChatGPT-Web security threat model

## Scope and evidence

This model covers all of `hack1ngcat/ChatGPT-Web`, including the browser application, backend, model integration and repository-local deployment configuration. References below are repository-relative. Update the model when these boundaries, deployment assumptions or accepted risks change.

**Owner-confirmed assumptions and requirements:** assume internet exposure and include anonymous and malicious authenticated users. Protect private chats, API/access tokens, the access password, API spend and availability. No accepted risks. Requested finding tags: Component, CVE, Compliance scope and Owning team. Other code-derived observations below are not user risk decisions.

## System and components

| Stable component | Code covered | Responsibility and trust boundary |
| --- | --- | --- |
| Web Client | `src/**`, root frontend manifest/lockfile and Vite configuration | Vue/Pinia application in the user's browser; accepts prompts, displays model output and imports/exports chats. Browser storage and untrusted content meet the application origin. |
| API Access | `service/src/index.ts`, `service/src/middleware/**`, other service bootstrap/build/dependency files | Express server, static assets and HTTP entry points; crosses from remote clients to the server and its shared credentials. |
| Model Gateway | `service/src/chatgpt/**`, `service/src/types/**`, `service/src/utils/**` | Sends prompts and conversation identifiers to OpenAI or an unofficial ChatGPT reverse proxy; crosses from the trusted server into external providers/proxies. |
| Deployment | `Dockerfile`, `docker-compose/**`, `.github/workflows/**` and container/build configuration | Container packaging, Nginx and CI/image publishing; crosses from contributors/build systems and image registries into deployed runtime. |

Flow: browser → optional Nginx → Express → model API/reverse proxy (optionally through SOCKS) → streamed response → browser renderer/storage. No separate database or worker is defined in the reviewed application code.

## Entry points and reachability

- Express listens on port 3002, serves `public`, parses JSON and mounts the same router at both `/` and `/api` (`service/src/index.ts`).
- POST `/chat-process` accepts a prompt and conversation context, passes through `auth` and streams model output. POST `/config`, `/session` and `/verify` are reachable without that middleware.
- `AUTH_SECRET_KEY` is an optional shared password. When set, chat requests require its Bearer value; when absent the middleware allows requests (`service/src/middleware/auth.ts`). There is no per-user identity in this middleware.
- Compose publishes app port 3002 and Nginx port 80; Nginx proxies `/api` to the app (`docker-compose/docker-compose.yml`, `docker-compose/nginx/nginx.conf`). Actual firewall, ingress, TLS termination and identity-proxy configuration are unconfirmed.
- The browser accepts imported chat JSON (`src/components/common/Setting/General.vue`) and renders model output through MarkdownIt, KaTeX and syntax highlighting into `v-html` (`src/views/chat/components/Message/Text.vue`). Treat imported content and model responses as untrusted.
- CI processes pushes/PRs; the Docker workflow logs in to a registry and publishes images using repository secrets (`.github/workflows/ci.yml`, `build_docker.yml`). Trusted contributor and CI permissions are separate from runtime access.

## Assets and sensitivity

- OpenAI API keys or ChatGPT access tokens in the server environment; optional shared access password; registry credentials used by CI.
- Prompts, responses, conversation identifiers, imported/exported histories and the browser's access token. `chatStorage` and `SECRET_TOKEN` use the unencrypted, non-expiring `ss` localStorage wrapper (`src/store/modules/{chat,auth}/helper.ts`, `src/utils/storage/local.ts`).
- Provider spending/quota, service availability and integrity of deployed images and browser code.
- The repository owner confirmed that private chats, API/access tokens, the access password, API spend and availability require protection; none of these risks is accepted. User content and credentials cross the provider/proxy boundary; permitted providers, retention and any regulated data classes are unknown.

## Attacker profiles

The repository owner explicitly confirmed internet exposure as the threat-model assumption and inclusion of anonymous and malicious authenticated users. Additional code-based attacker profiles: malicious websites interacting with browsers; attackers supplying imported chat content or influencing model output; compromised dependencies, image registries or CI contributors; and untrusted external proxies. No attacker class has been ruled out by the user.

## Existing controls and limits observed

- Shared-password middleware on chat processing when configured; server-side provider credentials loaded using dotenv; provider debug logging disabled.
- Browser API requests attach a Bearer token (`src/utils/request/axios.ts`); user-written messages use `v-text`. MarkdownIt is initialized without enabling raw HTML, but plugin/generated HTML remains a distinct rendering boundary.
- Express JSON parsing and a configured model-call timeout exist. These observations do not establish complete input validation, resource controls or endpoint authorization.
- API base URL, reverse-proxy URL and SOCKS configuration come from server environment variables (`service/src/chatgpt/index.ts`). Third-party services receive data according to the selected mode.
- HTTPS, private networking, an identity-aware proxy, backups, per-user isolation and external rate limits have not been verified. Public/private exposure must not be asserted from examples alone.

## Accepted risk, exclusions and prioritization

The repository owner explicitly accepts no risks and has excluded no surfaces or attackers. Begin with the scanner's severity, including its exploitability assessment. The confirmed broad protection goals will normally leave severities unchanged. Raise a finding only in the rare case where a scanner-confirmed, readily exploitable path reaches an asset the repository owner named as critical. Code-derived assumptions alone cannot raise or lower a finding. Do not raise hard-to-exploit or unverified findings.

Use the stable component names above for `Component`. The repository owner also requested `CVE` (a confirmed applicable CVE identifier), `Compliance scope` (an explicitly applicable framework/control or data scope) and `Owning team` (an owner established from repository ownership metadata or owner input). Omit values that are not supported; no team mapping or compliance framework has been supplied.

## Open questions

1. Actual production ingress/TLS, external abuse controls and provider/proxy trust are unconfirmed; use the owner-approved internet-exposure assumption rather than assuming protective infrastructure.
2. Exact owning-team mappings, compliance frameworks, regulated data classes and relative criticality among protected assets are unspecified. Do not invent assignments or obligations.
3. Repository-local Docker/Nginx configuration was included; no separate infrastructure repository was supplied or identified by repository-name searches.
