# =============================================================================
# FASE DE CONSTRUCCION (Build Stage)
# Usamos Node 20 Alpine (imagen minimalista) para compilar la aplicacion React
# No incluimos Node en la imagen final -> imagen de produccion mas pequena y segura
# =============================================================================
FROM node:20-alpine AS builder

# Directorio de trabajo dentro del contenedor
WORKDIR /app

# OPTIMIZACION DE CAPAS:
# Copiamos PRIMERO solo los archivos de dependencias (package.json y package-lock.json)
# Docker guarda esta capa en cache. Si el codigo cambia pero las dependencias no,
# Docker NO vuelve a ejecutar "npm ci" -> builds hasta 3x mas rapidos en CI/CD
COPY package.json package-lock.json ./

# Instalacion con npm ci: mas estricto y reproducible que npm install
# Garantiza que el entorno de CI/CD es identico al de desarrollo
RUN npm ci

# Copiamos el resto del codigo fuente DESPUES de instalar dependencias
# Solo esta capa se invalida cuando cambia el codigo fuente
COPY . .

# Compilamos la aplicacion React para produccion
# Vite genera archivos estaticos optimizados (minificados, tree-shaken) en /app/dist
RUN npm run build

# =============================================================================
# FASE DE PRODUCCION (Production Stage)
# Usamos la imagen oficial de Nginx sin privilegios de root
# Solo copiamos los archivos compilados de la fase anterior -> imagen final ~25MB
# =============================================================================
FROM nginxinc/nginx-unprivileged:alpine

# SEGURIDAD - Usuario no root:
# La imagen nginxinc/nginx-unprivileged ya viene configurada para correr como
# el usuario "nginx" (UID 101) en lugar de root. Esto significa que si existe
# una vulnerabilidad en Nginx o en la aplicacion, el atacante no obtiene
# privilegios de root sobre el host Docker ni sobre la instancia EC2.

# Copiamos nuestra configuracion de Nginx personalizada
# Esta config escucha en el puerto 8080 (no privilegiado, sin necesitar root)
# y define las reglas de proxy reverso hacia el backend
COPY nginx.conf /etc/nginx/nginx.conf

# Copiamos los archivos compilados por Vite desde la etapa anterior (builder)
# Solo los archivos estaticos del build (HTML, JS, CSS, imagenes)
# El codigo fuente Node.js, los node_modules y las herramientas de desarrollo
# NO estan presentes en esta imagen final
COPY --from=builder /app/dist /usr/share/nginx/html

# Documentamos el puerto en que corre el servicio dentro del contenedor
# El mapeo real (host:80 -> contenedor:8080) se define en docker-compose.yml
EXPOSE 8080

# La imagen nginxinc/nginx-unprivileged ya define el CMD correcto para iniciar Nginx
# No necesitamos redefinirlo aqui