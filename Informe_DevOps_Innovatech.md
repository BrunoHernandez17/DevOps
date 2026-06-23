# Informe de DevOps y CI/CD: Proyecto Innovatech Chile

**Autor:** Bruno Hernández Baro  
**Repositorio:** [BrunoHernandez17/DevOps](https://github.com/BrunoHernandez17/DevOps.git)  
**Fecha:** 22 de Junio de 2026  
**Curso:** DevOps e Infraestructura Cloud (2025/2026)  

---

## 1. Resumen Ejecutivo
El presente informe documenta la evaluación, optimización y propuesta arquitectónica para el pipeline de Integración y Despliegue Continuo (CI/CD) de la plataforma **Innovatech Chile**. 

El sistema original, basado en contenedores Docker locales, fue automatizado a través de GitHub Actions utilizando un mecanismo de despliegue sin agente SSH mediante **AWS Systems Manager (SSM) Send-Command** sobre instancias EC2. Además, se detalla la propuesta técnica para migrar esta solución hacia un clúster completamente orquestado bajo estándares de producción (ECS/EKS).

---

## 2. Tabla de Métricas de Ejecución del Pipeline

Tras aplicar técnicas de optimización y depuración, el pipeline de GitHub Actions se ejecuta con los siguientes tiempos por job:

| Job | Descripción | Tiempo Inicial | Tiempo Optimizado | Impacto de la Optimización |
| :--- | :--- | :--- | :--- | :--- |
| **🔍 Validar Archivos YAML** (`lint-yaml`) | Linter preliminar que valida la sintaxis de todos los archivos YAML del repositorio. | N/A | **~10s** | **Nuevo**. Evita fallas tempranas en compilación y despliegue por errores tipográficos. |
| **🧪 Pruebas Unitarias Backend** (`test-backend`) | Ejecución de pruebas unitarias (`mvn test`) utilizando Java JDK 17 y base de datos H2. | N/A | **~42s** | **Nuevo**. Emplea caché nativa de dependencias Maven (`cache: 'maven'`), reduciendo el tiempo de descarga de librerías. |
| **🐳 Build y Push Docker** (`build-and-push`) | Compilación de imágenes del Backend (Spring) y Frontend (React + Nginx) y subida a Docker Hub. | ~3m 15s | **~1m 20s** | **Reducción del 59%** gracias al uso del sistema de caché de capas de Docker (`cache-from/to: type=gha`). |
| **🚀 Deploy en EC2 via SSM** (`deploy`) | Conexión segura e inyección de comando en la instancia EC2 utilizando AWS CLI. | ~50s | **~45s** | **Ejecución Directa**. El uso del Agente SSM elimina la latencia de negociación de llaves y handshake SSH. |
| **Total del Pipeline** | Ciclo completo desde el Push hasta el despliegue funcional en vivo. | ~4m 05s | **~2m 57s** | **Optimización general del 28%** en la duración total de la entrega. |

---

## 3. Registro de Problemas Encontrados y Resoluciones

Durante el ciclo de desarrollo se identificaron y solucionaron 4 cuellos de botella técnicos:

### A. Confusión en la Región de AWS (`us-west-2` vs `us-east-1`)
* **Problema:** El portal de AWS Academy opera su capa de autenticación en Oregón (`us-west-2`), inyectando dicho valor en la cabecera de las credenciales temporales. No obstante, las instancias EC2 y subredes fueron desplegadas físicamente en N. Virginia (`us-east-1`). El pipeline fallaba al no ubicar la instancia en la región incorrecta.
* **Resolución:** Se fijó la variable `aws-region` del pipeline en `us-east-1`, manteniendo las credenciales de Academy funcionales para conectarse de forma cruzada.

### B. Fallo de Pruebas Unitarias por Contexto de Base de Datos
* **Problema:** El job de test (`mvn test`) fallaba en el runner de GitHub porque las pruebas de integración (`SpringBootTest`) intentaban conectarse a la base de datos MySQL de producción, la cual requiere variables de entorno del host y no existe dentro del runner de compilación.
* **Resolución:** Se configuró la anotación `@ActiveProfiles("test")` en las clases de prueba para forzar el uso de base de datos en memoria **H2** (especificada en `application-test.properties`).

### C. Advertencias de Dockerfile Depreciado en el Runner
* **Problema:** GitHub Actions emitía advertencias amarillas debido al uso de la propiedad obsoleta `dockerfile` en la acción de compilación de Docker.
* **Resolución:** Se reemplazó por la propiedad `file` para alinearse con la especificación actual de `docker/build-push-action@v5`.

### D. Carga de Secretos Locales en Windows
* **Problema:** El script automatizado `configure_secrets.py` fallaba con `ModuleNotFoundError` en entornos sin privilegios de administrador, ya que `pip` instalaba las librerías `requests` y `pynacl` en el espacio de usuario, el cual no estaba en el `sys.path` por defecto.
* **Resolución:** Se programó una inyección dinámica que añade `site.getusersitepackages()` al listado de rutas de importación de Python al atrapar la excepción `ImportError`.

---

## 4. Mejoras Implementadas e Impacto Esperado

1. **Caché en GitHub Actions (Maven y Docker):** Reducción drástica del tiempo de build (de 4 minutos a menos de 3). Se reduce el uso de minutos de cómputo del runner de GitHub.
2. **Versionamiento Semántico Dinámico:** El pipeline ahora detecta tags de git tipo `v*` (ej: `v1.0.3`) y publica la imagen en Docker Hub bajo esa versión. Esto asegura la trazabilidad del código y permite realizar **Rollbacks** inmediatos en producción si se detectan bugs.
3. **Linter YAML Integrado:** Detiene la ejecución si hay un error de indentación en los Compose o workflows, ahorrando tiempo de procesamiento en el runner.

---

## 5. Diseño y Sustento de Arquitectura para Producción (ECS vs EKS)

Para migrar Innovatech Chile a un modelo de orquestación empresarial, se exponen los siguientes argumentos técnicos:

### A. Selección del Orquestador: AWS ECS con Fargate
Se selecciona **AWS ECS (Elastic Container Service) con Fargate** como la arquitectura destino ideal.
* **Justificación Operativa:** EKS (Kubernetes) introduce una sobrecarga operativa innecesaria para una aplicación web tradicional de tres capas. Requiere administrar actualizaciones de API de Kubernetes, balanceadores Ingress complejos e inyectores de seguridad. ECS Fargate proporciona un entorno **Serverless (sin servidores)** que permite centrarse en el contenedor y escala nativamente sin aprovisionar nodos EC2.
* **Capacity Providers:** Permiten mezclar tareas `Fargate` normales para producción y `Fargate Spot` para pruebas, ahorrando hasta un 70% de costos de infraestructura.
* **Roles IAM:**
  * **ECS Task Execution Role:** Utilizado por el agente de ECS para descargar las imágenes de ECR y registrar logs en CloudWatch.
  * **ECS Task Role:** Otorga permisos específicos al contenedor en ejecución (por ejemplo, escribir a un bucket S3 o conectarse a RDS).

### B. Segmentación de Redes, Subredes y Security Groups
La infraestructura en AWS se estructura bajo el principio de menor privilegio con 3 capas lógicas en una VPC:

1. **Capa Pública (Subredes Públicas):**
   * Aloja al **Application Load Balancer (ALB)**.
   * **Security Group (SG-ALB):** Permite entrada pública en puertos `80` (HTTP) y `443` (HTTPS) desde cualquier origen (`0.0.0.0/0`).
2. **Capa de Aplicación (Subredes Privadas):**
   * Aloja los contenedores del Frontend (React/Nginx) y del Backend (Spring Boot).
   * **SG-Frontend:** Solo permite conexiones en el puerto `3000` si provienen del `SG-ALB`.
   * **SG-Backend:** Solo permite conexiones en el puerto `8080` si provienen de la subred del Frontend o del `SG-ALB`.
3. **Capa de Datos (Subredes Privadas de Datos):**
   * Aloja el servicio de base de datos MySQL (o Amazon RDS MySQL).
   * **SG-Datos:** Bloquea todo acceso exterior. Solo permite conexiones entrantes en el puerto `3306` que provengan estrictamente del `SG-Backend`.

### C. Estrategia de Autoscaling (Target Tracking)
* **Mecanismo:** Target Tracking sobre uso de CPU al **70%** y Memoria al **80%**.
* **Justificación:** Spring Boot tiene un consumo de memoria inicial alto debido a la JVM (Java Virtual Machine), por lo que el escalamiento por memoria previene caídas por desbordamiento (`OutOfMemoryError`). La CPU al 70% actúa como indicador preventivo ante ráfagas transaccionales, levantando nuevos contenedores antes de que se degrade el tiempo de respuesta.

### D. Monitoreo y Tolerancia a Fallos
* **CloudWatch Container Insights:** Habilitado para recopilar métricas de salud de las tareas de ECS.
* **Tolerancia a fallos:** Despliegue Multi-AZ equilibrado. Si la zona de disponibilidad `us-east-1a` se cae, el ALB redirige el 100% de las peticiones a la réplica corriendo en `us-east-1b` de forma transparente para el cliente final.
* **Auto-recuperación (Self-Healing):** Si una tarea de ECS falla en su healthcheck HTTP local, ECS la destruye y levanta una nueva tarea de forma automática para mantener el número deseado de réplicas activas.
