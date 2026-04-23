# Continuous Authentication + MFA MVP

This project implements an MVP for continuous authentication in enterprise web applications using:

- Behavioral biometrics (`keystroke dynamics`, `mouse behavior`)
- Risk scoring (baseline + context risk)
- Adaptive MFA (WebAuthn primary, TOTP fallback)
- Shadow-mode rollout support and tuning metrics
- PostgreSQL + Redis persistence

## What Is Included

- Event schema with explicit PII exclusions:
  - no raw key values
  - no plaintext IP/user-agent (hashes only)
- Feature extraction on rolling behavioral samples
- Risk score from `0-100`
- Policy engine:
  - `>= 80`: allow
  - `50-79`: step-up MFA
  - `< 50`: lock/reauth
- Enrollment profile with drift-safe decay updates
- Shadow mode override with decision metrics
- Real WebAuthn registration/authentication endpoints via `@simplewebauthn/server`

## Quick Start

1. Install dependencies:

```bash
npm install
```

2. Copy environment file:

```bash
cp .env.example .env
```

3. Start development server:

```bash
npm run dev
```

Server runs on `http://localhost:4000` by default.

## API Endpoints

- `GET /health`
- `POST /api/ingestion/events`
- `POST /api/auth/mfa/start`
- `POST /api/auth/mfa/verify`
- `POST /api/auth/webauthn/register/options`
- `POST /api/auth/webauthn/register/verify`
- `POST /api/auth/webauthn/auth/options`
- `POST /api/auth/webauthn/auth/verify`
- `GET /api/shadow/metrics`

## Persistence

- If both `DATABASE_URL` and `REDIS_URL` are present, profile and credential persistence is enabled.
- If either is missing or unavailable, the app falls back to in-memory mode.
- PostgreSQL tables are auto-created on startup:
  - `enrollment_profiles`
  - `webauthn_credentials`

## Run With Docker

```bash
docker compose up --build
```

Services started:
- API: `http://localhost:4000`
- PostgreSQL: `localhost:5432`
- Redis: `localhost:6379`

## CI

GitHub Actions workflow at `.github/workflows/ci.yml` runs:
- `npm ci`
- `npm run lint`
- `npm run build`
- `docker build`

## Example Ingestion Payload

```json
{
  "context": {
    "userId": "user_123",
    "sessionId": "sess_1234567890",
    "eventType": "periodic_sample",
    "route": "/dashboard",
    "ipAddressHash": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    "userAgentHash": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
    "deviceFingerprint": "device_fingerprint_1",
    "timestamp": 1713850000000
  },
  "signals": [
    {
      "timestamp": 1713850000100,
      "holdTimeMs": 120,
      "flightTimeMs": 95,
      "typoRate": 0.04,
      "backspaceRate": 0.06,
      "mouseVelocity": 800,
      "mouseAcceleration": 2600,
      "clickCadence": 2.2,
      "movementEntropy": 3.4
    }
  ],
  "metadata": {
    "sdkVersion": "1.0.0",
    "nonce": "n_123456789abc",
    "signature": "sig_1234567890abcdef"
  }
}
```

## Shadow Mode Rollout

- Keep `SHADOW_MODE=true` during pilot.
- System computes risk/policy but returns allow decisions.
- Observe `GET /api/shadow/metrics` to tune thresholds.
- Move to `SHADOW_MODE=false` once lock/step-up rates are acceptable.
