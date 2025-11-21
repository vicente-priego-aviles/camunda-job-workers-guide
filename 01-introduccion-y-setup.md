# 01 - Introducción y Setup

## Tabla de Contenidos
- [Conceptos Fundamentales](#conceptos-fundamentales)
- [Instalación de Docker Compose](#instalación-de-docker-compose)
- [Verificación de Servicios](#verificación-de-servicios)
- [Acceso a Interfaces Web](#acceso-a-interfaces-web)
- [Instalación de HTTPie](#instalación-de-httpie)
- [Troubleshooting](#troubleshooting)

---

## Conceptos Fundamentales

### ¿Qué es un Job Worker?

Un **Job Worker** es un servicio (aplicación Java/Spring Boot) capaz de realizar una tarea particular dentro de un proceso BPMN. Piensa en él como un microservicio especializado que:

- Escucha por trabajos específicos que necesitan ejecutarse
- Procesa esos trabajos cuando están disponibles
- Reporta el resultado (éxito o fallo) de vuelta a Camunda

**Analogía del mundo real:**

Imagina una pizzería 🍕:
- El **proceso BPMN** es la receta completa para hacer una pizza
- Cada **Job Worker** es un empleado especializado:
  - Worker "amasar": Prepara la masa
  - Worker "agregar-ingredientes": Añade salsa, queso, etc.
  - Worker "hornear": Cocina la pizza
  - Worker "empaquetar": Empaca para entregar

Cada trabajador solo sabe hacer su tarea específica y espera a que le llegue trabajo.

### ¿Qué es un Job?

Un **Job** representa una instancia específica de trabajo que necesita realizarse. Cuando un proceso BPMN llega a una Service Task, Zeebe (el motor de Camunda) crea un Job.

**Propiedades de un Job:**

```
┌─────────────────────────────────────────┐
│              JOB                        │
├─────────────────────────────────────────┤
│ Type: "procesar-pago"                   │
│ Key: 2251799813685249                   │
│ Custom Headers:                         │
│   - metodo-pago: "tarjeta"              │
│   - proveedor: "stripe"                 │
│ Variables:                              │
│   - pedidoId: "PED-001"                 │
│   - monto: 150.50                       │
│   - clienteEmail: "cliente@email.com"   │
└─────────────────────────────────────────┘
```

#### 1. **Type (Tipo)**
- Identifica qué tipo de trabajo es
- Los workers se registran para tipos específicos
- Debe coincidir entre el BPMN y el worker

```java
// En el BPMN:
<zeebe:taskDefinition type="procesar-pago" />

// En el Worker:
@JobWorker(type = "procesar-pago")
public void procesarPago() { ... }
```

#### 2. **Key (Clave)**
- Identificador único del job
- Se usa para reportar resultados o fallos
- Generado automáticamente por Zeebe

#### 3. **Custom Headers (Cabeceras Personalizadas)**
- Metadata estática definida en el proceso BPMN
- Configuración reutilizable para workers
- No cambia entre ejecuciones del proceso

**Ejemplo de uso:**
```xml
<!-- En el BPMN -->
<zeebe:taskHeaders>
  <zeebe:header key="timeout-seconds" value="30" />
  <zeebe:header key="max-retries" value="3" />
</zeebe:taskHeaders>
```

```java
// En el Worker
@JobWorker(type = "mi-tarea")
public void miTarea(ActivatedJob job) {
    String timeout = job.getCustomHeaders().get("timeout-seconds");
    String maxRetries = job.getCustomHeaders().get("max-retries");
    // Usar configuración...
}
```

#### 4. **Variables**
- Datos de contexto del proceso
- Datos de negocio necesarios para ejecutar el trabajo
- Cambian según cada instancia del proceso

### Ciclo de Vida de un Job

```
┌──────────────┐
│   CREADO     │  ← Service Task activada en el proceso
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  EN COLA     │  ← Esperando que un worker lo solicite
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  ACTIVADO    │  ← Worker lo recibe y comienza a procesarlo
└──────┬───────┘
       │
       ├──────────────┐
       │              │
       ▼              ▼
┌──────────────┐  ┌──────────────┐
│ COMPLETADO   │  │   FALLIDO    │
└──────────────┘  └──────┬───────┘
                         │
                         ├─── Reintentos > 0 → Vuelve a EN COLA
                         │
                         └─── Reintentos = 0 → INCIDENTE
```

**Estados explicados:**

1. **CREADO**: Zeebe crea el job cuando se activa una Service Task
2. **EN COLA**: El job espera a que un worker lo solicite
3. **ACTIVADO**: Un worker recibe el job y tiene un tiempo límite (timeout) para completarlo
4. **COMPLETADO**: El worker terminó exitosamente y reportó resultados
5. **FALLIDO**: El worker reportó un fallo
   - Si quedan reintentos → vuelve a cola
   - Si no quedan reintentos → se crea un INCIDENTE (requiere intervención manual)

### Arquitectura Básica

```
┌─────────────────────────────────────────────────────────┐
│                    CAMUNDA 8 CLUSTER                    │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              ZEEBE (Motor)                       │   │
│  │  - Ejecuta procesos BPMN                         │   │
│  │  - Gestiona jobs queue                           │   │
│  │  - Coordina workers                              │   │
│  └────────┬─────────────────────────────────────────┘   │
│           │                                             │
│           │ gRPC (26500) / REST (8088)                  │
│           │                                             │
└───────────┼─────────────────────────────────────────────┘
            │
            │  Jobs disponibles
            │
    ┌───────┴────────┐
    │                │
    ▼                ▼
┌─────────┐    ┌─────────┐
│ Worker  │    │ Worker  │
│ Type: A │    │ Type: B │
└─────────┘    └─────────┘
```

**Componentes principales:**

1. **Zeebe**: Motor de workflow que ejecuta procesos BPMN
2. **Operate**: Interfaz web para monitorear instancias de procesos
3. **Tasklist**: Interfaz web para gestionar tareas de usuario
4. **Elasticsearch**: Almacena datos para Operate/Tasklist
5. **Job Workers**: Tus aplicaciones Spring Boot que procesan jobs

### ¿Cómo se comunican Workers y Zeebe?

**Polling (Solicitación activa):**

```
┌─────────┐                           ┌─────────┐
│ Worker  │  1. "¿Hay jobs tipo A?"   │  Zeebe  │
│         │ ─────────────────────────>│         │
│         │                           │         │
│         │  2. "Sí, aquí está job X" │         │
│         │ <─────────────────────────│         │
│         │                           │         │
│         │  3. Procesa job X...      │         │
│         │                           │         │
│         │  4. "Job X completado"    │         │
│         │ ─────────────────────────>│         │
└─────────┘                           └─────────┘
```

**Conceptos clave:**
- Los workers **solicitan** jobs de forma activa (pull model)
- Zeebe **no empuja** jobs a los workers (por defecto)
- Workers controlan su propio ritmo de trabajo
- Múltiples workers pueden solicitar el mismo tipo de job (escalamiento horizontal)

---

## Instalación de Docker Compose

### Paso 1: Verificar Prerrequisitos

Antes de comenzar, verifica que tienes Docker instalado:

```bash
# Verificar Docker
docker --version
# Salida esperada: Docker version 24.x.x o superior

# Verificar Docker Compose
docker compose version
# Salida esperada: Docker Compose version v2.x.x o superior
```

⚠️ **Nota**: Si ves `docker-compose` (con guion) en lugar de `docker compose` (espacio), estás usando la versión legacy. La documentación usa la nueva sintaxis, pero ambas funcionan.

### Paso 2: Descargar Camunda 8.8 Docker Compose

```bash
# Crear directorio para Camunda
mkdir -p ~/camunda-local
cd ~/camunda-local

# Descargar el paquete oficial
curl -L https://github.com/camunda/camunda-distributions/releases/download/docker-compose-8.8/docker-compose-8.8.zip -o camunda-8.8.zip

# Verificar descarga
ls -lh camunda-8.8.zip
```

### Paso 3: Extraer el Archivo

```bash
# Extraer contenido del zip
unzip camunda-8.8.zip

# Ver contenido extraído en el directorio actual
ls -la
```

**Archivos principales que verás:**

```
~/camunda-local/
├── docker-compose.yaml              ← Configuración LIGERA (la que usaremos)
├── docker-compose-full.yaml         ← Configuración completa
├── docker-compose-web-modeler.yaml  ← Solo Web Modeler
├── connector-secrets.txt            ← Secretos para connectors
└── .env                             ← Variables de entorno
```

⚠️ **Nota importante**: Los archivos se extraen directamente en el directorio actual (`~/camunda-local/`), no se crea una subcarpeta `camunda-8.8`.

### Paso 4: Entender las Configuraciones

#### Configuración Ligera (docker-compose.yaml) ✅ **LA QUE USAREMOS**

**Incluye:**
- ✅ **Orchestration Cluster** (Zeebe, Operate, Tasklist consolidados)
- ✅ **Connectors** (Integraciones out-of-the-box)
- ✅ **Elasticsearch** (Almacenamiento)

**Ideal para:**
- Desarrollo local
- Aprender Camunda
- Modelar, desplegar y probar procesos
- Menor consumo de recursos

**Puertos expuestos:**
- `26500`: Zeebe gRPC API
- `8088`: Orchestration Cluster (Operate, Tasklist, REST API)
- `9200`: Elasticsearch

#### Configuración Completa (docker-compose-full.yaml)

**Incluye TODO lo anterior MÁS:**
- Console (gestión de clusters)
- Optimize (análisis de procesos)
- Web Modeler (modelado en navegador)
- Identity + Keycloak (gestión de usuarios/autenticación)
- PostgreSQL (base de datos)

**Requiere:**
- Más RAM (~8-16 GB)
- Más CPU
- Autenticación OAuth

### Step 5: Levantar Camunda

```bash
# Asegúrate de estar en el directorio donde extrajiste los archivos
# En nuestro caso: ~/camunda-local/
docker compose up -d
```

**¿Qué significa cada flag?**
- `up`: Inicia los servicios
- `-d`: Modo detached (en segundo plano)

**Salida esperada:**

```
[+] Running 6/6
 ✔ Network camunda-local_camunda  Created                                0.0s 
 ✔ Volume camunda-local_elastic   Created                                0.0s 
 ✔ Volume camunda-local_zeebe     Created                                0.0s 
 ✔ Container elasticsearch        Healthy                               11.0s 
 ✔ Container orchestration        Healthy                               16.4s 
 ✔ Container connectors           Started                               16.5s
```

**📝 Nota sobre nombres de red y volúmenes:**

Docker Compose genera automáticamente los nombres usando el patrón: `<directorio>_<nombre-red>` y `<directorio>_<nombre-volumen>`.

En este caso:
- Directorio: `camunda-local`
- Red definida en `docker-compose.yaml`: `camunda`
- Resultado: `camunda-local_camunda`

**¿Quieres personalizar el nombre?**

**Opción A - Cambiar el nombre del directorio:**

```bash
# En lugar de ~/camunda-local/, usa un nombre más corto
mkdir ~/camunda
cd ~/camunda
curl -L https://github.com/camunda/camunda-distributions/releases/download/docker-compose-8.8/docker-compose-8.8.zip -o camunda-8.8.zip
unzip camunda-8.8.zip
docker compose up -d

# Resultado: camunda_camunda
```

**Opción B - Definir un nombre de proyecto personalizado:**

```bash
# Usar el flag -p (project name)
docker compose -p mi-camunda up -d

# Resultado: mi-camunda_camunda, mi-camunda_elastic, etc.
```

**Opción C - Modificar el docker-compose.yaml:**

```yaml
# Editar docker-compose.yaml y añadir al inicio:
name: camunda-cluster

# Luego ejecutar:
docker compose up -d

# Resultado: camunda-cluster_camunda, camunda-cluster_elastic, etc.
```

Para esta documentación, continuaremos usando `camunda-local` como nombre de directorio.

⏱️ **Tiempo de inicio**: Los servicios tardan **2-4 minutos** en estar completamente operativos. Elasticsearch es el más lento en iniciar.

### Paso 6: Monitorear el Inicio

```bash
# Ver logs en tiempo real de todos los servicios
docker compose logs -f

# Ver logs solo de Orchestration Cluster
docker compose logs -f orchestration

# Ver logs solo de Elasticsearch
docker compose logs -f elasticsearch
```

**Busca estas líneas para confirmar que está listo:**

```
orchestration      | Started StandaloneCamunda in X.XXX seconds (process running for X.XXX)
orchestration      | Tomcat started on port 8080 (http) with context path '/'
orchestration      | Partition-1 recovered, marking it as healthy
elasticsearch      | Cluster health status changed from [YELLOW] to [GREEN]
connectors         | Started ConnectorRuntimeApplication in X.XXX seconds
```

**Líneas clave que indican que todo está OK:**
- ✅ `Started StandaloneCamunda` - Aplicación principal iniciada
- ✅ `Tomcat started on port 8080` - Servidor web listo
- ✅ `Partition-1 recovered, marking it as healthy` - Broker Zeebe operativo
- ✅ `Elasticsearch cluster health: Green` - Base de datos lista

⚠️ **Es normal ver algunos WARN** durante el inicio:
- `Username and/or password for are empty` - Normal en configuración sin autenticación
- `MigrationSnapshotDirector failed` - Temporal, se resuelve automáticamente
- `The API is unprotected` - Esperado en configuración ligera para desarrollo

Presiona `Ctrl+C` para salir de los logs.

---

## Verificación de Servicios

### Comando Básico: Ver Estado de Contenedores

```bash
docker compose ps
```

**Salida esperada (todos con STATUS "Up" o "Healthy"):**

```
NAME                 IMAGE                                    STATUS
connectors           camunda/connectors-bundle:8.8.3          Up X minutes
elasticsearch        docker.elastic.co/elasticsearch/...      Up X minutes (healthy)
orchestration        camunda/camunda:8.8.4                    Up X minutes (healthy)
```

### Verificar Salud de Servicios

```bash
# Health check de Orchestration Cluster (puerto 9600 - actuator)
curl http://localhost:9600/actuator/health

# Salida esperada:
{"status":"UP"...}
```

```bash
# Health check de Elasticsearch
curl http://localhost:9200/_cluster/health

# Salida esperada (cluster_name y status):
{"cluster_name":"docker-cluster","status":"green",...}
```

### Comandos Útiles de Docker Compose

```bash
# Ver servicios corriendo
docker compose ps

# Ver logs de todos los servicios
docker compose logs

# Ver logs de un servicio específico
docker compose logs orchestration

# Seguir logs en tiempo real
docker compose logs -f

# Ver uso de recursos
docker stats

# Detener servicios (sin borrar datos)
docker compose stop

# Iniciar servicios detenidos
docker compose start

# Reiniciar un servicio específico
docker compose restart orchestration

# Detener y eliminar contenedores + volúmenes (⚠️ BORRA DATOS)
docker compose down -v

# Detener pero mantener volúmenes (datos persisten)
docker compose down
```

### Verificar Puertos en Uso

```bash
# macOS
lsof -nP -iTCP -sTCP:LISTEN | grep -E '(26500|8088|9200|9600)'

# Linux
netstat -tuln | grep -E '(26500|8088|9200|9600)'

# Windows (PowerShell)
netstat -an | findstr "26500 8088 9200 9600"
```

**Salida esperada (macOS):**

```
com.docke  1234  usuario   48u  IPv6 0x... 0t0  TCP *:26500 (LISTEN)
com.docke  1234  usuario   49u  IPv6 0x... 0t0  TCP *:8088 (LISTEN)
com.docke  1234  usuario   50u  IPv6 0x... 0t0  TCP *:9200 (LISTEN)
com.docke  1234  usuario   51u  IPv6 0x... 0t0  TCP *:9600 (LISTEN)
```

**Alternativa más simple (macOS y Linux):**

```bash
# Verificar puerto específico
lsof -i :26500
lsof -i :8088
lsof -i :9200
lsof -i :9600
```

---

## Acceso a Interfaces Web

Una vez que los servicios estén corriendo, puedes acceder a las interfaces web de Camunda.

### 1. Operate - Monitoreo de Procesos

**URL**: http://localhost:8088/operate

**Credenciales**:
- Usuario: `demo`
- Contraseña: `demo`

**¿Qué verás?**

Operate es la interfaz para monitorear y gestionar instancias de procesos en ejecución.

```
┌─────────────────────────────────────────────────────────┐
│  OPERATE                                    [demo]  ⚙️  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  📊 Procesos                                            │
│  ├─ proceso-pedidos (v1)         3 instancias activas   │
│  ├─ proceso-aprobacion (v2)      1 instancia activa     │
│  └─ proceso-facturacion (v1)     0 instancias           │
│                                                         │
│  🔍 Instancias                                          │
│  ├─ Instance 2251799813685249    ✅ Completada          │
│  ├─ Instance 2251799813685250    🔄 En ejecución        │
│  └─ Instance 2251799813685251    ❌ Con incidente       │
│                                                         │
│  ⚠️  Incidentes                                         │
│  └─ 1 incidente requiere atención                       │
└─────────────────────────────────────────────────────────┘
```

**Funcionalidades principales:**
- Ver procesos desplegados
- Monitorear instancias activas
- Ver variables de proceso
- Inspeccionar incidentes
- Ver el progreso de cada instancia en el diagrama BPMN

**Captura de lo que verás inicialmente:**

Al entrar por primera vez, verás una pantalla vacía indicando, esto es normal, aún no hemos desplegado ningún proceso.

![[operate.png]]
### 2. Tasklist - Gestión de Tareas de Usuario

**URL**: http://localhost:8088/tasklist

**Credenciales**: 
- Usuario: `demo`
- Contraseña: `demo`

**¿Qué verás?**

Tasklist es la interfaz para que usuarios completen tareas manuales (User Tasks) dentro de procesos.

```
┌─────────────────────────────────────────────────────────┐
│  TASKLIST                                   [demo]  ⚙️  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  📋 Mis Tareas (0)                                      │
│                                                         │
│  Sin tareas asignadas actualmente                       │
│                                                         │
│  💡 Las tareas aparecerán aquí cuando:                  │
│     - Un proceso llegue a una User Task                 │
│     - La tarea te sea asignada                          │
└─────────────────────────────────────────────────────────┘
```

**Funcionalidades principales:**
- Ver tareas asignadas al usuario actual
- Completar tareas con formularios
- Reclamar tareas no asignadas
- Filtrar tareas por proceso

![[tasklist.png]]

**Nota**: Tasklist solo muestra User Tasks, no Service Tasks (que son las que procesan los Job Workers).

### 3. Zeebe gRPC API

**Endpoint**: `localhost:26500`

**Uso**: Comunicación gRPC con Zeebe (alternativa a REST)

**Verificación:**

```bash
# Usando grpcurl (si lo tienes instalado)
grpcurl -plaintext localhost:26500 list

# Salida esperada: lista de servicios gRPC disponibles
```

**Nota**: En esta documentación usaremos principalmente REST, pero gRPC está disponible si lo prefieres.

### 4. Orchestration Cluster REST API 

**URL Base**: http://localhost:8088/v2

**Autenticación**: Ninguna (configuración ligera sin autenticación)

**Swagger UI**: http://localhost:8088/swagger-ui/index.html

**OpenAPI Specification**: http://localhost:8088/v3/api-docs

📚 **Documentación interactiva con Swagger UI**

   Swagger UI incluye **tres APIs diferentes** que puedes explorar:

   **Opción 1: Orchestration Cluster API** (⭐ Principal - Para Job Workers)
   ```
   http://localhost:8088/swagger-ui/index.html?urls.primaryName=Orchestration+Cluster+API
   ```
   - Gestión de procesos, jobs, workers
   - Despliegue de recursos BPMN
   - Creación de instancias de proceso
   - **Esta es la API que usaremos principalmente para Job Workers**

   **Opción 2: Operate Public API** (Monitoreo de procesos)
   ```
   http://localhost:8088/swagger-ui/index.html?urls.primaryName=Operate-v1
   ```
   - Consultar instancias de procesos
   - Ver variables de proceso
   - Gestionar incidentes

   **Opción 3: Tasklist REST API** (Gestión de tareas de usuario)
   ```
   http://localhost:8088/swagger-ui/index.html?urls.primaryName=Tasklist-v1
   ```
   - Gestionar User Tasks
   - Asignar y completar tareas
   - Trabajar con formularios

   También puedes cambiar entre APIs usando el menú desplegable **"Select a definition"** dentro de Swagger UI.

**Endpoints principales (Orchestration Cluster API):**

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/topology` | GET | Información del cluster |
| `/deployments` | POST | Desplegar recursos BPMN |
| `/process-instances` | POST | Crear instancia de proceso |
| `/jobs/search` | POST | Buscar jobs |

**Verificación rápida con HTTPie:**

```bash
# Obtener topología del cluster
http --body localhost:8088/v2/topology

# Salida esperada: JSON con información del cluster
{
    "brokers": [
        {
            "host": "172.18.0.3",
            "nodeId": 0,
            "partitions": [
                {
                    "health": "healthy",
                    "partitionId": 1,
                    "role": "leader"
                }
            ],
            "port": 26501,
            "version": "8.8.4"
        }
    ],
    "clusterSize": 1,
    "gatewayVersion": "8.8.4",
    "lastCompletedChangeId": "-1",
    "partitionsCount": 1,
    "replicationFactor": 1
}
```

**Probar con HTTPie (alternativa a Swagger UI):**

```bash
# Obtener topología con formato mejorado
http --pretty=all localhost:8088/v2/topology
```

### 5. Elasticsearch (Opcional)

**URL**: http://localhost:9200

**Uso**: Almacenamiento de datos para Operate/Tasklist

**Verificación:**

```bash
http localhost:9200

# Salida esperada:
HTTP/1.1 200 OK
X-elastic-product: Elasticsearch
content-encoding: gzip
content-length: 329
content-type: application/json

{
    "cluster_name": "docker-cluster",
    "cluster_uuid": "lg9fRiHOQ5O4YyO2ZhhnMA",
    "name": "60b4eb61a489",
    "tagline": "You Know, for Search",
    "version": {
        "build_date": "2025-08-08T08:36:52.872377389Z",
        "build_flavor": "default",
        "build_hash": "e5c4b2af120c131ea2885730f6693cb7d40a0a63",
        "build_snapshot": false,
        "build_type": "docker",
        "lucene_version": "9.12.0",
        "minimum_index_compatibility_version": "7.0.0",
        "minimum_wire_compatibility_version": "7.17.0",
        "number": "8.17.10"
    }
}
```

**Nota**: No necesitas interactuar directamente con Elasticsearch en el desarrollo normal.

### Resumen de Endpoints

```
┌────────────────────────────────────────────────────────────────────────┐
│  SERVICIO              │  URL                                          │
├────────────────────────┼───────────────────────────────────────────────┤
│  Operate               │  http://localhost:8088/operate                │
│  Tasklist              │  http://localhost:8088/tasklist               │
│  REST API              │  http://localhost:8088/v2  (base)             │
│  Swagger UI            │  http://localhost:8088/swagger-ui/index.html  │
│  OpenAPI Spec          │  http://localhost:8088/v3/api-docs            │
│  Zeebe gRPC            │  localhost:26500                              │
│  Actuator (Monitoring) │  http://localhost:9600/actuator               │
│  Elasticsearch         │  http://localhost:9200                        │
└────────────────────────────────────────────────────────────────────────┘

🔑 Credenciales: demo / demo
```

---

## Instalación de HTTPie

HTTPie es un cliente HTTP moderno, amigable y con sintaxis intuitiva. Lo usaremos en todos los ejemplos de esta documentación para interactuar con la REST API de Camunda.

### ¿Por qué HTTPie sobre curl?

**Comparación de sintaxis:**

```bash
# Con curl (tradicional)
curl -X POST http://localhost:8088/v2/process-instances \
  -H "Content-Type: application/json" \
  -d '{"bpmnProcessId":"proceso-pedidos","variables":{"pedidoId":"123"}}'

# Con HTTPie (más simple) ✨
http POST localhost:8088/v2/process-instances \
  bpmnProcessId=proceso-pedidos \
  variables:='{"pedidoId":"123"}'
```

**Ventajas de HTTPie:**
- ✅ Sintaxis más limpia y legible
- ✅ Coloreado de sintaxis automático
- ✅ JSON por defecto (no necesitas especificar Content-Type)
- ✅ Formato de respuestas más legible
- ✅ Autocompletado en terminal

### Instalación según Sistema Operativo

#### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install httpie

# Verificar instalación
http --version
```

#### Linux (Fedora/CentOS/RHEL)

```bash
sudo dnf install httpie

# Verificar instalación
http --version
```

#### macOS

```bash
# Con Homebrew
brew install httpie

# Verificar instalación
http --version
```

#### Windows

**Opción 1: Con Chocolatey**
```powershell
choco install httpie

# Verificar instalación
http --version
```

**Opción 2: Con pip (Python)**
```powershell
pip install httpie

# Verificar instalación
http --version
```

**Opción 3: Con Scoop**
```powershell
scoop install httpie
```

#### Instalación Universal con pip

Si tienes Python instalado en cualquier sistema:

```bash
pip install httpie

# O con pip3
pip3 install httpie

# Verificar instalación
http --version
```

### Configuración Inicial (Opcional)

HTTPie guarda configuración en `~/.config/httpie/config.json` (Linux/Mac) o `%APPDATA%\httpie\config.json` (Windows).

**Configuración recomendada:**

```bash
# Crear/editar archivo de configuración
mkdir -p ~/.config/httpie
cat > ~/.config/httpie/config.json << 'EOF'
{
    "default_options": [
        "--pretty=all",
        "--style=monokai",
        "--timeout=30"
    ]
}
EOF
```

### Ejemplos Básicos de HTTPie

#### GET Request

```bash
# Obtener topología del cluster
http GET localhost:8088/v2/topology

# Forma corta (GET es por defecto)
http localhost:8088/v2/topology
```

#### POST Request con JSON

```bash
# Crear instancia de proceso
http POST localhost:8088/v2/process-instances \
  bpmnProcessId=mi-proceso \
  variables:='{"dato":"valor"}'
```

#### Headers Personalizados

```bash
http POST localhost:8088/v2/deployments \
  Content-Type:application/json \
  X-Custom-Header:valor
```

#### Mostrar Solo el Body

```bash
http --body localhost:8088/v2/topology
```

#### Mostrar Solo Headers

```bash
http --headers localhost:8088/v2/topology
```

#### Pretty Print de JSON

```bash
# Por defecto está activo
http localhost:8088/v2/topology

# Desactivar pretty print
http --pretty=none localhost:8088/v2/topology
```

#### Guardar Respuesta en Archivo

```bash
http localhost:8088/v2/topology > topology.json
```

#### Verbose Mode (Ver Request y Response)

```bash
http -v POST localhost:8088/v2/process-instances \
  bpmnProcessId=test
```

### HTTPie vs curl - Chuleta

| Operación | curl | HTTPie |
|-----------|------|--------|
| GET simple | `curl http://api.com` | `http api.com` |
| POST JSON | `curl -X POST -H "Content-Type: application/json" -d '{"key":"val"}' http://api.com` | `http POST api.com key=val` |
| PUT | `curl -X PUT http://api.com/item` | `http PUT api.com/item` |
| DELETE | `curl -X DELETE http://api.com/item` | `http DELETE api.com/item` |
| Headers | `curl -H "Auth: token" http://api.com` | `http api.com Auth:token` |
| Query params | `curl http://api.com?key=val` | `http api.com key==val` |
| Upload file | `curl -F file=@path http://api.com` | `http -f POST api.com file@path` |

### Verificación de Instalación con Camunda

Prueba que HTTPie funciona correctamente con Camunda:

```bash
# Test 1: Obtener topología
http --body localhost:8088/v2/topology

# Deberías ver algo como:
{
    "brokers": [
        {
            "host": "172.18.0.3",
            "nodeId": 0,
            "partitions": [
                {
                    "health": "healthy",
                    "partitionId": 1,
                    "role": "leader"
                }
            ],
            "port": 26501,
            "version": "8.8.4"
        }
    ],
    "clusterSize": 1,
    "gatewayVersion": "8.8.4",
    "lastCompletedChangeId": "-1",
    "partitionsCount": 1,
    "replicationFactor": 1
}
```

✅ Si ves una respuesta similar, HTTPie está correctamente instalado y Camunda está accesible.

---

## Troubleshooting

### Problema 1: Servicios no inician

**Síntoma:**
```bash
docker compose ps
# Servicios con STATUS "Exited" o "Restarting"
```

**Soluciones:**

```bash
# 1. Ver logs para identificar error
docker compose logs orchestration

# 2. Verificar que no haya conflictos de puertos
netstat -tuln | grep -E '(26500|8088|9200)'

# 3. Reiniciar servicios
docker compose restart

# 4. Si persiste, eliminar todo y empezar de cero
docker compose down -v
docker compose up -d
```

### Problema 2: Puerto 26500 ya en uso

**Síntoma:**
```
Error: bind: address already in use
```

**Solución:**

```bash
# Identificar qué proceso usa el puerto
lsof -i :26500

# Matar el proceso (reemplaza PID con el número mostrado)
kill -9 <PID>

# O cambiar el puerto en docker-compose.yaml
# Editar la sección de orchestration:
#   ports:
#     - "26501:26500"  # Usar 26501 en lugar de 26500
```

### Problema 3: Elasticsearch no inicia (memoria insuficiente)

**Síntoma:**
```
elasticsearch-1 | ERROR: [1] bootstrap checks failed
elasticsearch-1 | max virtual memory areas vm.max_map_count [65530] is too low
```

**Solución (Linux):**

```bash
# Aumentar vm.max_map_count
sudo sysctl -w vm.max_map_count=262144

# Hacer permanente
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

**Solución (Windows/Mac con Docker Desktop):**

1. Abrir Docker Desktop
2. Settings → Resources → Advanced
3. Aumentar memoria a al menos 4 GB
4. Apply & Restart

### Problema 4: No puedo acceder a Operate (404)

**Síntoma:**
```
http://localhost:8088/operate → 404 Not Found
```

**Soluciones:**

```bash
# 1. Verificar que orchestration está corriendo
docker compose ps orchestration

# 2. Ver logs para errores
docker compose logs orchestration | grep -i error

# 3. Verificar que Elasticsearch esté saludable
curl http://localhost:9200/_cluster/health

# 4. Esperar más tiempo (puede tardar 3-4 minutos)
docker compose logs -f orchestration
# Busca: "Started CamundaOrchestrationApplication"

# 5. Intentar con la IP directa
http://127.0.0.1:8088/operate
```

### Problema 5: HTTPie no reconoce JSON

**Síntoma:**
```bash
http POST localhost:8088/v2/test data=value
# Error: Content-Type not recognized
```

**Solución:**

```bash
# Usar := para valores JSON literales
http POST localhost:8088/v2/test data:='{"key":"value"}'

# O usar = para strings simples
http POST localhost:8088/v2/test data="value"
```

### Problema 6: "Connection refused" al hacer requests

**Síntoma:**
```bash
http localhost:8088/v2/topology
# Connection refused
```

**Soluciones:**

```bash
# 1. Verificar que servicios estén UP
docker compose ps

# 2. Verificar que el puerto esté escuchando
netstat -tuln | grep 8088

# 3. Verificar desde dentro del container
docker compose exec orchestration curl localhost:8080/actuator/health

# 4. Revisar firewall local
sudo ufw status  # Linux
# Asegúrate de que 8088 esté permitido
```

### Problema 7: Procesos desplegados no aparecen en Operate

**Síntoma:**
Desplegaste un proceso pero no aparece en Operate.

**Soluciones:**

```bash
# 1. Verificar que el despliegue fue exitoso
http POST localhost:8088/v2/deployments \
  resources:='[...]'
# Debe retornar 200 OK con deployment key

# 2. Forzar refresh en Operate
# Navega a: http://localhost:8088/operate
# Presiona Ctrl+F5 (hard refresh)

# 3. Verificar índices de Elasticsearch
curl http://localhost:9200/_cat/indices?v | grep operate

# 4. Reiniciar Orchestration Cluster
docker compose restart orchestration
```

### Problema 8: Contenedores consumen mucha RAM

**Síntoma:**
```bash
docker stats
# Elasticsearch usa >2GB RAM
```

**Soluciones:**

```bash
# 1. Limitar memoria de Elasticsearch
# Editar docker-compose.yaml:
# 
# elasticsearch:
#   environment:
#     - "ES_JAVA_OPTS=-Xms512m -Xmx512m"

# 2. Usar configuración ligera en lugar de completa
docker compose -f docker-compose.yaml up -d

# 3. Limpiar datos no usados de Docker
docker system prune -a --volumes
```

### Comandos Útiles para Debugging

```bash
# Ver todos los logs
docker compose logs

# Ver logs de los últimos 100 líneas
docker compose logs --tail=100

# Seguir logs en tiempo real
docker compose logs -f

# Ver logs de un servicio específico
docker compose logs orchestration

# Inspeccionar un contenedor
docker compose exec orchestration bash

# Ver variables de entorno de un contenedor
docker compose exec orchestration env

# Ver procesos dentro de un contenedor
docker compose exec orchestration ps aux

# Verificar conectividad de red entre contenedores - TODO...
docker compose exec orchestration ping elasticsearch

# Ver uso de recursos en tiempo real
docker stats

# Ver información detallada de un contenedor
docker inspect orchestration
```

### Logs Comunes y Su Significado

| Log | Significado | Acción |
|-----|-------------|--------|
| `Started CamundaOrchestrationApplication in X seconds` | ✅ Orchestration está listo | Ninguna, todo OK |
| `"message":"started"` en elasticsearch | ✅ Elasticsearch está listo | Ninguna, todo OK |
| `Connection refused` | ❌ Servicio no está escuchando | Verificar que el servicio esté UP |
| `OutOfMemoryError` | ❌ Memoria insuficiente | Aumentar RAM de Docker Desktop |
| `BindException: Address already in use` | ❌ Puerto en uso | Cambiar puerto o matar proceso |
| `Bootstrap checks failed` | ❌ Configuración de Elasticsearch incorrecta | Aplicar fix de vm.max_map_count |

### ¿Dónde Buscar Ayuda?

1. **Documentación Oficial**: https://docs.camunda.io/
2. **Camunda Forum**: https://forum.camunda.io/
3. **GitHub Issues**: https://github.com/camunda/camunda/issues
4. **Stack Overflow**: Tag `camunda` o `zeebe`
5. **Logs de Docker**: Siempre el primer lugar para buscar errores

---

## ✅ Checklist de Verificación

Antes de continuar al siguiente documento, verifica que:

- [ ] Docker Compose está instalado y funcionando
- [ ] Descargaste e instalaste Camunda 8.8
- [ ] Los servicios están corriendo (`docker compose ps` muestra todo "Up")
- [ ] Puedes acceder a Operate en http://localhost:8088/operate
- [ ] Puedes acceder a Tasklist en http://localhost:8088/tasklist
- [ ] La REST API responde en http://localhost:8088/v2/topology
- [ ] HTTPie está instalado y funciona correctamente
- [ ] Puedes hacer requests con HTTPie a la API de Camunda

Si todos los checks están ✅, ¡estás listo para continuar!

---

**Siguiente:** [02 - Primeros Pasos](./02-primeros-pasos.md) →

En el siguiente documento configuraremos un proyecto Spring Boot 3.x con el nuevo Camunda Spring Boot Starter y crearemos nuestro primer Job Worker.
