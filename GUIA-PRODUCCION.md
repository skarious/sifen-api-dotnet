# Guía Paso a Paso para Poner en Producción SifenApi

Esta guía está diseñada para personas sin experiencia en C# que necesitan desplegar este sistema de facturación electrónica.

## Tabla de Contenidos
1. [¿Qué es este sistema?](#qué-es-este-sistema)
2. [Requisitos previos](#requisitos-previos)
3. [Instalación del software necesario](#instalación-del-software-necesario)
4. [Configuración inicial](#configuración-inicial)
5. [Despliegue con Docker (Recomendado)](#despliegue-con-docker-recomendado)
6. [Despliegue manual (Alternativa)](#despliegue-manual-alternativa)
7. [Verificación del sistema](#verificación-del-sistema)
8. [Configuración de seguridad en producción](#configuración-de-seguridad-en-producción)
9. [Monitoreo y mantenimiento](#monitoreo-y-mantenimiento)
10. [Solución de problemas comunes](#solución-de-problemas-comunes)

---

## ¿Qué es este sistema?

**SifenApi** es una API que te permite:
- Generar facturas electrónicas válidas para Paraguay
- Enviar documentos al sistema SIFEN del gobierno
- Firmar digitalmente los documentos
- Generar códigos QR y PDFs de las facturas

Es como un puente entre tu sistema de ventas y el sistema de facturación electrónica del gobierno paraguayo.

---

## Requisitos previos

### Lo que DEBES tener antes de empezar:

1. **Un servidor o computadora** donde instalar el sistema:
   - **Mínimo**: 2 procesadores, 4GB RAM, 50GB disco
   - **Recomendado**: 4 procesadores, 8GB RAM, 100GB disco
   - Sistema operativo: Windows Server, Ubuntu Linux, o similar

2. **Certificado digital SIFEN**:
   - Un archivo `.pfx` o `.pem` proporcionado por la SET (Subsecretaría de Estado de Tributación)
   - La contraseña de ese certificado
   - Este certificado es OBLIGATORIO para comunicarte con SIFEN

3. **Datos de acceso a SIFEN**:
   - RUC de tu empresa
   - Usuario y contraseña del sistema SIFEN
   - URL del ambiente (test o producción)

4. **Acceso a internet** con los siguientes puertos abiertos:
   - Puerto 80 (HTTP)
   - Puerto 443 (HTTPS)
   - Acceso a `sifen.set.gov.py`

---

## Instalación del software necesario

### Opción A: Usando Docker (MÁS FÁCIL - RECOMENDADO)

#### Windows:

1. **Descargar Docker Desktop**:
   - Ve a: https://www.docker.com/products/docker-desktop
   - Descarga "Docker Desktop for Windows"
   - Ejecuta el instalador
   - Reinicia tu computadora cuando te lo pida

2. **Verificar la instalación**:
   - Abre "PowerShell" o "Símbolo del sistema"
   - Escribe: `docker --version`
   - Deberías ver algo como: `Docker version 24.0.x`

#### Linux (Ubuntu/Debian):

1. **Instalar Docker**:
   ```bash
   # Actualizar el sistema
   sudo apt update
   sudo apt upgrade -y

   # Instalar Docker
   curl -fsSL https://get.docker.com -o get-docker.sh
   sudo sh get-docker.sh

   # Agregar tu usuario al grupo docker
   sudo usermod -aG docker $USER

   # Instalar Docker Compose
   sudo apt install docker-compose -y

   # Reiniciar sesión o ejecutar:
   newgrp docker
   ```

2. **Verificar la instalación**:
   ```bash
   docker --version
   docker-compose --version
   ```

### Opción B: Sin Docker (manual)

#### Windows:

1. **Descargar .NET 9**:
   - Ve a: https://dotnet.microsoft.com/download/dotnet/9.0
   - Descarga ".NET 9.0 Runtime" (Hosting Bundle para Windows)
   - Ejecuta el instalador

2. **Instalar SQL Server**:
   - Descarga SQL Server 2022 Express: https://www.microsoft.com/sql-server/sql-server-downloads
   - Instala también SQL Server Management Studio (SSMS)

#### Linux (Ubuntu):

```bash
# Instalar .NET 9
wget https://dot.net/v1/dotnet-install.sh -O dotnet-install.sh
chmod +x dotnet-install.sh
./dotnet-install.sh --channel 9.0

# Agregar al PATH
echo 'export PATH=$PATH:$HOME/.dotnet' >> ~/.bashrc
source ~/.bashrc

# Instalar SQL Server (Ubuntu)
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/mssql-server-2022.list)"
sudo apt-get update
sudo apt-get install -y mssql-server
sudo /opt/mssql/bin/mssql-conf setup
```

---

## Configuración inicial

### 1. Descargar el código

```bash
# Navegar a donde quieras instalar el sistema
cd /ruta/donde/quieras/instalarlo

# Si tienes git instalado:
git clone https://github.com/turepositorio/sifen-api-dotnet.git
cd sifen-api-dotnet

# Si no tienes git, descarga el ZIP desde GitHub y descomprímelo
```

### 2. Preparar el certificado digital

```bash
# Crear carpeta para certificados
mkdir -p certificados

# Copiar tu certificado a esa carpeta
# En Windows:
copy "C:\ruta\a\tu\certificado.pfx" "certificados\certificado.pfx"

# En Linux:
cp /ruta/a/tu/certificado.pfx certificados/certificado.pfx
```

### 3. Configurar el archivo de producción

Necesitas crear un archivo llamado `appsettings.Production.json` en la carpeta `src/SifenApi.WebApi/`

```bash
# Navegar a la carpeta correcta
cd src/SifenApi.WebApi

# Copiar el archivo de ejemplo
cp appsettings.json appsettings.Production.json
```

Ahora edita `appsettings.Production.json` con un editor de texto (Notepad++, nano, vim, VS Code, etc.) y configura lo siguiente:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=SifenApiDb;User Id=sa;Password=TU_PASSWORD_SEGURO;TrustServerCertificate=true;",
    "Redis": "localhost:6379"
  },
  "Jwt": {
    "Key": "GENERA_UNA_CLAVE_SUPER_SEGURA_DE_AL_MENOS_256_BITS_AQUI_12345678901234567890",
    "Issuer": "SifenApi",
    "Audience": "SifenApiUsers",
    "DurationInMinutes": 60
  },
  "Sifen": {
    "UrlTest": "https://sifen-test.set.gov.py/de/ws/sync",
    "UrlProd": "https://sifen.set.gov.py/de/ws/sync",
    "UrlConsultaPublica": "https://ekuatia.set.gov.py/consultas",
    "CertificatePath": "/app/certificados/certificado.pfx",
    "CertificatePassword": "TU_PASSWORD_DEL_CERTIFICADO"
  },
  "Storage": {
    "BasePath": "/app/storage"
  },
  "Email": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": 587,
    "SmtpUser": "tu-email@gmail.com",
    "SmtpPassword": "tu-password-de-aplicacion",
    "FromEmail": "noreply@tu-empresa.com.py",
    "FromName": "Sistema Facturación"
  },
  "Cors": {
    "AllowedOrigins": ["https://tu-dominio-produccion.com"]
  },
  "ApiKeys": {
    "ValidKeys": ["GENERA_UNA_API_KEY_SEGURA_AQUI_123456"]
  },
  "AllowedHosts": "*"
}
```

**IMPORTANTE - Lee esto:**

- **DefaultConnection**: Cambia `TU_PASSWORD_SEGURO` por una contraseña fuerte para SQL Server
- **Jwt.Key**: Cambia esto por una cadena larga y aleatoria (mínimo 64 caracteres)
- **Sifen.CertificatePassword**: Pon la contraseña de tu certificado digital
- **Email**: Configura tu servidor de correo (opcional pero recomendado)
- **ApiKeys.ValidKeys**: Genera claves API seguras (puedes usar https://www.uuidgenerator.net/)

### 4. Crear variables de entorno seguras (RECOMENDADO)

Para mayor seguridad, las contraseñas NO deberían estar en el archivo. Usa variables de entorno:

**En Linux:**
```bash
export Sifen__CertificatePassword="tu-password-real"
export ConnectionStrings__DefaultConnection="Server=localhost;Database=SifenApiDb;User Id=sa;Password=TU_PASSWORD_SQL;"
export Jwt__Key="TU_CLAVE_JWT_SUPER_SEGURA"
```

**En Windows:**
```powershell
$env:Sifen__CertificatePassword="tu-password-real"
$env:ConnectionStrings__DefaultConnection="Server=localhost;Database=SifenApiDb;User Id=sa;Password=TU_PASSWORD_SQL;"
$env:Jwt__Key="TU_CLAVE_JWT_SUPER_SEGURA"
```

---

## Despliegue con Docker (RECOMENDADO)

Esta es la forma más fácil y segura de poner el sistema en producción.

### 1. Modificar docker-compose para producción

Crea un archivo `docker-compose.production.yml` en la raíz del proyecto:

```yaml
version: '3.8'

services:
  sifenapi:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "80:80"
      - "443:443"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=http://+:80
    volumes:
      - ./certificados:/app/certificados:ro
      - ./storage:/app/storage
      - ./appsettings.Production.json:/app/appsettings.Production.json:ro
    depends_on:
      - sqlserver
      - redis
    restart: unless-stopped
    networks:
      - sifen-network

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      SA_PASSWORD: "TU_PASSWORD_SEGURO_SQL_123!"
      ACCEPT_EULA: "Y"
      MSSQL_PID: "Express"
    ports:
      - "1433:1433"
    volumes:
      - sqlserver_data:/var/opt/mssql
    restart: unless-stopped
    networks:
      - sifen-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped
    networks:
      - sifen-network

volumes:
  sqlserver_data:
  redis_data:

networks:
  sifen-network:
    driver: bridge
```

### 2. Construir e iniciar los contenedores

```bash
# Volver a la raíz del proyecto
cd /ruta/a/sifen-api-dotnet

# Construir las imágenes
docker-compose -f docker-compose.production.yml build

# Iniciar los servicios
docker-compose -f docker-compose.production.yml up -d

# Ver los logs para verificar que todo está bien
docker-compose -f docker-compose.production.yml logs -f
```

### 3. Crear la base de datos

```bash
# Esperar 30 segundos a que SQL Server inicie completamente
sleep 30

# Aplicar las migraciones de base de datos
docker-compose -f docker-compose.production.yml exec sifenapi \
  dotnet ef database update --project /src/src/SifenApi.Infrastructure --startup-project /src/src/SifenApi.WebApi
```

**Si el comando anterior no funciona**, hazlo manualmente:

```bash
# Entrar al contenedor
docker-compose -f docker-compose.production.yml exec sifenapi bash

# Dentro del contenedor, crear la base de datos
# (Tendrás que hacerlo con scripts SQL si no tienes Entity Framework CLI instalado)
```

### 4. Verificar que está funcionando

```bash
# Ver el estado de los contenedores
docker-compose -f docker-compose.production.yml ps

# Deberías ver algo como:
# NAME                COMMAND             STATUS          PORTS
# sifenapi           dotnet SifenApi...   Up             0.0.0.0:80->80/tcp
# sqlserver          /opt/mssql/bin...    Up             0.0.0.0:1433->1433/tcp
# redis              redis-server         Up             0.0.0.0:6379->6379/tcp

# Probar la API
curl http://localhost/health
# Debería responder: "Healthy" o similar
```

---

## Despliegue manual (Alternativa)

Si no quieres usar Docker, sigue estos pasos:

### 1. Compilar el proyecto

```bash
# Navegar a la raíz del proyecto
cd /ruta/a/sifen-api-dotnet

# Compilar en modo Release
dotnet build -c Release

# Publicar la aplicación
dotnet publish src/SifenApi.WebApi/sifen-webapi.csproj -c Release -o /opt/sifenapi
```

### 2. Configurar SQL Server

```bash
# Conectarte a SQL Server con SSMS o sqlcmd
sqlcmd -S localhost -U sa -P "TU_PASSWORD"

# Crear la base de datos
CREATE DATABASE SifenApiDb;
GO

# Salir
exit
```

### 3. Aplicar migraciones

```bash
cd /ruta/a/sifen-api-dotnet

dotnet ef database update \
  --project src/SifenApi.Infrastructure/sifen-infrastructure.csproj \
  --startup-project src/SifenApi.WebApi/sifen-webapi.csproj \
  --connection "Server=localhost;Database=SifenApiDb;User Id=sa;Password=TU_PASSWORD;"
```

### 4. Ejecutar la aplicación

```bash
cd /opt/sifenapi

# Ejecutar
dotnet SifenApi.WebApi.dll

# O mejor, crear un servicio systemd (Linux):
```

**Crear servicio en Linux:**

```bash
sudo nano /etc/systemd/system/sifenapi.service
```

Contenido:

```ini
[Unit]
Description=SifenApi Service
After=network.target

[Service]
WorkingDirectory=/opt/sifenapi
ExecStart=/usr/bin/dotnet /opt/sifenapi/SifenApi.WebApi.dll
Restart=always
RestartSec=10
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
```

Activar el servicio:

```bash
sudo systemctl enable sifenapi
sudo systemctl start sifenapi
sudo systemctl status sifenapi
```

---

## Verificación del sistema

### 1. Health Check

```bash
curl http://localhost/health
# Debería responder: "Healthy"
```

### 2. Swagger (Documentación de la API)

Abre un navegador y ve a:
```
http://localhost/swagger
```

Deberías ver la documentación interactiva de la API.

### 3. Probar la conexión con SIFEN (Test)

```bash
# Reemplaza TU_API_KEY con la que configuraste
curl -X GET "http://localhost/api/v1/health/sifen" \
  -H "X-API-Key: TU_API_KEY"
```

---

## Configuración de seguridad en producción

### 1. HTTPS (SSL/TLS)

**IMPORTANTE**: En producción DEBES usar HTTPS.

#### Opción A: Usar un certificado SSL

```bash
# Obtener certificado gratuito con Let's Encrypt
sudo apt install certbot
sudo certbot certonly --standalone -d tu-dominio.com

# El certificado estará en:
# /etc/letsencrypt/live/tu-dominio.com/fullchain.pem
# /etc/letsencrypt/live/tu-dominio.com/privkey.pem
```

Actualiza tu `docker-compose.production.yml` para usar HTTPS:

```yaml
services:
  sifenapi:
    environment:
      - ASPNETCORE_URLS=https://+:443;http://+:80
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/app/ssl/cert.pem
      - ASPNETCORE_Kestrel__Certificates__Default__KeyPath=/app/ssl/key.pem
    volumes:
      - /etc/letsencrypt/live/tu-dominio.com/fullchain.pem:/app/ssl/cert.pem:ro
      - /etc/letsencrypt/live/tu-dominio.com/privkey.pem:/app/ssl/key.pem:ro
```

#### Opción B: Usar un proxy reverso (Nginx)

Más recomendado para producción:

```bash
# Instalar Nginx
sudo apt install nginx

# Configurar Nginx
sudo nano /etc/nginx/sites-available/sifenapi
```

Contenido:

```nginx
server {
    listen 80;
    server_name tu-dominio.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name tu-dominio.com;

    ssl_certificate /etc/letsencrypt/live/tu-dominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tu-dominio.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Activar:

```bash
sudo ln -s /etc/nginx/sites-available/sifenapi /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 2. Firewall

```bash
# Linux (ufw)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22/tcp  # SSH
sudo ufw enable

# Windows Firewall
# Abrir Panel de Control > Firewall de Windows > Configuración avanzada
# Agregar reglas de entrada para puertos 80 y 443
```

### 3. Respaldo de la base de datos

Configura respaldos automáticos:

```bash
# Crear script de backup
sudo nano /usr/local/bin/backup-sifenapi.sh
```

Contenido:

```bash
#!/bin/bash
BACKUP_DIR="/backups/sifenapi"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup de SQL Server
docker exec sqlserver /opt/mssql-tools/bin/sqlcmd \
  -S localhost -U sa -P "TU_PASSWORD" \
  -Q "BACKUP DATABASE SifenApiDb TO DISK = '/var/opt/mssql/backup/sifen_$DATE.bak'"

# Copiar el backup fuera del contenedor
docker cp sqlserver:/var/opt/mssql/backup/sifen_$DATE.bak $BACKUP_DIR/

# Eliminar backups antiguos (más de 7 días)
find $BACKUP_DIR -name "sifen_*.bak" -mtime +7 -delete
```

Hacerlo ejecutable y programar:

```bash
sudo chmod +x /usr/local/bin/backup-sifenapi.sh

# Agregar a crontab (ejecutar diariamente a las 2 AM)
sudo crontab -e
# Agregar línea:
0 2 * * * /usr/local/bin/backup-sifenapi.sh
```

---

## Monitoreo y mantenimiento

### 1. Ver logs en tiempo real

```bash
# Con Docker
docker-compose -f docker-compose.production.yml logs -f sifenapi

# Sin Docker (systemd)
sudo journalctl -u sifenapi -f
```

### 2. Monitorear recursos

```bash
# CPU, RAM, Disco
docker stats

# O sin Docker
htop  # (instalar con: sudo apt install htop)
```

### 3. Endpoints de monitoreo

La API tiene endpoints para verificar el estado:

- `GET /health` - Estado general de la API
- `GET /health/ready` - ¿Está lista para recibir tráfico?
- `GET /health/live` - ¿Está viva la aplicación?

Puedes configurar un servicio de monitoreo como UptimeRobot, Pingdom, o Zabbix para que verifique estos endpoints cada minuto.

### 4. Reiniciar servicios

```bash
# Con Docker
docker-compose -f docker-compose.production.yml restart

# Solo la API
docker-compose -f docker-compose.production.yml restart sifenapi

# Sin Docker (systemd)
sudo systemctl restart sifenapi
```

---

## Solución de problemas comunes

### Problema 1: "No se puede conectar a la base de datos"

**Síntomas**: Error 500, logs muestran "Cannot connect to SQL Server"

**Solución**:

```bash
# Verificar que SQL Server esté corriendo
docker-compose -f docker-compose.production.yml ps sqlserver

# Si no está corriendo, iniciarlo
docker-compose -f docker-compose.production.yml up -d sqlserver

# Verificar la contraseña en appsettings.Production.json
```

### Problema 2: "Error al conectar con SIFEN"

**Síntomas**: Error al enviar facturas, timeout

**Solución**:

```bash
# Verificar conectividad
ping sifen.set.gov.py

# Verificar que el certificado sea válido
openssl pkcs12 -info -in certificados/certificado.pfx

# Verificar logs de la API
docker-compose -f docker-compose.production.yml logs sifenapi | grep -i sifen
```

### Problema 3: "Certificado expirado o inválido"

**Síntomas**: Error 401, "Certificate validation failed"

**Solución**:

1. Contactar a la SET para renovar el certificado
2. Reemplazar el archivo en `certificados/certificado.pfx`
3. Reiniciar la API

### Problema 4: "API responde lento"

**Síntomas**: Timeouts, respuestas lentas

**Solución**:

```bash
# Verificar recursos
docker stats

# Incrementar recursos si es necesario
# Editar docker-compose.production.yml y agregar:
services:
  sifenapi:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G

# Verificar logs de SQL Server
docker-compose -f docker-compose.production.yml logs sqlserver

# Limpiar cache de Redis
docker-compose -f docker-compose.production.yml exec redis redis-cli FLUSHALL
```

### Problema 5: "Disco lleno"

**Síntomas**: Error al escribir, aplicación se cae

**Solución**:

```bash
# Verificar espacio
df -h

# Limpiar logs antiguos
docker system prune -a --volumes

# Limpiar archivos temporales
rm -rf /app/storage/temp/*

# Implementar rotación de logs
```

---

## Checklist de producción

Antes de declarar el sistema "listo para producción", verifica:

- [ ] Certificado digital SIFEN instalado y válido
- [ ] Base de datos creada y migraciones aplicadas
- [ ] Contraseñas fuertes configuradas (SQL, JWT, API Keys)
- [ ] HTTPS configurado con certificado SSL válido
- [ ] Firewall configurado correctamente
- [ ] Backup automático configurado
- [ ] Monitoreo configurado (health checks)
- [ ] Logs siendo almacenados y rotados
- [ ] Documentación de la API accesible (Swagger)
- [ ] Probado en ambiente de test primero
- [ ] Variables de entorno configuradas (no contraseñas en archivos)
- [ ] Proxy reverso configurado (Nginx/Apache)
- [ ] Dominio apuntando al servidor
- [ ] Email SMTP configurado para notificaciones
- [ ] Contacto técnico de la SET disponible

---

## Próximos pasos

Una vez que tengas el sistema funcionando:

1. **Prueba en el ambiente de test de SIFEN** antes de ir a producción
2. **Integra tu sistema de ventas** usando la API
3. **Configura alertas** para errores críticos
4. **Documenta** cualquier configuración específica de tu empresa
5. **Capacita** a tu equipo en el uso de la API

---

## Soporte

Si tienes problemas:

1. Revisa los logs: `docker-compose logs -f`
2. Consulta la documentación de SIFEN: https://www.set.gov.py/
3. Revisa la documentación técnica en la carpeta `docs/`
4. Abre un issue en el repositorio de GitHub

---

## Glosario para principiantes

- **API**: Interfaz que permite que tu sistema hable con el sistema SIFEN
- **Docker**: Software que empaqueta la aplicación con todo lo necesario para funcionar
- **SQL Server**: Base de datos donde se guardan las facturas
- **Redis**: Sistema de cache para hacer la API más rápida
- **HTTPS**: Protocolo seguro de internet (el candado en el navegador)
- **Certificado digital**: Archivo que identifica a tu empresa ante SIFEN
- **RUC**: Registro Único de Contribuyentes (identificador de tu empresa)
- **CDC**: Código de Control del documento electrónico
- **Endpoint**: Una URL específica de la API (por ejemplo: `/api/v1/facturas`)
- **JSON**: Formato de texto para intercambiar datos
- **Backup**: Copia de seguridad de los datos

---

**Última actualización**: Diciembre 2025
**Versión**: 1.0
