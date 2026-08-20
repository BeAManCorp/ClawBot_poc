# Deploying this to Render — BeAManCorp/ClawBot_poc

This is `openclaw/openclaw` (MIT-licensed) with exactly one addition: a
6-line Dockerfile block so Render's real SSH client access works, not just
its browser Shell tab. Everything else — `render.yaml`, `Dockerfile`,
`docker-compose.yml`, `.env.example` — is upstream, unmodified.

**Why not the from-scratch bundle from earlier:** OpenClaw's own maintainers
already ship a tested `render.yaml` Blueprint and an official guide
(`docs/install/render.mdx`, published at
[docs.openclaw.ai/install/render](https://docs.openclaw.ai/install/render)).
That's authoritative in a way nothing I hand-built could be — so this
supersedes the compose/entrypoint bundle sent earlier. Discard that one.

```diff
 USER node

+# --- Render deployment addition (BeAManCorp/ClawBot_poc) ---
+# Render's SSH client access (as opposed to the Dashboard's browser Shell tab)
+# requires the image to pre-create ~/.ssh for the running user with 0700 perms:
+# https://render.com/docs/ssh
+RUN install -d -m 0700 -o node -g node /home/node/.ssh
+# --- end Render deployment addition ---
+
 # Start gateway server with default config.
```

## Push this to your repo

This directory is a shallow clone of `openclaw/openclaw` with that patch
committed on top. From here:

```bash
git remote set-url origin https://github.com/BeAManCorp/ClawBot_poc.git
git push -u origin HEAD:main
```

(I can't push this myself — this sandbox has no credentials for your GitHub
account.)

## Deploy

Once pushed, either:

- **One click**: `https://render.com/deploy?repo=https://github.com/BeAManCorp/ClawBot_poc`
- **Dashboard**: New → Blueprint → select the `BeAManCorp/ClawBot_poc` repo. Render reads `render.yaml` and pre-fills everything.

`render.yaml` as shipped:

```yaml
services:
  - type: web
    name: openclaw
    runtime: docker
    plan: starter
    dockerCommand: node openclaw.mjs gateway --allow-unconfigured
    healthCheckPath: /startupz
    envVars:
      - key: OPENCLAW_GATEWAY_PORT
        value: "8080"
      - key: OPENCLAW_STATE_DIR
        value: /data/.openclaw
      - key: OPENCLAW_WORKSPACE_DIR
        value: /data/workspace
      - key: OPENCLAW_GATEWAY_TOKEN
        generateValue: true # auto-generated, this is your control-UI password
    disk:
      name: openclaw-data
      mountPath: /data
      sizeGB: 1
```

`--allow-unconfigured` is what lets the gateway boot on a completely fresh
disk (no `openclaw.json` yet) instead of exiting — there's no shell available
before the first successful boot, so onboarding has to happen after
deployment, not during it (see below).

**Plan note:** Blueprint defaults to `starter` ($7/mo — see
[render.com/pricing](https://render.com/pricing) for current numbers), which
is what real SSH client access needs anyway (Free tier web services don't
support it, full stop — see `render.com/docs/ssh`). Free tier also can't
attach the persistent disk, so WhatsApp linkage and config would reset on
every redeploy; only use `plan: free` (and delete the `disk:` block, Render
rejects the combination) for a throwaway test.

## After it's deployed

### Get a shell

Two options, different capability:

- **Dashboard → your service → Shell** — a browser terminal, works
  immediately, no extra setup on your end.
- **Real `ssh` client** — add your SSH public key in the Render account
  settings first, then `render ssh <service-name>` (needs the
  [Render CLI](https://render.com/docs/cli)) or the connection string Render
  shows you. This is what the Dockerfile patch above enables.

Either shell drops you into the running container; use whichever's handy for
the two steps below.

### Auth with your Claude Code subscription (not an API key)

You said you're using your Claude Code CLI login, not a pay-per-token
`ANTHROPIC_API_KEY` — that's already how this is set up; there's no API key
anywhere in `render.yaml`. On your own machine (wherever Claude Code is
already logged into your subscription):

```bash
claude setup-token
```

That prints a token tied to your subscription login. Then, in the Render
shell:

```bash
openclaw models auth paste-token --provider anthropic
# paste the token from `claude setup-token` when prompted
```

### Link WhatsApp

WhatsApp pairing is QR-only and the code expires fast, so do this from a live
shell session (either option above), not a screenshot:

```bash
openclaw channels login --channel whatsapp --account main
```

Scan with WhatsApp → Linked Devices. Credentials land under
`/data/.openclaw/credentials/whatsapp/main/creds.json` — on the persistent
disk, so it survives redeploys.

### Using it

Control UI: `https://<your-service>.onrender.com/`, log in with
`OPENCLAW_GATEWAY_TOKEN` (Dashboard → Environment). Message the linked
WhatsApp number to talk to the agent.

### Backups

```bash
openclaw backup create
```

from the Render shell — worth doing before any risky config change, per the
official docs' backup/restore flow.

## Sources

- [OpenClaw on Render (official guide)](https://docs.openclaw.ai/install/render)
- [OpenClaw repo](https://github.com/openclaw/openclaw)
- [Render SSH docs](https://render.com/docs/ssh)
- [Render pricing](https://render.com/pricing)
