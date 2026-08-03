# Guía de uso — DICOM Migration Manager

Manual operativo, solapa por solapa, con casos de ejecución de punta a punta.
Pensado para administradores PACS e ingenieros de implementación.

> Este documento es la **guía de uso**. Para descargar la aplicación y los primeros
> pasos, ver [README.md](README.md).

---

## Índice

1. [Conceptos mínimos antes de empezar](#1-conceptos-mínimos-antes-de-empezar)
2. [Primer arranque](#2-primer-arranque)
3. [Solapa **Configuración**](#3-solapa-configuración) — definir PACS y probar conectividad
4. [Solapa **DICOM Explorer**](#4-solapa-dicom-explorer) — explorar y analizar headers
5. [Solapa **Query / Retrieve**](#5-solapa-query--retrieve) — buscar y migrar selecciones puntuales
6. [Solapa **Migraciones**](#6-solapa-migraciones) — por lista y por catálogo (masivo)
7. [Solapa **Jobs Históricos**](#7-solapa-jobs-históricos) — controlar y validar migraciones
8. [Solapa **Dashboard**](#8-solapa-dashboard) — monitoreo en vivo + estadísticas
9. [Solapa **Logs**](#9-solapa-logs) — errores y auditoría
10. [Casos de ejecución completos](#10-casos-de-ejecución-completos)
11. [Glosario y solución de problemas](#11-glosario-y-solución-de-problemas)

---

## 1. Conceptos mínimos antes de empezar

| Término | Qué es |
|---|---|
| **AE Title** | Nombre lógico de un nodo DICOM (origen, destino o esta app). Sensible a mayúsculas. |
| **Host / Puerto** | Dirección de red del PACS. Puerto DICOM típico: **104** (o 11112). |
| **C-ECHO** | Verificación DICOM: valida que dos nodos negocian asociación (AE titles incluidos). Botón *Probar conexión*. |
| **Ping (TCP)** | Chequeo de **red** a host:puerto, sin DICOM. Botón *Ping*. Sirve para distinguir un problema de red de uno de AE/DICOM. |
| **C-FIND** | Consulta de metadata (qué estudios/series hay). No mueve imágenes. |
| **C-MOVE** | El PACS **origen empuja** las imágenes directo al destino. La app no las toca. |
| **C-GET** | La **app recupera** las imágenes a disco para reenviarlas. |
| **C-STORE** | Enviar imágenes a un nodo (lo usa el reenvío y `storescp`). |
| **StudyInstanceUID** | Identificador único e inmutable de un estudio. Clave de deduplicación. |

**Idea central:** la app **orquesta**; el transporte masivo lo hacen binarios DCMTK
externos. Todo lo que migrás queda registrado en una base **SQLite local** (`data/`)
junto al ejecutable, con checkpoint por estudio (reanudable).

> **Copiar de cualquier grilla:** en todas las tablas (Jobs, estudios, Query/Retrieve,
> catálogo, Logs, Migraciones, headers del Explorer) podés **seleccionar filas/celdas
> y copiar con Ctrl+C** o **click derecho → Copiar / Copiar con encabezados**. Se copia
> como texto tabulado, pegable directo en Excel.

---

## 2. Primer arranque

1. Ejecutá `DICOMMigrationManager.exe` (build portable) o `python -m app.main` (dev).
2. En el primer arranque se crea la carpeta **`data/`** junto al exe: base SQLite,
   logs (3 canales) y caché. **Es portable**: copiar la carpeta mueve todo.
3. La ventana abre con **7 pestañas**. El orden recomendado de trabajo es:
   **Configuración → (Explorer / Query) → Migraciones / Query → Jobs → Dashboard / Logs.**
4. Menú **Ayuda → Acerca de…**: versión y contacto del autor.

> Antes de cualquier operación DICOM necesitás al menos **una conexión** y que
> **DCMTK esté disponible** (panel de binarios en Configuración).

---

## 3. Solapa Configuración

Es el punto de partida: aquí se definen los PACS, se prueba conectividad y se
verifica que los binarios DCMTK estén presentes.

### 3.1 Crear / editar una conexión

1. Click en **Nuevo**.
2. Completá el **Perfil de conexión**:
   - **Nombre** — etiqueta libre (ej. *PACS Central*).
   - **AE Title remoto** — el AE del PACS origen o destino.
   - **Host** y **Puerto** — dirección de red (puerto por defecto 104).
   - **AE Title local** — con qué AE se presenta esta app (default `DICOMTOOL`).
     **Importante:** este AE debe estar **dado de alta en el PACS** para que acepte
     la asociación; para C-MOVE, el **AE destino** también debe estar configurado
     como nodo conocido en el **PACS origen** (si no, rechaza el move).
   - **Timeouts** (Connect / ACSE / DIMSE) — subilos si el PACS es lento o la red
     tiene latencia alta.
   - **Máx. asociaciones** — cuántas asociaciones DICOM simultáneas tolera **este**
     PACS (default 4). La concurrencia real de una migración es `min(workers, este
     valor)`: **bajalo para no saturar un PACS lento**. Se aplica al instante al
     guardar (no hace falta reiniciar).
3. (Opcional) **TLS** — campos de CA/cert/key de cliente. *La negociación TLS real
   está pendiente; los campos se guardan.*
4. **Guardar**.

### 3.2 Probar conexión (C-ECHO) y Ping (red)

- **Probar conexión** ejecuta un **C-ECHO** real (nivel DICOM: negocia AE titles) y
  muestra latencia (ej. *"C-ECHO OK 47 ms"*) o el error. **Hacelo siempre antes de migrar.**
- **Ping** hace un chequeo de **red TCP** a host:puerto, **sin** DICOM. Úsalo para
  **aislar el problema**: si el Ping anda pero el C-ECHO falla, el problema es de
  **AE title / configuración DICOM** (no de red); si falla el Ping, es **red/firewall/
  host/puerto**.

### 3.3 Estado de binarios DCMTK

Panel que lista `echoscu`, `findscu`, `movescu`, `getscu`, `storescu`, `storescp`:
cada uno como **OK** o **no encontrado**. Si aparecen "no encontrado", colocá los
binarios en `vendor/dcmtk/` (incluido `dicom.dic`).

### 3.4 SCP de almacenamiento (storescp)

Panel para **Iniciar/Detener** un receptor DICOM embebido (AE + puerto + directorio).
Útil para recibir un C-MOVE de prueba o como destino temporal. Muestra estado y dir.

> **Borrar** una conexión está restringido a rol **ADMIN** (control de acceso básico).

---

## 4. Solapa DICOM Explorer

Para **consultar estudios puntuales con varios filtros** y obtener un **análisis
completo de los DICOM headers**. Navegación jerárquica perezosa: Estudio → Serie →
Instancia (cada expansión dispara su propio C-FIND).

### Paso a paso

1. Elegí **Conexión** (origen a explorar).
2. Cargá filtros: **Nombre paciente** (acepta wildcards DICOM, ej. `DOE*`), Patient ID,
   accession, modalidad, institución, médico referente, descripción, Study UID y
   **rango de fechas**. Para la fecha, tildá **Filtrar por fecha** (arranca en hoy; el
   botón **Hoy** fija ambos extremos al día); destildado = sin filtro de fecha.
3. **Buscar estudios** → aparecen los estudios que matchean (árbol).
4. **Expandí** un estudio para ver sus series, y una serie para ver sus instancias.
   Cada nivel se consulta on-demand (no se trae todo de golpe).
5. Seleccioná una **instancia** y click en **Ver headers completos**:
   la app **recupera esa instancia** (C-GET) y muestra **todos los tags** del objeto.

### Cuándo usarla

- Diagnóstico fino: verificar qué tags trae realmente un objeto (ej. confirmar si
  `InstitutionName` existe en el origen — ver caso 10.5).
- Comparar **origen vs destino** abriendo la misma instancia en cada PACS.
- Inspección puntual sin crear ningún job.

> El Explorer **no migra**; es solo lectura/análisis. Para migrar usá Query/Retrieve
> o Migraciones.

---

## 5. Solapa Query / Retrieve

Para **buscar estudios y crear un job de migración** con la selección. Es el camino
directo cuando ya sabés qué buscar y el volumen es manejable (la grilla y el C-FIND
tienen tope de resultados; para millones de estudios usá **Migraciones → Por catálogo**).

### 5.1 Buscar

1. **Conexión origen**.
2. **Nivel** de consulta: `STUDY` / `SERIES` / `IMAGE`.
3. **Modelo** de Query/Retrieve:
   - **Study Root** — buscás directo por estudio (lo habitual).
   - **Patient Root** — la jerarquía arranca en paciente (útil si filtrás por paciente).
4. Filtros: **Nombre paciente** (acepta wildcards, ej. `DOE*`), Patient ID, accession,
   modalidad, institución, médico referente, descripción y **rango de fechas**. Para
   la fecha tildá **Filtrar por fecha** (default hoy + botón **Hoy**); si *desde =
   hasta*, consulta por fecha exacta.
5. **Buscar** → la grilla virtualizada lista los resultados (con conteos de series/
   instancias cuando el PACS los devuelve).

### 5.2 Crear Job de migración

1. **Seleccioná** una o varias filas de la grilla.
2. **Crear Job de migración** → abre el asistente *Crear Job* (ver [§5.3](#53-asistente-crear-job)).

### 5.3 Asistente "Crear Job"

Campos:

- **Nombre** del job.
- **Conexión destino**.
- **Estrategia**:
  | Estrategia | Flujo | Cuándo |
  |---|---|---|
  | **C-MOVE directo** | Origen → Destino (el origen empuja; no pasa por la app) | Hay ruteo C-MOVE configurado en el origen. La más eficiente. |
  | **C-GET + reenvío** | Origen → App → Destino (transitorio, sin persistir) | No hay ruteo C-MOVE entre nodos. |
  | **Store & Forward** | Origen → caché en disco → Destino (durable, con reintentos) | Redes inestables o alto volumen; sobrevive caídas sin re-descargar. |
- **Workers** (1 / 5 / 10 / 20) — paralelismo. La concurrencia real contra un PACS es
  `min(workers, max_associations del nodo)`: **nunca abre más asociaciones de las que
  el PACS tolera**.
- **Deduplicación**:
  | Política | Qué hace |
  |---|---|
  | **Ninguna** | No verifica; siempre transfiere. |
  | **Omitir (SKIP)** | Si el `StudyInstanceUID` ya existe en destino → no lo envía (queda **SKIPPED**). |
  | **Comparar (COMPARE)** | Existe **y** el conteo de instancias coincide → omite; si difiere (parcial) → lo re-transfiere. Ideal para reanudar. |
  | **Sobrescribir (OVERWRITE)** | No verifica; re-envía aunque exista. |
  > La dedup compara por **StudyInstanceUID**, no por Accession Number.
- **Dry run** (checkbox) — **simulación**: agrega estudios/series/imágenes/bytes
  estimados **sin transferir nada** ni cambiar estados. Úsalo para dimensionar antes
  de lanzar de verdad.

Al confirmar, el job queda creado en estado inicial y aparece en **Jobs Históricos**,
desde donde se **inicia y controla**.

---

## 6. Solapa Migraciones

Dos sub-solapas: **Por lista** y **Por catálogo (masivo)**.

### 6.1 Por lista (Modalidad B)

Migrar a partir de un **archivo** con identificadores (CSV / TXT / Excel).

**Formato del archivo:**
- **Una sola columna** (la primera) o CSV con encabezado reconocible.
- **Un único tipo de identificador por archivo**, elegido en *Tipo de identificador*
  o detectado por el encabezado: **Study Instance UID**, **Series Instance UID**,
  **SOP Instance UID**, **Accession Number** o **Patient ID**.
- Validación de formato: UIDs `^\d+(\.\d+)*$` (≤64), Accession ≤16, Patient ID ≤64.
  Las filas inválidas se reportan (no frenan a las válidas).

**Paso a paso:**
1. **Conexión origen**.
2. **Tipo de identificador**.
3. **Elegir archivo…** y seleccioná el CSV/TXT/Excel.
4. **Resolver** — la app hace C-FIND contra el origen para resolver cada identificador
   a su estudio, mostrando el **avance en %** (`Resolviendo… 45% (9/20)`). Al terminar
   lista **resueltos** (grilla) y **no resueltos** (con motivo).
5. **Crear Job** — abre el mismo asistente de [§5.3](#53-asistente-crear-job).

> Ejemplo de CSV por Accession Number:
> ```
> AccessionNumber
> A0001
> A0002
> ```

**Guardar / reusar la lista resuelta (sin volver a resolver):**
- **Guardar lista…** — guarda los estudios resueltos con un nombre. Útil para reenviar
  los mismos estudios a **otro destino** más adelante sin re-resolver contra el origen.
  Guardar con un nombre existente lo **reemplaza**.
- **Cargar lista…** — abre el selector de listas guardadas (Cargar / Borrar). Al cargar,
  llena la grilla; después elegís **origen** y **Crear Job** al destino que quieras.

### 6.2 Por catálogo (masivo)

Para **PACS de millones de estudios**. En vez de una sola query gigante, **cataloga**
el origen en la DB local con **muchas consultas chicas por rango de fecha**, y después
migrás **por lotes** desde esa base. Sin topes de grilla ni de memoria.

**Fase A — Catalogar (harvest):**
1. **Origen** a catalogar.
2. (Opcional) **Modalidad** e **Institución** para acotar.
3. **Rango de fechas** (Desde / Hasta).
4. **Granularidad**: **día / semana / mes** — tamaño de cada partición de consulta.
   Si una partición vuelve truncada, la app la **sub-divide automáticamente** por
   modalidad y luego por franja horaria (split adaptativo).
5. **Catalogar** — corre en segundo plano (no congela la UI); progreso en vivo por
   partición. **Cancelar** lo detiene de forma cooperativa.
   - Es **reanudable**: las particiones ya completadas se saltean al re-lanzar.
6. Al terminar, la tabla del catálogo se puebla (paginada por **keyset**, nunca carga
   todo en memoria).

**Fase B — Migrar desde el catálogo:**
1. En el filtro, elegí **Origen / Modalidad / Fechas** y **Aplicar filtro**.
2. Navegá con **Página anterior / siguiente** (el contador muestra el total).
3. **Crear Job de migración masiva** — abre el asistente de [§5.3](#53-asistente-crear-job).
   El job se siembra desde el catálogo **por lotes** y el orquestador recorre los
   pendientes con cursor (sin materializar millones de filas).

> **Recomendación de escala:** Store & Forward + dedup **Comparar** para migraciones
> largas; así reanudás sin re-descargar y sin re-enviar lo que ya llegó completo.

---

## 7. Solapa Jobs Históricos

Centro de control de las migraciones. Tabla de jobs + drill-down de estudios.

### 7.1 Controlar un job

Seleccioná un job y usá:
- **Iniciar** — arranca la migración en segundo plano.
- **Pausar** / **Reanudar** — la pausa es **cooperativa**: corta **entre** estudios,
  nunca a mitad de un C-MOVE. Reanuda desde donde quedó (checkpoint por estudio).
- **Cancelar** — detiene el job.
- **Editar** — solo para jobs **no iniciados (PENDING)**: cambia nombre, destino,
  estrategia, workers, dedup y dry-run. Un job ya iniciado queda inmutable.
- **Reintentar fallidos** — reencola los estudios FAILED/PARTIAL a PENDING y reinicia
  la migración. Abre un diálogo con los **tipos de error y su conteo** (ej.
  `Out of Resources (1240)`, `Timeout (83)`): **tildás cuáles reintentar** y solo esos
  se reencolan. No recrea el job.
- **Validar** / **Actualizar** — ver §7.3 / refresco manual (también se refresca solo).
- **Borrar** / **Vaciar** (a la derecha, destructivos, requieren **ADMIN** + confirmación):
  **Borrar** elimina el job seleccionado (con sus estudios/errores); **Vaciar** borra
  todos los jobs que **no** estén RUNNING. No se puede borrar un job en ejecución.

La columna **Progreso** muestra el **% procesado** en vivo (ej. `80% — 4/10 (fallidos: 1,
omitidos: 1)`); la columna **Dedup** muestra la política de deduplicación del job.

> **Escala:** el refresco en vivo está **throttleado** y el drill-down es **paginado**
> (ver §7.2), así la UI **no se congela** en migraciones masivas (decenas de miles de
> estudios) — antes se tildaba por recargar toda la lista en cada evento.

> Tras un **cierre inesperado**, al reabrir la app los jobs que estaban corriendo
> quedan **PAUSED** y los estudios a medias vuelven a **PENDING**: reanudables con un click.

### 7.2 Drill-down de estudios (paginado, ordenable, filtrable)

Al seleccionar un job se listan sus estudios en una tabla **paginada** (200 por
página — nunca carga los 2,5M de golpe). Columnas: **Patient ID, Paciente, Accession,
Fecha, Descripción, Modalidad, Estado, Inst. recibidas / total, Reintentos, Study UID**
y **Error** (categoría del último error DICOM; mensaje completo en el tooltip).

- **Filtro de estado** (combo): Todos / Pendientes / Completados / Fallidos / Parciales /
  Fallidos+parciales / Omitidos — ideal para ir directo a los fallidos en un job grande.
- **Ordenar**: click en el encabezado ordena asc/desc **sobre todo el job** (no solo la
  página), server-side. Ej. ordenar por **Reintentos** o por **Estado**.
- **Navegación**: botones **◀ Anterior / Siguiente ▶** + contador *"Página X/Y — N estudios"*.

Estados posibles: `PENDING`, `RUNNING`, `COMPLETED`, `PARTIAL`, `FAILED`, `SKIPPED`.

> Los errores se **persisten en la DB apenas ocurren** (por estudio), así que aunque
> cierres la app por Task Manager **no se pierden**: los ves acá (filtrando "Fallidos")
> o en la solapa **Logs**.

### 7.3 Validar (posterior a la migración)

- **Validar** — compara, por estudio, **lo esperado vs lo encontrado en el destino**
  (C-FIND) y reporta discrepancias de conteo. No modifica datos.
- **Validar series** — sobre el estudio seleccionado: C-FIND nivel SERIES en origen y
  destino y reporta **qué series faltan** de un estudio ya forwardeado (útil ante
  errores tipo *out of resources* que dejan estudios incompletos — ver caso 10.4).

---

## 8. Solapa Dashboard

Monitoreo en una sola vista (panel superior **en vivo** + panel inferior **histórico**,
en un splitter ajustable).

- **En vivo:** tarjetas de jobs activos/completados/fallidos, throughput
  (estudios/min, imágenes/s, MB/s), totales procesados y **ETA** estimado. Se
  actualiza solo durante una migración.
- **Histórico (Estadísticas):** gráficos de barras (estudios/día, por modalidad, por
  método) y tablas (tiempo promedio de transferencia, rendimiento por par
  origen/destino). Botón **Actualizar** para recalcular. Si `pyqtgraph` falla, cada
  gráfico **degrada a una tabla equivalente** (no rompe la vista).

> C-MOVE reporta `MB/s = 0` (los píxeles no pasan por la app); C-GET/Store&Forward sí
> miden MB/s real.

---

## 9. Solapa Logs

Auditoría y diagnóstico unificados (errores + eventos en una grilla).

1. Filtrá por **Canal** (operativo / DICOM / técnico) y **Nivel** (DEBUG…CRITICAL).
2. **Actualizar** recarga (máx. 500 filas por fuente, carga asíncrona).
3. **Exportar CSV…** para llevarte el detalle.

Los errores de migración se clasifican por dominio/categoría/severidad y quedan
ligados al estudio que falló (nunca se traga un fallo en silencio). Para el detalle
crudo, mirá también los archivos en `data/logs/` (ej. `dicom.log`).

---

## 10. Casos de ejecución completos

### 10.1 Verificar conectividad con un PACS nuevo
1. **Configuración → Nuevo** → cargar AE/host/puerto → **Guardar**.
2. **Probar conexión** → esperar *C-ECHO OK*.
3. Si falla: revisar que el **AE local** esté dado de alta en el PACS y que el puerto
   sea correcto; subir timeouts si la red es lenta.

### 10.2 Migrar 10.000 estudios CT entre dos PACS (selección por query)
1. Probar C-ECHO de **origen** y **destino** (Configuración).
2. **Query / Retrieve**: origen, nivel STUDY, modalidad **CT**, rango de fechas → **Buscar**.
3. Seleccionar las filas → **Crear Job de migración**.
4. Asistente: destino, estrategia **C-MOVE directo** (si hay ruteo) o **Store &
   Forward**, **Workers** acorde a lo que el PACS tolera, dedup **Comparar**.
   Opcional: tildar **Dry run** primero para dimensionar.
5. **Jobs Históricos → Iniciar**. Monitorear en **Dashboard**.
6. Al terminar: **Validar** para confirmar conteos en destino.

### 10.3 Migrar desde un PACS con 2.500.000 estudios
1. **Migraciones → Por catálogo (masivo)**: origen, rango amplio de fechas,
   granularidad **mes** (o **semana** si hay mucho volumen por mes) → **Catalogar**.
   Dejarlo correr (es reanudable; se puede cancelar y retomar).
2. Cuando el catálogo está poblado: **filtrar** por modalidad/fechas → **Crear Job de
   migración masiva** con **Store & Forward** + dedup **Comparar**.
3. Iniciar en **Jobs Históricos**. El recorrido es por lotes con checkpoint; si se
   corta, reanuda sin perder lo hecho.

### 10.4 Detectar y completar un estudio con series faltantes
1. **Jobs Históricos** → seleccionar el job → en el drill-down, **filtrar por "Fallidos"
   o "Parciales"** (combo de estado) y/o **ordenar por la columna Error** para agrupar
   por tipo de fallo.
2. Seleccionar el estudio → **Validar series**: muestra qué `SeriesInstanceUID` faltan
   en destino.
3. Re-lanzar con dedup **Comparar** (re-transfiere lo incompleto, saltea lo presente).

### 10.4b Reintentar en masa una tanda de fallos del mismo tipo
1. **Jobs Históricos** → seleccionar el job → **Reintentar fallidos**.
2. En el diálogo, ves los **tipos de error con su conteo** (ej. `Out of Resources (1240)`).
   Tildá solo el/los que quieras (ej. reintentar *Out of Resources* cuando el PACS ya
   está descargado, y dejar los *Timeout* para otro momento).
3. Confirmá: se reencolan solo esos y el job reinicia. Repetí por tipo según convenga.

### 10.5 "El sitio del estudio aparece como unknown" tras migrar
- C-MOVE **no recorta** metadata: el destino recibe el objeto completo tal como está
  en el origen. Un campo "unknown" suele ser porque **el origen no lo tenía**, porque
  es un atributo de **nivel serie/instancia** (no aparece en una vista de nivel
  estudio), o porque **el destino lo coacciona** al recibir.
- Para confirmarlo: **DICOM Explorer → Ver headers completos** sobre una instancia del
  **origen** y del **destino**, y comparar el tag real (ej. `InstitutionName 0008,0080`).
- Para confirmar que el estudio se movió **completo**: estado **COMPLETED** en Jobs +
  **Validar** (conteo de instancias) y la línea del C-MOVE en `data/logs/dicom.log`.

### 10.6 Recuperar tras un cierre inesperado
1. Reabrir la app: los jobs en curso quedan **PAUSED**, los estudios a medias en
   **PENDING**.
2. **Jobs Históricos** → seleccionar el job → **Reanudar**.

### 10.7 Reenviar la misma lista a otro destino (sin re-resolver)
1. **Migraciones → Por lista**: resolvé la lista una vez → **Guardar lista…** con un nombre.
2. Más adelante (otro día, otro destino): **Cargar lista…** → elegí la lista guardada.
3. Se llena la grilla: seleccioná filas, **Crear Job** y elegí el **nuevo destino**.
   No hace falta volver a correr el resolver contra el origen.

### 10.8 Diagnosticar por qué no conecta un PACS
1. **Configuración** → seleccioná la conexión → **Ping**.
   - **Ping falla** → problema de **red** (host/puerto/firewall/ruta). Revisá eso.
   - **Ping OK pero C-ECHO falla** → la red está bien; el problema es **DICOM**: el
     **AE local** no está dado de alta en el PACS, o el AE remoto está mal.

---

## 11. Glosario y solución de problemas

| Síntoma | Causa probable / Acción |
|---|---|
| *C-ECHO falla* | AE local no dado de alta en el PACS, puerto/host errados, o firewall. Usá **Ping** para separar red de DICOM (ver [10.8](#108-diagnosticar-por-qué-no-conecta-un-pacs)). |
| *Un PACS satura / se cuelga con envíos masivos* | Bajá **Máx. asociaciones** de esa conexión (Configuración). La concurrencia real es `min(workers, ese valor)`. |
| *La UI se congela con un job de decenas de miles* | El drill-down es **paginado** y el refresco **throttleado**: si igual pasa, actualizá al build nuevo (versiones viejas recargaban toda la lista en cada evento). |
| *Muchos estudios FAILED del mismo tipo (ej. Out of Resources)* | **Reintentar fallidos** → elegí ese tipo en el diálogo (ver [10.4b](#104b-reintentar-en-masa-una-tanda-de-fallos-del-mismo-tipo)). |
| *No encuentro los fallidos en un job enorme* | En el drill-down, **filtro de estado → Fallidos** y/o **ordenar por la columna Error**. |
| *Quiero reenviar lo resuelto a otro destino* | **Guardar lista…** y luego **Cargar lista…** en Migraciones → Por lista (ver [10.7](#107-reenviar-la-misma-lista-a-otro-destino-sin-re-resolver)). |
| *No encuentro por qué un campo no filtra* | El **Nombre paciente** acepta wildcards (`DOE*`). La fecha solo filtra si tildás **Filtrar por fecha**. |
| *Una query no devuelve nada (y Explorer sí)* | Revisá nivel/modelo y que las fechas no sean un rango vacío. El Explorer fija nivel STUDY; igualá criterios. |
| *Binarios "no encontrado"* | Colocar DCMTK en `vendor/dcmtk/` (incluido `dicom.dic`). |
| *C-MOVE rechazado ("Move Destination Unknown")* | El **AE destino** no está configurado como nodo conocido en el **PACS origen**. |
| *Estudio queda PARTIAL/FAILED* | Ver columna **Error** en Jobs y la solapa **Logs**. Reintentar con **Comparar**; usar **Store & Forward** en redes inestables. |
| *Campos "unknown" en destino* | Ver caso [10.5](#105-el-sitio-del-estudio-aparece-como-unknown-tras-migrar). |
| *No quiero transferir aún, solo dimensionar* | Tildar **Dry run** en el asistente de Job. |
| *Una conexión nueva no aparece en otra solapa* | Las solapas refrescan sus combos al mostrarse; cambiá de pestaña y volvé. |

---

*DICOM Migration Manager — Pablo Bravo · pablosbrv@gmail.com*
