---
title: Incident Response OpenEnv
emoji: 🛡️
colorFrom: blue
colorTo: blue
sdk: docker
app_port: 7860
tags:
  - openenv
  - reinforcement-learning
  - sre
short_description: AI-powered SRE incident triage environment by Team BitShift
---

# Incident Response OpenEnv

A real-world OpenEnv environment where an AI agent acts as an on-call SRE — triaging system alerts by classifying severity, routing to the correct team, and drafting status updates. Built by **Team BitShift** for the OpenEnv Hackathon.

## What Makes This Different

Most environments have 2-3 tasks. This one has **8 tasks**, **14 unique alerts**, **3 extreme challenge modes**, **daily challenges**, **AI-powered explainers**, and a full product-grade web UI — all wired to a proper RL reward function.

## Environment Description

Incident response is a high-stakes task every SRE performs daily: read an alert, decide how critical it is, page the right team, and communicate status. This environment formalizes that as a reinforcement learning problem with partial-credit rewards, urgency decay, persona overrides, noise injection, and adversarial false positives.

## Action Space

```python
Action(
    severity: Literal["P1", "P2", "P3", "P4"],
    incident_type: Literal["database", "network", "security", "application", "infrastructure"],
    team: Literal["backend", "infra", "security", "database", "frontend"],
    status_update: Optional[str]        # required for "hard" task
    is_false_positive: Optional[bool]   # required for "adversarial" task
)
```

## Observation Space

```python
Observation(
    alert: Alert(id, title, source, body, timestamp),
    step: int,
    total_steps: int,
    task_id: str,
    persona: Optional[str],
    cascading_context: Optional[str]   # hint when alerts are causally related
)
```

## Tasks

| Task               | Alerts | Graded Fields                    | Difficulty |
|--------------------|--------|----------------------------------|------------|
| easy               | 5      | severity                         | ⭐         |
| medium             | 8      | severity + team                  | ⭐⭐       |
| hard               | 10     | severity + team + status_update  | ⭐⭐⭐     |
| adversarial        | 4      | severity + false_positive        | ⭐⭐⭐⭐   |
| persona_startup    | 7      | severity + team                  | ⭐⭐⭐     |
| persona_enterprise | 7      | severity + team                  | ⭐⭐⭐     |
| noisy_easy         | 5      | severity + team (30% noise)      | ⭐⭐⭐     |
| noisy_hard         | 10     | severity + team (70% noise)      | ⭐⭐⭐⭐⭐ |

## Reward Function

- **Severity**: 1.0 (exact), 0.5 (off by one P-level), 0.0 (wrong)
- **Team routing**: 1.0 (exact), 0.0 (wrong)
- **Status update**: keyword coverage score (0.0–1.0)
- **False positive**: 1.0 (correct detection), 0.0 (wrong); penalizes false alarms on real alerts
- **Urgency decay**: P1 alerts triaged late lose reward linearly (0.15 per step after deadline)

Final reward = mean across graded fields per step, averaged over the episode (range: 0.0–1.0).

## Unique Features

### Creative Environment Mechanics
- **Urgency decay** — P1s triaged late lose reward. Every second counts.
- **Cascading context** — alerts hint that they're causally related (DB outage → payment failure)
- **Persona system** — same alert has different expected severity for startup vs enterprise SRE
- **Noise injection** — 30–70% word corruption tests robustness to garbled logs
- **Adversarial false positives** — alerts disguised as real incidents require `is_false_positive=True`

### Challenge Modes (Unique)
- **⚡ Blitz Mode** — 10 alerts, 60 seconds, score × speed multiplier (2× if done in 30s)
- **🔴 Blackout Mode** — alert body hidden, triage on title + source only
- **████ Redacted Mode** — key words blacked out like a classified document

### AI-Powered Features (Unique)
- **Alert DNA Fingerprinting** (`POST /fingerprint`) — auto-suggests severity + team from alert text with confidence scores
- **SRE Mentor Explainer** (`GET /explain/sre`) — after wrong answers, explains what a senior SRE would do and why
- **Severity Drift Detector** (`GET /drift`) — detects if you're systematically over/under-escalating
- **Confidence Calibration** (`POST /step/confident`, `GET /calibration`) — tracks if high-confidence decisions are actually more accurate

### Daily Challenge (Unique)
- **`GET /daily/challenge`** — same 5 alerts for everyone today, seeded by date (like Wordle for SREs)
- **`GET /daily/leaderboard`** — global daily rankings, resets at midnight UTC

### Analytics & Research
- **`GET /autopsy`** — full post-mortem: per-mistake breakdown with corrective advice
- **`GET /heatmap`** — confusion matrix for severity and team routing mistakes
- **`GET /compare`** — side-by-side session comparison (human vs AI, or two models)
- **`GET /skills`** — skill profile by alert source, severity level, and incident type
- **`POST /step/timed`** — time-to-triage scoring with speed bonus/penalty
- **`GET /feed`** — SSE live alert stream like a real PagerDuty feed

## API Endpoints

```
GET  /                    Web UI
POST /reset               Start episode
POST /step                Submit triage action
GET  /tasks               List all tasks
GET  /hint                Hint for current alert
GET  /explain             Explain last action
GET  /streak              Current/best streak
GET  /timeline            Full episode replay
GET  /difficulty          Per-alert difficulty scores
GET  /analytics           Accuracy stats by team/severity
GET  /autopsy             Post-mortem analysis
GET  /heatmap             Confusion matrix
GET  /compare             Session comparison
GET  /skills              Skill profile
GET  /drift               Severity bias detector
GET  /calibration         Confidence calibration
POST /fingerprint         Alert DNA auto-suggest
GET  /explain/sre         Senior SRE mentor explainer
GET  /daily/challenge     Today's daily challenge
POST /daily/reset         Start daily challenge
GET  /daily/leaderboard   Daily rankings
POST /step/timed          Timed triage with speed bonus
POST /blitz/reset         Start Blitz Mode
POST /blitz/step          Blitz Mode step
POST /blackout/reset      Start Blackout Mode
POST /blackout/step       Blackout Mode step
POST /redacted/reset      Start Redacted Mode
POST /redacted/step       Redacted Mode step
POST /benchmark           Composite benchmark score
POST /replay              Deterministic episode replay
GET  /leaderboard         Global leaderboard
POST /leaderboard         Save score
GET  /session/export      Export session as JSON
GET  /health              Service status
```

## Setup

```bash
pip install -r requirements.txt
pip install -e .
```

## Usage

```python
from env import IncidentResponseEnv
from env.models import Action

env = IncidentResponseEnv()
obs = env.reset("medium")

done = False
while not done:
    action = Action(severity="P1", incident_type="database", team="database")
    next_obs, reward, done, info = env.step(action)
    print(reward.value, reward.breakdown)
    if not done:
        obs = next_obs

print(env.state())
```

## Validate

```bash
openenv validate
# or
python openenv_cli.py validate
```

## Baseline Inference

Required env vars:
- `API_BASE_URL` — LLM API endpoint (default: `https://router.huggingface.co/v1`)
- `MODEL_NAME` — model identifier (default: `Qwen/Qwen2.5-72B-Instruct`)
- `HF_TOKEN` — your Hugging Face / API key

```bash
HF_TOKEN=hf_... python inference.py
```

Baseline scores (Qwen2.5-72B-Instruct, temperature=0):

| Task   | Mean Reward |
|--------|-------------|
| easy   | 0.880       |
| medium | 0.750       |
| hard   | 0.640       |

## Docker

```bash
docker build -t incident-response-env .

# Run HTTP server (default — for HF Space)
docker run --rm -p 7860:7860 incident-response-env

# Run openenv validate
docker run --rm incident-response-env openenv validate

# Run inference
docker run --rm \
  -e API_BASE_URL=https://router.huggingface.co/v1 \
  -e MODEL_NAME=Qwen/Qwen2.5-72B-Instruct \
  -e HF_TOKEN=hf_... \
  incident-response-env python inference.py
```

## Hugging Face Spaces

Live demo: https://huggingface.co/spaces/aadya4517/incident-response-env

Tag: `openenv`  
Deploy by pushing this repo to a HF Space with Docker SDK selected. The server starts on port 7860.
