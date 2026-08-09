# ms-meli-token-manager

El servicio que mantiene vivo el token de MercadoLibre: lo renueva antes de que expire y deja
la versión fresca en GCP Secret Manager, para que ninguna app de Notorios360 tenga que saber
cómo se hace un refresh de OAuth.

---

## §0 — Cómo se trabaja acá (leer antes de tocar nada)

**Tu foco es este repositorio.** Resolvés acá lo que es de acá.

**Podés leer afuera.** Otros repos, `notorios-vm`, logs, Odoo: mirar para entender está bien y
es parte del trabajo. La VM es el corazón de Notorios360; visitarla no es salirse del carril.

**No arregles lo que encontrás afuera.** Si ves algo raro en otro módulo o en la VM, lo
levantás y seguís con lo tuyo:

```bash
fleet-need add "<síntoma>" --modulo <repo> --pide <equipo> \
  --bloquea "<qué se frena>" --repro "<cómo reproducirlo>"
```

Producto lo evalúa y lo corrige. **Que sea obvio y urgente no te autoriza a arreglarlo vos.**
Un arreglo hecho de paso, en territorio ajeno, sin que el dueño del repo se entere, es
exactamente el modo en que este sistema se vuelve impredecible.

**Lo que sólo puede decidir Sam se escala, no se adivina.** Irreversible, lo ve un cliente,
toca dinero, o pisa un experimento vivo:

```bash
fleet-log escala <task> --team <perfil> --tenant <tenant> --pregunta "..." \
  --porque irreversible|cliente|dinero|experimento-vivo \
  --evidencia "..." --opcion "..." --recomendacion "..."
```

Con evidencia y con opciones. *«Si tengo que ir a investigar para poder responder, ese
escalamiento no sirvió»* — el manifiesto, `notorios-apps-orquesta/fleet/VISION.md`.

**Este README es la visión de largo plazo, y no se cambia solo.** Acá va lo que **no** cambia:
qué problema resuelve esto y para qué existe — los **objetivos**, no los medios. El medio
cambia; el objetivo es lo que le da sentido al cambio.

Si lo que te piden hoy contradice lo que dice acá, **preguntás antes de escribir**: «el README
dice X, me estás pidiendo Y, ¿cambiamos la visión de largo plazo?». Sólo con un sí explícito
se toca este archivo. Sin ese sí, lo que hay es una tarea que cree ser una doctrina.

**Lo que sí cambia va en `PARITY_TRACKER.md`**: decisiones tomadas, con fecha y con su por
qué. Nunca planes ni pendientes — un plan se pudre y después miente; una decisión con fecha no
promete nada, así que envejece sin mentir.

**No hay más archivos de doctrina que estos.** `README.md` es la verdad; `CLAUDE.md` y
`CODEX.md` son punteros de una línea. **`AGENTS.md` no existe**: se borró del árbol entero el
2026-08-09 y no se vuelve a crear — ni a mano, ni desde una herramienta. No escribas doctrina
en los punteros, ni en `.agents/`, ni en un documento nuevo al lado. Dos copias garantizan que
una envejezca y que un agente lea lo que el otro ya corrigió.

**Contexto de dónde estás parado:** Odoo son los datos duros. **Notorios360** es la capa
heurística — los módulos que saben cómo se hace cada cosa; este repo es uno de ellos.
**Orquesta** es la capa de casuística: los teams que aplican esa heurística al caso concreto y
la corrigen cuando falla. Una excepción que la heurística no supo resolver no se arregla a
mano y se olvida: se le pregunta al humano una vez y la respuesta se guarda como regla nueva.
La doctrina global está en `../README.md`.

---

## §1 — Para qué existe

El token de acceso de MercadoLibre **expira a las 6 horas**. Sin nadie que lo renueve, toda
integración con MeLi se cae sola en menos de un día — y el modo de falla es especialmente
malo: no es un error de despliegue ni de código, es un `401` silencioso a las tres de la
mañana, en un canal por el que entra plata.

El objetivo es que **ninguna app tenga que preocuparse por eso**. Este servicio existe para
que exista, en todo momento, **un access token de MeLi válido en un lugar conocido**, y para
que conseguirlo sea leer un secreto en vez de implementar OAuth otra vez en cada módulo.

De eso se derivan tres objetivos que no cambian aunque cambie la implementación:

- **Que el token nunca esté vencido cuando alguien lo pide.** Se renueva con margen sobre la
  ventana de expiración, no en el último minuto ni bajo demanda.
- **Que haya una sola copia vigente.** No versiones viejas conviviendo con la nueva, ni
  «cuál de estos tres es el bueno». Una, la última, y las anteriores destruidas.
- **Que el refresh token se conserve.** Es el único activo que no se puede regenerar sin un
  humano frente a un navegador autorizando la app. Perderlo no es un bug: es una interrupción
  operativa con intervención manual.

Lo que este repo **no** busca ser: no es un cliente de la API de MercadoLibre, no publica
productos, no sincroniza stock ni pedidos. Eso vive en otros módulos (ver §3). Acá sólo se
resuelve la credencial.

---

## §2 — Dónde vive

**Producción: la VM `notorios-vm`, proyecto GCP `notorios`.**

**No es un contenedor.** Es una unidad de systemd, y esto importa porque en `docker ps` no
aparece y ya hubo confusión al respecto:

```
meli-token-rotate.service   (systemd, enabled + active — verificado 2026-08-02)
  ExecStart: /opt/meli-token-manager/.venv/bin/python -m meli_token_manager.cli rotate
```

Corre el **loop continuo** (sin `--once`): un proceso de larga vida que refresca cada
`ROTATION_INTERVAL_SECONDS` (default 14400 s = 4 h) contra una expiración de 6 h.

**Despliegue: manual.** Este repo no tiene `Dockerfile`, ni `docker-compose.yml`, ni
`.github/workflows/`. No hay CI y no hay deploy automático: se instala en
`/opt/meli-token-manager` con su propio venv y se maneja con `systemctl`. Después de tocar
código, el servicio **no se entera hasta que se reinicia**.

**Configuración y secretos.** `config/config_vars.yaml` (gitignored; el template versionado es
`config/config_vars.yaml.example`) declara las variables que `env-manager` resuelve. En
producción, `SECRET_ORIGIN=gcp` y `GCP_PROJECT_ID=notorios`: los valores salen de GCP Secret
Manager. Obligatorias: `MELI_APP_ID`, `MELI_CLIENT_SECRET`, `MELI_REDIRECT_URI`,
`MELI_TOKENS_SECRET_NAME`, `GCP_PROJECT_ID`. Opcionales: `MELI_REFRESH_TOKEN` (sólo para
precargar uno existente), `MELI_TOKEN_FILE` (default `tokens.json`) y
`ROTATION_INTERVAL_SECONDS`.

**Estado local.** El JSON de tokens se escribe en disco en `MELI_TOKEN_FILE` además de en el
secreto. Está gitignored y **es la fuente que gana al arrancar**: el rotador lee primero el
archivo y sólo si no existe cae al secreto de GSM. Un `tokens.json` viejo en la VM manda sobre
lo que diga Secret Manager hasta el próximo refresh.

**IAM.** La SA de la VM (`495979799441-compute@`) ya tiene `secretmanager.admin` a nivel
proyecto: no hace falta IAM extra para que este servicio cree, escriba y destruya versiones.

**Observabilidad.** No hay healthcheck ni endpoint: este proceso no expone nada. La única
señal es `journalctl -u meli-token-rotate`. El loop está escrito para no morirse nunca —
si un refresh falla, loguea, duerme `min(600, interval)` y reintenta —, así que **un servicio
`active (running)` no prueba que el token se esté renovando**. Es exactamente el modo de falla
que describe §5 del README global: un semáforo que sólo sabe ponerse verde.

---

## §3 — Su lugar en Notorios360

**Qué consume**

- **MercadoLibre OAuth**: `https://auth.mercadolibre.cl/authorization` (bootstrap interactivo)
  y `https://api.mercadolibre.com/oauth/token` (canje de `code` y `refresh_token`).
- **GCP Secret Manager** del proyecto `notorios`, vía `google-cloud-secret-manager`.
- **`env-manager`** (`notoriosti-env-manager`, desde PyPI) para resolver configuración.

**Qué expone**

- **El secreto de tokens en GSM**, con el nombre que diga `MELI_TOKENS_SECRET_NAME`. Payload
  JSON: `access_token`, `refresh_token`, `token_type`, `scope`, `expires_in`, `expires_at`,
  `updated_at`. **Ésta es la interfaz pública real del servicio**: quien necesita el token lee
  ese secreto.
- **Helpers Python** para leerlo sin reimplementar nada: `get_access_token()` y
  `get_token_payload()` (`from meli_token_manager import ...`).
- **La CLI** `meli-token-rotate` (equivalente a `python -m meli_token_manager.cli`), con
  subcomandos `init` y `rotate`.

**Quién depende de él — y quién no**

- Los tokens que rota viven en los secretos **`MELI_*` de GSM, compartidos multi-app**. El
  README global los lista explícitamente entre los secretos que **quedan individuales a
  propósito** y que no hay que consolidar.
- **Despacha NO depende de esto, y no es un duplicado.** Despacha refresca sus propios tokens
  de MeLi desde `channel_credentials` en su propia DB (`mercadoLibreTokenRefreshWorker`).
  Verificado el 2026-08-02 y anotado en el `README.md` de Despacha con una advertencia
  explícita: son **dos caminos independientes; no apagar uno creyendo que es duplicado del
  otro**. Los contenedores históricos `meli-token-rotator*` sí se apagaron; este
  `meli-token-rotate.service` es otra cosa y sigue vivo a propósito.
- **Cataloga tampoco.** Sus operaciones de MeLi van por los proxies de sólo lectura de
  Despacha (`/mercadolibre/catalog/...`), no por acá. Concilia, para el billing de MeLi, pasa
  por el broker de Despacha con su propio token de integración.
- **Ningún repo del checkout local importa este paquete** (verificado 2026-08-09: no aparece
  como dependencia en ningún `pyproject.toml` ni `requirements.txt` de `~/Desktop/Notorios360`).
  Su acoplamiento con el resto es **por el secreto, no por el import**. Consecuencia práctica:
  cambiarle el nombre o el formato al payload rompe consumidores que no se ven grepeando este
  repo.

---

## §4 — Reglas duras

**Al rotar se destruyen las versiones viejas. Este repo es el patrón de referencia.**
`write_secret()` agrega la versión nueva y después llama a `_destroy_prior_versions()`, que
destruye todas las demás (salta las ya `DESTROYED` y traga errores de API para no romper la
rotación). El README global cita este comportamiento como el patrón que copian los demás
módulos: **«se destruyen las versiones viejas tras rotar (se paga por versión activa; patrón
meli-token-manager)»**. No es una optimización opcional ni un detalle de implementación: es la
razón por la que este repo se nombra en la doctrina global. Quien lo quite, quita el ejemplo.
(La decisión es anterior a esta doctrina: commit `e94af4c`, *«Destruir secretos antiguos en
vez de desactivarlos»* — antes se desactivaban, y desactivadas se siguen pagando.)

**Los secretos `MELI_*` quedan individuales. No los consolides.** La regla general de
Notorios360 es un único secreto JSON `<app>-config` por app; `MELI_*` está en la lista de
excepciones deliberadas del README global, por ser compartidos multi-app. Consolidarlos acá
rompería a los consumidores de afuera.

**Acá NO se cachea, y es a propósito.** La doctrina global dice «cachear siempre: leer
secretos una vez al arrancar y servir desde memoria». `token_access.py` hace lo contrario:
construye un `ConfigManager` nuevo **en cada llamada** para no servir un token vencido desde
caché. Es la excepción deliberada de este módulo — el objeto que reparte es justamente el que
cambia cada 4 horas. Documentada acá para que nadie la «arregle».

**El rotador sí cachea sus credenciales al boot — y eso tiene consecuencia operativa.**
`TokenRotator.__init__` lee `MELI_APP_ID`, `MELI_CLIENT_SECRET`, `MELI_TOKENS_SECRET_NAME` y
`GCP_PROJECT_ID` una sola vez. Como en producción corre el loop infinito, **si rotás a mano
cualquiera de esos secretos, el servicio sigue con el valor viejo hasta que lo reinicies**
(`systemctl restart meli-token-rotate`). Es el mismo problema que costó 28 horas de
`localfeed` el 2026-08-08.

**El margen de rotación es una regla, no un default cualquiera.** El token expira a las 6 h y
se refresca cada 4 h. Ese colchón de 2 h es lo que absorbe un reinicio, una caída de la API de
MeLi o un par de reintentos. Subir `ROTATION_INTERVAL_SECONDS` cerca o por encima de la
ventana de expiración convierte el servicio en decorativo.

**El refresh token no se pierde.** MeLi puede devolver uno nuevo en cada refresh; el código
guarda el nuevo y, si no viene, conserva el anterior (`payload.get("refresh_token") or
refresh_token`). Si se pierde igual, **no hay recuperación automática**: hay que rehacer el
bootstrap OAuth interactivo, que exige un humano autorizando la app en el navegador y pegando
el `code`. Eso es un evento a escalar, no a improvisar.

**Ningún agente ve valores de secretos.** Ni Claude, ni Codex, ni ninguno. Los secretos se
crean vacíos y los rellena el usuario, o se mueven por script sin imprimirlos jamás. Nunca
imprimas un token en un log, en una salida de terminal ni en un PR.

**Nunca commitear credenciales.** `config/config_vars.yaml` y `tokens.json` están en
`.gitignore` y ahí se quedan. Lo que se versiona es `config/config_vars.yaml.example`, con las
claves y sin los valores. Tampoco entran credenciales de GCP ni el `client_secret`.

**`env-manager` viene de PyPI.** La dependencia es `notoriosti-env-manager` (`>=0.2.3,<1.0.0`),
migrada desde la dependencia git en el commit `f776d2c`. La copia en `libraries/env-manager`
está **DEPRECADA** (y `libraries/` está gitignored acá).

**Estilo de código.** PEP 8, indentación de 4 espacios. `snake_case` para funciones y
variables, `CapWords` para clases, `UPPER_SNAKE_CASE` para constantes. Los módulos nuevos van
bajo `src/meli_token_manager/` y siguen el orden de imports existente (stdlib, third-party,
local). **No hay formateador impuesto**: mantené la consistencia con el código de al lado.
Python **>=3.13** (`pyproject.toml`); el README viejo decía 3.11+ y estaba vencido.

**Tests.** No existen todavía. Cuando se agreguen: bajo `tests/`, con nombres `test_*.py`,
framework `pytest` (ya está como dependencia dev), corridos con `poetry run pytest`. Si se
agregan gates de cobertura, van cableados en CI y documentados acá.

**Commits y PRs.** Los commits recientes usan Conventional Commits en español (`docs:`,
`fix:`, `feat:`); la historia vieja usa frases cortas capitalizadas en inglés («Added base
package»). Manda lo reciente. Un PR incluye resumen breve, notas de testing y **cualquier
cambio de configuración** — sobre todo claves nuevas obligatorias en
`config/config_vars.yaml.example`, porque una clave nueva sin su entrada en el ejemplo es un
despliegue roto en la VM.

---

## §5 — Cómo se corre

**Setup local**

```bash
poetry install
cp config/config_vars.yaml.example config/config_vars.yaml   # completar valores
```

**Bootstrap inicial (una sola vez, o tras perder el refresh token)**

Interactivo — imprime la URL de autorización y pide el `code` del redirect:

```bash
poetry run python -m meli_token_manager.cli init \
  --secret-origin gcp --config config/config_vars.yaml
```

Con el `code` ya obtenido:

```bash
poetry run python -m meli_token_manager.cli init \
  --secret-origin gcp --config config/config_vars.yaml --code "<code>"
```

Genera `tokens.json` (o la ruta de `MELI_TOKEN_FILE`) y crea o actualiza el secreto en GCP. Si
el secreto no existe, se crea con replicación automática.

**Rotación**

Loop continuo — es lo que corre en producción bajo systemd:

```bash
poetry run python -m meli_token_manager.cli rotate \
  --secret-origin gcp --config config/config_vars.yaml
```

Una sola pasada, para probar o para un cron externo:

```bash
poetry run python -m meli_token_manager.cli rotate --once \
  --secret-origin gcp --config config/config_vars.yaml
```

Flags: `--gcp-project-id` sobrescribe el proyecto, `--interval-seconds` ajusta el intervalo del
loop (ignorado con `--once`). Sin subcomando, el default es `rotate`. La CLI empaquetada es
`poetry run meli-token-rotate --help`.

**Leer el token desde otro servicio**

```python
from meli_token_manager import get_access_token

token = get_access_token(
    config_path="/ruta/config/config_vars.yaml",
    secret_origin="gcp",
)
```

Payload completo (incluye `refresh_token`, `expires_at`, metadatos):

```python
from meli_token_manager import get_token_payload

tokens = get_token_payload(config_path="/ruta/config/config_vars.yaml", secret_origin="gcp")
```

**Build**

```bash
poetry build
```

**Tests**

No hay suite todavía. El fleet declara `python3 -m pytest -q` en
`notorios-apps-orquesta/fleet/tests.json`, marcado `validated: false` — hoy no corre nada.

**Operar en la VM**

```bash
systemctl status meli-token-rotate
journalctl -u meli-token-rotate -n 100
systemctl restart meli-token-rotate    # única forma de que tome secretos rotados a mano
```

---

Lo que cambia —decisiones tomadas, con fecha y con su por qué— vive en `PARITY_TRACKER.md`.
