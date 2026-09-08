# shalom-api-skill

[![skills.sh](https://skills.sh/b/ronnaldrangel/shalom-api-skill)](https://skills.sh/ronnaldrangel/shalom-api-skill)

Skill de IA (formato `SKILL.md`, estándar de Agent Skills) que enseña a cualquier agente — Claude Code, Cursor, Codex, OpenCode, VS Code — a integrar **Shalom API Perú**: rastrear envíos, buscar agencias con coordenadas, cotizar tarifas, validar DNI, crear guías en Shalom Pro y verificar webhooks firmados.

> Plataforma independiente compatible con Shalom Pro. No está afiliada a Shalom Empresarial S.A.C.

## Instalación

```bash
npx skills add ronnaldrangel/shalom-api-skill
```

### Manual (cualquier agente)

```bash
mkdir -p .agents/skills/shalom-api
curl -o .agents/skills/shalom-api/SKILL.md https://shalom-api.lat/skill/SKILL.md
```

Tras instalar, inicia una nueva sesión del agente y pruébalo:

> "Usa la skill shalom-api: rastrea la guía 66479331 con código 3KTH y dime en qué estado está."

## Qué incluye el SKILL.md

- **Autenticación** con header `x-api-key` y validación contra `GET /validate`
- **Endpoints esenciales** con request/response reales (tracking, agencias, ubicaciones, envíos Shalom Pro, cotizaciones, webhooks)
- **5 recetas paso a paso**: rastrear guía · agencia más cercana (`near=lat,lng`) · cotizar · crear envío Pro (instancias) · verificar firma HMAC de webhook
- **Manejo de errores** (400/401/403/404/429/500) con estrategias de reintento
- **Reglas anti-alucinación**: no inventar endpoints, límites de formato (orderNumber 8 dígitos, orderCode 4 caracteres, DNI 8 dígitos), rate limit 1000 req/min

## Publicación y actualización

Este archivo está sincronizado con la versión servida en producción
(`https://shalom-api.lat/skill/SKILL.md`), que es la fuente canónica generada
desde `shalom-api/src/lib/skill.ts`.

```bash
# Publicar (primera vez) — desde esta carpeta
gh repo create ronnaldrangel/shalom-api-skill --public --source . --push \
  --description "AI skill: integra Shalom Perú por API — tracking, agencias, guías Shalom Pro, webhooks"

# Sembrar el listado en skills.sh (primer install = indexación)
npx skills add ronnaldrangel/shalom-api-skill --agent claude-code -y

# Verificar el listado (visible ~30-60 s después)
# https://skills.sh/ronnaldrangel/shalom-api-skill
```

> Reemplaza `ronnaldrangel` por tu usuario de GitHub en el badge y los comandos
> una vez creado el repo.

## Fuentes

- Documentación pública: https://shalom-api.lat/docs
- SKILL.md servido en vivo: https://shalom-api.lat/skill/SKILL.md
- Cómo instalar la skill: https://shalom-api.lat/docs/skill
- Directorio de agencias: https://shalom-api.lat/agencias

## Licencia

MIT — ver [LICENSE](LICENSE).
