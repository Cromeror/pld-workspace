---
type: agent-reference
date: 2026-05-12
status: aplicado
affects: pld-api/apps/auth-users/src/main.ts, pld-api/apps/auth-users/src/users/dto/create-user.dto.ts
---

# 001 — Validación estricta de DTOs + fix regex RFC

**Contexto**: `ValidationPipe` tenía `whitelist: false` haciendo inefectivo `forbidNonWhitelisted`. Campos extra pasaban sin rechazo. Regex de RFC tenía `&amp;` (HTML entity) en lugar de `&`, aceptando caracteres inválidos.

**Decisión**: `whitelist: true` + corregir regex a `[A-ZÑ&]`. Verificado con casos reales incluyendo RFCs con `&` (válidos según SAT).

**Consecuencia**: requests con campos no declarados en el DTO ahora reciben 400.
