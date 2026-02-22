# ChittyXL Charter

## Classification
- **Canonical URI**: `chittycanon://core/services/chittyxl`
- **Tier**: 5 (Application)
- **Organization**: chittyapps
- **Domain**: chittyxl.chitty.cc

## Mission

Extended session and token management service for the ChittyOS ecosystem.

## Scope

### IS Responsible For
- Session persistence, token tracking, checkpoint management

### IS NOT Responsible For
- Identity generation (ChittyID)
- Token provisioning (ChittyAuth)

## Dependencies

| Type | Service | Purpose |
|------|---------|---------|
| Upstream | ChittyAuth | Authentication |

## API Contract

**Base URL**: https://chittyxl.chitty.cc

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Service health |

## Ownership

| Role | Owner |
|------|-------|
| Service Owner | chittyapps |

## Compliance

- [ ] Registered in ChittyRegister
- [ ] Health endpoint operational at /health
- [ ] CLAUDE.md present
- [ ] CHARTER.md present
- [ ] CHITTY.md present

---
*Charter Version: 1.0.0 | Last Updated: 2026-02-21*