# Bonitasoft

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Bonitasoft is the French open-source company behind **Bonita**, a BPMN 2.0 business process
management and process automation platform. **In June 2026 the company rebranded to Ofelia** — the
product is still Bonita and the GitHub organization is still `bonitasoft`, but every web,
documentation and community host moved from `bonitasoft.com` to `ofelia.com` behind 301 redirects.

- Website: <https://www.ofelia.com/>
- Documentation: <https://documentation.ofelia.com/bonita/latest/>
- API reference: <https://api-documentation.ofelia.com/latest/>
- Source: <https://github.com/bonitasoft> (157 public repositories; engine LGPL-2.1, OpenAPI GPL-2.0)

## The API

Bonita is **software the customer runs** — on premises, in a container, or as managed Bonita Cloud —
so the REST API is served by each deployment at `<host>/bonita/API`, not by a vendor host. There is
no single production base URL, and the `servers[]` block in the published spec correctly declares a
labelled local sample.

Every HTTP-reachable Bonita feature is described by one first-party **OpenAPI 3.0.2** document —
**153 paths, 224 operations, 162 schemas** — which the vendor calls "the single source of truth for
the Bonita features accessible through HTTP". It is versioned independently of the product (semver,
currently **1.0.9**, released 2026-06-19) and every release ships the bundled YAML plus a generated
Postman collection.

Captured verbatim in this repository at `openapi/bonitasoft-bonita-openapi.yml`.

## What this profile records

| Area | Finding |
|---|---|
| Contract | First-party OpenAPI 3.0.2, harvested from the provider's own host and byte-identical to the GitHub release asset |
| Auth | Session cookie (`JSESSIONID`) + CSRF header (`X-Bonita-API-Token`); OIDC bearer on Enterprise. **Not** OAuth scopes — authorization is profile/permission based |
| Conventions | Required `p`/`c` pagination, `content-range` response header, repeatable `d=` field expansion documented in prose but never declared per operation |
| Idempotency | **None.** No idempotency key anywhere in 224 operations |
| Errors | `{ "message": "..." }` only. No RFC 9457, no error codes, no error reference page |
| Rate limits | Exactly one — Community Edition case-creation quota, `429` with `Retry-After` as an **absolute date-time** |
| Deprecation | 33 of 224 operations marked `deprecated: true`, but **no published deprecation policy** and no status page |
| SDKs | One first-party client (`org.bonitasoft.web:bonita-java-client`, Java). Docker Official Image `bonita`. No JS/Python/Go/Ruby/.NET client |
| Sandbox | `docker run -p 8080:8080 -d bonita`, published default credentials, and a vendor `docker-compose.yaml` that starts Swagger UI against a live runtime |
| Security | Real vulnerability reporting policy (`product-security@ofelia.com`) with MITRE CVE coordination; ISO 27001 (Bureau Veritas, 2022) for Bonita Cloud. **No** `/.well-known/security.txt`, no bug bounty, no trust centre |
| Agent surfaces | **No MCP server, no A2A agent card, no llms.txt, no `/.well-known/` document, no webhooks, no AsyncAPI.** First-party AI connectors let a Bonita process *call* a model, which is the other side of the boundary |

Five packaged Agent Skills grounded in verified `operationId`s are in `skills/`.

Source: portfolio company of [serena](https://github.com/api-evangelist/serena)
