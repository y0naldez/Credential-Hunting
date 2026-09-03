# CredsHunter — documentación en español

[English](../README.md) | **Español (este documento)**

CredsHunter es una herramienta de búsqueda local de credenciales en modo de sólo lectura para post-explotación autorizada, laboratorios, CTF y evaluaciones internas. Examina rutas locales y presenta contraseñas reutilizables, material de autenticación, contenedores, secretos cifrados y pistas que pueden conducir a otros artefactos.

- [`credshunter.sh`](../credshunter.sh): Linux.
- [`credshunter.ps1`](../credshunter.ps1): Windows.

No intenta autenticarse, no realiza password spraying, no explota servicios, no modifica los archivos examinados y no necesita acceso a la red. Úsalo únicamente con autorización.

## Cómo funciona

```text
Stage 1   Sistema y usuarios       registro, servicios, historiales, vaults, sesiones y workspaces
Stage 2   Contenedores             kdbx, ppk, pfx, keytab, axx, enc, gpg y archivos renombrados
Stage 3   Archivos de alto valor   llaves, .env, respaldos, bases de datos, capturas y configs
Stage 4   Nombres sospechosos      password, secret, credential, backup, vault, etc.
Stage 5   Contenido                patrones para credenciales reutilizables y secretos cifrados
```

Stage 1 comprueba ubicaciones conocidas del sistema y aplicaciones. Las etapas 2–5 recorren las rutas indicadas. Stage 5 analiza por defecto extensiones seleccionadas; `--all` o `-All` amplía el análisis a todo archivo de texto legible.

## Qué encuentra

| Categoría | Ejemplos |
|---|---|
| Credenciales directas | `password=...`, cadenas de conexión, basic auth, comandos e historiales |
| Hallazgos en servicios de Windows | Credenciales hardcodeadas en `ImagePath`, argumentos, URLs o valores de texto bajo `Parameters`; las cuentas de servicio no integradas se conservan como leads para revisión |
| Material de autenticación | Llaves SSH/PuTTY, PFX/P12, keytabs, SAM/SYSTEM y GPP `cpassword` |
| Contenedores | KeePass, `.axx`, `.enc`, `.gpg`, `.pgp` y archivos comprimidos renombrados |
| Secretos cifrados | Bloques con formato de vault, valores cifrados estructurados y datos secretos sellados o gestionados por aplicaciones |
| Referencias | Rutas encontradas en historiales, accesos directos, sesiones y workspaces |
| Artefactos de usuario | Estado guardado de editores y aplicaciones que requiere una revisión completa |
| Archivos interesantes | `.env`, respaldos, configuraciones, volcados y capturas |

La búsqueda de tokens cloud/SaaS se limita intencionalmente para evitar ruido. Algunos archivos locales de clientes cloud sí pueden aparecer como artefactos interesantes.

### Servicios de Windows

Stage 1 revisa `ImagePath`, `ObjectName` y los valores de texto de la subclave `Parameters` de cada servicio. Una contraseña literal presente en argumentos, URLs o campos con nombre de credencial se clasifica como `[HIGH]`. Un servicio ejecutado con una cuenta no integrada se presenta como pista para revisión, aunque no exista una contraseña visible.

Las contraseñas configuradas normalmente mediante el Service Control Manager se almacenan como secretos LSA protegidos. CredsHunter no intenta extraerlas y no debe interpretarse una cuenta de servicio como prueba de que su contraseña esté disponible en texto claro.

## Sesiones y archivos que requieren revisión

Una sesión guardada puede contener texto sin guardar, servidores, usuarios, comandos, rutas, notas y varias credenciales. Ambos motores reconocen formatos comunes de sesiones serializadas, workspaces, configuraciones, accesos directos y estado de aplicaciones sin exigir que el operador conozca cada ubicación específica de cada producto.

Estos archivos se conservan como leads aunque no exista una coincidencia `[HIGH]`:

```text
[LEAD] USER_ARTIFACT/app_session  <archivo-de-sesion>
```

Cuando una contraseña activa el hallazgo:

```text
[HIGH] app_session/password_assign  <archivo-de-sesion>: línea <N>
       Password: <valor-detectado> | REVIEW ENTIRE SESSION FILE; additional useful information or credentials may be present.
[LEAD] USER_ARTIFACT/app_session  <archivo-de-sesion>
```

La contraseña fue el disparador, pero se debe revisar el archivo completo. CredsHunter no reduce una sesión a una sola combinación host/usuario/password porque puede haber más destinos, comandos, rutas u otras credenciales.

## Tags y categorías

`CRITICAL`, `HIGH` y `KEY` son hallazgos directamente accionables. Un lead indica dónde continuar investigando; no confirma por sí solo una credencial reutilizable.

| Tag o categoría | Significado | Acción recomendada |
|---|---|---|
| `[CRITICAL]` | Contenedor confirmado o artefacto altamente sensible | Proteger y revisar el contenedor completo |
| `[HIGH]` | Password, hash, cadena de conexión, GPP `cpassword` o credencial de comando reutilizable | Validar sólo dentro del alcance y revisar el origen |
| `[KEY]` | Llave privada o material como SAM/SYSTEM | Proteger e identificar la cuenta o sistema relacionado |
| `ENCRYPTED_CREDENTIAL_LEAD` | Secreto cifrado que necesita llave, contraseña o descifrador | Identificar formato, llave y configuración asociada |
| `CREDENTIAL_LEAD` | Artefacto que probablemente contiene o apunta a credenciales | Revisar el artefacto y archivos cercanos |
| `REFERENCE` | Ruta encontrada en historial, sesión, workspace, recientes o shortcut | Seguir la ruta referenciada |
| `USER_ARTIFACT/app_session` | Sesión con posibles destinos, comandos, identidades o credenciales adicionales | Revisar el archivo completo |
| `[INTEREST]` | Archivo de alto valor sin credencial confirmada | Priorizar manualmente según categoría y ruta |
| `[NAME]` | Nombre de archivo sospechoso | Revisar; el nombre por sí solo no es evidencia |
| `[CHECK]` | Ubicación comprobada | Informativo; no indica hallazgo |
| `[SKIP]` | Archivo omitido por tamaño, binario, permisos o ruido | Ajustar opciones sólo si es relevante |

En modo limpio, las categorías de pistas aparecen bajo el tag visual `[LEAD]`:

```text
[LEAD] ENCRYPTED_CREDENTIAL_LEAD/encrypted_block  <archivo-cifrado>
[LEAD] CREDENTIAL_LEAD/referenced_file          /ruta/config
[LEAD] REFERENCE                                history -> /ruta/config
[LEAD] USER_ARTIFACT/app_session                <archivo-de-sesion>
```

Prioriza `[CRITICAL]`, `[HIGH]` y `[KEY]`; después revisa sesiones completas, sigue `REFERENCE`/`CREDENTIAL_LEAD` e identifica el mecanismo requerido por cada `ENCRYPTED_CREDENTIAL_LEAD`.

En modo limpio, las llaves privadas aparecen antes que las coincidencias de
password. Las coincidencias completamente comentadas se mueven a una sección
separada de pistas históricas. La vista omite documentación de paquetes y
dependencias instaladas (por ejemplo Ruby gems, archivos `.jar`,
`node_modules` y Python site-packages). También omite coincidencias basadas sólo
en el nombre para certificados públicos, scripts del sistema, logs rutinarios,
respaldos del gestor de paquetes y componentes de Cassandra sin datos de
autenticación. Las entradas PHP repetidas de catálogos de idioma, como
`ftp_login_pass => "FTP Password"`, también se filtran como ruido cuando la
misma clave aparece con traducciones diferentes en al menos tres archivos de
idioma del mismo directorio; los valores aislados siguen visibles. `NOISE_SUPPRESSED`
indica cuántos elementos crudos se ocultaron.
El modo completo los conserva para una revisión exhaustiva.

Los formatos cifrados explícitos como `.axx` se conservan como pistas de
credenciales cifradas. Los respaldos genéricos como `.zip` quedan como archivos
interesantes de menor prioridad. Los formatos conocidos basados en ZIP,
incluidos paquetes como `.jar`, `.war`, `.whl` y `.nupkg`, no se convierten en
pistas sólo por su contenedor. Los nombres sospechosos y el contenido con
credenciales se siguen reportando por separado.

El modo limpio muestra todas las entradas restantes. Ningún hallazgo o pista se
oculta detrás de un límite de presentación. Las credenciales idénticas que se
repiten en el mismo archivo se muestran una vez con todas sus líneas y el total
de apariciones.

La detección de llaves se basa en el contenido, no sólo en el nombre: un archivo
sin extensión como `~/.ssh/keys/root` se clasifica como `[KEY]` si contiene una
cabecera privada real. Las cadenas PEM incrustadas dentro de código fuente y los
keyrings públicos de gestores de paquetes no se clasifican como credenciales.
El ruido de baja confianza suprimido tampoco cambia a `1` el código de salida en
modo limpio.

## Uso

Linux:

```bash
sudo ./credshunter.sh -p / -o loot.txt
./credshunter.sh -p /home -p /var/www -p /opt --clean --no-color -o loot.txt
./credshunter.sh -p /home -x /home/user/cache
./credshunter.sh -p /home --all
```

Windows:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\credshunter.ps1 `
  -Path C:\Users -Clean -NoColor -OutputFile .\loot.txt

powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\credshunter.ps1 `
  -Path C:\Users,C:\inetpub -OutputFile .\loot.txt

powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\credshunter.ps1 `
  -Path D:\ -IncludeData
```

No es obligatorio usar `root` o Administrador, pero los privilegios elevados permiten leer más perfiles y ubicaciones protegidas.

## Opciones principales

| Objetivo | Linux | Windows |
|---|---|---|
| Ruta | `-p PATH`, `--path PATH` | `-Path C:\Ruta` |
| Exclusión | `-x PATH`, `--exclude PATH` | `-ExcludePath C:\Ruta` |
| Modo limpio | `--clean` | `-Clean` |
| Sin colores | `--no-color` | `-NoColor` |
| Guardar log | `-o loot.txt` | `-OutputFile .\loot.txt` |
| Todo texto legible | `-a`, `--all` | `-All` |
| Tamaño máximo | `-m N`, `--max-size N` | `-MaxFileSizeMB N` |
| Sin límite normal | `--no-size-limit` | `-NoSizeLimit` |
| Omitir Stage 1 | `-s`, `--skip-system`, `--no-stage1` | `-SkipSystem`, `-NoStage1` |
| Omitir etapa | `--no-stageN` | `-NoStageN` |
| Incluir SQL/CSV | `--all` | `-IncludeData` |

El límite predeterminado es 5 MB. Usa `--no-color` / `-NoColor` al redirigir o procesar resultados. El log puede contener secretos en texto claro: protégelo como evidencia sensible y elimínalo de forma segura cuando deje de ser necesario.

## Códigos de salida

| Código | Significado |
|---|---|
| `0` | Sin `CRITICAL`, `HIGH` o `KEY`; puede haber leads/intereses |
| `1` | Al menos un `CRITICAL`, `HIGH` o `KEY` |
| `2` | Error de argumentos, entrada/salida o error fatal |
| `130` | Interrumpido con Ctrl+C o señal equivalente |

Los leads, intereses y nombres sospechosos no cambian por sí solos `0` a `1`.

## Solución de problemas

- Si falta una contraseña, confirma ruta, permisos, tamaño, exclusiones y extensión; prueba `--all` / `-All`.
- Si aparecen muchos resultados, limita las rutas, conserva el límite de tamaño y usa modo limpio.
- Si aparece `REVIEW ENTIRE SESSION FILE`, abre la ruta indicada y revisa todo el artefacto autorizado.
- Si un archivo aparece como `[SKIP]`, revisa la razón antes de ampliar límites.

## Uso responsable y contribuciones

Utiliza CredsHunter únicamente con autorización. Los patrones y listas de tipos están en secciones identificadas cerca del inicio de ambos scripts. Al contribuir, valida sintaxis PowerShell/Bash y prueba casos positivos, falsos positivos y contenido serializado.
