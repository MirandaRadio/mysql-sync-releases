# MySQL Sync · descargas

Este repositorio publica los **binarios e instaladores** de MySQL Sync, una herramienta para
comparar la **estructura** de esquemas MySQL / Amazon Aurora MySQL (tablas, columnas, claves
primarias y foráneas, índices, checks, triggers, vistas, rutinas y eventos) y generar el SQL que
deja cada esquema igual que el de referencia. Aquí solo hay releases: el código fuente se
mantiene en un repositorio privado.

La aplicación es de **solo lectura por defecto**: no escribe nada en una base de datos hasta que
confirmas explícitamente la ejecución del SQL, y antes de aplicar puede guardar una copia local de
los objetos afectados.

## Descargar

Ve a [Releases](../../releases/latest) y elige el fichero según tu caso:

| Fichero | Para qué |
|---|---|
| `mysql-sync-setup-vX.Y.Z-windows-x64.exe` | **Aplicación de escritorio para Windows** (recomendado). Instalador por usuario, sin permisos de administrador. Incluye actualización automática. |
| `mysql-sync-vX.Y.Z-windows-x64.zip` | Versión portable para Windows: `mysql-sync-gui.exe` (escritorio) y `mysql-sync.exe` (línea de comandos). Sin actualización automática. |
| `mysql-sync-vX.Y.Z-linux-x64.tar.gz` | Herramienta de línea de comandos para Linux x64 (`mysql-sync`), con el README y el CHANGELOG. |
| `*.sha256` | Suma SHA-256 de cada fichero, para comprobar la descarga. |
| `*.sig` y `latest.json` | Firma del instalador y manifiesto que usa la aplicación para actualizarse sola. No hace falta descargarlos. |

Requisitos: Windows 10/11 de 64 bits para la aplicación de escritorio; Linux x64 con glibc para la
CLI. Servidores MySQL 5.7 y 8.x y Aurora MySQL 2 y 3.

## Instalar y actualizar

1. Descarga el instalador `mysql-sync-setup-…-windows-x64.exe` y ejecútalo. Se instala en tu
   perfil de usuario y crea un acceso en el menú Inicio.
2. Cuando exista una versión nueva, la aplicación lo avisa en la barra lateral. Pulsa
   **Instalar y reiniciar**: descarga el instalador de este repositorio, comprueba su firma, lo
   instala en silencio y vuelve a abrirse con la nueva versión.
3. Las notas de cada versión están en la propia release y en el fichero `CHANGELOG.md` incluido
   en el `tar.gz`.

Si Windows SmartScreen muestra un aviso al ejecutar el instalador, es porque el ejecutable no
lleva firma de código de Microsoft (la firma que se verifica es la del actualizador, ver abajo).
Comprueba la suma SHA-256 y continúa con *Más información → Ejecutar de todas formas*.

## Verificar una descarga

Suma SHA-256 (PowerShell en Windows, `sha256sum` en Linux):

```powershell
Get-FileHash .\mysql-sync-setup-v0.1.4-windows-x64.exe -Algorithm SHA256
Get-Content .\mysql-sync-setup-v0.1.4-windows-x64.exe.sha256
```

```bash
sha256sum -c mysql-sync-v0.1.4-linux-x64.tar.gz.sha256
```

Firma del instalador ([minisign](https://jedisct1.github.io/minisign/)): cada `.exe` del
instalador va acompañado de un `.sig` generado con la clave privada del proyecto. La clave pública
es esta y no cambia entre versiones:

```
untrusted comment: minisign public key: 753E0EB1F994FBC6
RWTG+5T5sQ4+dY7WsfDn9vitA3XfmfFNI7B0dIretMA0s/XXd4ufMyLz
```

```bash
minisign -Vm mysql-sync-setup-v0.1.4-windows-x64.exe -P RWTG+5T5sQ4+dY7WsfDn9vitA3XfmfFNI7B0dIretMA0s/XXd4ufMyLz
```

La aplicación hace exactamente esta comprobación antes de instalar una actualización.

## Qué hace la aplicación

- **Proyectos**: una conexión y esquema de referencia frente a N esquemas de clientes. *Comparar
  todo* carga los esquemas en paralelo y muestra, por cliente, qué hay que crear, modificar o
  eliminar, en un árbol acción → tabla → columna con diff en una sola línea.
- **SQL de sincronización**: se genera a partir de los cambios que marques. Se puede copiar,
  guardar, hacer una **copia de seguridad local** de los objetos afectados y aplicarlo con
  confirmación, viendo el estado de cada sentencia en tiempo real.
- **Conexiones**: contraseña en el almacén de credenciales del sistema, variable de entorno,
  AWS Secrets Manager o token IAM de RDS/Aurora con la sesión local de AWS. TLS por defecto, con
  el bundle de CAs de Amazon RDS incluido. Entorno (producción / test / integración), carpetas
  multinivel e importación masiva desde JSON.
- **Seguridad**: todas las conexiones, credenciales y sentencias aplicadas quedan en un registro de
  auditoría local. Nada se ejecuta sin escribir el nombre del esquema destino.

### Línea de comandos

```bash
mysql-sync conn add prod -H cluster.xxxx.eu-west-1.rds.amazonaws.com -u sync_ro --ssl --keyring --ask-password
mysql-sync conn add cliente-a -H cliente-a.xxxx.eu-west-1.rds.amazonaws.com -u sync_ro --ssl --aws-iam
mysql-sync diff prod/app cliente-a/app                     # informe de texto
mysql-sync diff prod/app cliente-a/app --format sql > sync.sql
mysql-sync backup -c cliente-a -s app --script sync.sql    # copia local antes de aplicar
mysql-sync --help
```

Permisos mínimos en el servidor: `SELECT` sobre `information_schema` del esquema comparado y
`SHOW VIEW` / privilegios de rutina para leer definiciones de vistas y procedimientos. Para
aplicar el SQL o hacer copias con datos hacen falta, además, los privilegios correspondientes
sobre el esquema destino.

## Dónde guarda las cosas

| Qué | Windows | Linux |
|---|---|---|
| Conexiones y proyectos | `%APPDATA%\mysql-sync\config.toml` | `~/.config/mysql-sync/config.toml` |
| Registro de auditoría | `%LOCALAPPDATA%\mysql-sync\logs\audit.log` | `~/.local/share/mysql-sync/logs/audit.log` |
| Copias de seguridad | `%LOCALAPPDATA%\mysql-sync\backups\<conexión>\<fecha>\` | `~/.local/share/mysql-sync/backups/…` |

Las contraseñas nunca se guardan en `config.toml` desde la aplicación de escritorio: van al
Administrador de credenciales de Windows (o al llavero del sistema) o se piden al conectar.

## Incidencias

Abre una [issue](../../issues) en este repositorio indicando la versión (la ves en *Rutas y
versión*, en la barra lateral), el sistema operativo y, si es un error de comparación, la
definición de los objetos implicados. No incluyas contraseñas ni datos de clientes.

## Licencia

Software propietario. Su uso está sujeto a las condiciones acordadas con el propietario; la
redistribución de los binarios no está permitida.
