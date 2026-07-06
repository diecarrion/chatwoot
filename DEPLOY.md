# Deploy Instructions - Chatwoot (Heroku)

> **IMPORTANTE**: Este archivo contiene información sensible del proyecto. NO debe ser commiteado al repositorio.

> **REGLA DE ORO**: Cualquier cambio temporal realizado durante el deploy (ej: aumentar memoria) DEBE ser revertido al finalizar el deploy.

## Información del Proyecto

- **App Heroku**: `chatwoot-sysentify`
- **Rama de Deploy**: `heroku-release`
- **Rama Principal**: `develop`
- **Repositorio GitHub**: `https://github.com/diecarrion/chatwoot.git`
- **Repositorio Upstream**: `https://github.com/chatwoot/chatwoot.git`

## Configuración de Dynos

- **Web Dyno**: Standard-2X (1 GB RAM)
- **Worker Dyno**: Standard-2X (1 GB RAM)
- **NODE_OPTIONS**: `--max-old-space-size=512` (runtime)
- **NODE_OPTIONS (build)**: `--max-old-space-size=4096` (temporal para builds)

## Pre-requisitos

1. Heroku CLI instalado y autenticado
2. Git configurado con acceso al repositorio
3. Ruby (usar `rbenv` con la versión en `.ruby-version`)

```bash
# Verificar heroku CLI
heroku --version

# Verificar apps disponibles
heroku apps
```

## Proceso de Actualización a Nueva Versión

### 1. Verificar Remoto de Heroku

```bash
# Verificar remotos actuales
git remote -v

# Si no existe el remoto 'heroku', agregarlo:
heroku git:remote -a chatwoot-sysentify
```

### 2. Fetch de Tags y Versión Deseada

```bash
# Traer todos los tags del upstream
git fetch --all --tags

# Verificar que el tag existe (reemplazar X.X.X con la versión deseada)
git tag -l "vX.X.X"

# Ver commits del tag
git log vX.X.X --oneline -5
```

### 3. Actualizar Rama heroku-release

```bash
# Asegurarse de estar en la rama heroku-release
git checkout heroku-release

# Reset a la versión deseada
git reset --hard vX.X.X

# Verificar que estás en la versión correcta
git log -1 --oneline
```

### 3.5. Re-aplicar patch del Procfile (release phase smoke)

`git reset --hard vX.X.X` sobreescribe el `Procfile` con la versión upstream. Este repo tiene un patch local que agrega `Rails.application.eager_load!` en la `release:` phase para detectar errores de carga de código **antes** de que Heroku swapee el slug a producción. Re-aplicar después de cada reset:

```bash
# Reemplazar la línea "release:" del Procfile para incluir el smoke de boot.
python3 - <<'PY'
import pathlib
p = pathlib.Path('Procfile')
lines = p.read_text().splitlines()
new = []
for line in lines:
    if line.startswith('release:'):
        new.append(
            "release: POSTGRES_STATEMENT_TIMEOUT=600s bundle exec rails db:chatwoot_prepare "
            "&& bundle exec rails runner 'Rails.application.eager_load!; puts \"[smoke] eager_load OK\"' "
            "&& echo $SOURCE_VERSION > .git_sha"
        )
    else:
        new.append(line)
p.write_text("\n".join(new) + "\n")
print("Procfile patched")
PY

# Stagear el patch (se va con el push del deploy)
git add Procfile
git commit -m "chore(deploy): add eager_load smoke to release phase"
```

> **Por qué este paso existe**: `eager_load!` carga TODAS las clases Ruby del bundle. Si una gem es incompatible con la versión de Ruby/Rails del nuevo tag, o un `require` en el código nuevo está roto, falla acá. El release phase aborta y el slug viejo de prod sigue sirviendo (cero downtime).

### 4. Aumentar Memoria para Build (IMPORTANTE)

**Antes de hacer el deploy**, aumentar temporalmente la memoria de Node.js:

```bash
# Aumentar a 4GB para el build (necesario para Vite)
heroku config:set -a chatwoot-sysentify NODE_OPTIONS=--max-old-space-size=4096
```

> ⚠️ **Race condition**: cualquier `heroku config:set` en Chatwoot dispara el `release:` phase (porque el Procfile lo tiene definido). Mientras ese release phase corre (~30-60s), `heroku config:get NODE_OPTIONS` **devuelve el valor viejo** (con un warning `Release command executing: this config change will not be available until the command succeeds`). **No hacer `git push` hasta confirmar que el release succeeded**, sino el build puede arrancar con `NODE_OPTIONS=512` y morir por OOM:
> ```bash
> # Esperar a que el release v(N+1) succeeded antes de pushear
> until [ "$(heroku releases -a chatwoot-sysentify --num 1 --json | python3 -c 'import sys,json;print(json.load(sys.stdin)[0]["status"])')" != "pending" ]; do sleep 10; done
> heroku config:get NODE_OPTIONS -a chatwoot-sysentify  # debe devolver --max-old-space-size=4096
> ```

### 5. Deploy a Heroku

```bash
# Force push de heroku-release a main de Heroku
git push heroku heroku-release:main --force
```

Este comando:
- ✅ Compilará los assets con Vite (necesita 4GB de memoria)
- ✅ Ejecutará las migraciones de base de datos automáticamente
- ✅ Reiniciará los dynos con la nueva versión

**IMPORTANTE**: El build puede tomar 5-10 minutos. No interrumpir el proceso.

### 6. Reducir Memoria a Runtime Normal

**Después del deploy exitoso**, reducir la memoria a un valor apropiado para runtime:

```bash
# Reducir a 512MB para runtime normal
heroku config:set -a chatwoot-sysentify NODE_OPTIONS=--max-old-space-size=512
```

### 7. Verificar Deploy

```bash
# Verificar que los dynos están corriendo
heroku ps -a chatwoot-sysentify

# Ver logs en tiempo real (opcional)
heroku logs --tail -a chatwoot-sysentify

# Verificar releases
heroku releases -n 5 -a chatwoot-sysentify
```

### 8. Verificar Aplicación

1. Abrir la aplicación en el navegador: https://chatwoot-sysentify-e7748e8d7d48.herokuapp.com
2. Verificar que la versión en la UI sea la correcta (esquina superior izquierda)
3. Probar login y funcionalidades básicas

## Autenticación Heroku desde Claude Code

Cuando el CLI de Heroku falla con `Invalid credentials` o `Device not configured`:

**El problema**: Claude Code (extensión VSCode) no soporta `stdin.setRawMode` directamente, por lo que `heroku login` falla al intentar leer el keypress.

**La solución** (funciona desde Claude Code):
```bash
printf ' ' | script -q /dev/null heroku login
```
Esto crea un pseudo-TTY, envía un keypress automático, el CLI genera la URL de autenticación y abre el browser. El usuario aprueba en el browser y el CLI queda autenticado.

## Troubleshooting

### Error: "JavaScript heap out of memory"

**Síntoma**: El build falla con `FATAL ERROR: Reached heap limit`

**Solución**: Aumentar NODE_OPTIONS antes del deploy
```bash
# Si falla con 2GB, probar con 4GB
heroku config:set -a chatwoot-sysentify NODE_OPTIONS=--max-old-space-size=4096
```

### Error: "Push rejected"

**Síntoma**: `! [remote rejected] heroku-release -> main (pre-receive hook declined)`

**Solución**: Verificar los logs del build
```bash
heroku logs --tail -a chatwoot-sysentify
```

### Dynos no inician después del deploy

**Síntoma**: Dynos en estado "crashed" o "starting" indefinidamente

**Solución**: Verificar logs y reiniciar manualmente
```bash
# Ver logs
heroku logs --tail -a chatwoot-sysentify

# Reiniciar dynos
heroku restart -a chatwoot-sysentify
```

### Migraciones no se ejecutaron

**Síntoma**: Errores relacionados con columnas o tablas faltantes

**Solución**: Ejecutar migraciones manualmente
```bash
heroku run bundle exec rake db:migrate -a chatwoot-sysentify
```

### Rollback a Versión Anterior

Si algo sale mal, hacer rollback:
```bash
# Ver releases disponibles
heroku releases -n 10 -a chatwoot-sysentify

# Rollback al release anterior
heroku rollback -a chatwoot-sysentify

# O a un release específico
heroku rollback vXX -a chatwoot-sysentify
```

## Comandos Útiles

### Ver configuración actual
```bash
heroku config -a chatwoot-sysentify
```

### Ver estado de la base de datos
```bash
heroku pg:info -a chatwoot-sysentify
```

### Ver estado de Redis
```bash
heroku redis:info -a chatwoot-sysentify
```

### Acceder a la consola de Rails
```bash
heroku run bundle exec rails console -a chatwoot-sysentify
```

### Ejecutar comando bash
```bash
heroku run bash -a chatwoot-sysentify
```

## Proceso Resumido (Quick Reference)

Para deploys rápidos cuando todo está configurado:

```bash
# 1. Fetch tags
git fetch --all --tags

# 2. Cambiar a heroku-release y reset a la versión
git checkout heroku-release
git reset --hard vX.X.X

# 3. Re-aplicar patch del Procfile (smoke en release phase) — ver paso 3.5
#    git reset --hard wipea el Procfile, hay que re-patchearlo cada deploy.

# 4. Aumentar memoria para build
heroku config:set -a chatwoot-sysentify NODE_OPTIONS=--max-old-space-size=4096

# 5. Deploy
git push heroku heroku-release:main --force

# 6. Esperar a que termine el build (~5-10 min). El release phase corre eager_load:
#    si falla, el deploy se aborta y prod sigue intacta.

# 7. Reducir memoria a runtime normal
heroku config:set -a chatwoot-sysentify NODE_OPTIONS=--max-old-space-size=512

# 8. Verificar
heroku ps -a chatwoot-sysentify
```

## Notas Importantes

- ⚠️ **NO hacer push a main/develop** - solo hacer deploy desde `heroku-release`
- ⚠️ **Siempre aumentar memoria antes del build** - Vite necesita 4GB para compilar
- ⚠️ **CRÍTICO: Siempre reducir memoria después del deploy** - Runtime solo necesita 512MB
  - **4GB es SOLO para el build temporal**
  - **512MB es para runtime normal**
  - **NO dejar en 4GB después del deploy** - causaría problemas de memoria en los dynos Standard-2X (1GB RAM)
- ⚠️ **Force push es necesario** - Estamos reseteando a tags específicos
- ✅ **Las migraciones se ejecutan automáticamente** - No es necesario ejecutarlas manualmente
- ✅ **Los dynos se reinician automáticamente** - No es necesario reiniciar manualmente

## ⚠️ IMPORTANTE: Estado Antes vs Después del Deploy

### Estado ANTES del Deploy (Normal)
```
NODE_OPTIONS: --max-old-space-size=512
```

### Estado DURANTE el Deploy (Temporal)
```
NODE_OPTIONS: --max-old-space-size=4096  ⚠️ SOLO TEMPORAL
```

### Estado DESPUÉS del Deploy (Debe volver a Normal)
```
NODE_OPTIONS: --max-old-space-size=512  ✅ REVERTIDO
```

**REGLA DE ORO**: Todo cambio temporal debe ser revertido al finalizar el deploy.

## Historial de Deploys

> **Cómo redactar `Notas` en este historial**: describir qué trae la release (features, fixes, hotfixes, migraciones o cambios funcionales relevantes para la instancia). No documentar pasos operativos del deploy como `git push`, releases de Heroku, cambios temporales de `NODE_OPTIONS` o reinicios de dynos, salvo que haya habido un incidente excepcional que sea importante recordar.

| Versión | Fecha      | Estado     | Notas |
|---------|------------|------------|-------|
| v4.10.0 | 2026-02-08 | ✅ Exitoso | Primer deploy documentado. Requirió 4GB para build. |
| v4.11.1 | 2026-02-27 | ✅ Exitoso | Hotfix Google OAuth. Migraciones: EnableCaptainTasks + AddIndexReportingEvents. |
| v4.11.2 | 2026-03-09 | ✅ Exitoso | Hotfix filtros. Migraciones: AddSecretToWebhooks, BackfillWebhookSecrets, DisableReportRollupForAllAccounts. |
| v4.12.0 | 2026-03-17 | ✅ Exitoso | Assignment V2 para todas las instalaciones, mejoras en Help Center y nuevas integraciones/capacidades en Linear, TikTok, WhatsApp Cloud y widget móvil. |
| v4.12.1 | 2026-04-02 | ✅ Exitoso | Fix para error 404 de AI Assist en Community Edition y corrección de la regresión en webhooks `message_created`/`message_updated`, que volvieron a enviar el contenido raw del mensaje en lugar de HTML o formatos renderizados por canal. |
| v4.13.0 | 2026-04-23 | ✅ Exitoso | Release menor con mejoras de Captain (nuevas columnas `edited` y sync status en assistant responses/documents) y toggle Assignment V2 para nuevas cuentas. Migraciones: `EnableAssignmentV2ForNewAccounts`, `AddEditedToCaptainAssistantResponses`, `AddSyncColumnsToCaptainDocuments`, `BackfillEditedOnCaptainAssistantResponses`. |
| v4.14.0 | 2026-05-20 | ✅ Exitoso | Nuevas capacidades de empresa (atributos adicionales, banners de plataforma), Help Center con acciones en lote y traducción IA, autenticación IMAP en canal email, voz extendida en Twilio SMS (TwiML app + API key secret), seguridad reforzada (webhooks, SAML). Migraciones: `AddVoiceToChannelTwilioSms`, `DropChannelVoice`, `CreatePlatformBanners`, `AddAdditionalAttributesToCompanies`, `RepurposeTwilioContentTemplatesFlagForCaptainDocumentAutoSync`, `RenameCompanyConditionKeyInAutomationRules`, `BackfillCaptainDocumentSyncMetadata`, `AddSyncStatsIndexToCaptainDocuments`, `RepurposeReportV4FlagForCaptainV1ActionClassifier`, `AddImapAuthenticationToChannelEmail`, `EnqueueValidateOpenaiHooksJob`. |
| v4.14.1 | 2026-06-09 | ✅ Exitoso | Patch sobre 4.14.0. Nuevo layout estilo documentación para Help Center + switcher, soporte de BSUID en payloads de WhatsApp, adjuntos XML/PFX, fix de overflow en code blocks del message bubble. Mejoras en unread counts / orden de sidebar / badges, voice-call UX (WhatsApp/Twilio Cloud Calling), bulk label removal y cambio de categoría en lote, reliability en IMAP/auto-assign/WhatsApp/CSAT/widgets, nuevos webhook events de inbox y seguridad (SafeFetch, allowlist para webhooks privados de inbox). Migraciones: `RepurposeChannelTwitterFlagForConversationUnreadCounts`, `ChangeCaptainDocumentExternalLinkToText`. |
| v4.15.1 | 2026-06-17 | ✅ Exitoso | Salto 4.14.1→4.15.1 (incluye 4.14.2, minor 4.15.0 y patch 4.15.1). Minor 4.15.0: unread counts + badges/orden/filtros de sidebar, Help Center rediseñado (layout, personalización por locale, color de íconos de categorías, editor mejorado), mejoras de llamadas WhatsApp/Twilio (voice messages, mute, settings de inbox), media viewer y filtros avanzados en contactos/empresas, página dedicada de resultados de búsqueda, tablas redimensionables, hardening de seguridad (sesiones, SafeFetch, webhooks, OAuth, secrets) y reliability en IMAP/SMTP/asignación/widgets/redes. Patch 4.15.1: revert de "Sidebar unread counts for filters" (CW-7262) por regresión. Migraciones aplicadas en release: `AddProviderConfigToChannelTwilioSms`, `AddIconColorToCategories`, `CreateUserSessions`. Smoke `eager_load` del release phase pasó. Post-deploy: `web.1`+`worker.1` up, HTTP 200. |

---

**Última actualización**: 2026-06-17
