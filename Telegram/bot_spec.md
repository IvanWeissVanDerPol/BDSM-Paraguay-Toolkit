# Maskarada Bot — Technical Specification (MVP)

## 1. Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Telegram   │────▶│  Bot Server  │────▶│   SQLite     │
│   Users      │◀────│  (Python)    │◀────│   Database   │
└──────────────┘     └──────────────┘     └──────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │  Admin       │
                     │  Telegram    │
                     │  Group       │
                     └──────────────┘
```

- **Bot framework:** python-telegram-bot v20+
- **Database:** SQLite3 (single file, backups via cron)
- **Hosting:** VPS with Docker (or direct Python service)
- **Admin notifications:** Forward to private admin Telegram group

---

## 2. Database Schema

```sql
CREATE TABLE members (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    telegram_id INTEGER UNIQUE NOT NULL,
    telegram_username TEXT,
    display_name TEXT,
    real_name_encrypted TEXT,  -- AES-256 encrypted
    application_date TEXT,
    status TEXT DEFAULT 'pending',  -- pending, approved, rejected, banned
    approved_date TEXT,
    approved_by TEXT,
    notes TEXT,
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    description TEXT,
    date TEXT NOT NULL,
    time TEXT NOT NULL,
    venue TEXT,
    capacity INTEGER,
    price TEXT,
    event_type TEXT,  -- maskarada, munch, workshop, sexpo
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE reports (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    reporter_telegram_id INTEGER,
    report_text TEXT NOT NULL,
    event_id INTEGER,
    is_anonymous BOOLEAN DEFAULT 1,
    status TEXT DEFAULT 'new',  -- new, reviewing, resolved, archived
    created_at TEXT DEFAULT CURRENT_TIMESTAMP,
    resolved_at TEXT,
    resolved_by TEXT,
    resolution_notes TEXT
);

CREATE TABLE applications (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    telegram_id INTEGER NOT NULL,
    step INTEGER DEFAULT 1,  -- which question they're on
    answers TEXT,  -- JSON blob of collected answers
    status TEXT DEFAULT 'in_progress',  -- in_progress, submitted, reviewed
    submitted_at TEXT,
    reviewed_at TEXT,
    reviewed_by TEXT,
    decision TEXT,  -- approved, rejected
    created_at TEXT DEFAULT CURRENT_TIMESTAMP
);
```

---

## 3. Commands — MVP

### 3.1 /start
**Trigger:** User sends /start or opens bot for first time
**Response:**
```
¡Bienvenido/a a Maskarada! 🎭

Somos la comunidad de bienestar sexual y kink de Paraguay.

Comandos disponibles:
/apply — Solicitar membresía
/events — Ver próximos eventos
/rules — Reglas de la comunidad
/report — Reportar un incidente (confidencial)
/help — Ver todos los comandos

Para más información: @maskarada_events
```

### 3.2 /apply
**Flow:**
1. Check if user already has a pending/approved application → if yes, show status
2. Ask Question 1: "¿Cuál es tu nombre o alias? (Puede ser un pseudónimo)"
3. Ask Question 2: "¿Cómo nos conociste? (Instagram, amigo/a, evento, otro)"
4. Ask Question 3: "¿Qué experiencia tenés con comunidades kink/BDSM? (Ninguna está bien)"
5. Ask Question 4: "¿Por qué querés unirte a Maskarada?"
6. Ask Question 5: "¿Tenés perfil en FetLife u otra comunidad? (Opcional — podés decir 'no')"
7. Confirm: "Gracias por tu solicitud. Un administrador la revisará y te contactará para coordinar un munch (encuentro casual). Esto puede tardar 3-7 días."
8. Forward application summary to admin group
9. Store in applications table

**Admin flow:**
- Admin receives application in admin group
- Admin replies with `/approve [telegram_id]` or `/reject [telegram_id] [reason]`
- Bot notifies applicant of decision
- If approved: add to members table, add to private community group

### 3.3 /events
**Response:** List of next 5 upcoming events from events table
```
📅 Próximos Eventos:

1. Munch Mensual — Sábado 15/03, 19:00
   📍 Bar [nombre] — Entrada libre

2. Maskarada: Noche Roja — Sábado 22/03, 22:00
   📍 [Venue] — Gs. 150.000
   🎟️ Solo miembros

3. Taller: BDSM 101 — Domingo 30/03, 16:00
   📍 [Venue] — Gs. 80.000

/register [número] — Para inscribirte
```

**Admin flow:**
- Admin creates events with `/newevent` command
- Admin edits/cancels with `/editevent [id]` / `/cancelevent [id]`

### 3.4 /rules
**Response:** (static text, editable by admin)
```
📜 Reglas de Maskarada:

1. CONSENTIMIENTO es obligatorio. "No" significa "no". Siempre.
2. PRIVACIDAD es sagrada. No compartir identidades, fotos, ni información de otros miembros.
3. RESPETO a todos los géneros, orientaciones y experiencias.
4. SOBRIEDAD requerida para juegos (play). Alcohol moderado solo en zonas sociales.
5. FOTOGRAFÍA prohibida sin permiso explícito.

Palabras de seguridad:
🔴 ROJO = Parar todo
🟡 AMARILLO = Reducir intensidad
🟢 VERDE = Todo bien

Violaciones = expulsión permanente.
```

### 3.5 /report
**Flow:**
1. Ask: "¿Querés que este reporte sea anónimo? (Sí/No)"
2. Ask: "¿En qué evento ocurrió? (O escribí 'fuera de evento')"
3. Ask: "Describí lo que pasó. Incluí todos los detalles que puedas."
4. Confirm: "Tu reporte fue enviado. Un administrador lo revisará de forma confidencial. Si necesitás ayuda inmediata, contactá a [Safety Director contact]."
5. Forward to admin group (with or without reporter identity based on choice)
6. Store in reports table

---

## 4. Admin Commands

| Command | Access | Action |
|---|---|---|
| `/approve [telegram_id]` | Admin only | Approve membership application |
| `/reject [telegram_id] [reason]` | Admin only | Reject application with reason |
| `/ban [telegram_id] [reason]` | Admin only | Ban member, remove from groups |
| `/newevent` | Admin only | Create new event (interactive flow) |
| `/cancelevent [id]` | Admin only | Cancel event, notify registered attendees |
| `/members` | Admin only | Show member count and recent joins |
| `/reports` | Admin only | Show open reports |
| `/broadcast [message]` | Admin only | Send message to all approved members |

---

## 5. Security Requirements

| Requirement | Implementation |
|---|---|
| Admin authentication | Only Telegram IDs in admin list can use admin commands |
| Data encryption | Real names encrypted with AES-256 in database |
| Report confidentiality | Anonymous reports show no reporter ID to admins |
| Database backups | Automated daily backup via cron → encrypted → cloud storage |
| Access logging | Log all admin actions with timestamp and admin ID |
| Rate limiting | Max 10 commands per minute per user (prevent spam) |

---

## 6. What's NOT in MVP (Deferred to v2+)

| Feature | Why Deferred |
|---|---|
| Token/points tracking | Need to define token economy first; manual tracking works for <100 members |
| Automated event reminders | Manual broadcast is sufficient for Year 1 |
| Event check-in via QR | Physical check-in is sufficient for <100 attendees |
| Member matching/compatibility | Premature — community too small, could be creepy |
| Integration with WooCommerce | Not needed until shop has >200 active customers |
| Multi-language (Spanish/Guarani) | Spanish only for MVP; consider Guarani support in v2 |

---

## 7. Development Plan

| Phase | Timeline | Deliverables |
|---|---|---|
| **Phase 1** | Week 1-2 | Bot skeleton, /start, /rules, /help, database setup |
| **Phase 2** | Week 3-4 | /apply flow + admin approval commands |
| **Phase 3** | Week 5 | /events + /newevent admin commands |
| **Phase 4** | Week 6 | /report flow + admin report management |
| **Testing** | Week 7 | Beta test with 5-10 trusted community members |
| **Launch** | Week 8 | Deploy to production, announce to community |

**Total estimated development time:** 6-8 weeks
**Estimated cost:** Gs. 15,000,000 (~$2,027 USD) for contracted developer
