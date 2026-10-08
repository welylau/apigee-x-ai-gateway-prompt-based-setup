# Apigee X AI Gateway Labs

Turn your Apigee X environment into an **AI gateway** for LLM traffic. An AI coding agent writes the proxy, and you deploy and test it.

Your first goal is one endpoint that:

- routes callers to different models by a simple alias (`"model": "fast"` → the real backend model)
- keeps your backend API key in Apigee, so callers only ever use their own Apigee key

Then add controls one lab at a time: token quotas, spending budgets and Model Armor inspection.

> This repo is an **agent skill** (`apigee-x-ai-gateway`, version `1.0.0`). It isn't a ready-made proxy. Your agent reads it and generates a proxy bundle for your setup.

---

## How it works

```
 you ──prompt──▶ agent + skill ──writes──▶ proxy ZIP + setup steps + test plan
                                                   │
 you ◀──test (curl)── Apigee X ◀──import & deploy──┘   (you run every cloud command)
```

**Ground rules** (the agent follows these too):

1. The agent **writes files only**. You run every cloud command.
2. **Never paste secrets** into the chat. Keys stay in Apigee (for example, in an encrypted KVM).
3. Keep generated bundles and test notes **outside this repo**.
4. **Review what the agent generates.** Agent output is not automatically correct.

---

## What you need

- [ ] An **Apigee X org with a test environment**. An [evaluation org](https://docs.cloud.google.com/apigee/docs/api-platform/get-started/eval-orgs) works. The org itself is free, but model calls and later labs' services (Model Armor, Vertex AI) are billed separately.
- [ ] `gcloud`, signed in with rights to import and deploy proxies. Later steps also need rights to create KVM entries, API products and apps, and in Lab 4, Model Armor resources and IAM grants. Deploy rights alone may not cover these.
- [ ] An **OpenAI-compatible chat completions endpoint** (non-streaming), its API key, and **two model IDs** it serves
- [ ] Any AI coding agent that supports skills (a folder with a `SKILL.md` plus supporting files) and can read and write local files
- [ ] A basic idea of [API products](https://docs.cloud.google.com/apigee/docs/api-platform/publish/what-api-product): they control which apps can call your proxy, and later hold your quota limits

You don't need to know Apigee policies. The agent explains each decision it needs from you.

---

## Quickstart: your first routed call

### 1. Install the two skills

The gateway skill builds on a general proxy-development skill. Install both into the skills folder your agent loads from. Check your agent's documentation for its location, and set `SKILLS` to it:

```bash
SKILLS=/path/to/your/agent/skills
mkdir -p "$SKILLS"

# This skill
git clone https://github.com/rajeevramani/apigee-x-ai-gateway.git "$SKILLS/apigee-x-ai-gateway"

# Required dependency, pinned to the tested commit
git clone https://github.com/carlosmscabral/cabral-skills.git /tmp/cabral-skills
git -C /tmp/cabral-skills checkout f1171671874444b42d5a6d7a0e4ac63e80db2fe8
cp -R /tmp/cabral-skills/skills/apigee-x-proxy-development "$SKILLS/"
```

Keep each skill's `SKILL.md`, `references/` and `assets/` together. Start a **new** agent session so it picks up the skills. Optional: run [Lab 0](labs/lab-0-readiness.md) to check the agent found both skills before you build anything.

### 2. Store your backend key in Apigee

Put the backend API key in an **encrypted environment KVM** and note the map and entry names exactly. If you don't have one yet, ask the agent in step 3 and it will give you the commands.

### 3. Ask the agent to build it

```text
Help me build model routing on my Apigee X gateway.

Use proxy name ai-gateway, base path /ai-gateway and two caller
aliases, fast and smart, each mapped to one of my approved backend
models. Callers authenticate with x-api-key.

Ask me for the other details you need and explain any decisions I
have to make.

Save the source bundle and an importable ZIP in ./ai-gateway/lab-1,
with setup steps for me and a small manual test plan. Keep test calls
cheap: short "hello" messages, few calls and a small output limit.

I will do all cloud setup, deployment and testing myself. Don't access
my cloud account and never ask me to paste secrets.
```

The agent will ask for your project ID, environment, backend URL, the exact model IDs and your KVM names. Prefer to fill these in up front? Copy [`lab-config.example.md`](lab-config.example.md) and give the agent its path.

**You get:** proxy source, an importable ZIP, API product and app setup steps, and a short test plan.

### 4. Deploy

Follow the agent's setup steps. They cover creating the API product and developer app that give you a caller API key. Then import and deploy the ZIP. Typically:

```bash
ORG=your-project-id  ENV=your-environment  PROXY=ai-gateway
ZIP=ai-gateway/lab-1/<zip-name-from-handoff>.zip

# Import. The response includes the new revision number.
curl -s -X POST \
  "https://apigee.googleapis.com/v1/organizations/$ORG/apis?name=$PROXY&action=import" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -F "file=@$ZIP"

# Deploy that revision
gcloud apigee apis deploy REVISION --api=$PROXY --environment=$ENV --organization=$ORG
```

Use the commands generated for your configuration. **Stop if they conflict with what you agreed** (names, paths, environment) and ask the agent why.

### 5. Smoke test

Fill these in from your environment and the agent's handoff:

```bash
HOST=your-env-group-hostname   BASE=/ai-gateway   KEY=your-app-api-key

call() { curl -s -w '\nHTTP %{http_code}\n' "https://$HOST$BASE/ai-gateway/chat/completions" \
  -H "Content-Type: application/json" -H "x-api-key: $1" \
  -d "{\"model\":\"$2\",\"messages\":[{\"role\":\"user\",\"content\":\"hello\"}],\"max_tokens\":20}"; }

call "$KEY" fast        # expect: a normal completion
call "$KEY" smart       # expect: a normal completion
call "$KEY" gpt-9000    # expect: an error for the unknown alias, not a fallback to another model
call "bad-key" fast     # expect: a caller-authentication error
```

Exact status codes and error bodies follow the contract you agreed with the agent.

**All four behave as expected? Your smoke tests pass.** These are quick checks. For example, a response's `model` field hints at routing but doesn't confirm it. The [routing acceptance cases](references/model-routing.md#acceptance-cases) cover the full checks. Keep the `lab-1` folder, because the next lab builds on it.

Something off? See [Troubleshooting](#troubleshooting).

---

## Next: add controls, one lab at a time

Each lab extends the gateway you already have. Stay in the **same agent conversation**, paste the lab's short prompt, deploy, smoke test, then move on. You can stop after any lab.

For each new lab, the agent needs the previous lab's source folder, deployed revision and test results. Your configuration file alone isn't the previous implementation.

| Lab | Adds | Try it by… | Extra GCP setup |
|---|---|---|---|
| 0 *(optional)* | [Readiness check](labs/lab-0-readiness.md) | asking the agent what it can do | None |
| 1 | [Model routing](labs/lab-1-routing.md) | calling two aliases | Caller API key |
| 2 | [Token quota](labs/lab-2-token-quota.md) | using up a small token limit → local rejection | Quota set on the API product |
| 3 | [Spending budget](labs/lab-3-spending-budget.md) | using up a small budget at synthetic test prices | Prices set on the API product |
| 4 | [Model Armor](labs/lab-4-model-armor.md) | agreed synthetic test cases for your template's enabled filters, on both prompts and responses | Template + service account ([setup below](#model-armor-setup-lab-4)) |
| 5 *(optional)* | [Burst protection](labs/lab-5-burst-protection.md) | a bounded burst of prompt tokens → rate-limit rejection | None |
| 6 *(optional)* | [FAQ semantic cache](labs/lab-6-semantic-cache.md) | a repeat or reworded FAQ question → served from cache | Vertex AI Vector Search |

After each lab, tell the agent how it went so it can check the next lab builds on a sound base:

```text
I tested Lab N (revision R). Passed: … Failed: …
Record the results. Don't fix anything yet. Can the next lab build on this?
```

**Already comfortable?** Build Labs 1–4 in one go with the [combined prompt](labs/combined-build.md). Do the Model Armor setup first.

<a id="model-armor-setup-lab-4"></a>
<details>
<summary><strong>Model Armor setup (Lab 4 and the combined build)</strong></summary>

Run on your own workstation, not in the agent's environment. These commands create resources and grant IAM. Model Armor adds a processing location and usage charges. Reuse suitable existing resources instead of recreating them. Full detail: [Model Armor setup](references/model-armor-setup.md).

**1. Set values and check access.** Choose the location after checking [regional filter availability](https://docs.cloud.google.com/model-armor/feature-availability-by-region) and your data-processing requirements.

```bash
PROJECT_ID="__APIGEE_PROJECT_ID__"
LOCATION="__APPROVED_LOCATION__"
TEMPLATE_ID="__TEMPLATE_ID__"
SA_ID="__DEPLOYMENT_SERVICE_ACCOUNT_ID__"
DEPLOYER_MEMBER="user:__DEPLOYER_EMAIL__"   # or serviceAccount:EMAIL

SA_EMAIL="${SA_ID}@${PROJECT_ID}.iam.gserviceaccount.com"
PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"

# Per-command regional routing; no persistent gcloud configuration change.
armor() {
  CLOUDSDK_API_ENDPOINT_OVERRIDES_MODELARMOR="https://modelarmor.${LOCATION}.rep.googleapis.com/" \
    gcloud model-armor "$@" --project="$PROJECT_ID" --location="$LOCATION"
}

gcloud auth list --filter=status:ACTIVE --format='value(account)'
gcloud model-armor templates create --help
gcloud services list --enabled --project="$PROJECT_ID" \
  --filter='config.name:modelarmor.googleapis.com' --format='value(config.name)'
```

Stop if anything fails or a value is still a placeholder. Don't grant Owner/Editor to get past an error.

**2. Enable the API (if disabled) and check what exists.**

```bash
# Only if the check above showed the API is disabled
gcloud services enable modelarmor.googleapis.com --project="$PROJECT_ID"

armor templates list
armor templates describe "$TEMPLATE_ID" --format=json
gcloud iam service-accounts describe "$SA_EMAIL" --project="$PROJECT_ID"
gcloud projects get-iam-policy "$PROJECT_ID" --format=json
```

`NOT_FOUND` means create it. `PERMISSION_DENIED` means fix access first. If a resource already exists, check that it suits you and who else uses it, then skip creating it and don't overwrite its settings.

**3. Create the template (if missing).** These are evaluation choices, not recommendations. Confirm each filter is available in your location.

```bash
REQUEST_ID="$(python3 -c 'import uuid; print(uuid.uuid4())')"
armor templates create "$TEMPLATE_ID" \
  --request-id="$REQUEST_ID" \
  --rai-settings-filters='[{"filterType":"HATE_SPEECH","confidenceLevel":"MEDIUM_AND_ABOVE"},{"filterType":"HARASSMENT","confidenceLevel":"MEDIUM_AND_ABOVE"},{"filterType":"SEXUALLY_EXPLICIT","confidenceLevel":"MEDIUM_AND_ABOVE"},{"filterType":"DANGEROUS","confidenceLevel":"MEDIUM_AND_ABOVE"}]' \
  --pi-and-jailbreak-filter-settings-enforcement=enabled \
  --pi-and-jailbreak-filter-settings-confidence-level=medium-and-above \
  --malicious-uri-filter-settings-enforcement=enabled \
  --basic-config-filter-enforcement=enabled \
  --no-template-metadata-ignore-partial-invocation-failures \
  --no-template-metadata-log-operations \
  --no-template-metadata-log-sanitize-operations
```

On a timeout or conflict, describe the template before retrying, and reuse the same `REQUEST_ID` for the retry. Don't change the region or disable a filter just to make creation succeed.

**4. Create the service account (if missing) and grant access.** If a deployment identity already exists and suits you, reuse it; changing identity can affect other policies. Never create service-account keys.

```bash
# Only if the account is missing
gcloud iam service-accounts create "$SA_ID" \
  --project="$PROJECT_ID" --display-name="Apigee Model Armor proxy"

# Check existing bindings before adding grants
gcloud iam service-accounts get-iam-policy "$SA_EMAIL" \
  --project="$PROJECT_ID" --format=json
```

These are **unconditional evaluation grants**, so prefer a dedicated account. Run only the grants that are missing. If your organization requires conditional access, use administrator-approved conditions instead of `--condition=None`.

```bash

# Runtime: the proxy's service account can call Model Armor
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SA_EMAIL" --role=roles/modelarmor.user --condition=None
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:$SA_EMAIL" --role=roles/modelarmor.viewer --condition=None

# Deployment: you and the Apigee service agent can use this account
gcloud iam service-accounts add-iam-policy-binding "$SA_EMAIL" --project="$PROJECT_ID" \
  --member="$DEPLOYER_MEMBER" --role=roles/iam.serviceAccountUser --condition=None
gcloud iam service-accounts add-iam-policy-binding "$SA_EMAIL" --project="$PROJECT_ID" \
  --member="serviceAccount:service-${PROJECT_NUMBER}@gcp-sa-apigee.iam.gserviceaccount.com" \
  --role=roles/iam.serviceAccountTokenCreator --condition=None
```

Import permission is separate from these grants. The deployer needs `iam.serviceAccounts.actAs` on this account, and the Apigee service agent needs token-creation access.

**5. Read back and give the agent the results.** Run from a private directory, and don't overwrite an earlier readback you want to keep.

```bash
armor templates describe "$TEMPLATE_ID" --format=json > model-armor-template.json
gcloud iam service-accounts describe "$SA_EMAIL" --project="$PROJECT_ID"
gcloud projects get-iam-policy "$PROJECT_ID" --format=json
gcloud iam service-accounts get-iam-policy "$SA_EMAIL" \
  --project="$PROJECT_ID" --format=json
gcloud services list --enabled --project="$PROJECT_ID" \
  --filter='config.name:modelarmor.googleapis.com' --format='value(config.name)'
printf 'Deployment service account: %s\n' "$SA_EMAIL"
```

Check that:

- the account is enabled
- the grants apply to the intended principals
- the template's filters, thresholds and logging match your choices

Effective IAM and runtime inspection are confirmed later, in the lab tests. If `describe` fails, don't hand the agent an empty or error file as the readback.

Add the template's full `name` and the service-account email to your private config file. Give the agent `model-armor-template.json` so it uses your actual filters. When deploying, **select this service account** for the revision; importing the ZIP doesn't attach it.

</details>

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Backend `404` | Path is duplicated in the target URL. Ask the agent to check the TargetEndpoint path. |
| Backend `401/403` | The KVM map or entry name is wrong, or the key isn't populated. Caller auth is separate. |
| Caller auth fails with a valid key | The app's API product doesn't include this proxy or environment. |
| Quota never triggers | You edited a different product than the app uses, or the product cache hasn't refreshed yet (wait a few minutes). |

More detail: [routing](references/model-routing.md) · [API products](references/api-product-setup.md) · [token quota](references/token-quota.md) · [cost budget](references/cost-budget.md) · [Model Armor](references/model-armor.md) · [burst protection](references/prompt-token-limit.md) · [semantic cache](references/semantic-cache.md) · [Vertex setup for caching](references/semantic-cache-vertex-setup.md)

---

## Good to know

- **Not supported yet:** streaming responses, Anthropic-format conversion, and automatic fallback between models.
- **Test prices are synthetic.** They show that enforcement works; they aren't your real bill.
- **FAQ caching is a gated trial.** It stays disabled until the [trial checklist](references/semantic-cache.md#cache-trial-and-enablement-checklist) passes.
- **How far each capability has been tested:** see [feature status](references/capability-status.md#current-status) and [what the statuses mean](references/capability-status.md#what-the-statuses-mean).
- **Evaluating the skill itself** (agent evaluation runs, evidence rules, offline tests): see [EVALUATION.md](EVALUATION.md).

---

## Support and issues

Bugs, feature requests and questions all go in [GitHub Issues](https://github.com/rajeevramani/apigee-x-ai-gateway/issues/new/choose). Pick the matching form.

Issues are public. Before posting, remove API keys, tokens, KVM values, project IDs, organization names and anything from `lab-config.md` or `evidence/`.

Report security problems privately, as described in the [security policy](SECURITY.md).

---

## Licence

Copyright 2026 Rajeev Ramani. Licensed under the [Apache License, Version 2.0](LICENSE).

The required `apigee-x-proxy-development` dependency is a separate project (also Apache-2.0), installed from its own repository.
