# Registering `capability-abstraction` v4 on Dockerized MISP (misp.counterintel.cz)

This updates the already-registered `capability-abstraction` object template (id 475 as of
2026-09-15 — MISP reassigns the numeric id on some reloads, always re-resolve it by `uuid`
rather than trusting a hardcoded id; see step 6),
uuid `9f2e6b8a-7c1d-4e3a-9b5f-1a2c3d4e5f60`) from v3 to **v4**, which adds two new required-in-practice
fixed-vocabulary attributes (not carried on any v2 event yet — re-register before the next
`--push` run so these two land in MISP instead of silently being new text the instance has
never seen before):

- `abstraction-layer` — one of: Tool/Binary, API/Export, COM/IPC, Registry Object, Protocol,
  Process Behavior, Network Behavior (SpecterOps "On Detection" / capability-abstraction
  mapping layer)
- `robustness-level` — one of: Ephemeral, Implementation / attacker-controlled,
  System-constrained interaction, Low-variance behavior, Invariant / core to technique
  (Center for Threat-Informed Defense "Summiting the Pyramid" robustness level)

The uuid is unchanged, so this is an in-place template update, not a new template. Same
procedure as the v1->v2 update below, with `version: 3` in step 6's expected output and
`abstraction-layer`/`robustness-level` in step 6's attribute-presence check instead of
`detection-query`/`detection-query-language` (those are still present from v2, unaffected).

## 1. Find the MISP compose project and core container

```bash
ssh user@misp-ip

# Locate the compose project directory (adjust if you already know it)
docker compose ls
# or, if that's empty / you're not sure which directory:
find / -maxdepth 4 -iname 'docker-compose*.yml' 2>/dev/null | grep -i misp

cd <path-to-misp-compose-dir>
docker compose ps
```

Identify the core app service/container — typically named `misp-core`, `misp_core`, or
`misp-server` depending on which compose project this deployment uses
(`coolacid/misp-docker` vs the official `MISP/misp-docker` name their services differently).
The rest of this doc uses `<misp-core-container>` as a placeholder — substitute the real
container name from `docker compose ps`.

## 2. Confirm the objects directory inside the container

```bash
docker exec -it <misp-core-container> ls /var/www/MISP/app/files/misp-objects/objects | grep capability-abstraction
```

Should print `capability-abstraction` — confirms v1 is already there from the original
registration and you're updating in place.

## 3. Copy the updated v2 definition.json into the container

From Your PC, Mac, with the updated file at
`/Your/File/Location/capability-abstraction/definition.json`
(now `"version": 4`, with `detection-query` / `detection-query-language` added):

```bash
# From Your PC, Mac: copy just the definition.json to the SSH host
scp /Your/File/Location/capability-abstraction/definition.json \
    user@misp-ip:/tmp/capability-abstraction-definition.json

# On the SSH host: copy from host into the running container, overwriting the v1 file
ssh user@misp-ip
docker cp /tmp/capability-abstraction-definition.json \
    <misp-core-container>:/var/www/MISP/app/files/misp-objects/objects/capability-abstraction/definition.json
```

## 4. Fix ownership/permissions (MISP runs as `www-data` inside the container)

```bash
docker exec -it <misp-core-container> chown www-data:www-data \
    /var/www/MISP/app/files/misp-objects/objects/capability-abstraction/definition.json
```

## 5. Reload object templates

Either via the web UI: **Administration → List Object Templates → "Update Object Templates"**,
or via the API (same call already confirmed live against this instance for the v1 registration):

```bash
curl -s -X POST "https://yourmisp.here.local/objectTemplates/update" \
  -H "Authorization: <your-api-key>" -H "Accept: application/json"
```

## 6. Verify the version bumped to 2 with the new fields present

```bash
curl -s "https://yourmisp.here.local/objectTemplates" \
  -H "Authorization: <your-api-key>" -H "Accept: application/json" \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
for t in d:
    tpl = t.get('ObjectTemplate', {})
    if tpl.get('name') == 'capability-abstraction':
        print('id:', tpl.get('id'), '| version:', tpl.get('version'), '| uuid:', tpl.get('uuid'))
"
```

Should print `version: 4` (was `3`) and `id: 480` (or whatever id this reload assigned — always
re-resolve it from this listing call). Then confirm the
new attributes registered, substituting the id printed above for `<template-id>`:

```bash
curl -s "https://yourmisp.here.local/objectTemplates/view/<template-id>" \
  -H "Authorization: <your-api-key>" -H "Accept: application/json" \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
elements = d.get('ObjectTemplateElement', [])
names = [e.get('ObjectTemplateElement', {}).get('object_relation') for e in elements]
print('detection-query present:', 'detection-query' in names)
print('detection-query-language present:', 'detection-query-language' in names)
"
```

Both should print `True`. If either is `False`, check the container's MISP error log
(`docker exec -it <misp-core-container> tail -50 /var/www/MISP/app/tmp/logs/error.log`) for a
schema-validation error MISP's own PHP-side loader raised — it can be stricter than the
Python-side `jsonschema` check already run locally against `schema_objects.json`.

## Note on the Git-centric design goal

If the long-term intent is "Git is the source of truth MISP pulls object templates from"
(rather than a manual `docker cp` each time the template changes), the cleaner long-term setup
is:

- Fork/mirror `MISP/misp-objects` (or maintain a small private overlay repo)
- Add `capability-abstraction/definition.json` to it and bump `version` on every future change
- Point this MISP instance's object-template Git remote/update mechanism at that fork
  (server-side config, not something doable via the REST API)
- Every `/objectTemplates/update` then pulls from your fork instead of upstream MISP,
  and every future version bump is a normal `git push` instead of a manual `docker cp`

That is server configuration work outside the JSON file itself — flagging it here rather than
guessing at commands, since it depends on how this specific MISP deployment's object-template
sync is configured (cron pulling from Git vs manual `git pull` vs docker image rebuild).
