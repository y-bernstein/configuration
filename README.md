# Supio Config Service – Configuration Authoring Guide

This guide explains **how to structure configuration**, **how overrides work**, and **how your config is rendered** by the the Supio Config Service .

The goal of this system is to:
- standardize configuration across services and environments
- keep configs **versioned in git**
- eliminate environment-specific template drift
- make overrides explicit, minimal, and reviewable

---
### 0. Terminology
- **Template** = *a templated configuration file your app needs* (ex: **.env, options.json, config.yaml etc**)
- **Vars file** = *yaml file containing a list of variables that are rendered into the templates*
- **Environment** = *a combination of aws region + env, where your app runs* (ex: **us-dev, us-stg, us-prod, can-prod, au-dev etc**)

### 1. Core Concepts

#### Service = App
In this system, **a service is the application**.  
There is no separate “app vs service” abstraction.

#### Templates are global and immutable per service
- Each service has **one canonical config template**
- Templates **do not vary by environment**
- Environment-specific behavior is achieved **only via variables**

This avoids config drift and keeps config shape consistent.

#### Variables are layered with strict precedence
Variables are defined at different levels and merged in order.
Higher-precedence layers override lower-precedence ones.

global --> global/service --> environment --> environment/service
---

### 2. Repository Structure

[globals/
	vars.yaml                      # global defaults (all services, all envs)

services/
	example/					   # service name
		config.yaml                # Jinja template (YAML / JSON / .env)
		vars.yaml                  # service-wide defaults

environments/
	us-prod/
		vars.yaml                  # environment-wide overrides

		services/
  			example/               # service name
    			vars.yaml          # env-specific service overrides
]

Only `vars.yaml` files vary by environment.  
`config.yaml` exists **only once per service**.

---

### 3. Variable Precedence

Variables are merged in the following order:

1. `globals/vars.yaml`
2. `globals/services/<service>/vars.yaml`
3. `environments/<env>/vars.yaml`
4. `environments/<env>/services/<service>/vars.yaml`


⸻

### 4. Merge Rules (Exact Semantics)

When merging variables:
	•	Dictionaries → merged recursively
	•	Scalars (string, int, bool, null) → overridden
	•	Lists → replaced entirely (no append)

**Example**:

**globals/vars.yaml**
`logging:
  level: INFO
  outputs: [stdout]`

**environments/us-prod/vars.yaml**
`logging:
  level: WARN`

**Result:**

`logging:
  level: WARN
  outputs: [stdout]`

⸻

### 5. Writing Templates

Templates are written using Jinja2.

**Example**: `globals/services/example/config.yaml`

`
server:
  port: {{ service.port }}
  log_level: {{ logging.level }}

database:
  host: {{ db.host }}
  user: {{ db.user }}
  password: "{{ secret(db.password_secret) }}"
  `

**Important rules**:
	•	Missing variables fail the render
	•	Templates should be deterministic
	•	Conditional logic should be minimal and explicit

⸻

### 6. Writing Variables

**Global Variables**

Used for defaults shared across all services and environments.

**globals/vars.yaml**
`logging:
  level: INFO

service:
  port: 8080`

**Service Variables**

Defaults for a specific service across all environments.

**globals/services/example/vars.yaml**
`service:
  port: 9000

db:
  host: gateway-db.internal
  user: gateway
  password_secret: /config/example/db_password`


⸻

**Environment Variables**

Overrides shared across all services in one environment.

**environments/us-prod/vars.yaml**
`logging:
  level: WARN

monitoring:
  enabled: true`

⸻

**Environment + Service Variables (Highest Precedence)**

Used when a service has a unique parameter and it is unique in a given environment

**environments/us-prod/services/example/vars.yaml**
`service:
  port: 9443

db:
  host: gateway-db.us-prod.internal
  name: db_only_this_service_uses`


⸻

### 7. Secrets

Secrets are resolved at render time using AWS Secrets Manager.

Template Usage

`password: "{{ secret('/config/us-prod/supio-prod-documents-cluster/db_password') }}"`

Key Points
	•	Secrets are never stored in git
	•	/explain endpoint never returns secret values
	•	Secret values are cached briefly in memory
	•	Use naming conventions for secrets (recommended)

⸻

### 8. DON'T

❌ Do NOT put secrets in vars.yaml
❌ Do NOT create env-specific templates
❌ Do NOT duplicate values across layers without intent
❌ Do NOT override values “just in case”

If a value is the same everywhere → move it up to a higher layer.

⸻

### 9. How Config Is Served

Request format:

`GET /<env>/<service>/<version>`

Example:

`GET /us-prod/gateway/main`

	•	version is a git ref (branch, tag, or commit)
	•	The service resolves the ref to a commit
	•	All files are read atomically from that commit
	•	The rendered config is returned as a baked file

⸻

### 10. Explain Mode (Debugging)

To understand why a config looks the way it does:

`GET /<env>/<service>/<version>/explain`

Returns:
	•	which vars files were loaded
	•	merge order
	•	merged variables (redacted)
	•	which secrets were referenced
	•	rendered preview (redacted)

This endpoint is designed for debugging and PR review.

⸻

### 11. Design Philosophy (Why This Exists)
	•	Config as code (git is the source of truth)
	•	Deterministic builds (commit → config)
	•	No hidden magic
	•	Minimal surface area
	•	Easy to reason about
	•	Lean and fits Supio platform, with no unneccessary overhead	
