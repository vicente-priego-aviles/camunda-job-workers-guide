# 02 - Primeros Pasos

## Tabla de Contenidos
- [Introducción](#introducción)
- [Creación del Proyecto con Spring Initializr](#creación-del-proyecto-con-spring-initializr)
- [Configuración Inicial](#configuración-inicial)
- [Primera Iteración: Worker Mínimo](#primera-iteración-worker-mínimo)
- [Segunda Iteración: Añadir Complejidad](#segunda-iteración-añadir-complejidad)
- [Despliegue de Procesos BPMN](#despliegue-de-procesos-bpmn)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)

---

## Introducción

En este documento aprenderás a:

✅ Crear un proyecto Spring Boot 3.5.8 con Java 25 desde cero  
✅ Configurar el nuevo Camunda Spring Boot Starter 8.8  
✅ Implementar tu primer Job Worker funcional  
✅ Diseñar y desplegar un proceso BPMN  
✅ Probar todo el flujo con HTTPie  
✅ Escribir tests automatizados  

**Enfoque iterativo**: Empezaremos con lo más simple que funcione y luego añadiremos complejidad paso a paso.

## Version compatibility

| Camunda Spring Boot Starter version | JDK  | Camunda version | Bundled Spring Boot version | Compatible Spring Boot version(s) |
| ----------------------------------- | ---- | --------------- | --------------------------- | --------------------------------- |
| 8.8.x                               | ≥ 17 | 8.8.x           | 3.5.x                       |                                   |

---

## Creación del Proyecto con Spring Initializr

### Opción 1: Desde el Navegador (Recomendado)

1. **Abre Spring Initializr** en tu navegador:
   ```
   https://start.spring.io
   ```

2. **Configura el proyecto:**

   ```
   ┌─────────────────────────────────────────────────────────┐
   │  Project                                                │
   │  ● Maven                                                │
   │  ○ Gradle                                               │
   │                                                         │
   │  Language                                               │
   │  ● Java                                                 │
   │  ○ Kotlin                                               │
   │  ○ Groovy                                               │
   │                                                         │
   │  Spring Boot                                            │
   │  [3.5.8] ▼  (o la última 4.x.x disponible)              │
   │                                                         │
   │  Project Metadata                                       │
   │  Group:         com.ejemplo                             │
   │  Artifact:      camunda-workers                         │
   │  Name:          camunda-workers                         │
   │  Description:   Job Workers con Camunda 8.8             │
   │  Package name:  com.ejemplo.camunda                     │
   │  Packaging:     ● Jar  ○ War                            │ 
   │  Configuration: ○ Properties  ● YAML                    │
   │  Java:         [25] ▼                                   │
   └─────────────────────────────────────────────────────────┘
   ```

3. **Añadir dependencias** (botón "ADD DEPENDENCIES"):

   En el buscador, añade las siguientes:
   
   - **Spring Web** (para REST controllers)
   - **Lombok** (para reducir boilerplate)
   - **Spring Boot DevTools** (opcional - para desarrollo)

   ⚠️ **Nota**: No de puede añadir Camunda desde Spring Initializr, lo añadiremos manualmente después.

4. **Generar el proyecto:**
   
   - Click en **"GENERATE"** (botón azul)
   - Se descargará `camunda-workers.zip`
   
![[spring-initlzr.png]]

5. **Extraer y abrir el proyecto:**

   ```bash
   # Ir a la carpeta de descargas
   cd ~/Downloads
   
   # Extraer
   unzip camunda-workers.zip
   
   # Mover a tu workspace
   mv camunda-workers ~/workspace/
   cd ~/workspace/camunda-workers
   
   # Abrir con tu IDE favorito
   # IntelliJ IDEA:
   idea .
   
   # VS Code:
   code .
   
   # Eclipse:
   # File > Open Projects from File System > Directory
   ```

### Opción 2: Desde la Línea de Comandos

```bash
# Crear proyecto con curl
curl https://start.spring.io/starter.zip \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.5.8 \
  -d baseDir=camunda-workers \
  -d groupId=com.ejemplo \
  -d artifactId=camunda-workers \
  -d name=camunda-workers \
  -d description="Job Workers con Camunda 8.8" \
  -d packageName=com.ejemplo.camunda \
  -d packaging=jar \
  -d javaVersion=25 \
  -d dependencies=web,lombok,devtools \
  -d configurationFileFormat=yaml \
  -o camunda-workers.zip

# Extraer
unzip camunda-workers.zip
cd camunda-workers
```

### Verificar la Estructura del Proyecto

Deberías ver esta estructura:

```
camunda-workers/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── ejemplo/
│   │   │           └── camunda/
│   │   │               └── CamundaWorkersApplication.java
│   │   └── resources/
│   │       ├── application.yaml
│   │       ├── static/
│   │       └── templates/
│   └── test/
│       └── java/
│           └── com/
│               └── ejemplo/
│                   └── camunda/
│                       └── CamundaWorkersApplicationTests.java
├── .gitattributes
├── .gitignore
├── mvnw
├── mvnw.cmd
├── pom.xml
└── HELP.md
```

---

## Configuración Inicial

### Paso 1: Añadir Dependencia de Camunda

Abre `pom.xml` y añade la dependencia de Camunda Spring Boot Starter y otras dependencias adicionales (Validation, Actuator, Micrometer):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.8<</version>
        <relativePath/> 
    </parent>
    
    <groupId>com.ejemplo</groupId>
    <artifactId>camunda-workers</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>camunda-workers</name>
    <description>Job Workers con Camunda 8.8</description>
    
    <properties>
        <java.version>25</java.version>
        <camunda.version>8.8.4</camunda.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        

        <!-- Camunda Spring Boot Starter -->
		<dependency>  
		    <groupId>io.camunda</groupId>  
		    <artifactId>camunda-spring-boot-starter</artifactId>  
		    <version>${camunda.version}</version>  
		</dependency>
        
        <!-- DevTools (opcional) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
		<!-- Validation (para @Valid en controllers) -->  
		<dependency>  
		    <groupId>org.springframework.boot</groupId>  
		    <artifactId>spring-boot-starter-validation</artifactId>  
		</dependency>  
		  
		<!-- Actuator (métricas y health checks) -->  
		<dependency>  
		    <groupId>org.springframework.boot</groupId>  
		    <artifactId>spring-boot-starter-actuator</artifactId>  
		</dependency>  
		  
		<!-- Micrometer (métricas detalladas) -->  
		<dependency>  
		    <groupId>io.micrometer</groupId>  
		    <artifactId>micrometer-core</artifactId>  
		</dependency>

    </dependencies>
    
    <build>
        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
					<annotationProcessorPaths>  
					    <path>       
						    <groupId>org.projectlombok</groupId>  
						    <artifactId>lombok</artifactId>  
					    </path>
					</annotationProcessorPaths>
                </configuration>
            </plugin>
            
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
            
        </plugins>
    </build>
</project>
```

**🔑 Aspectos importantes:**

- `camunda-spring-boot-starter`: Nuevo starter de Camunda 8.8.4 (reemplaza Spring Zeebe SDK)
- `maven-compiler-plugin` con `<parameters>true</parameters>`: Permite usar `@Variable` sin especificar el nombre
- Spring Boot 4.4.0 es compatible con Java 21-25

### Paso 2: Recargar Dependencias Maven

```bash
# Desde terminal en el directorio del proyecto
./mvnw clean install

# O desde tu IDE:
# IntelliJ: Click derecho en pom.xml > Maven > Reload Project
# Eclipse: Click derecho en proyecto > Maven > Update Project
# VS Code: Ctrl+Shift+P > Java: Clean Java Language Server Workspace
```

### Paso 3: Configurar Conexión a Camunda

Añade la siguiente configuración a `application.yml` :

**📁 src/main/resources/application.yml**

```yaml
# ============================================================================  
# CONFIGURACIÓN BÁSICA  
# ============================================================================
spring:
  application:
    name: camunda-workers

# ============================================================================  
# CAMUNDA 8.8.4 SELF-MANAGED (CONFIGURACIÓN MÍNIMA)  
# ============================================================================
camunda:
  client:
    mode: self-managed  
	grpc-address: ${CAMUNDA_GRPC_ADDRESS:http://localhost:26500}  
	rest-address: ${CAMUNDA_REST_ADDRESS:http://localhost:8080}  
	auth:  
	  method: none  
	worker:  
	  defaults:  
	    max-jobs-active: 64  
	    timeout: PT1M           # 1 minuto  
	    request-timeout: PT30S  # 30 segundos  
	    stream-enabled: true
	    
# ============================================================================  
# ACTUATOR  
# ============================================================================  
management:  
  endpoints:  
    web:  
      exposure:  
        include: health,info,metrics

# ============================================================================  
# LOGGING  
# ============================================================================  
logging:  
  level:  
    root: INFO  
    com.ejemplo.camunda: DEBUG  
    io.camunda: INFO
    
# ============================================================================  
# SERVIDOR  
# ============================================================================  
server:  
  port: 8090  
  
---  
# ============================================================================  
# PROFILE: DEV  
# ============================================================================  
spring:  
  config:  
    activate:  
      on-profile: dev  
  
logging:  
  level:  
    com.ejemplo.camunda: DEBUG  
  
---  
# ============================================================================  
# PROFILE: PROD  
# ============================================================================  
spring:  
  config:  
    activate:  
      on-profile: prod  
  
logging:  
  level:  
    root: WARN  
    com.ejemplo.camunda: INFO
```

⚠️ **Nota sobre Seguridad**: Para producción, usa variables de entorno:

**📁 .env (no commitear a git)**

```bash
CAMUNDA_GRPC_ADDRESS=http://localhost:26500
CAMUNDA_REST_ADDRESS=http://localhost:8088
```

### Paso 4: Verificar que la Aplicación Arranca

```bash
# Ejecutar la aplicación
./mvnw spring-boot:run
```

**Salida esperada:**

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.5.8)

... Tomcat started on port 8090 (http) with context path '/'
... Started CamundaWorkersApplication in 2.345 seconds
```

✅ Si ves esto, ¡tu aplicación Spring Boot está lista!

**Detener la aplicación:** Presiona `Ctrl+C`

---

## Primera Iteración: Worker Mínimo

Vamos a crear el worker más simple posible que funcione.

### Paso 1: Crear el Primer Job Worker

**📁 src/main/java/com/ejemplo/camunda/worker/SaludoWorker.java**

```java
package com.ejemplo.camunda.worker;

import io.camunda.zeebe.spring.client.annotation.JobWorker;
import io.camunda.zeebe.spring.client.annotation.Variable;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.Map;

/**
 * Worker simple que saluda a una persona.
 * 
 * Este es un Job Worker básico que demuestra:
 * - Cómo recibir variables del proceso
 * - Cómo procesar datos
 * - Cómo devolver resultados al proceso
 */
@Slf4j
@Component
public class SaludoWorker {
    
    /**
     * Procesa un saludo personalizado.
     * 
     * @param nombre El nombre de la persona a saludar (viene del proceso BPMN)
     * @return Map con el mensaje de saludo que se fusiona con las variables del proceso
     */
    @JobWorker(type = "saludar")
    public Map<String, Object> saludar(@Variable String nombre) {
        log.info("👋 Procesando saludo para: {}", nombre);
        
        // Simular procesamiento
        String mensaje = String.format("¡Hola %s! Bienvenido a Camunda 8.8", nombre);
        
        log.info("✅ Saludo generado: {}", mensaje);
        
        // Devolver resultado que se fusiona con las variables del proceso
        return Map.of(
            "mensaje", mensaje,
            "procesadoEn", System.currentTimeMillis()
        );
    }
}
```

**🔑 Conceptos clave:**

- `@Component`: Registra la clase como un bean de Spring
- `@JobWorker(type = "saludar")`: Define que este método procesa jobs de tipo "saludar"
- `@Variable String nombre`: Inyecta automáticamente la variable "nombre" del proceso
- `Map<String, Object>`: El resultado se fusiona con las variables del proceso
- `@Slf4j`: Añade logger automáticamente (por Lombok)

### Paso 2: Crear el Proceso BPMN Mínimo

Vamos a crear un proceso BPMN simple con 1 Service Task.

**📁 src/main/resources/bpmn/proceso-saludo.bpmn**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL"
                   xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
                   xmlns:dc="http://www.omg.org/spec/DD/20100524/DC"
                   xmlns:zeebe="http://camunda.org/schema/zeebe/1.0"
                   xmlns:di="http://www.omg.org/spec/DD/20100524/DI"
                   id="Definitions_1"
                   targetNamespace="http://bpmn.io/schema/bpmn">
  
  <bpmn:process id="proceso-saludo" name="Proceso de Saludo" isExecutable="true">
    
    <!-- Evento de Inicio -->
    <bpmn:startEvent id="StartEvent_1" name="Inicio">
      <bpmn:outgoing>Flow_1</bpmn:outgoing>
    </bpmn:startEvent>
    
    <!-- Service Task: Saludar -->
    <bpmn:serviceTask id="Task_Saludar" name="Saludar">
      <bpmn:extensionElements>
        <zeebe:taskDefinition type="saludar" />
      </bpmn:extensionElements>
      <bpmn:incoming>Flow_1</bpmn:incoming>
      <bpmn:outgoing>Flow_2</bpmn:outgoing>
    </bpmn:serviceTask>
    
    <!-- Evento de Fin -->
    <bpmn:endEvent id="EndEvent_1" name="Fin">
      <bpmn:incoming>Flow_2</bpmn:incoming>
    </bpmn:endEvent>
    
    <!-- Flujos -->
    <bpmn:sequenceFlow id="Flow_1" sourceRef="StartEvent_1" targetRef="Task_Saludar" />
    <bpmn:sequenceFlow id="Flow_2" sourceRef="Task_Saludar" targetRef="EndEvent_1" />
    
  </bpmn:process>
  
  <!-- Diagrama visual (opcional para visualización) -->
  <bpmndi:BPMNDiagram id="BPMNDiagram_1">
    <bpmndi:BPMNPlane id="BPMNPlane_1" bpmnElement="proceso-saludo">
      <bpmndi:BPMNShape id="StartEvent_1_di" bpmnElement="StartEvent_1">
        <dc:Bounds x="152" y="102" width="36" height="36" />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="Task_Saludar_di" bpmnElement="Task_Saludar">
        <dc:Bounds x="240" y="80" width="100" height="80" />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNShape id="EndEvent_1_di" bpmnElement="EndEvent_1">
        <dc:Bounds x="392" y="102" width="36" height="36" />
      </bpmndi:BPMNShape>
      <bpmndi:BPMNEdge id="Flow_1_di" bpmnElement="Flow_1">
        <di:waypoint x="188" y="120" />
        <di:waypoint x="240" y="120" />
      </bpmndi:BPMNEdge>
      <bpmndi:BPMNEdge id="Flow_2_di" bpmnElement="Flow_2">
        <di:waypoint x="340" y="120" />
        <di:waypoint x="392" y="120" />
      </bpmndi:BPMNEdge>
    </bpmndi:BPMNPlane>
  </bpmndi:BPMNDiagram>
  
</bpmn:definitions>
```

**📊 Diagrama Visual del Proceso:**

```
┌─────────┐       ┌──────────────┐       ┌─────────┐
│ Inicio  │──────>│   Saludar    │──────>│   Fin   │
│    ○    │       │ (saludar)    │       │   ◉     │
└─────────┘       └──────────────┘       └─────────┘
```

**🔑 Elementos importantes del BPMN:**

- `bpmn:process id="proceso-saludo"`: Identificador del proceso (úsalo para iniciarlo)
- `isExecutable="true"`: El proceso puede ejecutarse
- `<zeebe:taskDefinition type="saludar" />`: Conecta con el worker mediante el "type"
- Service Task → Worker mediante el "type" matching

### Paso 3: Configurar Despliegue Automático

Añade la anotación `@Deployment` en tu aplicación principal:

**📁 src/main/java/com/ejemplo/camunda/CamundaWorkersApplication.java**

```java
package com.ejemplo.camunda;

import io.camunda.zeebe.spring.client.annotation.Deployment;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@Deployment(resources = "classpath:bpmn/proceso-saludo.bpmn")
public class CamundaWorkersApplication {

    public static void main(String[] args) {
        SpringApplication.run(CamundaWorkersApplication.class, args);
    }
}
```

**🔑 La anotación `@Deployment`:**
- Despliega automáticamente el proceso BPMN al iniciar la aplicación
- `resources = "classpath:bpmn/proceso-saludo.bpmn"`: Ruta al archivo BPMN
- Puedes especificar múltiples archivos: `resources = {"classpath:bpmn/*.bpmn"}`

### Paso 4: Crear un Controller para Iniciar el Proceso

**📁 src/main/java/com/ejemplo/camunda/controller/ProcesoController.java**

```java
package com.ejemplo.camunda.controller;

import io.camunda.zeebe.client.ZeebeClient;
import io.camunda.zeebe.client.api.response.ProcessInstanceEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

/**
 * Controller REST para gestionar procesos de Camunda.
 */
@Slf4j
@RestController
@RequestMapping("/api/procesos")
@RequiredArgsConstructor
public class ProcesoController {

    private final ZeebeClient zeebeClient;

    /**
     * Inicia una instancia del proceso de saludo.
     *
     * @param request Mapa con las variables iniciales (debe incluir "nombre")
     * @return Información de la instancia creada
     */
    @PostMapping("/saludo")
    public ResponseEntity<Map<String, Object>> iniciarProcesoSaludo(
            @RequestBody Map<String, Object> request) {
        
        log.info("📝 Iniciando proceso de saludo con datos: {}", request);
        
        // Validar que viene el nombre
        if (!request.containsKey("nombre")) {
            return ResponseEntity.badRequest()
                    .body(Map.of("error", "El campo 'nombre' es requerido"));
        }
        
        try {
            // Iniciar instancia del proceso
            ProcessInstanceEvent processInstance = zeebeClient
                    .newCreateInstanceCommand()
                    .bpmnProcessId("proceso-saludo")
                    .latestVersion()
                    .variables(request)
                    .send()
                    .join();
            
            log.info("✅ Proceso iniciado. Process Instance Key: {}", 
                    processInstance.getProcessInstanceKey());
            
            return ResponseEntity.ok(Map.of(
                    "processInstanceKey", processInstance.getProcessInstanceKey(),
                    "bpmnProcessId", processInstance.getBpmnProcessId(),
                    "version", processInstance.getVersion(),
                    "mensaje", "Proceso iniciado correctamente"
            ));
            
        } catch (Exception e) {
            log.error("❌ Error iniciando proceso", e);
            return ResponseEntity.internalServerError()
                    .body(Map.of("error", e.getMessage()));
        }
    }
    
    /**
     * Endpoint de prueba rápida.
     */
    @GetMapping("/saludo/test")
    public ResponseEntity<Map<String, Object>> test() {
        return iniciarProcesoSaludo(Map.of("nombre", "Desarrollador"));
    }
}
```

### Paso 5: Ejecutar Todo el Flujo

1. **Asegúrate de que Camunda está corriendo:**
   ```bash
   cd ~/camunda-local
   docker compose ps
   # Todos los servicios deben estar "Up"
   ```

2. **Inicia tu aplicación Spring Boot:**
   ```bash
   cd ~/workspace/camunda-workers
   ./mvnw spring-boot:run
   ```

3. **Verifica que el proceso se desplegó:**
   
   En los logs deberías ver algo como:
   ```
   Deploying resource proceso-saludo.bpmn
   Deployment created with key: 2251799813685249
   ```

4. **Inicia una instancia del proceso con HTTPie:**

   ```bash
   # Opción 1: Endpoint de prueba
   http GET localhost:8080/api/procesos/saludo/test
   
   # Opción 2: Con tu propio nombre
   http POST localhost:8080/api/procesos/saludo nombre="Juan"
   ```

5. **Observa los logs de tu aplicación:**

   Deberías ver:
   ```
   📝 Iniciando proceso de saludo con datos: {nombre=Juan}
   ✅ Proceso iniciado. Process Instance Key: 2251799813685250
   👋 Procesando saludo para: Juan
   ✅ Saludo generado: ¡Hola Juan! Bienvenido a Camunda 8.8
   ```

6. **Verifica en Operate:**
   
   - Abre http://localhost:8088/operate
   - Login: demo / demo
   - Verás tu proceso "Proceso de Saludo" con instancias completadas

**🎉 ¡Felicidades! Has completado tu primer flujo completo con Camunda 8.8**

---

## Segunda Iteración: Añadir Complejidad

Ahora vamos a añadir dos workers más que procesan diferentes tipos de datos.

### Worker 2: Procesar Números

**📁 src/main/java/com/ejemplo/camunda/worker/CalculadoraWorker.java**

```java
package com.ejemplo.camunda.worker;

import io.camunda.zeebe.spring.client.annotation.JobWorker;
import io.camunda.zeebe.spring.client.annotation.Variable;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.Map;

/**
 * Worker que realiza cálculos matemáticos simples.
 */
@Slf4j
@Component
public class CalculadoraWorker {
    
    @JobWorker(type = "calcular")
    public Map<String, Object> calcular(
            @Variable Integer numero1,
            @Variable Integer numero2,
            @Variable String operacion) {
        
        log.info("🔢 Calculando: {} {} {}", numero1, operacion, numero2);
        
        int resultado;
        
        switch (operacion) {
            case "suma" -> resultado = numero1 + numero2;
            case "resta" -> resultado = numero1 - numero2;
            case "multiplicacion" -> resultado = numero1 * numero2;
            case "division" -> {
                if (numero2 == 0) {
                    throw new IllegalArgumentException("No se puede dividir por cero");
                }
                resultado = numero1 / numero2;
            }
            default -> throw new IllegalArgumentException("Operación no válida: " + operacion);
        }
        
        log.info("✅ Resultado: {}", resultado);
        
        return Map.of(
                "resultado", resultado,
                "operacionRealizada", String.format("%d %s %d = %d", 
                        numero1, operacion, numero2, resultado)
        );
    }
}
```

### Worker 3: Procesar Objetos JSON

Primero, crea un record para el modelo de datos:

**📁 src/main/java/com/ejemplo/camunda/model/Usuario.java**

```java
package com.ejemplo.camunda.model;

import lombok.Builder;

/**
 * Modelo de datos para un usuario.
 * Usando Java Record para inmutabilidad y menos boilerplate.
 */
@Builder
public record Usuario(
        String nombre,
        String email,
        Integer edad,
        String ciudad
) {
    /**
     * Valida que el usuario tenga datos correctos.
     */
    public boolean esValido() {
        return nombre != null && !nombre.isBlank() &&
               email != null && email.contains("@") &&
               edad != null && edad >= 18;
    }
}
```

**📁 src/main/java/com/ejemplo/camunda/worker/ValidadorUsuarioWorker.java**

```java
package com.ejemplo.camunda.worker;

import com.ejemplo.camunda.model.Usuario;
import io.camunda.zeebe.spring.client.annotation.JobWorker;
import io.camunda.zeebe.spring.client.annotation.VariablesAsType;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

/**
 * Worker que valida datos de usuario.
 */
@Slf4j
@Component
public class ValidadorUsuarioWorker {
    
    @JobWorker(type = "validar-usuario")
    public Map<String, Object> validarUsuario(@VariablesAsType Usuario usuario) {
        
        log.info("🔍 Validando usuario: {}", usuario.nombre());
        
        List<String> errores = new ArrayList<>();
        boolean esValido = true;
        
        // Validar nombre
        if (usuario.nombre() == null || usuario.nombre().isBlank()) {
            errores.add("El nombre es requerido");
            esValido = false;
        }
        
        // Validar email
        if (usuario.email() == null || !usuario.email().contains("@")) {
            errores.add("El email no es válido");
            esValido = false;
        }
        
        // Validar edad
        if (usuario.edad() == null || usuario.edad() < 18) {
            errores.add("El usuario debe ser mayor de 18 años");
            esValido = false;
        }
        
        if (esValido) {
            log.info("✅ Usuario válido: {}", usuario.email());
        } else {
            log.warn("⚠️ Usuario inválido. Errores: {}", errores);
        }
        
        return Map.of(
                "usuarioValido", esValido,
                "erroresValidacion", errores,
                "validadoEn", System.currentTimeMillis()
        );
    }
}
```

**🔑 Nota sobre `@VariablesAsType`:**
- Deserializa automáticamente todas las variables del proceso en un objeto
- Muy útil cuando el proceso tiene muchas variables relacionadas
- El objeto debe tener un constructor sin parámetros o ser un Record

### Proceso BPMN Completo con 3 Service Tasks

**📁 src/main/resources/bpmn/proceso-completo.bpmn**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<bpmn:definitions xmlns:bpmn="http://www.omg.org/spec/BPMN/20100524/MODEL"
                   xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
                   xmlns:dc="http://www.omg.org/spec/DD/20100524/DC"
                   xmlns:zeebe="http://camunda.org/schema/zeebe/1.0"
                   xmlns:di="http://www.omg.org/spec/DD/20100524/DI"
                   id="Definitions_1"
                   targetNamespace="http://bpmn.io/schema/bpmn">
  
  <bpmn:process id="proceso-completo" name="Proceso Completo de Ejemplo" isExecutable="true">
    
    <!-- Evento de Inicio -->
    <bpmn:startEvent id="StartEvent_1" name="Inicio">
      <bpmn:outgoing>Flow_1</bpmn:outgoing>
    </bpmn:startEvent>
    
    <!-- Service Task 1: Saludar -->
    <bpmn:serviceTask id="Task_Saludar" name="Saludar Usuario">
      <bpmn:extensionElements>
        <zeebe:taskDefinition type="saludar" />
      </bpmn:extensionElements>
      <bpmn:incoming>Flow_1</bpmn:incoming>
      <bpmn:outgoing>Flow_2</bpmn:outgoing>
    </bpmn:serviceTask>
    
    <!-- Service Task 2: Calcular -->
    <bpmn:serviceTask id="Task_Calcular" name="Realizar Cálculo">
      <bpmn:extensionElements>
        <zeebe:taskDefinition type="calcular" />
      </bpmn:extensionElements>
      <bpmn:incoming>Flow_2</bpmn:incoming>
      <bpmn:outgoing>Flow_3</bpmn:outgoing>
    </bpmn:serviceTask>
    
    <!-- Service Task 3: Validar Usuario -->
    <bpmn:serviceTask id="Task_ValidarUsuario" name="Validar Usuario">
      <bpmn:extensionElements>
        <zeebe:taskDefinition type="validar-usuario" />
      </bpmn:extensionElements>
      <bpmn:incoming>Flow_3</bpmn:incoming>
      <bpmn:outgoing>Flow_4</bpmn:outgoing>
    </bpmn:serviceTask>
    
    <!-- Evento de Fin -->
    <bpmn:endEvent id="EndEvent_1" name="Fin">
      <bpmn:incoming>Flow_4</bpmn:incoming>
    </bpmn:endEvent>
    
    <!-- Flujos -->
    <bpmn:sequenceFlow id="Flow_1" sourceRef="StartEvent_1" targetRef="Task_Saludar" />
    <bpmn:sequenceFlow id="Flow_2" sourceRef="Task_Saludar" targetRef="Task_Calcular" />
    <bpmn:sequenceFlow id="Flow_3" sourceRef="Task_Calcular" targetRef="Task_ValidarUsuario" />
    <bpmn:sequenceFlow id="Flow_4" sourceRef="Task_ValidarUsuario" targetRef="EndEvent_1" />
    
  </bpmn:process>
  
</bpmn:definitions>
```

**📊 Diagrama Visual:**

```
┌─────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌─────────┐
│ Inicio  │──>│   Saludar    │──>│   Calcular   │──>│   Validar    │──>│   Fin   │
│    ○    │   │  (String)    │   │  (Numbers)   │   │   (Object)   │   │   ◉     │
└─────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └─────────┘
```

### Actualizar la Aplicación Principal

**📁 src/main/java/com/ejemplo/camunda/CamundaWorkersApplication.java**

```java
package com.ejemplo.camunda;

import io.camunda.zeebe.spring.client.annotation.Deployment;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@Deployment(resources = {
        "classpath:bpmn/proceso-saludo.bpmn",
        "classpath:bpmn/proceso-completo.bpmn"
})
public class CamundaWorkersApplication {

    public static void main(String[] args) {
        SpringApplication.run(CamundaWorkersApplication.class, args);
    }
}
```

### Añadir Controller para el Proceso Completo

**📁 src/main/java/com/ejemplo/camunda/controller/ProcesoController.java** (actualizar)

Añade este método al controller existente:

```java
/**
 * Inicia una instancia del proceso completo.
 */
@PostMapping("/completo")
public ResponseEntity<Map<String, Object>> iniciarProcesoCompleto(
        @RequestBody Map<String, Object> request) {
    
    log.info("📝 Iniciando proceso completo con datos: {}", request);
    
    try {
        ProcessInstanceEvent processInstance = zeebeClient
                .newCreateInstanceCommand()
                .bpmnProcessId("proceso-completo")
                .latestVersion()
                .variables(request)
                .send()
                .join();
        
        log.info("✅ Proceso iniciado. Process Instance Key: {}", 
                processInstance.getProcessInstanceKey());
        
        return ResponseEntity.ok(Map.of(
                "processInstanceKey", processInstance.getProcessInstanceKey(),
                "bpmnProcessId", processInstance.getBpmnProcessId(),
                "version", processInstance.getVersion(),
                "mensaje", "Proceso completo iniciado correctamente"
        ));
        
    } catch (Exception e) {
        log.error("❌ Error iniciando proceso", e);
        return ResponseEntity.internalServerError()
                .body(Map.of("error", e.getMessage()));
    }
}

/**
 * Endpoint de prueba para proceso completo.
 */
@GetMapping("/completo/test")
public ResponseEntity<Map<String, Object>> testCompleto() {
    return iniciarProcesoCompleto(Map.of(
            "nombre", "María García",
            "email", "maria@ejemplo.com",
            "edad", 25,
            "ciudad", "Madrid",
            "numero1", 10,
            "numero2", 5,
            "operacion", "suma"
    ));
}
```

### Probar el Proceso Completo

```bash
# Reiniciar la aplicación
./mvnw spring-boot:run

# Probar el proceso completo
http GET localhost:8080/api/procesos/completo/test

# O con datos personalizados
http POST localhost:8080/api/procesos/completo \
  nombre="Carlos" \
  email="carlos@test.com" \
  edad:=30 \
  ciudad="Barcelona" \
  numero1:=20 \
  numero2:=4 \
  operacion="multiplicacion"
```

**Logs esperados:**

```
📝 Iniciando proceso completo con datos: {nombre=Carlos, ...}
✅ Proceso iniciado. Process Instance Key: 2251799813685251
👋 Procesando saludo para: Carlos
✅ Saludo generado: ¡Hola Carlos! Bienvenido a Camunda 8.8
🔢 Calculando: 20 multiplicacion 4
✅ Resultado: 80
🔍 Validando usuario: Carlos
✅ Usuario válido: carlos@test.com
```

---

## Despliegue de Procesos BPMN

Existen **tres formas** de desplegar procesos BPMN en Camunda 8.8. Vamos a explorar todas.

### Opción 1: Despliegue Automático con @Deployment (Recomendado para Desarrollo)

Ya lo has usado. Es la forma más simple para desarrollo:

```java
@SpringBootApplication
@Deployment(resources = "classpath:bpmn/*.bpmn")
public class CamundaWorkersApplication {
    // Despliega automáticamente todos los .bpmn al iniciar
}
```

**✅ Ventajas:**
- Automático al iniciar la aplicación
- Ideal para desarrollo local
- Soporta wildcards (`*.bpmn`)

**⚠️ Desventajas:**
- Redespliega en cada reinicio (crea nuevas versiones)
- No adecuado para producción
- No permite control fino del despliegue

### Opción 2: Despliegue Manual con Código Java

Para más control, puedes desplegar programáticamente:

**📁 src/main/java/com/ejemplo/camunda/service/DespliegueService.java**

```java
package com.ejemplo.camunda.service;

import io.camunda.zeebe.client.ZeebeClient;
import io.camunda.zeebe.client.api.response.DeploymentEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.io.Resource;
import org.springframework.core.io.ResourceLoader;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.io.InputStream;

/**
 * Servicio para gestionar despliegues de procesos BPMN.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class DespliegueService {

    private final ZeebeClient zeebeClient;
    private final ResourceLoader resourceLoader;

    /**
     * Despliega un proceso BPMN desde el classpath.
     *
     * @param rutaBpmn Ruta al archivo BPMN (ej: "classpath:bpmn/mi-proceso.bpmn")
     * @return Información del despliegue
     */
    public DeploymentEvent desplegarProceso(String rutaBpmn) throws IOException {
        log.info("📤 Desplegando proceso: {}", rutaBpmn);

        Resource resource = resourceLoader.getResource(rutaBpmn);

        try (InputStream inputStream = resource.getInputStream()) {
            DeploymentEvent deployment = zeebeClient.newDeployResourceCommand()
                    .addResourceStream(inputStream, resource.getFilename())
                    .send()
                    .join();

            log.info("✅ Proceso desplegado. Deployment Key: {}", deployment.getKey());
            
            deployment.getProcesses().forEach(process -> 
                log.info("   - Proceso: {} v{}", 
                    process.getBpmnProcessId(), 
                    process.getVersion())
            );

            return deployment;
        }
    }

    /**
     * Despliega múltiples procesos en un solo despliegue.
     */
    public DeploymentEvent desplegarMultiplesProcesos(String... rutas) throws IOException {
        log.info("📤 Desplegando {} procesos", rutas.length);

        var command = zeebeClient.newDeployResourceCommand();

        for (String ruta : rutas) {
            Resource resource = resourceLoader.getResource(ruta);
            try (InputStream inputStream = resource.getInputStream()) {
                command.addResourceStream(inputStream, resource.getFilename());
            }
        }

        DeploymentEvent deployment = command.send().join();
        
        log.info("✅ {} procesos desplegados. Deployment Key: {}", 
            rutas.length, deployment.getKey());

        return deployment;
    }
}
```

**Usar el servicio desde un Controller:**

```java
@RestController
@RequestMapping("/api/despliegues")
@RequiredArgsConstructor
public class DespliegueController {

    private final DespliegueService despliegueService;

    @PostMapping("/proceso")
    public ResponseEntity<Map<String, Object>> desplegarProceso(
            @RequestParam String rutaBpmn) {
        
        try {
            DeploymentEvent deployment = despliegueService.desplegarProceso(rutaBpmn);
            
            return ResponseEntity.ok(Map.of(
                "deploymentKey", deployment.getKey(),
                "procesos", deployment.getProcesses().stream()
                    .map(p -> Map.of(
                        "bpmnProcessId", p.getBpmnProcessId(),
                        "version", p.getVersion(),
                        "resourceName", p.getResourceName()
                    ))
                    .toList()
            ));
            
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                .body(Map.of("error", e.getMessage()));
        }
    }
}
```

**Probar con HTTPie:**

```bash
http POST "localhost:8080/api/despliegues/proceso?rutaBpmn=classpath:bpmn/proceso-saludo.bpmn"
```

### Opción 3: Despliegue desde Camunda Modeler (Manual)

Para diseñar visualmente y desplegar:

1. **Descargar Camunda Modeler:**
   ```
   https://camunda.com/download/modeler/
   ```

2. **Instalar y abrir Camunda Modeler**

3. **Crear un nuevo diagrama BPMN:**
   - File → New File → BPMN Diagram

4. **Diseñar tu proceso visualmente:**
   - Arrastra elementos desde el panel izquierdo
   - Configura cada Service Task:
     - Click en la tarea
     - Panel derecho → Implementation: Job Type
     - Escribe el "type" (ej: `saludar`)

5. **Guardar el archivo:**
   - File → Save File As
   - Guarda en `src/main/resources/bpmn/mi-proceso.bpmn`

6. **Desplegar desde Modeler:**
   - Click en el icono de cohete (🚀) en la parte superior
   - Selecciona "Camunda 8 Self-Managed"
   - Configurar conexión:
     ```
     Cluster endpoint: http://localhost:26500
     Authentication: None
     ```
   - Click "Deploy"

7. **Verificar despliegue:**
   ```bash
   # Ver en Operate
   open http://localhost:8088/operate
   
   # O verificar con HTTPie
   http localhost:8088/v2/topology
   ```

**📊 Comparación de Métodos:**

| Método | Cuándo Usar | Ventajas | Desventajas |
|--------|-------------|----------|-------------|
| **@Deployment** | Desarrollo local | Automático, simple | Redespliega cada vez |
| **Código Java** | CI/CD, producción | Control total, versionado | Requiere código |
| **Modeler** | Diseño, pruebas rápidas | Visual, intuitivo | Manual |

---

## Testing

Vamos a crear tests para asegurar que nuestros workers funcionan correctamente.

### Configuración de Testing

Añade la dependencia de Camunda Process Test al `pom.xml`:

```xml
<dependency>
    <groupId>io.camunda</groupId>
    <artifactId>camunda-process-test-spring-junit5</artifactId>
    <version>${camunda.version}</version>
    <scope>test</scope>
</dependency>
```

### Test 1: Test Unitario de Worker

**📁 src/test/java/com/ejemplo/camunda/worker/SaludoWorkerTest.java**

```java
package com.ejemplo.camunda.worker;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Tests unitarios para SaludoWorker.
 */
@SpringBootTest
class SaludoWorkerTest {

    @Autowired
    private SaludoWorker saludoWorker;

    @Test
    void deberiaSaludarCorrectamente() {
        // Given
        String nombre = "Carlos";

        // When
        Map<String, Object> resultado = saludoWorker.saludar(nombre);

        // Then
        assertThat(resultado).isNotNull();
        assertThat(resultado.get("mensaje"))
            .isEqualTo("¡Hola Carlos! Bienvenido a Camunda 8.8");
        assertThat(resultado.get("procesadoEn")).isNotNull();
    }

    @Test
    void deberiaGenerarMensajeDiferente() {
        // Given
        String nombre1 = "Ana";
        String nombre2 = "Luis";

        // When
        Map<String, Object> resultado1 = saludoWorker.saludar(nombre1);
        Map<String, Object> resultado2 = saludoWorker.saludar(nombre2);

        // Then
        assertThat(resultado1.get("mensaje"))
            .isNotEqualTo(resultado2.get("mensaje"));
    }
}
```

### Test 2: Test Unitario de CalculadoraWorker

**📁 src/test/java/com/ejemplo/camunda/worker/CalculadoraWorkerTest.java**

```java
package com.ejemplo.camunda.worker;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@SpringBootTest
class CalculadoraWorkerTest {

    @Autowired
    private CalculadoraWorker calculadoraWorker;

    @ParameterizedTest
    @CsvSource({
        "10, 5, suma, 15",
        "10, 5, resta, 5",
        "10, 5, multiplicacion, 50",
        "10, 5, division, 2"
    })
    void deberiaCalcularCorrectamente(Integer num1, Integer num2, 
                                       String operacion, Integer esperado) {
        // When
        Map<String, Object> resultado = calculadoraWorker.calcular(num1, num2, operacion);

        // Then
        assertThat(resultado.get("resultado")).isEqualTo(esperado);
    }

    @Test
    void deberiaLanzarExcepcionAlDividirPorCero() {
        // Given
        Integer num1 = 10;
        Integer num2 = 0;
        String operacion = "division";

        // When & Then
        assertThatThrownBy(() -> 
            calculadoraWorker.calcular(num1, num2, operacion))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("No se puede dividir por cero");
    }

    @Test
    void deberiaLanzarExcepcionConOperacionInvalida() {
        // When & Then
        assertThatThrownBy(() -> 
            calculadoraWorker.calcular(10, 5, "potencia"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("Operación no válida");
    }
}
```

### Test 3: Test de Integración del Proceso Completo

**📁 src/test/java/com/ejemplo/camunda/ProcesoCompletoIntegrationTest.java**

```java
package com.ejemplo.camunda;

import io.camunda.zeebe.client.ZeebeClient;
import io.camunda.zeebe.client.api.response.ProcessInstanceEvent;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.TestPropertySource;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Test de integración para el proceso completo.
 * Requiere que Camunda Docker esté corriendo.
 */
@SpringBootTest
@TestPropertySource(properties = {
    "camunda.client.zeebe.grpc-address=http://localhost:26500",
    "camunda.client.zeebe.rest-address=http://localhost:8088"
})
class ProcesoCompletoIntegrationTest {

    @Autowired
    private ZeebeClient zeebeClient;

    @Test
    void deberiaEjecutarProcesoCompletoCorrectamente() {
        // Given
        Map<String, Object> variables = Map.of(
            "nombre", "Test User",
            "email", "test@ejemplo.com",
            "edad", 25,
            "ciudad", "Madrid",
            "numero1", 10,
            "numero2", 5,
            "operacion", "suma"
        );

        // When
        ProcessInstanceEvent processInstance = zeebeClient
            .newCreateInstanceCommand()
            .bpmnProcessId("proceso-completo")
            .latestVersion()
            .variables(variables)
            .send()
            .join();

        // Then
        assertThat(processInstance.getProcessInstanceKey()).isPositive();
        assertThat(processInstance.getBpmnProcessId()).isEqualTo("proceso-completo");
    }
}
```

⚠️ **Nota sobre Testing:**

Este test requiere que tu cluster de Camunda Docker esté corriendo. Para tests completamente aislados, considera usar Camunda Process Test (actualmente en desarrollo en 8.8).

### Ejecutar los Tests

```bash
# Ejecutar todos los tests
./mvnw test

# Ejecutar una clase específica
./mvnw test -Dtest=SaludoWorkerTest

# Ejecutar con verbose
./mvnw test -X
```

**Salida esperada:**

```
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.ejemplo.camunda.worker.SaludoWorkerTest
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
[INFO] Running com.ejemplo.camunda.worker.CalculadoraWorkerTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 5, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] BUILD SUCCESS
```

### Cobertura de Tests con JaCoCo (Opcional)

Añadir al `pom.xml`:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Generar reporte:

```bash
./mvnw clean test jacoco:report

# Ver reporte en:
open target/site/jacoco/index.html
```

---

## Troubleshooting

### Problema 1: Aplicación no arranca - "Connection refused"

**Síntoma:**
```
Failed to connect to localhost:26500
```

**Causa:** Camunda Docker no está corriendo.

**Solución:**
```bash
cd ~/camunda-local
docker compose ps

# Si no están corriendo:
docker compose up -d

# Verificar que estén healthy:
docker compose ps

# Verificar conectividad con HTTPie:
http localhost:8088/v2/topology
```

### Problema 2: Worker no procesa jobs

**Síntoma:**
- El proceso se inicia
- El job queda en estado "activatable"
- Worker no lo procesa

**Causas posibles:**

1. **El "type" no coincide:**
   ```java
   // BPMN: <zeebe:taskDefinition type="saludar" />
   // Worker: @JobWorker(type = "saludarr")  ❌ Typo
   ```

2. **Worker no está registrado como Component:**
   ```java
   // ❌ Falta @Component
   public class SaludoWorker {
   
   // ✅ Correcto
   @Component
   public class SaludoWorker {
   ```

3. **Aplicación no está corriendo:**
   ```bash
   # Verificar con HTTPie
   http localhost:8080/actuator/health
   ```

**Solución:**
- Verificar logs de la aplicación
- Buscar: "Registered job worker" en los logs
- Si no aparece, revisar anotaciones

### Problema 3: Proceso no se despliega

**Síntoma:**
```
No deployment found for process definition
```

**Soluciones:**

1. **Verificar que el archivo BPMN existe:**
   ```bash
   ls -la src/main/resources/bpmn/
   ```

2. **Verificar la anotación @Deployment:**
   ```java
   @Deployment(resources = "classpath:bpmn/mi-proceso.bpmn")
   //                      ^^^^^^^ Verificar esta ruta
   ```

3. **Ver logs de despliegue:**
   ```
   Buscar: "Deploying resource" en los logs de inicio
   ```

4. **Verificar sintaxis del BPMN:**
   - Abrir con Camunda Modeler
   - Verificar que no haya errores de validación

### Problema 4: Error "Variable not found"

**Síntoma:**
```
Expected to have variable with name 'nombre', but not found
```

**Causa:** El proceso se inició sin la variable necesaria.

**Solución:**

1. **Verificar que pasas todas las variables:**
   ```bash
   # ❌ Falta la variable "nombre"
   http POST localhost:8080/api/procesos/saludo
   
   # ✅ Correcto
   http POST localhost:8080/api/procesos/saludo nombre="Juan"
   ```

2. **Verificar la deserialización:**
   ```java
   // Si usas @VariablesAsType, asegúrate de que los nombres coincidan
   public record Usuario(
       String nombre,  // Este nombre debe coincidir con la variable del proceso
       String email
   ) {}
   ```

3. **Usar fetchVariables para debugging:**
   ```java
   @JobWorker(type = "saludar", fetchVariables = {"nombre"})
   public Map<String, Object> saludar(@Variable String nombre) {
       // Camunda solo pedirá la variable "nombre"
   }
   ```

### Problema 5: Job Worker timeout

**Síntoma:**
```
Job worker timeout exceeded
```

**Causa:** El worker tarda más que el timeout configurado.

**Soluciones:**

1. **Aumentar timeout del worker:**
   ```java
   @JobWorker(
       type = "tarea-larga",
       timeout = 600_000L  // 10 minutos en milisegundos
   )
   ```

2. **Aumentar timeout global en application.yml:**
   ```yaml
   camunda:
     client:
       zeebe:
         default-job-timeout: PT10M  // 10 minutos
   ```

3. **Optimizar el código del worker:**
   - Reducir tiempo de procesamiento
   - Mover lógica pesada a servicios asíncronos

### Problema 6: Lombok no funciona

**Síntoma:**
```
Cannot resolve symbol 'log'
Cannot resolve method 'builder'
```

**Causa:** Lombok no está configurado en el IDE.

**Soluciones:**

**IntelliJ IDEA:**
1. Settings → Plugins → Buscar "Lombok" → Install
2. Settings → Build, Execution, Deployment → Compiler → Annotation Processors
3. Activar "Enable annotation processing"
4. Rebuild project

**Eclipse:**
1. Descargar lombok.jar de https://projectlombok.org/download
2. Ejecutar: `java -jar lombok.jar`
3. Seleccionar Eclipse installation
4. Restart Eclipse

**VS Code:**
1. Instalar extensión "Lombok Annotations Support"
2. Reload window

### Problema 7: Tests fallan - "Cannot autowire ZeebeClient"

**Síntoma:**
```
No qualifying bean of type 'ZeebeClient' available
```

**Causa:** Falta configuración de test properties.

**Solución:**

```java
@SpringBootTest
@TestPropertySource(properties = {
    "camunda.client.zeebe.grpc-address=http://localhost:26500",
    "camunda.client.zeebe.rest-address=http://localhost:8088"
})
class MiTest {
    // Tests aquí
}
```

O crear **📁 src/test/resources/application-test.yml:**

```yaml
camunda:
  client:
    zeebe:
      grpc-address: http://localhost:26500
      rest-address: http://localhost:8088
```

### Problema 8: Puerto 8080 ya en uso

**Síntoma:**
```
Web server failed to start. Port 8080 was already in use.
```

**Soluciones:**

1. **Cambiar puerto en application.yml:**
   ```yaml
   server:
     port: 8081  // O cualquier puerto libre
   ```

2. **Identificar qué usa el puerto:**
   ```bash
   # macOS
   lsof -i :8080
   
   # Linux
   netstat -tuln | grep 8080
   
   # Matar el proceso
   kill -9 <PID>
   ```

### Problema 9: "Unable to parse BPMN file"

**Síntoma:**
```
Failed to parse BPMN file: proceso.bpmn
```

**Causas y Soluciones:**

1. **XML mal formado:**
   - Abrir con Camunda Modeler
   - Verificar errores de sintaxis
   - Guardar de nuevo

2. **Namespace incorrecto:**
   ```xml
   <!-- ✅ Correcto -->
   xmlns:zeebe="http://camunda.org/schema/zeebe/1.0"
   
   <!-- ❌ Incorrecto -->
   xmlns:zeebe="http://camunda.org/schema/zeebe/1.5"
   ```

3. **taskDefinition vacío:**
   ```xml
   <!-- ❌ Falta el type -->
   <zeebe:taskDefinition />
   
   <!-- ✅ Correcto -->
   <zeebe:taskDefinition type="saludar" />
   ```

### Comandos Útiles para Debugging

```bash
# Ver logs en tiempo real
./mvnw spring-boot:run | grep -i "error\|warn\|worker"

# Ver todos los endpoints disponibles
http localhost:8080/actuator/mappings

# Ver métricas de la aplicación
http localhost:8080/actuator/metrics

# Ver información del ambiente
http localhost:8080/actuator/env

# Ver health check
http localhost:8080/actuator/health

# Verificar procesos desplegados en Camunda
http localhost:8088/v2/topology
```

### Logs Importantes a Buscar

| Log | Significado | Acción |
|-----|-------------|--------|
| `Registered job worker for type 'saludar'` | ✅ Worker registrado | Ninguna |
| `Deploying resource proceso.bpmn` | ✅ Proceso desplegado | Ninguna |
| `Connection refused` | ❌ Camunda no accesible | Verificar Docker |
| `Variable not found` | ❌ Variable faltante | Verificar proceso/worker |
| `Job activation timeout` | ⚠️ Worker lento | Aumentar timeout |
| `Failed to parse BPMN` | ❌ BPMN inválido | Verificar sintaxis |

### Habilitar Debug Logging

Para ver más información detallada:

**📁 src/main/resources/application.yml**

```yaml
logging:
  level:
    root: INFO
    io.camunda: DEBUG              # Logs de Camunda
    io.camunda.zeebe: DEBUG        # Logs de Zeebe
    com.ejemplo.camunda: TRACE     # Tus logs
    
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
```

---

## ✅ Checklist de Verificación

Antes de considerar que tu setup está completo, verifica:

### Infraestructura
- [ ] Docker Compose está corriendo (`docker compose ps`)
- [ ] Todos los servicios están "healthy"
- [ ] Puedes acceder a Operate (http://localhost:8088/operate)
- [ ] Swagger UI funciona (http://localhost:8088/swagger-ui/index.html)

### Aplicación Spring Boot
- [ ] La aplicación arranca sin errores (`./mvnw spring-boot:run`)
- [ ] Los workers están registrados (ver logs: "Registered job worker")
- [ ] Los procesos se despliegan automáticamente (ver logs: "Deploying resource")
- [ ] Puedes acceder a los actuator endpoints

### Workers
- [ ] Cada worker tiene `@Component`
- [ ] Cada worker tiene `@JobWorker(type = "...")`
- [ ] El "type" coincide con el BPMN
- [ ] Los workers tienen logging (`@Slf4j`)

### Procesos BPMN
- [ ] Los archivos .bpmn están en `src/main/resources/bpmn/`
- [ ] Cada Service Task tiene `<zeebe:taskDefinition type="..." />`
- [ ] Los procesos tienen `isExecutable="true"`
- [ ] Los IDs de proceso no tienen espacios ni caracteres especiales

### Testing
- [ ] Los tests unitarios pasan (`./mvnw test`)
- [ ] Puedes iniciar procesos vía HTTP
- [ ] Los workers procesan los jobs correctamente
- [ ] Puedes ver las instancias completadas en Operate

### Conectividad
- [ ] HTTPie funciona: `http localhost:8080/actuator/health`
- [ ] Puedes iniciar un proceso: `http GET localhost:8080/api/procesos/saludo/test`
- [ ] Camunda REST API responde: `http localhost:8088/v2/topology`

---

## 🎯 Próximos Pasos

¡Felicidades! Has completado los primeros pasos con Camunda 8.8. Ahora estás listo para:

1. **Documento 03 - Ejemplo Completo**: Sistema de Pedidos
   - 4 workers más complejos
   - Manejo de errores avanzado
   - BPMN Errors y compensación
   - Integración con bases de datos

2. **Documento 04 - Conceptos Avanzados**:
   - Long Polling y Job Streaming
   - Backpressure y control de flujo
   - Timeouts dinámicos
   - Idempotencia
   - Métricas y monitoreo

3. **Explorar Camunda Modeler**:
   - Diseñar procesos visualmente
   - Gateways (exclusivos, paralelos)
   - Eventos (timers, mensajes, errores)
   - Subprocesos

4. **Profundizar en Testing**:
   - Mocking de servicios externos
   - Tests de integración más complejos
   - Test de carga con JMeter

---

## 📚 Recursos Adicionales

### Documentación Oficial
- [Camunda 8.8 Docs](https://docs.camunda.io/)
- [Java Client Documentation](https://docs.camunda.io/docs/apis-tools/java-client/getting-started/)
- [Spring Boot Starter](https://docs.camunda.io/docs/apis-tools/camunda-spring-boot-starter/getting-started/)
- [BPMN Tutorial](https://docs.camunda.io/docs/components/modeler/bpmn/bpmn-primer/)

### Herramientas
- [Camunda Modeler](https://camunda.com/download/modeler/)
- [HTTPie](https://httpie.io/)
- [Postman Collection](https://www.postman.com/camundateam/camunda-8-postman/collection/)

### Comunidad
- [Camunda Forum](https://forum.camunda.io/)
- [GitHub - Ejemplos](https://github.com/camunda/camunda-platform-examples)
- [GitHub - Vicente Priego](https://github.com/vicente-priego-aviles)

---

**¡Excelente trabajo! →** Continúa con [03 - Ejemplo Completo](./03-ejemplo-completo.md)

En el siguiente documento implementaremos un sistema completo de gestión de pedidos con:
- 4 Job Workers especializados
- Manejo de errores robusto
- Integración con servicios externos
- Logging y métricas detalladas
- Suite completa de tests