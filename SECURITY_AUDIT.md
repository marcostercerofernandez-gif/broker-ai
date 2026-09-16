# Auditoría de secretos — Broker AI v1

- **Fecha:** 2026-09-16
- **Repositorio:** `marcostercerofernandez-gif/broker-ai` (rama `main`, publicado en GitHub Pages)
- **Alcance:** árbol de trabajo actual + historial completo (`git log --all`: 32 commits, sin stash ni objetos inalcanzables)
- **Herramientas:** gitleaks 8.30.1 (binario oficial, checksum SHA-256 verificado), más una búsqueda propia de patrones por proveedor en cada revisión (`sk-ant-`, `AIza…`, webhooks `hook.*.make.com`, Apps Script, OAuth de Google, `token=`, `Bearer`, claves privadas, IDs de Google Sheets)
- **Estado del historial:** sin tocar. No se ha usado filter-repo ni BFG.

## 1. Resultado

| # | Proveedor probable | Tipo | Archivo | Primera aparición | ¿En HEAD antes del arreglo? | Valor enmascarado |
|---|---|---|---|---|---|---|
| 1 | **Finnhub** | Clave de API (`token=`), 40 caracteres | `index.html` (`const FINNHUB_KEY`) | `6e10a0c` · 2026-03-16 | No (cambió en `d27f20e`) | `…fsb0` |
| 2 | **Finnhub** (la misma clave, corregida) | Clave de API (`token=`), 40 caracteres | `index.html` (`const FINNHUB_KEY`, línea 670) | `d27f20e` · 2026-03-18 | **Sí** → sustituida por `REPLACE_WITH_FINNHUB_KEY` | `…fsbg` |
| 3 | Google Sheets (informativo) | ID de hoja para la API `gviz` | `index.html` (`const SHEET_ID`) | `dc46f0f` · 2026-03-16 | No | `…rc7M` |
| 4 | Google Sheets (informativo) | URL de "Publicar en la web" (CSV) | `index.html` (`const SHEET_URL`, línea 477) | posterior a `dc46f0f` | Sí, se mantiene | `…l5J1` |

**Notas sobre los hallazgos**

- **1 y 2 son la misma credencial.** Las claves de Finnhub tienen dos mitades de 20 caracteres; los dos valores solo difieren en el último carácter (`0` → `g`), lo que apunta a un error al copiarla y a su corrección posterior. Estuvo en 8 commits consecutivos (`6e10a0c` → `d27f20e`). Con rotar esa clave de Finnhub basta para cubrir ambos valores.
- **3 y 4 no son credenciales.** Son identificadores de una hoja que se leía sin autenticación porque era pública ("cualquiera con el enlace" o "Publicar en la web"). No se pueden rotar como una clave. Lo que exponen son **los datos de la hoja**, no acceso de escritura. No se han quitado porque son la fuente de datos del panel y quitarlos cambiaría su lógica. Revisa en el apartado 4 qué contiene la hoja.
- **No se ha encontrado nada de Anthropic, Make.com, Google Cloud ni Yahoo** en ningún commit: ni `sk-ant-`, ni `AIza…`, ni webhooks de Make, ni tokens OAuth. La llamada a Claude se hace desde el escenario de Make.com, así que esa clave vive en la conexión de Make y no en el HTML. Yahoo Finance (`query1.finance.yahoo.com`) y alternative.me (Fear & Greed) se usan sin clave.
- Que no aparezca en el repositorio **no garantiza** que una clave no se haya filtrado por otra vía (capturas, escenarios de Make compartidos, blueprints exportados). La checklist sigue incluyendo a Anthropic y Make por prudencia.

## 2. Cambios aplicados en HEAD

1. `index.html`: `const FINNHUB_KEY = 'REPLACE_WITH_FINNHUB_KEY';`. No se ha tocado nada más de la lógica.
   - **Efecto en el panel publicado:** Finnhub responderá 401 para AAPL, MSFT, SPY, NVDA y GLD. `fetchPrice()` ya captura el error y devuelve `null`, así que esas posiciones mostrarán `—`. Yahoo, Google Sheets y Fear & Greed siguen funcionando igual.
   - **No vuelvas a poner una clave real en este archivo.** En una web estática de GitHub Pages cualquier clave en el HTML es pública por definición, esté o no en `.env`. La única forma segura de usar Finnhub desde el navegador es un proxy o backend, que queda fuera del alcance de la v1.
2. `.gitignore` nuevo con `.env`, `.env.*` y `*.env` (se permite `.env.example`).
3. Verificación: `gitleaks dir` sobre el árbol y `gitleaks git` sobre el commit HEAD → **0 hallazgos**.

> Los cambios están en un **commit local, sin push**. La clave sigue siendo visible en la web publicada hasta que hagas `git push`, y en el historial de GitHub para siempre si no se reescribe.

## 3. ¿Reescribir el historial? Qué implicaría con GitHub Pages activo

**No se ha hecho.** Si quieres hacerlo más adelante, esto es lo que conlleva:

- **Rotar primero; reescribir es opcional.** La clave lleva pública desde el 2026-03-16 en un repositorio y una web públicos. Los bots que rastrean GitHub la capturan en minutos. Reescribir el historial no la "desfiltra": la única solución real es revocarla. Reescribir solo sirve para limpiar y para no dejar la clave a la vista de quien navegue por el repositorio.
- **Hace falta un `git push --force` a `main`.** Todos los SHA desde `6e10a0c` cambian (8 commits afectados y todos sus descendientes). Si `main` tiene protección de rama, hay que permitir temporalmente el force-push.
- **GitHub Pages se vuelve a desplegar** con el nuevo HEAD. El contenido publicado no cambia respecto a hacer un push normal. Puede haber un breve periodo de redespliegue y la caché del CDN puede servir la versión antigua durante unos minutos.
- **GitHub conserva los commits antiguos.** Siguen accesibles por su SHA (`github.com/…/commit/6e10a0c…`) y en vistas en caché. Para purgarlos hay que abrir un ticket con GitHub Support pidiendo que eliminen las vistas en caché y ejecuten el garbage collection del repositorio.
- **Forks y clones** (incluido `broker-ai-v2`, si se creó a partir de este) conservan el historial antiguo. Cada copia habría que re-clonarla o limpiarla.
- **Se rompen los enlaces** a commits concretos y cualquier referencia externa a esos SHA. Las ejecuciones o artefactos de Actions y Pages asociados a commits antiguos pueden seguir mostrando contenido previo hasta que caduquen o se borren.
- **Procedimiento, si decides hacerlo:** (1) rotar la clave; (2) hacer un mirror de respaldo; (3) `git filter-repo --replace-text` con un fichero que mapee los dos valores a `REPLACE_WITH_FINNHUB_KEY`; (4) volver a ejecutar gitleaks sobre todo el historial; (5) force-push; (6) ticket a GitHub Support; (7) re-clonar las copias locales.

Valoración: al ser una v1 congelada y una clave de un plan gratuito de Finnhub, **rotar la clave es suficiente**. La reescritura solo compensa si quieres que el repositorio público quede limpio de cara a terceros.

## 4. Checklist de rotación manual por proveedor

### Finnhub — **OBLIGATORIO (clave expuesta)**
- [ ] Entra en <https://finnhub.io/dashboard> con la cuenta dueña de la clave `…fsbg`.
- [ ] Regenera o revoca la API key. Si el panel no ofrece esa opción en tu plan, escribe a `support@finnhub.io` pidiendo que la invaliden o, como alternativa, crea una cuenta nueva y abandona la antigua.
- [ ] Comprueba que la clave antigua ya no funciona: `curl "https://finnhub.io/api/v1/quote?symbol=AAPL&token=<CLAVE_ANTIGUA>"` debe devolver 401.
- [ ] Revisa el uso o los límites de la cuenta en el dashboard: busca errores 429 o consumo que no corresponda a tu panel.
- [ ] Si tienes un plan de pago asociado, revisa la facturación desde el 2026-03-16.
- [ ] No pongas la clave nueva en `index.html` de la v1.

### Google Sheets / Google — **revisión de exposición de datos**
- [ ] Abre la hoja y revisa qué contiene. Si hay datos que no quieres públicos (posiciones, importes, notas), actúa.
- [ ] Hoja con `…l5J1`: *Archivo → Compartir → Publicar en la web → Detener publicación* (esto rompe el panel v1) o mueve los datos sensibles a otra pestaña no publicada.
- [ ] Hoja con `…rc7M`: *Compartir* → cambia "Cualquier persona con el enlace" a "Restringido" si ya no la usa nada.
- [ ] En <https://myaccount.google.com/permissions>, revisa qué apps tienen acceso a tu Google (Make.com incluido) y quita las que ya no uses.
- [ ] Si alguna vez creaste claves en Google Cloud para este proyecto: <https://console.cloud.google.com/apis/credentials> → borra las que no uses y revisa *APIs y servicios → Métricas*.

### Anthropic — **preventivo (no encontrada en el repositorio)**
- [ ] En <https://console.anthropic.com/settings/keys>, identifica la clave que usa Make.com.
- [ ] Si la v1 está congelada, **desactívala o bórrala**. Si el escenario sigue activo, crea una nueva, actualízala en Make y borra la antigua.
- [ ] En *Usage* y *Cost* de la consola, busca picos de tokens, modelos que no usas o actividad fuera de las 08:30 (hora del escenario diario).
- [ ] Configura un límite de gasto mensual en *Settings → Limits*.

### Make.com — **preventivo (no encontrado en el repositorio)**
- [ ] *Connections*: revisa la conexión de Anthropic y la de Google. Si la v1 queda congelada, desactiva el escenario y elimina las conexiones.
- [ ] *Webhooks*: si el escenario tiene webhooks, comprueba que sus URLs no se hayan compartido. Si hay dudas, bórralos y créalos de nuevo (la URL cambia).
- [ ] *Scenario → History* y *Organization → Usage*: busca ejecuciones u operaciones que no correspondan a la ejecución diaria.
- [ ] Si compartiste blueprints o capturas del escenario, recuerda que pueden incluir URLs de webhook.

### Yahoo Finance / alternative.me
- [ ] Nada que rotar: son endpoints públicos sin clave.

### GitHub
- [ ] *Settings → Code security*: activa **Secret scanning** y **Push protection** para bloquear futuros commits con claves.
- [ ] Revisa en *Security → Secret scanning alerts* si GitHub ya había detectado la clave y cierra la alerta como "revoked" tras rotarla.
- [ ] Haz `git push` del commit de esta auditoría para que la web publicada deje de servir la clave.
