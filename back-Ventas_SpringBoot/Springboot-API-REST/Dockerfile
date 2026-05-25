# =============================================================================
# FASE DE CONSTRUCCION (Build Stage)
# Usamos la imagen oficial de Maven con JDK 17 Alpine para compilar el proyecto
# Maven + JDK son herramientas pesadas que NO queremos en la imagen de produccion
# =============================================================================
FROM maven:3.9.6-eclipse-temurin-17-alpine AS builder

# Directorio de trabajo dentro del contenedor de compilacion
WORKDIR /app

# OPTIMIZACION DE CAPAS - Descarga de dependencias en cache:
# Copiamos SOLO el pom.xml primero y ejecutamos dependency:go-offline
# Docker cachea esta capa. Si el codigo cambia pero el pom.xml no,
# Maven NO vuelve a descargar dependencias de internet -> builds hasta 5x mas rapidos
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copiamos el codigo fuente DESPUES de cachear las dependencias
# Solo esta capa se invalida cuando el codigo fuente cambia
COPY src ./src

# Compilamos el proyecto y empaquetamos el JAR ejecutable
# -DskipTests: omitimos tests en el build de Docker (se ejecutan en el pipeline CI)
# -B: modo batch (sin colores), limpio para logs de CI/CD
RUN mvn package -DskipTests -B

# =============================================================================
# FASE DE PRODUCCION (Production Stage)
# Usamos eclipse-temurin JRE 17 Alpine: solo el entorno de ejecucion de Java
# NO incluye Maven, JDK completo ni el codigo fuente -> imagen ~175MB vs ~600MB
# =============================================================================
FROM eclipse-temurin:17-jre-alpine

# SEGURIDAD - Creacion de usuario y grupo no privilegiado:
# Creamos el grupo "spring" y el usuario "spring" sin shell de login ni directorio home
# Si hay una vulnerabilidad RCE (Remote Code Execution) en la aplicacion,
# el atacante queda atrapado como el usuario "spring" sin privilegios sobre el host EC2
RUN addgroup -S spring && adduser -S spring -G spring

# Directorio de trabajo para la aplicacion
WORKDIR /app

# Copiamos UNICAMENTE el JAR compilado desde la etapa anterior (builder)
# El resto del filesystem del builder (codigo fuente, dependencias Maven, JDK)
# queda descartado automaticamente por Docker -> imagen final mas pequeña y segura
COPY --from=builder /app/target/*.jar app.jar

# Asignamos la propiedad del archivo JAR al usuario spring
# Garantizamos que el usuario no root pueda leer y ejecutar la aplicacion
RUN chown spring:spring app.jar

# Cambiamos al usuario no privilegiado "spring" para la ejecucion
# A partir de esta instruccion, todos los comandos siguientes corren como "spring"
USER spring:spring

# Documentamos el puerto en que escucha la aplicacion Spring Boot
# El mapeo real al puerto del host (3001:8080) se define en docker-compose.yml
EXPOSE 3001

# Comando de inicio: ejecuta el JAR de Spring Boot
# La JVM recibe las variables de entorno inyectadas por Docker Compose o EC2
ENTRYPOINT ["java", "-jar", "app.jar"]