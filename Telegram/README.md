# Maskarada Community Bot — Telegram

## Overview

This directory contains the specification and (future) code for the Maskarada community Telegram bot. The bot manages member applications, event announcements, safety reporting, and basic community tools.

## Status: SPECIFICATION ONLY

No code exists yet. This README and `bot_spec.md` define what needs to be built.

## MVP Scope

The bot has 4 core commands for v1:

| Command | Purpose |
|---|---|
| `/apply` | Start membership application process |
| `/events` | List upcoming events |
| `/rules` | Display community rules |
| `/report` | Submit confidential incident report |

Everything else (token tracking, automated vetting, matchmaking, etc.) is deferred to v2+.

## Tech Stack (Recommended)

- **Language:** Python 3.10+ (python-telegram-bot library)
- **Database:** SQLite for MVP (migrate to PostgreSQL if >500 members)
- **Hosting:** Small VPS (DigitalOcean, Hetzner, or Contabo) — ~$5-10/month
- **Deployment:** Docker container for portability

## Budget

- Development: Gs. 15,000,000 (~$2,027 USD) — contract to Python developer
- Monthly hosting: Gs. 75,000 (~$10/month)
- Estimated timeline: 4-6 weeks for MVP

## Files

- `README.md` — This file
- `bot_spec.md` — Detailed technical specification
