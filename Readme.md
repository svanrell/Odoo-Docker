# Despliegue y Configuración de Odoo 19 con Docker Compose

## 1. Conceptos Fundamentales de Docker

### 1.1 Comandos de Docker

A continuación se detalla la función técnica de los comandos y modificadores utilizados en la administración de contenedores:

#### Gestión del Ciclo de Vida de Contenedores
* **`docker run`**: Crea una capa de lectura/escritura sobre la imagen indicada y arranca el contenedor.
  * `-d` (*detach*): Ejecuta el contenedor en segundo plano, devolviendo el control de la terminal al anfitrión.
  * `-it` (`-i` interactivo + `-t` pseudo-TTY): Conecta la entrada estándar (stdin) y asigna un terminal para interactuar con el proceso o inspeccionar trazas en vivo.
  * `--name <nombre>`: Asigna un identificador alfanumérico fijo al contenedor para su referenciación por CLI o scripts.
  * `-p <puerto_host>:<puerto_contenedor>`: Publica y redirige tráfico TCP/UDP del puerto físico al contenedor mediante reglas de NAT en *iptables*.
  * `-v <origen>:<destino>`: Vincula un volumen o directorio del anfitrión (*bind mount*) en una ruta interna para persistencia de datos.
  * `-e <VARIABLE=VALOR>`: Inyecta variables de entorno en el espacio de usuario del proceso principal.
  * `--env-file <archivo>`: Carga en lote variables de entorno desde un fichero externo (ej. `.env`).
  * `--network <red>`: Conecta el contenedor a una red puente (*bridge*) definida por el usuario.
  * `--restart <política>`: Establece el comportamiento ante fallos o reinicio del host (`no`, `on-failure`, `unless-stopped`, `always`).
  * `--rm`: Destruye automáticamente el contenedor y su capa de escritura efímera al finalizar su ejecución.
  * `--user <UID:GID>`: Fija el usuario de ejecución del proceso interno (ej. `--user root` en desarrollo para resolver conflictos de permisos sobre volúmenes montados).
  * `--link <contenedor>:<alias>` (*legacy*): Mecanismo heredado de comunicación entre contenedores mediante entradas estáticas en `/etc/hosts`. Reemplazado por el DNS interno de las redes de Docker.
* **`docker ps` / `docker ps -a`**: Lista los contenedores activos o todos los existentes (incluyendo detenidos y estados de salida).
* **`docker stop` / `docker start` / `docker restart`**: Envía señales al proceso principal (`SIGTERM` con tiempo de gracia y posteriormente `SIGKILL`) para pausar, arrancar o reiniciar la instancia.
* **`docker exec -it <contenedor> <comando>`**: Ejecuta un proceso adicional dentro de un contenedor en ejecución (ej. `bash` para mantenimiento o `psql` para consultas directas).
* **`docker logs` / `docker logs -f`**: Extrae o sigue en tiempo real las salidas estándar (*stdout* y *stderr*) emitidas por el proceso principal (PID 1).
* **`docker rm` / `docker rm -f`**: Elimina contenedores detenidos o fuerza su parada y borrado.

#### Gestión de Imágenes
* **`docker images` / `docker image ls`**: Lista las imágenes descargadas o compiladas en el almacenamiento local de Docker.
* **`docker pull` / `docker push`**: Descarga o publica capas de imágenes desde/hacia registros de contenedores (Docker Hub o registros privados OCI).
* **`docker build -t <nombre:tag> .`**: Compila una nueva imagen siguiendo las directivas empaquetadas en un archivo `Dockerfile`.
* **`docker rmi`**: Elimina una imagen del almacenamiento local siempre que no existan contenedores asociados a sus capas.

#### Gestión de Almacenamiento y Redes
* **`docker volume (ls / create / inspect / rm)`**: Administra el ciclo de vida de los volúmenes gestionados en `/var/lib/docker/volumes/`.
* **`docker network (ls / create / inspect / rm)`**: Administra las redes virtuales y su configuración de aislamiento.

#### Orquestación con Docker Compose
* **`docker compose up -d`**: Lee el archivo de manifiesto (`compose.yml`), valida dependencias, crea volúmenes y redes, y arranca los servicios en segundo plano.
* **`docker compose down`**: Detiene los servicios, elimina los contenedores y destruye las redes creadas. La adición de la bandera `-v` elimina además los volúmenes asociados.
* **`docker compose ps`**: Muestra el estado operativo de los servicios definidos en el archivo de composición actual.
* **`docker compose logs -f`**: Consolida y monitoriza los registros de todos los servicios del proyecto simultáneamente.
* **`docker compose exec <servicio> <comando>`**: Ejecuta comandos dentro del servicio especificado en el stack.

#### Parámetros Adicionales de la Práctica
* **`--dev=all`**: Modificador de arranque del binario de Odoo que activa el modo de recarga en caliente de código Python y plantillas XML, e inhabilita la minificación de activos estáticos web.
* **`sudo chmod -R 777 <directorio>`**: Ajuste de permisos POSIX en el sistema anfitrión para permitir lectura y escritura indiscriminada a los usuarios internos del contenedor (`odoo:101`, `postgres:999`), evitando bloqueos de permisos en desarrollo.

---

### 1.2 Imágenes, Contenedores y Volúmenes

| Componente | Definición Técnica | Mecanismo de Funcionamiento |
| :--- | :--- | :--- |
| **Imagen** | Plantilla estática e inmutable de solo lectura conforme al estándar OCI (*Open Container Initiative*). | Construida en capas superpuestas mediante sistemas de archivos de unión (*UnionFS*, comúnmente *Overlay2*). Contiene el sistema base mínimo (*rootfs*), bibliotecas, dependencias, binarios y metadatos de ejecución (CMD, ENTRYPOINT, ENV). |
| **Contenedor** | Instancia aislada de ejecución generada a partir de una imagen. | Agrega una capa delgada de lectura y escritura (*container layer*) sobre las capas inmutables de la imagen. El aislamiento no utiliza hipervisores de virtualización de hardware, sino características nativas del kernel Linux:<br>• **Namespaces**: Aislamiento de procesos (PID), red (NET), puntos de montaje (MNT), IPC, UTS (hostname) y usuarios (USER).<br>• **Control Groups (cgroups)**: Monitorización y límite de consumo de recursos físicos (CPU, memoria, operaciones de E/S). |
| **Volumen** | Mecanismo de persistencia administrado por Docker independiente del ciclo de vida del contenedor. | Omite el sistema *Copy-on-Write* (CoW) del contenedor para escribir directamente en el almacenamiento persistente del anfitrión. |

#### Tipos de Montaje de Persistencia:
1. **Volúmenes Nombrados (*Named Volumes*)**: Gestionados completamente por el demonio de Docker en `/var/lib/docker/volumes/`. Proporcionan el mayor rendimiento de E/S, no sufren penalizaciones de traducción de sistema de archivos en entornos virtualizados (WSL2 / macOS) y aíslan los datos de manipulaciones accidentales en el host. Idóneos para motores de bases de datos relacionales (PostgreSQL).
2. **Montajes Vinculados (*Bind Mounts*)**: Mapean una ruta arbitraria del sistema anfitrión dentro de una ruta del contenedor. Fundamentales para entornos de desarrollo donde se requiere modificar código fuente localmente y reflejarlo de forma inmediata en el contenedor sin reconstruir la imagen.
3. **Montajes en Memoria (*tmpfs mounts*)**: Almacenamiento no persistente ubicado directamente en la memoria RAM del host. Empleado para datos volátiles o credenciales temporales donde la escritura en disco representa un riesgo de seguridad.

---

### 1.3 Bloques del Archivo compose.yml

El archivo `compose.yml` estructura la infraestructura dividiendo su configuración entre bloques globales y directivas a nivel de servicio:

#### Bloques Globales (Top-Level)
* **`services:`**: Contenedor principal donde se definen y configuran cada uno de los microservicios que componen la arquitectura.
* **`volumes:`**: Declara los volúmenes nombrados compartidos o persistentes que deben ser aprovisionados por el motor de Docker.
* **`networks:`**: Define las redes virtuales (típicamente controladas por el *driver* `bridge`) para aislar la comunicación interna de los contenedores.
* **`secrets:`**: Declara credenciales y claves sensibles gestionadas de forma segura que se montarán en memoria temporal dentro de los servicios.

#### Directivas de Servicio (Service-Level)
* **`image:`**: Especifica la imagen de contenedor y etiqueta exacta (*tag*) a descargar del registro (ej. `postgres:18`, `odoo:19`).
* **`container_name:`**: Asigna un nombre estático al contenedor en lugar de utilizar el esquema predeterminado `<proyecto>_<servicio>_<n>`.
* **`environment:`**: Define pares clave-valor inyectados directamente como variables de entorno del contenedor.
* **`env_file:`**: Especifica archivos externos de variables de entorno (ej. `.env`), separando la configuración del archivo de orquestación.
* **`secrets:`**: Asocia los secretos declarados a nivel global con el contenedor, haciéndolos accesibles bajo `/run/secrets/`.
* **`ports:`**: Expone puertos del contenedor al sistema anfitrión siguiendo la sintaxis `PUERTO_HOST:PUERTO_CONTENEDOR`.
* **`expose:`**: Hace que un puerto sea accesible únicamente para otros servicios conectados a la misma red interna, sin exponerlo al host exterior.
* **`volumes:`**: Asigna volúmenes nombrados o rutas locales (*bind mounts*) a directorios dentro del contenedor.
* **`networks:`**: Asocia el servicio a una o varias redes virtuales definidas.
* **`depends_on:`**: Controla el orden de inicialización de los servicios. Permite evaluar estados de salud (`condition: service_healthy`) para asegurar que una dependencia esté lista antes de arrancar el servicio consumidor.
* **`restart:`**: Define la política de reactivación del contenedor ante caídas o reinicios del servidor.
* **`command:`**: Sobrescribe la instrucción `CMD` por defecto configurada en la imagen base.
* **`profiles:`**: Agrupa servicios bajo etiquetas lógicas (ej. `dev`, `prod`), permitiendo arrancar selectivamente partes del stack.
* **`healthcheck:`**: Define un comando recurrente que verifica desde el interior del contenedor si el servicio se encuentra operativo y listo para recibir tráfico.
* **`deploy.resources:`**: Fija cuotas estrictas de recursos de hardware:
  * `limits`: Techo máximo absoluto de CPU y memoria RAM que el contenedor puede consumir.
  * `reservations`: Recursos mínimos garantizados por el anfitrión para el servicio.
* **`user:`**: Define el UID/GID con el que se ejecutará el proceso dentro del contenedor.

---

## 2. Eficiencia de Recursos y Gestión Segura de Credenciales

### 2.1 Optimización del Uso de Recursos

#### 1. Prevención del OOM Killer y Asignación de Cuotas
Tanto PostgreSQL como Odoo ejecutan operaciones que pueden experimentar picos transitorios de memoria (consultas analíticas pesadas, cálculo de balances o generación de informes PDF con QWeb). Si los contenedores no tienen límites definidos, un consumo descontrolado provocará que el kernel del sistema operativo active el **Out-Of-Memory Killer (OOM Killer)**, finalizando procesos críticos.
* **Contenedor PostgreSQL (`db`)**: Limitado a **1.5 CPUs** y **1024 MB de RAM**, con una reserva garantizada de **512 MB**.
* **Contenedor Odoo (`odoo-dev` / `odoo-prod`)**: Limitado a **2.0 CPUs** y **2048 MB de RAM**, con reserva de **1024 MB** en producción.

#### 2. Rendimiento de E/S: Named Volumes vs Bind Mounts
* **PostgreSQL (Base de Datos)**: Requiere escrituras constantes de alta velocidad para mantener la integridad de las transacciones (WAL y tablas). El uso de *bind mounts* en sistemas con capas de virtualización (como WSL2 en Windows o macOS) introduce una penalización de rendimiento crítica debido a la sincronización de sistemas de archivos no nativos. Por ello, se implementa un **volumen nombrado (`db_data`)**, que opera directamente sobre el sistema de archivos nativo de Docker con rendimiento de E/S óptimo.
* **Odoo (Módulos y Código)**: El uso de *bind mounts* se restringe exclusivamente al directorio de módulos propios (`volumes/addons`), donde la prioridad es la inmediatez de la edición de código sobre la velocidad de disco.

#### 3. Dimensionamiento y Cálculo de Workers en Odoo
Odoo emplea un servidor HTTP basado en Gevent / Werkzeug. Su rendimiento depende directamente de la asignación de trabajadores (*workers*):
* **Fórmula oficial de dimensionamiento de Workers**:
  $$\text{Workers} = (\text{Núcleos de CPU} \times 2) + 1$$
* **Fórmula de cálculo de memoria RAM requerida**:
  $$\text{RAM Requerida} = \text{Workers} \times ((\text{limit\_memory\_soft} \times 0.8) + (\text{limit\_memory\_hard} \times 0.2))$$
* **Entorno de Desarrollo**: Se establece `workers = 0`. Esto fuerza la ejecución en un **único hilo**, lo cual es indispensable para permitir la depuración interactiva (mediante `pdb` o herramientas de inspección), emitir trazas unificadas en la terminal y permitir la recarga en vivo sin bifurcación de procesos concurrentes.
* **Entorno de Producción**: Se fija `workers = 2` (o superior según CPU) y `max_cron_threads = 1`. De este modo, los procesos atienden peticiones HTTP concurrentes mientras un proceso independiente ejecuta las tareas automáticas del sistema (cron jobs) sin degradar la respuesta a los usuarios.

#### 4. Huella de Recursos y Rendimiento de Aplicación
En producción se elimina el modificador `--dev=all`. Este modificador obliga al framework a recompilar plantillas XML y recalcular hojas de estilo en cada solicitud; su exclusión en producción permite el uso de memoria caché en el navegador y en el servidor, reduciendo el uso de ciclos de reloj de CPU a una fracción.

---

### 2.2 Gestión Segura de Credenciales: .env vs Docker Secrets

| Nivel de Seguridad | Mecanismo | Nivel de Riesgo | Justificación Técnica |
| :--- | :--- | :--- | :--- |
| **Inseguro** | Variables en texto plano en `compose.yml` | **Crítico** | Expuestas en el historial de Git, en inspecciones de infraestructura (`docker compose config`) y visibles para cualquier persona con acceso al repositorio. |
| **Básico** | Archivo `.env` / `env_file` | **Medio** | Desacopla la configuración del código fuente y evita subir claves a Git (al incluirse en `.gitignore`). Sin embargo, las variables son inyectadas en el entorno global del contenedor, quedando expuestas en `docker inspect <container>` y legibles por cualquier proceso hijo en `/proc/1/environ`. |
| **Avanzado (Óptimo)** | **Docker Secrets** | **Alto (Recomendado)** | Las claves no se inyectan como variables de entorno del sistema. Docker monta las credenciales como archivos de solo lectura en un sistema de archivos temporal en memoria RAM (**tmpfs**) bajo `/run/secrets/<nombre_secreto>`. No dejan rastro en el disco del host, no son visibles mediante `docker inspect` y no se propagan a procesos no autorizados. |

#### Implementación Práctica de Docker Secrets
1. La contraseña de la base de datos se almacena en el archivo local `secrets/db_password.txt` (excluido en `.gitignore`).
2. En `compose.yml`, el servicio `db` monta el secreto `db_password`.
3. PostgreSQL implementa soporte nativo para la directiva `POSTGRES_PASSWORD_FILE=/run/secrets/db_password`, leyendo la credencial directamente desde el fichero en memoria RAM sin recibir la contraseña en texto plano por variable de entorno.

---

## 3. Comparativa de Configuración de Odoo: odoo.conf vs compose.yml

### 3.1 Tabla Comparativa de Capacidades

| Criterio de Comparación | Configuración vía `compose.yml` (Variables de Entorno) | Configuración vía `odoo.conf` (Archivo de Configuración) |
| :--- | :--- | :--- |
| **Alcance y Granularidad** | Reducido. El script de arranque oficial de Odoo solo procesa variables básicas de conexión (`HOST`, `PORT`, `USER`, `PASSWORD`). | **Total**. Permite parametrizar el núcleo completo del ERP (`addons_path`, `workers`, `limit_memory_*`, `limit_time_*`, `db_filter`, `proxy_mode`, `log_level`, configuración de correo SMTP, etc.). |
| **Seguridad de Acceso** | **Baja**. Las variables de entorno son auditables mediante `docker inspect`, aparecen en registros de diagnóstico y pueden filtrarse en volcados de memoria. | **Alta**. El archivo reside en el anfitrión con permisos restrictivos (`chmod 600` o `640`), se monta como solo lectura (`:ro`) y permite almacenar contraseñas hasheadas (`admin_passwd` con hash PBKDF2-SHA512). |
| **Agilidad en Desarrollo Local** | **Lenta**. Cualquier alteración de una variable requiere recrear el contenedor (`docker compose up --force-recreate`). | **Inmediata**. Permite modificar rutas de módulos, niveles de log o límites de tiempo editando el archivo directamente en el editor sin reiniciar la infraestructura de orquestación. |
| **Portabilidad y Desacoplamiento** | Altamente acoplada a Docker Compose. | Independiente de la plataforma (reutilizable en Docker, Kubernetes, VPS o entornos bare-metal). |

---

### 3.2 Orden Oficial de Precedencia en Odoo

Cuando un mismo parámetro se especifica en distintos orígenes, Odoo resuelve el valor aplicando la siguiente jerarquía de prioridad (de mayor a menor):

1. **Línea de comandos / Argumentos directos de arranque**: Valores pasados en el `command:` del contenedor (ej. `["odoo", "--dev=all"]`). Tienen la máxima prioridad.
2. **Variables de entorno del sistema**: Variables inyectadas en el contenedor (`environment:` o `.env`). Sobrescriben lo indicado en el archivo de configuración.
3. **Archivo de configuración (`/etc/odoo/odoo.conf`)**: Directivas leídas del archivo montado en el contenedor.
4. **Valores por defecto del framework**: Valores definidos internamente en el código fuente de Odoo si no se detecta ninguna parametrización externa.

---

### 3.3 Razonamiento de Criterio: Seguridad y Practicidad

* **¿Qué opción es más segura?**  
  **`odoo.conf` es más seguro.** En `compose.yml`, las credenciales e identificadores viajan por variables del kernel que pueden ser inspeccionadas por cualquier usuario o herramienta que ejecute `docker inspect` o lea `/proc/<PID>/environ`. Además, `odoo.conf` permite proteger la contraseña maestra del gestor de bases de datos de Odoo almacenando su hash criptográfico (`admin_passwd = $pbkdf2-sha512$...`), evitando exponer el secreto en claro.
* **¿Qué opción es más práctica para desarrollar en local?**  
  **`odoo.conf` es más práctico para desarrollo.** Centraliza la definición de rutas de módulos adicionales (`addons_path`), ajustes de registros detallados (`log_level = debug`) y ampliación de límites temporales (`limit_time_cpu = 600`) para evitar que el servidor cancele peticiones mientras se analizan breakpoints en depuración. Esto mantiene el manifiesto `compose.yml` limpio y centrado exclusivamente en la orquestación de red y almacenamiento.

---

## 4. Gestión de Entornos: Desarrollo vs Producción en Docker Compose

### 4.1 Diferencias Arquitectónicas entre Entornos

| Parámetro / Componente | Entorno de Desarrollo (`odoo-dev`) | Entorno de Producción (`odoo-prod`) | Justificación Técnica |
| :--- | :--- | :--- | :--- |
| **Modo de Ejecución** | `--dev=all` activo | `--dev=all` inhabilitado | Permite recarga dinámica de plantillas y código Python en dev; garantiza máxima velocidad y uso de caché en prod. |
| **Modelo de Workers** | `workers = 0` (Monohilo) | `workers = 2` (Multiproceso) | Monohilo permite depuración interactiva con `pdb`; multiproceso atiende tráfico concurrente sin bloqueos de peticiones. |
| **Hilos de Tareas Cron** | `max_cron_threads = 1` | `max_cron_threads = 1` dedicado | Separa la ejecución de tareas programadas del hilo principal en producción. |
| **Nivel de Registro (*Log Level*)** | `log_level = debug` | `log_level = info` | Registros detallados para trazabilidad en dev; registros limpios para evitar saturación de almacenamiento en prod. |
| **Límites de Tiempo (CPU/Real)** | 600s / 1200s (holgados) | 60s / 120s (estrictos) | Evita timeouts durante la inspección manual de código; en producción corta peticiones colgadas para liberar recursos. |
| **Puerto de Base de Datos** | Cerrado al exterior o local | Aislado en red interna | En prod la base de datos jamás debe ser accesible fuera de la red interna del stack Docker. |
| **Usuario de Ejecución** | `user: root` | Usuario no privilegiado (`odoo:101`) | En dev evita bloqueos de permisos al editar archivos; en prod aplica el principio de mínimo privilegio para mitigar escaladas. |
| **Política de Reinicio** | `restart: unless-stopped` | `restart: always` | Garantiza disponibilidad continua del servicio empresarial ante caídas imprevistas o reinicios del nodo. |
| **Soporte Proxy Inverso** | `proxy_mode = False` | `proxy_mode = True` | Habilita lectura de cabeceras `X-Forwarded-For` y `X-Forwarded-Proto` cuando opera tras Nginx/Traefik con SSL. |

---

### 4.2 Estrategia de Unificación con Perfiles de Docker Compose

Para evitar duplicar código en múltiples archivos y garantizar la coherencia del stack, se implementa la directiva nativa **`profiles`** de Docker Compose. El servicio de base de datos (`db`) es común a toda la infraestructura, mientras que los servicios de aplicación se activan bajo demanda:

* **Para desplegar el entorno de Desarrollo**:
  ```bash
  docker compose --profile dev up -d
  ```
  *(Inicia `db` y `odoo-dev`, aplicando `odoo-dev.conf` y `--dev=all`).*

* **Para desplegar el entorno de Producción**:
  ```bash
  docker compose --profile prod up -d
  ```
  *(Inicia `db` y `odoo-prod`, aplicando `odoo-prod.conf`, modo multiproceso y política `restart: always`).*

---

## 5. Seguridad y Gestión de Directorios en el Repositorio de Código

Para garantizar la integridad del código fuente, el cumplimiento de las normativas legales de protección de datos y evitar el almacenamiento innecesario de binarios pesados en el historial de Git, los directorios de trabajo se clasifican rigurosamente:

| Directorio | Estado en Git | Justificación Técnica y Normativa |
| :--- | :--- | :--- |
| **`volumes/addons/`** | **SÍ Versionado** | **Código fuente propio**. Contiene los módulos personalizados desarrollados para la empresa (modelos Python, vistas XML, reportes QWeb, controladores y flujos de negocio). Es el activo intelectual del proyecto y debe versionarse obligatoriamente para posibilitar trabajo colaborativo, control de versiones y canalizaciones de CI/CD. |
| **`volumes/filestore/`** | **NO Versionado** | **Binarios y datos dinámicos**. Almacena todos los archivos adjuntos indexados por su hash SHA1 (facturas en PDF, recibos, imágenes de producto, contratos escaneados).<br>1. *Rendimiento*: Provocaría un crecimiento desmesurado e irreversible del historial del repositorio Git (*repository bloat*).<br>2. *Legalidad y Seguridad*: Contiene datos de carácter personal y financiero. Versionarlo en un repositorio vulneraría directamente el **RGPD** y la **LOPDGDD**, exponiendo a la empresa a sanciones graves y fugas de información. |
| **`volumes/sessions/`** | **NO Versionado (Crítico)** | **Sesiones web activas**. Almacena los tokens y cookies de autenticación de los usuarios que han iniciado sesión en el ERP.<br>• *Riesgo Crítico de Secuestro de Sesión (Session Hijacking)*: Si este directorio se incluye en el repositorio, un atacante con acceso al código podría sustraer las cookies de sesión y autenticarse en el ERP con privilegios de administrador sin necesidad de conocer la contraseña maestra ni pasar por la pasarela de login. |
| **`volumes/postgres/`** (o volumen `db_data`) | **NO Versionado** | **Motor de Base de Datos (PGDATA)**. Almacena las tablas físicas relacionales en formato binario. Versionar estos archivos genera corrupción de transacciones, bloqueos de concurrencia e incompatibilidades entre distintas plataformas de hardware. |

---

## 6. Configuración Práctica del Proyecto

### 6.1 Estructura del Repositorio

```text
Entrega Docker/
├── .env.example                       # Plantilla de variables de entorno no sensibles
├── .gitignore                         # Reglas de exclusión de Git (secretos, binarios y sesiones)
├── README.md                          # Documentación técnica completa
├── compose.yml                        # Manifiesto de orquestación (perfiles dev y prod)
├── config/
│   ├── odoo-dev.conf                  # Configuración de Odoo para desarrollo
│   └── odoo-prod.conf                 # Configuración de Odoo para producción
├── secrets/
│   ├── .gitkeep                       # Mantiene el directorio en Git
│   ├── db_password.txt.example        # Plantilla de ejemplo de contraseña de BD
│   └── odoo_admin_password.txt.example # Plantilla de ejemplo de contraseña maestra
└── volumes/
    ├── addons/                        # Módulos propios (SÍ versionado en Git)
    │   └── .gitkeep
    ├── filestore/                     # Adjuntos de Odoo (ignorado en Git)
    │   └── .gitkeep
    └── sessions/                      # Sesiones de usuario (ignorado en Git)
        └── .gitkeep
```

---

### 6.2 Archivo `.gitignore`

```gitignore
.env
*.env.local

# Secretos
secrets/*.txt
!secrets/*.txt.example
!secrets/.gitkeep

# Persistencia Odoo y PostgreSQL
volumes/filestore/*
!volumes/filestore/.gitkeep
volumes/sessions/*
!volumes/sessions/.gitkeep
volumes/postgres/*
!volumes/postgres/.gitkeep

# Python / IDEs
**/__pycache__/
*.py[cod]
.venv/
.idea/
.vscode/
.DS_Store
```

---

### 6.3 Archivo `.env.example`

```bash
POSTGRES_DB=odoo
POSTGRES_USER=odoo
ODOO_HOST_PORT=8069
POSTGRES_DEV_PORT=5432
```

---

### 6.4 Archivos de Secretos (`secrets/`)

Para inicializar los secretos de Docker se crean los archivos correspondientes (protegidos contra subida a Git por `.gitignore`):

* `secrets/db_password.txt`:
  ```text
  SuperSecure_DB_Pass_2026!
  ```
* `secrets/odoo_admin_password.txt`:
  ```text
  SuperMaster_Admin_Pass_2026!
  ```

---

### 6.5 Archivos de Configuración de Odoo

#### `config/odoo-dev.conf` (Desarrollo)
```ini
[options]
addons_path = /mnt/extra-addons
data_dir = /var/lib/odoo

db_host = db
db_port = 5432
db_user = odoo
db_password = SuperSecure_DB_Pass_2026!

; 0 workers fuerza ejecucion monohilo para depuracion
workers = 0
max_cron_threads = 1

limit_time_cpu = 600
limit_time_real = 1200
limit_memory_hard = 2684354560
limit_memory_soft = 2147483648

log_level = debug
```

#### `config/odoo-prod.conf` (Producción)
```ini
[options]
addons_path = /mnt/extra-addons
data_dir = /var/lib/odoo

db_host = db
db_port = 5432
db_user = odoo
db_password = SuperSecure_DB_Pass_2026!

; Multiproceso para produccion
workers = 2
max_cron_threads = 1
proxy_mode = True

limit_time_cpu = 60
limit_time_real = 120
limit_memory_hard = 2684354560
limit_memory_soft = 2147483648

log_level = info
```

---

### 6.6 Manifiesto de Despliegue: `compose.yml`

```yaml
services:
  db:
    image: postgres:18
    container_name: odoo_postgres_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-odoo}
      POSTGRES_USER: ${POSTGRES_USER:-odoo}
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      PGDATA: /var/lib/postgresql/data/pgdata
    secrets:
      - db_password
    volumes:
      - db_data:/var/lib/postgresql/data/pgdata
    networks:
      - odoo_internal_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-odoo} -d ${POSTGRES_DB:-odoo}"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s
    deploy:
      resources:
        limits:
          cpus: '1.5'
          memory: 1024M
        reservations:
          memory: 512M

  odoo-dev:
    image: odoo:19
    container_name: odoo_app_dev
    profiles: ["dev"]
    user: root
    command: ["odoo", "--dev=all"]
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "${ODOO_HOST_PORT:-8069}:8069"
    secrets:
      - db_password
      - odoo_admin_password
    volumes:
      - ./config/odoo-dev.conf:/etc/odoo/odoo.conf:ro
      - ./volumes/addons:/mnt/extra-addons
      - ./volumes/filestore:/var/lib/odoo/filestore
      - ./volumes/sessions:/var/lib/odoo/sessions
    networks:
      - odoo_internal_network
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2048M

  odoo-prod:
    image: odoo:19
    container_name: odoo_app_prod
    profiles: ["prod"]
    restart: always
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "${ODOO_HOST_PORT:-8069}:8069"
    secrets:
      - db_password
      - odoo_admin_password
    volumes:
      - ./config/odoo-prod.conf:/etc/odoo/odoo.conf:ro
      - ./volumes/addons:/mnt/extra-addons
      - ./volumes/filestore:/var/lib/odoo/filestore
    networks:
      - odoo_internal_network
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2048M
        reservations:
          memory: 1024M

volumes:
  db_data:

secrets:
  db_password:
    file: ./secrets/db_password.txt
  odoo_admin_password:
    file: ./secrets/odoo_admin_password.txt

networks:
  odoo_internal_network:
    driver: bridge
```

---

## 7. Guía de Ejecución y Puesta en Marcha

### 1. Inicializar Variables de Entorno y Directorios
```bash
cp .env.example .env
mkdir -p secrets config volumes/addons volumes/filestore volumes/sessions
```

### 2. Configurar Secretos Locales
```bash
echo "SuperSecure_DB_Pass_2026!" > secrets/db_password.txt
echo "SuperMaster_Admin_Pass_2026!" > secrets/odoo_admin_password.txt
chmod 600 secrets/*.txt
```

### 3. Asignar Permisos en Volúmenes de Desarrollo
```bash
sudo chmod -R 777 volumes/addons volumes/filestore volumes/sessions
```
*(Previene discrepancias de permisos entre el usuario del host y los usuarios internos del contenedor de Odoo).*

### 4. Despliegue de los Servicios

* **Arranque en Modo Desarrollo**:
  ```bash
  docker compose --profile dev up -d
  ```

* **Arranque en Modo Producción**:
  ```bash
  docker compose --profile prod up -d
  ```

### 5. Verificación y Diagnóstico

* **Comprobar el estado y la salud de los contenedores**:
  ```bash
  docker compose ps
  ```

* **Inspeccionar registros en tiempo real**:
  ```bash
  docker compose logs -f
  ```

* **Acceso al Servicio Web**:
  Abra el navegador en `http://localhost:8069`. El asistente de inicialización de Odoo permitirá seleccionar la base de datos y comenzar la configuración inicial del ERP.
