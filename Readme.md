# Despliegue y Configuración de Odoo 19 con Docker Compose


## 1. Conceptos Fundamentales de Docker

### 1.1 Comandos de Docker

A continuación se detalla la función técnica de los comandos y modificadores utilizados en la administración de contenedores:

#### Gestión del Ciclo de Vida de Contenedores
* **`docker run`**: Crea una capa de lectura/escritura sobre la imagen indicada y arranca un contenedor nuevo.
  * `-d` (*detach*): Ejecuta el contenedor en segundo plano, liberando la terminal para seguir ejecutando comandos mientras el servicio corre.
  * `-it` (`-i` interactivo + `-t` pseudo-TTY): Conecta la entrada estándar (stdin) y asigna un terminal. Si arrancamos sin `-d`, nos permite ver los logs en vivo en la consola durante el arranque para diagnosticar errores rápidamente.
  * `--name <nombre>`: Asigna un identificador alfanumérico fijo al contenedor en lugar de asignarle un identificador o hash aleatorio.
  * `-p <puerto_host>:<puerto_contenedor>`: Redirige puertos de la máquina anfitriona hacia dentro del contenedor mediante reglas de NAT (ej. `-p 8069:8069` nos permite abrir Odoo directamente desde el navegador en `http://localhost:8069`).
  * `-v <origen>:<destino>`: Vincula un volumen o directorio del anfitrión (*bind mount*) en una ruta interna para persistencia de datos o sincronización de código.
  * `-e <VARIABLE=VALOR>`: Inyecta variables de entorno en el espacio de usuario del proceso (ej. credenciales de conexión a base de datos).
  * `--env-file <archivo>`: Carga en lote variables de entorno desde un fichero externo (ej. `.env`), evitando exponerlas en comandos.
  * `--network <red>`: Conecta el contenedor a una red virtual definida por el usuario para aislar su tráfico.
  * `--restart <política>`: Establece el comportamiento ante caídas o reinicio del host (`no`, `on-failure`, `unless-stopped`, `always`).
  * `--rm`: Destruye automáticamente el contenedor y su capa de escritura efímera al finalizar su ejecución.
  * `--user <UID:GID>`: Fuerza a que los procesos internos se ejecuten bajo un usuario concreto (ej. `--user root` en desarrollo para resolver conflictos de permisos sobre carpetas montadas).
  * `--link <contenedor>:<alias>` (*legacy*): Mecanismo antiguo para comunicar contenedores por nombre de host en `/etc/hosts`. Actualmente en desuso porque Docker Compose gestiona redes internas automáticamente donde los servicios se resuelven por su nombre de servicio.
* **`docker ps` / `docker ps -a`**: Lista los contenedores activos o todos los existentes (incluyendo detenidos y estados de salida).
* **`docker stop` / `docker start` / `docker restart`**: Pausa, arranca o reinicia la instancia enviando señales al proceso principal (`SIGTERM` con tiempo de espera y posteriormente `SIGKILL`).
* **`docker exec -it <contenedor> <comando>`**: Ejecuta un comando interactivo dentro de un contenedor en ejecución (ej. `bash` para mantenimiento o `psql` para consultas directas a PostgreSQL).
* **`docker logs` / `docker logs -f`**: Consulta o sigue en tiempo real (*follow*) los registros estándar emitidos por el proceso principal (PID 1).
* **`docker rm` / `docker rm -f`**: Elimina contenedores detenidos o fuerza su parada y borrado.

#### Gestión de Imágenes
* **`docker images` / `docker image ls`**: Lista las imágenes descargadas o compiladas en el almacenamiento local de Docker.
* **`docker pull`**: Descarga una imagen desde Docker Hub a la máquina local sin ejecutarla inmediatamente.
* **`docker push`**: Publica capas de imágenes hacia un registro de contenedores (Docker Hub o registro privado OCI).
* **`docker build -t <nombre:tag> .`**: Compila una nueva imagen siguiendo las instrucciones de un archivo `Dockerfile`.
* **`docker rmi`**: Elimina una imagen del almacenamiento local siempre que ningún contenedor dependa de ella.

#### Gestión de Almacenamiento y Redes
* **`docker volume (ls / create / inspect / rm)`**: Administra el ciclo de vida de los volúmenes gestionados en `/var/lib/docker/volumes/`.
* **`docker network (ls / create / inspect / rm)`**: Administra las redes virtuales y su configuración de aislamiento.

#### Orquestación con Docker Compose
* **`docker compose up -d`**: Lee el archivo de manifiesto (`compose.yml`), valida dependencias, crea volúmenes y redes, y arranca todos los servicios en segundo plano a la vez.
* **`docker compose down`**: Detiene y elimina los contenedores y redes creados por Compose, manteniendo los volúmenes para no perder datos (salvo que se añada la bandera `-v`).
* **`docker compose ps`**: Muestra el estado operativo de los servicios definidos en el archivo de composición actual.
* **`docker compose logs -f`**: Consolida y monitoriza los registros de todos los servicios del proyecto simultáneamente en una sola terminal.
* **`docker compose exec <servicio> <comando>`**: Ejecuta comandos dentro del servicio especificado en el stack.

#### Parámetros Adicionales de la Práctica
* **`--dev=all`**: Modificador de arranque de Odoo que activa la recarga en caliente de código Python y plantillas XML sin reiniciar el servicio, y desactiva la minificación de activos estáticos web (CSS/JS).
* **`sudo chmod -R 777 <directorio>`**: Ajuste de permisos en el sistema anfitrión para permitir lectura y escritura a los usuarios internos del contenedor (`odoo:101`, `postgres:999`), evitando bloqueos de permisos en desarrollo.

---

### 1.2 Imágenes, Contenedores y Volúmenes

* **¿Qué es una imagen?**:  
  Una imagen es una plantilla inmutable de solo lectura construida conforme al estándar OCI (*Open Container Initiative*). Contiene el sistema operativo base mínimo (*rootfs*), bibliotecas, dependencias, binarios y metadatos de ejecución (CMD, ENTRYPOINT, variables). Se construye mediante capas superpuestas (*UnionFS / Overlay2*) y sirve como molde inalterable para instanciar contenedores.

* **¿Qué es un contenedor?**:  
  Es una instancia en ejecución de una imagen. Funciona como un entorno de ejecución aislado (una "caja" de procesos) que comparte el kernel de la máquina anfitriona sin virtualizar hardware completo. El aislamiento se logra mediante dos mecanismos del kernel Linux:
  * **Namespaces**: Proveen aislamiento de procesos (PID), red (NET), puntos de montaje (MNT), usuarios (USER) e IPC.
  * **Control Groups (cgroups)**: Limitan y monitorizan el consumo de hardware físico (CPU, memoria RAM, operaciones de E/S).  
  > **Nota sobre efimeridad**: Los contenedores son **efímeros por diseño**. Añaden una capa delgada de lectura/escritura sobre la imagen; si se detiene o destruye el contenedor sin haber vinculado un volumen persistente, todos los datos y modificaciones internas se pierden de forma irreversible.

* **¿Qué es un volumen?**:  
  Es el mecanismo nativo que proporciona Docker para almacenar datos de forma persistente fuera del ciclo de vida del contenedor, omitiendo el sistema *Copy-on-Write* (CoW) para escribir directamente en el almacenamiento persistente del anfitrión. Permite que, aunque destruyamos, recreemos o actualicemos los contenedores de Odoo o PostgreSQL, la base de datos y los ficheros subidos por los usuarios permanezcan intactos.

#### Tipos de Montaje de Persistencia:
1. **Volúmenes Nombrados (*Named Volumes*)**: Gestionados completamente por el demonio de Docker en `/var/lib/docker/volumes/`. Proporcionan el mayor rendimiento de E/S, no sufren penalizaciones de traducción de sistema de archivos en entornos virtualizados (WSL2 / macOS) y aíslan los datos de manipulaciones accidentales en el host. Idóneos para bases de datos relacionales (PostgreSQL).
2. **Montajes Vinculados (*Bind Mounts*)**: Mapean una ruta arbitraria de nuestra máquina física dentro del contenedor. Son fundamentales en desarrollo para sincronizar código fuente localmente y reflejarlo de forma inmediata sin reconstruir la imagen.
3. **Montajes en Memoria (*tmpfs mounts*)**: Almacenamiento no persistente ubicado directamente en la memoria RAM del host. Empleado para datos volátiles o credenciales temporales (Docker Secrets) donde la escritura en disco representaría un riesgo de seguridad.

---

### 1.3 Bloques del Archivo compose.yml

El archivo `compose.yml` permite definir toda la infraestructura multi-contenedor de forma declarativa. Sus bloques se dividen en:

#### Bloques Globales (Top-Level)
* **`services:`**: Bloque raíz donde se declaran cada uno de los componentes o contenedores de la aplicación (la base de datos `db` y los servicios de aplicación).
* **`volumes:`**: Declara volúmenes nombrados independientes gestionados directamente por el motor de Docker para persistir datos.
* **`networks:`**: Configura redes virtuales internas para aislar la comunicación interna entre contenedores.
* **`secrets:`**: Declara credenciales y claves sensibles que se montarán en memoria temporal dentro de los servicios.

#### Directivas de Servicio (Service-Level)
* **`image:`**: Especifica la imagen base a descargar de Docker Hub, indicando su versión o etiqueta (ej. `postgres:18`, `odoo:19`).
* **`container_name:`**: Asigna un nombre fijo al contenedor en el sistema operativo.
* **`environment:`**: Inyecta variables necesarias para la configuración y conexión interna entre servicios (como `HOST=db`, `USER=odoo`).
* **`env_file:`**: Permite cargar variables desde uno o varios archivos externos (como `.env`), desacoplando la parametrización del código YAML.
* **`secrets:`**: Asocia los secretos declarados globalmente con el contenedor, haciéndolos accesibles bajo `/run/secrets/`.
* **`ports:`**: Define el mapeo de puertos entre el host y el contenedor siguiendo el formato `"PUERTO_HOST:PUERTO_CONTENEDOR"` (ej. `"8069:8069"`).
* **`volumes:`**: A nivel de servicio asocia rutas locales (*bind mounts*) o volúmenes nombrados para persistir addons, filestore o las tablas de PostgreSQL.
* **`networks:`**: Conecta el servicio a redes virtuales internas para comunicarse de forma segura con otros contenedores del stack.
* **`depends_on:`**: Marca el orden de inicio de los servicios. Asegura que la base de datos arranque y esté saludable (`condition: service_healthy`) antes de que Odoo intente conectarse.
* **`restart:`**: Define la política de recuperación ante caídas del proceso o reinicios del host (`no`, `on-failure`, `unless-stopped`, `always`).
* **`command:`**: Sobrescribe el comando por defecto de la imagen (utilizado en desarrollo para pasar modificadores como `["odoo", "--dev=all"]`).
* **`profiles:`**: Permite etiquetar servicios para activarlos selectivamente (ej. `dev` o `prod`) sin requerir archivos YAML independientes.
* **`healthcheck:`**: Define un comando recurrente que comprueba desde el interior del contenedor si el servicio está operativo y listo para recibir tráfico.
* **`deploy.resources:`**: Fija cuotas de hardware (`limits` máximos y `reservations` mínimas de CPU y RAM) para evitar caídas del sistema.
* **`user:`**: Define el usuario de ejecución del proceso interno (ej. `root`).

---

## 2. Eficiencia de Recursos y Gestión Segura de Credenciales

### 2.1 Optimización del Uso de Recursos

1. **Límites de CPU y RAM (Prevención del OOM Killer)**:  
   Por defecto, un contenedor puede consumir toda la memoria del host. Tanto PostgreSQL como Odoo ejecutan operaciones que pueden experimentar picos transitorios de memoria (consultas complejas o generación de informes PDF con QWeb). Sin cuotas, el kernel activa el **Out-Of-Memory Killer (OOM Killer)** y finaliza servicios críticos. Mediante `deploy.resources.limits` fijamos topes de contención:
   * **PostgreSQL (`db`)**: Límite de **1.5 CPUs** y **1024 MB de RAM** (reserva de 512 MB).
   * **Odoo (`odoo-dev` / `odoo-prod`)**: Límite de **2.0 CPUs** y **2048 MB de RAM** (reserva de 1024 MB en producción).

2. **Rendimiento de E/S: Named Volumes vs Bind Mounts**:  
   Para la base de datos (PostgreSQL), el uso de *bind mounts* en sistemas con capas de virtualización (como WSL2 o macOS) introduce una penalización severa de rendimiento debido a la traducción del sistema de archivos. Por ello, la base de datos utiliza un **volumen nombrado (`db_data`)** que escribe a velocidad nativa, reservando los *bind mounts* locales únicamente para el desarrollo de módulos en `volumes/addons`.

3. **Cálculo y Dimensionamiento de Workers en Odoo**:  
   Odoo gestiona la concurrencia mediante trabajadores (*workers*):
   * **Fórmula oficial de Workers**: $\text{Workers} = (\text{Núcleos de CPU} \times 2) + 1$
   * **Memoria requerida**: $\text{RAM} = \text{Workers} \times ((\text{limit\_memory\_soft} \times 0.8) + (\text{limit\_memory\_hard} \times 0.2))$
   * **En desarrollo (`workers = 0`)**: Se fuerza la ejecución en un **único hilo**. Esto es indispensable para permitir la depuración interactiva (usar `pdb` o breakpoints sin bloqueos), ver las trazas unificadas en la terminal y recargar código en vivo sin bifurcación de procesos.
   * **En producción (`workers = 2`, `max_cron_threads = 1`)**: Se activa el modo multiproceso para atender múltiples peticiones concurrentes y desacoplar las tareas automáticas del sistema (cron jobs).

4. **Optimización en Producción**:  
   Se excluye el parámetro `--dev=all`. En producción, deshabilitar el modo dev permite que el navegador y el servidor cacheen los activos web y no recompilen plantillas XML en cada petición, ahorrando ciclos de CPU.

---

### 2.2 Gestión Segura de Variables de Entorno: .env vs Docker Secrets

| Nivel de Seguridad | Mecanismo | Nivel de Riesgo | Justificación Técnica |
| :--- | :--- | :--- | :--- |
| **Inseguro** | Variables en texto plano en `compose.yml` | **Crítico** | Expuestas en el historial de Git, en inspecciones de infraestructura (`docker compose config`) y visibles para cualquiera con acceso al repositorio. |
| **Básico** | Archivo `.env` / `env_file` | **Medio** | Desacopla credenciales del código y evita subirlas a Git (mediante `.gitignore`). Sin embargo, las variables quedan inyectadas en el entorno global del contenedor, siendo visibles con `docker inspect <container>` o leyendo `/proc/1/environ`. |
| **Avanzado (Óptimo)** | **Docker Secrets** | **Alto (Recomendado)** | Las claves no se inyectan como variables de entorno del sistema. Docker monta las credenciales como archivos de solo lectura en memoria RAM (**tmpfs**) bajo `/run/secrets/<nombre_secreto>`. No dejan rastro en el disco del host, no son visibles en `docker inspect` y ofrecen el máximo aislamiento. |

#### Implementación Práctica:
PostgreSQL implementa soporte nativo mediante `POSTGRES_PASSWORD_FILE=/run/secrets/db_password`, leyendo la contraseña directamente desde el archivo montado en memoria sin exponerla jamás en variables de entorno.

---

## 3. Configuración de Odoo: odoo.conf vs variables en compose.yml

### 3.1 ¿Qué diferencias hay entre configurar en compose.yml y odoo.conf?

* **`compose.yml` (environment)**: Se utiliza principalmente para la infraestructura y conectar contenedores entre sí (definir el host de la base de datos `HOST=db`, usuario, contraseña y puertos expuestos). Su alcance está limitado a las variables básicas reconocidas por el script de entrada del contenedor.
* **`odoo.conf`**: Es el archivo de configuración interno de Odoo para parametrizar la aplicación en sí. Permite un control exhaustivo del ERP: definir las rutas de módulos (`addons_path`), la contraseña maestra (`admin_passwd`), el número de procesos (`workers`), límites de memoria, filtros de base de datos (`db_filter`), modo proxy y niveles de registro.

| Criterio | Configuración vía `compose.yml` | Configuración vía `odoo.conf` |
| :--- | :--- | :--- |
| **Alcance** | Básico (conexión a red y base de datos). | Exhaustivo (motor del ERP, addons, workers, logs). |
| **Seguridad** | Media/Baja (visible en `docker inspect`). | Alta (permisos `chmod 600`, montado en `:ro`). |
| **Desarrollo** | Lento (obliga a recrear el contenedor ante cambios). | Inmediato (permite editar rutas o logs sin reiniciar Compose). |
| **Portabilidad** | Acoplado a Docker Compose. | Independiente (exportable a VPS, contenedores o bare-metal). |

---

### 3.2 ¿Qué opción es más segura?

**Es más seguro `odoo.conf`.**  
Si se definen contraseñas en las variables de entorno de `compose.yml`, cualquier usuario con acceso al demonio de Docker puede verlas en texto plano ejecutando `docker inspect <contenedor>`.  
En cambio, con `odoo.conf`:
1. El archivo reside en el anfitrión con permisos restringidos de Linux (`chmod 600`), accesible únicamente por el usuario correspondiente.
2. Se monta dentro del contenedor en modo de solo lectura con `:ro` (*read only*), impidiendo que se modifique desde el contenedor.
3. Permite almacenar la contraseña maestra hasheada criptográficamente (`admin_passwd = $pbkdf2-sha512$...`), evitando exponer el secreto en claro.

---

### 3.3 ¿Y para desarrollar en local, qué opción es más práctica?

**Es mucho más práctico `odoo.conf`.**  
Durante el desarrollo de nuevos módulos, es habitual necesitar cambiar rutas en `addons_path`, ajustar el nivel de trazas a `log_level = debug` o ampliar los límites de tiempo (`limit_time_cpu = 600`) para que el servidor no cancele la petición mientras estamos analizando el código en un depurador.  
Con `odoo.conf`, basta con editar el archivo localmente desde nuestro editor de código (VSCode) y reiniciar el servicio de Odoo, sin tener que alterar el manifiesto de infraestructura `compose.yml` ni recrear contenedores.

---

### 3.4 Orden Oficial de Precedencia en Odoo

Cuando un mismo parámetro se define en distintos sitios, Odoo resuelve el conflicto aplicando el siguiente orden de prioridad (de mayor a menor):

1. **Argumentos de línea de comandos**: Valores pasados en el `command:` del contenedor (ej. `["odoo", "--dev=all"]`).
2. **Variables de entorno**: Variables inyectadas en el contenedor (`environment:` o `.env`).
3. **Archivo de configuración (`/etc/odoo/odoo.conf`)**: Directivas leídas del archivo montado.
4. **Valores por defecto**: Valores predeterminados en el código fuente de Odoo.

---

## 4. Configuración en Producción vs Desarrollo y cómo configurarlo en Compose

### 4.1 Diferencias entre la Configuración de Producción y Desarrollo

| Parámetro | Entorno de Desarrollo (`odoo-dev`) | Entorno de Producción (`odoo-prod`) | Justificación Técnica |
| :--- | :--- | :--- | :--- |
| **Modo de Ejecución** | `--dev=all` activo | `--dev=all` inhabilitado | Permite recarga dinámica de plantillas y código Python en dev; garantiza máxima velocidad y caché web en prod. |
| **Workers (Procesos)** | `workers = 0` (Monohilo) | `workers = 2` (Multiproceso) | Monohilo permite depuración interactiva con `pdb`; multiproceso atiende tráfico concurrente sin bloqueos. |
| **Tareas Cron** | Compartidas en hilo principal | `max_cron_threads = 1` dedicado | Separa la ejecución de tareas programadas del hilo HTTP en producción. |
| **Nivel de Logs** | `log_level = debug` | `log_level = info` | Registros detallados para diagnosticar en dev; registros limpios para no saturar disco en prod. |
| **Límites de Tiempo** | 600s / 1200s (holgados) | 60s / 120s (estrictos) | Evita timeouts durante la inspección manual de código; en producción corta peticiones colgadas. |
| **Puerto PostgreSQL** | Puerto 5432 accesible en local | Aislado en red interna | En producción la base de datos jamás debe ser accesible fuera de la red interna privada del stack. |
| **Usuario Interno** | `user: root` | Usuario sin privilegios (`odoo:101`) | En dev evita problemas de permisos en carpetas; en prod aplica el principio de mínimo privilegio. |
| **Política de Reinicio** | `restart: unless-stopped` | `restart: always` | Garantiza disponibilidad continua del servicio empresarial ante caídas imprevistas del host. |
| **Proxy Inverso** | `proxy_mode = False` | `proxy_mode = True` | Habilita cabeceras `X-Forwarded-For` cuando opera tras Nginx/Traefik con certificado SSL. |

---

### 4.2 Cómo mantener ambas opciones configuradas en Compose

Existen dos estrategias principales para gestionar ambos entornos sin duplicar código:

1. **Estrategia por Perfiles Nativos de Docker Compose (`profiles`) [Implementada en este proyecto]**:  
   Permite mantener toda la infraestructura centralizada en un único archivo `compose.yml`. Se define un servicio común de base de datos (`db`) y dos variantes del servicio de Odoo etiquetadas con `profiles: ["dev"]` y `profiles: ["prod"]`.  
   Para levantar el entorno deseado se utiliza una sola instrucción:
   * **Desarrollo**: `docker compose --profile dev up -d`
   * **Producción**: `docker compose --profile prod up -d`
   * *Ventaja*: Evita la dispersión de archivos, garantiza que la red y la base de datos sean idénticas y permite alternar de entorno con un simple comando.

2. **Estrategia por Archivos Override (Multi-fichero)**:  
   Divide la configuración en tres archivos:
   * `compose.yml` (Base común): Contiene las imágenes base, redes y persistencia de PostgreSQL.
   * `compose.override.yml` (Desarrollo): Se aplica automáticamente al ejecutar `docker compose up -d`, inyectando puertos, volúmenes de desarrollo y `--dev=all`.
   * `compose.prod.yml` (Producción): Se invoca explícitamente con `docker compose -f compose.yml -f compose.prod.yml up -d`, aplicando políticas estrictas, proxies y workers.

---

## 5. Gestión y Seguridad de Directorios de Odoo en el Repositorio

A la hora de gestionar el proyecto con Git, debemos separar el código fuente de los datos dinámicos generados por la aplicación:

* **`volumes/addons/` (Módulos personalizados) $\rightarrow$ SÍ se sube al repositorio**:  
  **Por qué**: Aquí se guarda el código fuente de los módulos y desarrollos propios que creamos o adaptamos para el negocio (modelos Python, vistas XML, reportes). Debe estar versionado para posibilitar el trabajo colaborativo, auditoría de cambios y despliegues automáticos (CI/CD).
* **`volumes/filestore/` (Adjuntos y binarios) $\rightarrow$ NO se sube al repositorio**:  
  **Por qué**: Contiene los archivos binarios que suben los usuarios (facturas en PDF, contratos firmados, imágenes de producto). Subirlo saturaría el historial de Git (*repository bloat*) y supone una infracción grave de normativas de privacidad (**RGPD** y **LOPDGDD**), ya que expondría datos de clientes y documentos confidenciales.
* **`volumes/sessions/` (Sesiones web de usuarios) $\rightarrow$ NO se sube al repositorio (Crítico)**:  
  **Por qué**: Guarda los ficheros temporales con las sesiones y cookies de autenticación de los usuarios que inician sesión. Si se sube a Git, cualquier persona con acceso al repositorio podría sustraer esas sesiones y entrar al ERP como administrador sin necesidad de contraseña (**Session Hijacking / Secuestro de Sesión**).
* **`volumes/postgres/` (Datos del motor relacional) $\rightarrow$ NO se sube al repositorio**:  
  **Por qué**: Almacena las tablas físicas de la base de datos en formato binario. Subirlo genera bloqueos de concurrencia, corrupción de datos e incompatibilidades entre diferentes sistemas operativos.

| Directorio | Estado en Git | Motivo Principal |
| :--- | :---: | :--- |
| `volumes/addons/` | **SÍ** | Código fuente del negocio y módulos propios. |
| `volumes/filestore/` | **NO** | Archivos binarios pesados y protección de datos (**RGPD** / **LOPDGDD**). |
| `volumes/sessions/` | **NO** | Riesgo crítico de secuestro de sesiones (**Session Hijacking**). |
| `volumes/postgres/` | **NO** | Integridad del motor de base de datos relacional. |

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
