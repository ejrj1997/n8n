# n8n on Render (free) + Supabase Postgres

Instancia de [n8n](https://github.com/n8n-io/n8n) desplegada en Render con el
plan **free**, usando **Supabase Postgres** como base de datos persistente.

- **URL de n8n:** https://n8n-service-7gac.onrender.com
- **Base de datos:** Supabase Postgres (project ref `kyemexorbnqdzmvgtclh`)
- **Imagen Docker:** `docker.io/n8nio/n8n:latest` (versión 2.33.7)
- **Despliegue:** Blueprint `render.yaml` sincronizado con la rama `main` de
  este repositorio (autoSync de Render).
- **Región de Render:** `oregon` · **Plan:** `free` · **1 instancia**

## Cómo funciona la persistencia

El filesystem del plan free de Render es **efímero** (se pierde al dormir o
reiniciar el servicio), por lo que n8n guarda todos sus datos —workflows,
credenciales, ejecuciones, cuenta de usuario— en Postgres alojado en
Supabase. La primera vez que n8n arranca, crea sus tablas (`workflow_entity`,
`credentials_entity`, `execution_entity`, etc.) en el schema `public`.

## Configuración importante

- **Transaction Pooler de Supabase (puerto 6543):** obligatorio. El Session
  Pooler (5432) congela las conexiones TCP inactivas a los ~25 s y el ping de
  BD de n8n falla con *"Database connection timed out"* desde el datacenter de
  Render → el servicio entra en un ciclo de reinicio (HTTP 502). El puerto
  6543 recicla las conexiones y es estable.
- **`NODE_OPTIONS=--max-old-space-size=384`:** sin esto, Node usa ~256 MB de
  heap en el contenedor de 512 MB y n8n muere con *"JavaScript heap out of
  memory"*.
- **Credenciales de BD:** la contraseña **no** está en `render.yaml` (el
  repositorio es público). Se aplica como secreto vía la API de Render.

### ⚠️ Después de cada push a este repositorio

El autoSync de Render sobrescribe las variables de entorno del servicio con
los valores de `render.yaml`, y en ese archivo `DB_POSTGRESDB_PASSWORD` es un
**placeholder**. Por tanto, tras cualquier push hay que volver a aplicar la
contraseña real con la API:

```bash
curl -s -X PUT \
  "https://api.render.com/v1/services/srv-d9rbet7avr4c738s1si0/env-vars" \
  -H "Authorization: Bearer <RENDER_API_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"envVars":[{"key":"DB_POSTGRESDB_PASSWORD","value":"<PASSWORD_REAL>"}]}'
```

Esto dispara un redeploy automático con la contraseña correcta.

## Limitaciones del plan free de Render

- El servicio **duerme** tras ~15 min sin tráfico. La primera carga después de
  dormir tarda hasta ~1 min en despertar.
- Los **webhooks entrantes no se ejecutan** mientras el servicio esté dormido.
- **512 MB de RAM / 0.1 vCPU:** si se agotan los recursos, reduce el límite de
  heap de Node o desactiva runners pesados.
- Los **task runners de Python** no están disponibles (Python 3 no está en la
  imagen). Usa los nodos de JavaScript o el modo externo de runners.
