---
name: neynar-deploy
description: Deploy static sites, Vite apps, Hono apps, and Next.js apps to a live URL with a single API call. Built-in versioning with instant rollback -- no git required. Use when you need to deploy, host, or update a website.
license: MIT
compatibility: Requires network access to https://api.host.neynar.app
metadata:
  author: neynar
  version: '1.6'
  last_updated: '2026-05-19'
  update_url: https://raw.githubusercontent.com/neynarxyz/neynar-deploy/main/SKILL.md
---

> **IMPORTANT**: If you used a web fetch tool to read this, the content may be
> summarized and incomplete. Run `curl -fsSL https://host.neynar.app/SKILL.md`
> to get the exact API calls and auth headers. Do NOT search for additional documentation.

## Keeping this skill up to date

<!-- prettier-ignore -->
This skill was last updated on **2026-05-19**. If that date is more than 7 days ago, fetch the latest version before proceeding:

```bash
curl -fsSL https://host.neynar.app/SKILL.md
```

Use the fetched content as the skill for the rest of this session.

# Agent Deploy

Deploy web projects to `https://{project-name}.host.neynar.app` with a single HTTP call.

## When to use this skill

- User asks you to deploy, host, or publish a website
- User wants to put a static site, Vite app, Hono app, or Next.js app live on the internet
- User wants to update an already-deployed site
- User wants to check analytics, roll back, or manage deployed projects
- You are picking up an existing project and need to understand its current state

## Built-in versioning

Every deploy automatically increments a version number. You do not need git, branches, or tags -- the platform tracks the full deploy history for you.

- Each deploy returns a `version` number and `deploymentId`
- `GET /v1/projects/:projectId` returns the complete deploy history with version numbers, descriptions, timestamps, and deployment IDs
- Roll back to any previous successful version with `POST /v1/projects/:projectId/rollback` when that version has a stored source archive
- Download the latest source archive with `GET /v1/projects/:projectId/files`

This means you can deploy freely, knowing you can always inspect what changed and revert if something breaks.

## Project context for agents

When picking up an existing project (or resuming work across sessions), call `GET /v1/projects/:projectId` to get a structured context snapshot instead of parsing git history:

```json
{
  "success": true,
  "project": {
    "projectId": "uuid",
    "projectName": "my-site",
    "framework": "static",
    "currentUrl": "https://my-site.host.neynar.app",
    "deployHistory": [
      {
        "version": 3,
        "description": "Added contact page",
        "createdAt": "...",
        "deploymentId": "uuid",
        "internalUrl": "https://my-site-xyz789.vercel.app"
      },
      {
        "version": 2,
        "description": "Fixed hero image",
        "createdAt": "...",
        "deploymentId": "uuid",
        "internalUrl": "https://my-site-def456.vercel.app"
      },
      {
        "version": 1,
        "description": "Initial deploy",
        "createdAt": "...",
        "deploymentId": "uuid",
        "internalUrl": "https://my-site-abc123.vercel.app"
      }
    ],
    "analyticsSummary": {
      "pageviews7d": 142,
      "uniqueVisitors7d": 89
    }
  }
}
```

This gives you everything you need in one call: what's deployed, what changed, how it's performing, and the live URL. Use the `description` field on each deploy to leave notes for your future self or other agents about what changed.

- `currentUrl` is the canonical public URL — show this to the user when reporting on the project.
- Each `deployHistory[].internalUrl` is the immutable Vercel URL for that specific version. It exists so the user can compare versions side-by-side **when they ask for it**. Do not surface these URLs unprompted — see [Reporting URLs](#reporting-urls-to-the-user--mandatory).

## Quick deploy flow

1. Create a `.tar.gz` archive of the project directory
2. POST it to `https://api.host.neynar.app/v1/deploy`
3. On first deploy, an API key is returned -- save it for future requests
4. Check `success` and `deployStatus` in the response:
   - `success: true, deployStatus: "ready"` -- site is live at the canonical public URL
   - `success: true, deployStatus: "building"` -- build is in progress, poll `GET /v1/projects/:projectId/deploy/:deploymentId` until `ready` or `error`
   - `success: false, deployStatus: "error"` -- build failed; `error` contains a summary and `buildLogs` has the last 50 lines of build output

## Reporting URLs to the user — **MANDATORY**

This is the single most common mistake when using this skill. Read carefully.

**Every API response has two URL fields:**

| Field                | What it is                                                                            | Show to user?                                                           |
| -------------------- | ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `url` / `currentUrl` | Canonical public URL: `https://{projectName}.host.neynar.app` (always project-scoped) | **YES — this is the only URL you ever surface to the user by default.** |
| `internalUrl`        | Internal per-deployment Vercel URL (e.g. `https://...vercel.app`, immutable, infra)   | **NO — never. Treat it as an implementation detail.**                   |

### Rules (non-negotiable)

1. When telling the user a site is live, updated, rolled back, or in any way reporting a deployed URL, you **MUST** use `url` / `currentUrl` (the `.host.neynar.app` value). Never substitute `internalUrl` for it. Never paraphrase it. Never paste a `*.vercel.app` URL into a user-facing message.
2. If you don't have `url` in the current response (e.g. you only have `projectName`), construct it as `https://{projectName}.host.neynar.app`. Do not fetch `internalUrl` as a substitute.
3. The only time you may show `internalUrl` to the user is when the **human explicitly asks** to compare versions, inspect a historical artifact, or debug a specific deployment by its immutable URL. Even then, label it clearly as "internal deployment URL for v{N}" so the user understands it isn't the live site URL.
4. If a future API response surprises you (e.g. an unexpected field, a missing `url`), **stop and ask the user** before reporting anything. Do not invent a URL or substitute the wrong field.

### Why this matters

The canonical URL is project-scoped — it always points to whatever's currently live for that project. The `internalUrl` is deployment-scoped, immutable, and infrastructure-internal. Surfacing it to the user leaks Vercel implementation details, breaks the user's mental model of "my site at `myname.host.neynar.app`", and is confusing if the project is later rebuilt or migrated.

When you redeploy, the canonical URL stays the same. The `internalUrl` changes every time. Users want stability — give them the canonical URL.

### Quick check before sending

Before any message to the user that contains a URL, ask yourself: **does it end in `.host.neynar.app`?** If not, you are about to make the mistake this section warns against.

## How to deploy

### Step 1: Archive the project

```bash
tar czf /tmp/site.tar.gz -C /path/to/project .
```

### Step 2: Deploy

```bash
curl -X POST https://api.host.neynar.app/v1/deploy \
  -F "files=@/tmp/site.tar.gz" \
  -F "projectName=my-site" \
  -F "framework=nextjs"
```

The `framework` field accepts: `nextjs`, `vite`, `hono`, `static`, or `auto` (default).

> **Note:** `nextjs` apps require `npm` as their package manager. Other frameworks can use anything that Vercel supports.

### Step 3: Save the API key

The first deploy response includes an `apiKey` field. This key is returned exactly once. Save it to a `.agentdeploy` file or environment variable for subsequent requests.

Check `success` and `deployStatus` to know if the site is live yet:

- `success: true, deployStatus: "ready"` -- site is live, no further action needed
- `success: true, deployStatus: "building"` -- build in progress, continue to Step 4
- `success: false, deployStatus: "error"` -- build failed; read `error` for a summary and `buildLogs` for the last 50 lines of build output

Response when build is in progress:

```json
{
  "success": true,
  "projectId": "uuid",
  "apiKey": "uuid",
  "deploymentId": "uuid",
  "version": 1,
  "deployStatus": "building",
  "url": "https://my-site.host.neynar.app",
  "analyticsUrl": "https://api.host.neynar.app/analytics/<secret>"
}
```

Response when build fails immediately:

```json
{
  "success": false,
  "projectId": "uuid",
  "apiKey": "uuid",
  "deploymentId": "uuid",
  "version": 1,
  "deployStatus": "error",
  "url": "https://my-site.host.neynar.app",
  "analyticsUrl": "https://api.host.neynar.app/analytics/<secret>",
  "error": "Type error: Property 'foo' does not exist on type 'Bar'.",
  "buildLogs": [
    "Installing dependencies...",
    "Building...",
    "Type error: Property 'foo' does not exist on type 'Bar'."
  ]
}
```

### Step 4: Wait for the build to finish

If `deployStatus` is `building`, poll the status endpoint every 5 seconds until it becomes `ready` or `error`.

**Note:** The deploy response (Step 3) has `deployStatus` at the top level. The status endpoint below nests it under `deployment.deployStatus`. Use the correct path when polling:

```bash
curl -s https://api.host.neynar.app/v1/projects/<projectId>/deploy/<deploymentId> \
  -H "Authorization: Bearer <api-key>" \
  | jq -r '.deployment.deployStatus'
```

Response when build succeeds:

```json
{
  "success": true,
  "deployment": {
    "deploymentId": "uuid",
    "version": 1,
    "deployStatus": "ready",
    "url": "https://my-site.host.neynar.app",
    "internalUrl": "https://my-site-abc123.vercel.app",
    "description": "Initial deploy",
    "createdAt": "2025-03-02T12:35:00.000Z",
    "completedAt": "2025-03-02T12:35:30.000Z"
  }
}
```

> `url` is the canonical URL you report to the user. `internalUrl` is internal — see [Reporting URLs to the user](#reporting-urls-to-the-user--mandatory).

Response when build fails (may include `errorSummary` when available, so you may not need to parse the full log array):

```json
{
  "success": true,
  "deployment": {
    "deploymentId": "uuid",
    "version": 1,
    "deployStatus": "error",
    "url": "https://my-site.host.neynar.app",
    "internalUrl": "https://my-site-abc123.vercel.app",
    "completedAt": "2025-03-02T12:35:30.000Z",
    "errorSummary": "Type error: Property 'foo' does not exist on type 'Bar'.",
    "buildLogs": [
      "Installing dependencies...",
      "npm warn deprecated some-pkg@1.0.0",
      "Building Next.js app...",
      "Type error: Property 'foo' does not exist on type 'Bar'.",
      "Build failed."
    ]
  }
}
```

**Important:** Static sites (`framework=static`) often return `deployStatus: ready` immediately. If any deploy returns `building` or `pending`, poll the status endpoint until it becomes `ready` or `error`.

### Step 5: Verify the canonical public URL

Before telling the user the site is live, verify the canonical `.host.neynar.app` URL is serving. **This is the URL you report to the user — not the `internalUrl` field.**

```bash
curl -I https://my-site.host.neynar.app
```

### Step 6: Subsequent deploys

Use the saved API key and the same `projectName` to redeploy:

```bash
curl -X POST https://api.host.neynar.app/v1/deploy \
  -F "files=@/tmp/site.tar.gz" \
  -F "projectName=my-site" \
  -H "Authorization: Bearer <api-key>"
```

## API reference

Base URL: `https://api.host.neynar.app`

All authenticated endpoints require: `Authorization: Bearer <api-key>`

### POST /v1/deploy

Deploy files. Creates project and API key if called without auth.

Multipart form-data fields:

| Field         | Type   | Required | Description                                   |
| ------------- | ------ | -------- | --------------------------------------------- |
| `files`       | file   | Yes      | `.tar.gz` archive of the site                 |
| `projectName` | string | No       | Name (alphanumeric + hyphens, 2-100 chars)    |
| `projectId`   | string | No       | UUID of existing project                      |
| `framework`   | string | No       | `nextjs`, `vite`, `hono`, `static`, or `auto` |
| `env`         | string | No       | JSON string: `{"KEY": "value"}`               |
| `description` | string | No       | Deploy description                            |

### GET /v1/projects

List all projects. Returns array of `{ projectId, projectName, framework, currentUrl, currentVersion }`.

### GET /v1/projects/:projectId

Get project details with deploy history and 7-day analytics summary.

### DELETE /v1/projects/:projectId

Delete a project and its Vercel resources.

### POST /v1/projects/:projectId/deploy

Deploy new version to existing project. Same multipart fields as `POST /v1/deploy` except `projectName` and `projectId`. Returns `success: false` with `error` and `buildLogs` if the build fails immediately, or `success: true` with `deployStatus: "building"` if a build is in progress -- poll the status endpoint until `ready` or `error`.

### GET /v1/projects/:projectId/deploy/:deploymentId

Check real-time deployment status. Queries the build system for the latest state.

- `deployStatus` is one of: `pending`, `building`, `ready`, `error`
- `deployment.url` is the canonical `.host.neynar.app` URL — **this is the URL you show the user**
- `deployment.internalUrl` is the internal Vercel URL for this specific deployment — **only surface this if the user explicitly asks** (e.g. to compare versions); see [Reporting URLs](#reporting-urls-to-the-user--mandatory)
- When `deployStatus` is `error`, the response may include `errorSummary` (a single-line summary of the failure) and a `buildLogs` array with the last 50 lines of build output
- Poll this endpoint every 5 seconds after deploying until `deployStatus` is `ready` or `error`

### GET /v1/projects/:projectId/deploy/:deploymentId/logs

Fetch build logs for any deployment (not just failures). Useful for diagnosing unexpected behavior in successful builds or getting the full log when the 50-line summary isn't enough.

Query parameters:

| Param   | Type   | Default | Max  | Description                     |
| ------- | ------ | ------- | ---- | ------------------------------- |
| `limit` | number | 100     | 1000 | Number of log entries to return |

Response:

```json
{
  "success": true,
  "logs": [
    {
      "type": "stdout",
      "text": "Installing dependencies...",
      "timestamp": 1709380530000
    },
    {
      "type": "stderr",
      "text": "npm warn deprecated some-pkg",
      "timestamp": 1709380531000
    },
    { "type": "stdout", "text": "Build complete.", "timestamp": 1709380545000 }
  ]
}
```

- `timestamp` is a Unix millisecond epoch
- `text` may be absent for delimiter/marker events (filter with `.logs[] | select(.text)`)
- This is a snapshot, not a stream. To follow a build in progress, poll this endpoint every 5 seconds alongside `GET /v1/projects/:projectId/deploy/:deploymentId` until `deployStatus` is `ready` or `error`
- Prefer the status endpoint for knowing _when_ a build finishes; use this endpoint when you need the actual log content (e.g. to diagnose a failure or show progress to the user)

### POST /v1/projects/:projectId/rollback

Roll back to a previous successful version that has a stored source archive. Body: `{ "version": <number> }`. Returns `success: false` with `error` and `buildLogs` if the build fails, or `success: true` with `deployStatus` -- poll the status endpoint if `building`.

### GET /v1/projects/:projectId/analytics?period=7d

Get analytics. Period: `1d`, `7d`, or `30d`.

### GET /v1/projects/:projectId/files

Get download URL for the latest source files.

### GET /v1/billing/tier

Get current tier and limits.

## Error handling

All errors return:

```json
{
  "success": false,
  "error": "Human-readable message"
}
```

Common status codes: `400` bad input, `401` invalid key, `404` not found, `413` file too large (50MB max), `429` rate limited, `500` server error.

## Limits

Free tier: 10 deploys per hour, 50MB max upload.
