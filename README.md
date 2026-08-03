# DICOM Migration Manager

Aplicación de escritorio **portable** para administración DICOM y **migración masiva de estudios
entre PACS**. Sin instalación: se descarga, se descomprime y se ejecuta.

Pensada para administradores PACS e ingenieros de implementación que necesitan mover grandes
volúmenes de estudios de forma controlada, reanudable y auditable.

## ⬇️ Descargar

**[Descargar la última versión](https://github.com/pablosbravo/dicom-migration-manager/releases/latest)**
→ `DICOMMigrationManager-vX.Y.Z-win64.zip`

1. Descomprimir el ZIP en cualquier carpeta (por ejemplo `C:\DICOMMigrationManager`).
2. Ejecutar **`DICOMMigrationManager.exe`**.

No requiere Python, ni instalador, ni permisos de administrador. **DCMTK ya viene incluido.**

> Windows puede mostrar un aviso de SmartScreen porque el ejecutable no está firmado
> digitalmente: *Más información → Ejecutar de todas formas*.

## Qué hace

- **Conectividad** — alta de nodos DICOM (AE Title / host / puerto), C-ECHO con latencia,
  ping TCP e historial de pruebas.
- **DICOM Explorer** — navegación Estudio → Serie → Instancia con carga perezosa y visor
  completo de metadata.
- **Query / Retrieve** — C-FIND con filtros (paciente, fecha, modalidad, institución) y creación
  de jobs de migración desde los resultados.
- **Migraciones masivas** — por lista de UIDs (CSV / XLSX / TXT) o por catálogo barrido en
  streaming; estrategias C-MOVE y C-GET + Store & Forward.
- **Control de jobs** — pausar, reanudar y cancelar; reanudación automática tras un cierre
  inesperado; concurrencia ajustable por nodo para no saturar el PACS.
- **Resiliencia** — deduplicación, dry-run, reintentos con backoff y validación posterior
  comparando origen contra destino.
- **Dashboard y logs** — métricas en vivo e históricas, errores clasificados, auditoría y
  exportación a Excel.

Diseñada para escala: procesa por lotes y streaming, con checkpoint por estudio.

## Primeros pasos

1. Abrir la solapa **Configuración** y dar de alta el PACS origen (AE Title, host, puerto).
2. Pulsar **Probar conexión** — debe responder `C-ECHO OK`.
   - El AE Title local de la app debe estar **dado de alta en el PACS** para que acepte la asociación.
   - Para C-MOVE, el **AE destino** también debe estar configurado como nodo conocido en el PACS origen.
3. Dar de alta el PACS destino y repetir la prueba.
4. Buscar estudios en **Query / Retrieve** y crear un job, o usar **Migraciones** para volumen alto.
5. Seguir el avance en **Jobs Históricos** y **Dashboard**.

📖 **Manual completo, solapa por solapa, con casos de uso de punta a punta:
[GUIA_DE_USO.md](GUIA_DE_USO.md)**

## Antes de usarla en serio

> ⚠️ **Herramienta en desarrollo activo. Probala primero contra PACS de prueba, no de producción.**

- En el primer arranque se crea una carpeta **`data/`** junto al ejecutable, con la base SQLite,
  los logs y la caché. **Puede contener datos de pacientes**: tratala como información sensible y
  no la compartas.
- Una migración masiva genera carga real sobre los PACS. Ajustá *workers* y *máx. asociaciones*
  según lo que tolere cada servidor.
- El software se entrega **sin garantía** (ver licencia). La responsabilidad de validar los datos
  migrados es de quien la opera.

## Requisitos

- **Windows x64**.
- Conectividad de red hacia los PACS y los **AE Titles dados de alta** en cada extremo.
- Aproximadamente 250 MB de espacio en disco, más lo que ocupen la base y los logs.

## Licencia

Publicada bajo licencia [MIT](LICENSE) © Pablo Bravo.

El paquete distribuido incluye componentes de terceros con sus propias licencias
(DCMTK © OFFIS e.V., Qt/PySide6 bajo LGPL v3, entre otros). El detalle está en el archivo
`TERCEROS.txt` incluido dentro del ZIP.

## Contacto

Pablo Bravo — pablosbrv@gmail.com

¿Encontraste un bug o querés proponer una mejora?
Abrí un [issue](https://github.com/pablosbravo/dicom-migration-manager/issues).
