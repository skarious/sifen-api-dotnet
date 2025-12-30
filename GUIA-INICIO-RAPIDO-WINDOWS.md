# Guía de Inicio Rápido - Windows + Docker Desktop

Esta guía te ayudará a levantar el sistema en tu PC Windows con Docker Desktop para hacer pruebas locales.

## 📋 Tabla de Contenidos

1. [Preparación del entorno local](#preparación-del-entorno-local)
2. [Configuración para testing](#configuración-para-testing)
3. [Levantar el sistema](#levantar-el-sistema)
4. [Flujo completo de facturación electrónica](#flujo-completo-de-facturación-electrónica)
5. [Ejemplos prácticos paso a paso](#ejemplos-prácticos-paso-a-paso)
6. [Integración con Nginx Proxy Manager](#integración-con-nginx-proxy-manager)
7. [Troubleshooting](#troubleshooting)

---

## Preparación del entorno local

### 1. Instalar Docker Desktop

```powershell
# 1. Descarga Docker Desktop para Windows:
# https://www.docker.com/products/docker-desktop

# 2. Instala y reinicia tu PC

# 3. Abre PowerShell y verifica:
docker --version
docker-compose --version

# Deberías ver algo como:
# Docker version 24.0.x
# Docker Compose version v2.x.x
```

### 2. Clonar o descargar el proyecto

```powershell
# Opción A: Si tienes Git
cd C:\Users\TuUsuario\Documents
git clone https://github.com/turepositorio/sifen-api-dotnet.git
cd sifen-api-dotnet

# Opción B: Si descargaste el ZIP
# Descomprime en: C:\Users\TuUsuario\Documents\sifen-api-dotnet
# Abre PowerShell en esa carpeta
```

### 3. Preparar el certificado SIFEN

```powershell
# Crear carpeta para certificados
mkdir certificados

# Copiar tu certificado (ajusta la ruta según donde lo tengas)
copy "C:\Users\TuUsuario\Downloads\certificado-sifen.pfx" "certificados\certificado-test.pfx"

# Verificar que está ahí
dir certificados
```

---

## Configuración para testing

### 1. Crear archivo de configuración para testing local

Crea el archivo: `src\SifenApi.WebApi\appsettings.Test.json`

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=sqlserver;Database=SifenApiDb;User Id=sa;Password=SifenTest123!;TrustServerCertificate=true;",
    "Redis": "redis:6379"
  },
  "Jwt": {
    "Key": "EstoEsUnaClaveDePruebaLocal123456789012345678901234567890",
    "Issuer": "SifenApiTest",
    "Audience": "SifenApiTestUsers",
    "DurationInMinutes": 120
  },
  "Sifen": {
    "UrlTest": "https://sifen-test.set.gov.py/de/ws/sync",
    "UrlProd": "https://sifen.set.gov.py/de/ws/sync",
    "UrlConsultaPublica": "https://ekuatia.set.gov.py/consultas",
    "CertificatePath": "/app/certificados/certificado-test.pfx",
    "CertificatePassword": "TU_PASSWORD_DEL_CERTIFICADO_AQUI"
  },
  "Storage": {
    "BasePath": "/app/storage"
  },
  "Email": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": 587,
    "SmtpUser": "test@test.com",
    "SmtpPassword": "test",
    "FromEmail": "noreply@test.com",
    "FromName": "Sistema TEST"
  },
  "Cors": {
    "AllowedOrigins": ["http://localhost:3000", "http://localhost:5173", "http://localhost:8080"]
  },
  "ApiKeys": {
    "ValidKeys": ["test-key-12345"]
  },
  "AllowedHosts": "*"
}
```

**IMPORTANTE**: En `CertificatePassword`, pon la contraseña REAL de tu certificado SIFEN.

### 2. Crear docker-compose para testing local

Crea el archivo: `docker-compose.local.yml`

```yaml
version: '3.8'

services:
  sifenapi:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:80"  # Accesible en http://localhost:8080
    environment:
      - ASPNETCORE_ENVIRONMENT=Test
      - ASPNETCORE_URLS=http://+:80
    volumes:
      - ./certificados:/app/certificados:ro
      - ./storage:/app/storage
      - ./src/SifenApi.WebApi/appsettings.Test.json:/app/appsettings.Test.json:ro
    depends_on:
      - sqlserver
      - redis
    networks:
      - sifen-network

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      SA_PASSWORD: "SifenTest123!"
      ACCEPT_EULA: "Y"
      MSSQL_PID: "Express"
    ports:
      - "1433:1433"
    volumes:
      - sqlserver_data:/var/opt/mssql
    networks:
      - sifen-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    networks:
      - sifen-network

volumes:
  sqlserver_data:

networks:
  sifen-network:
    driver: bridge
```

---

## Levantar el sistema

### 1. Construir e iniciar los contenedores

```powershell
# Asegúrate de estar en la carpeta del proyecto
cd C:\Users\TuUsuario\Documents\sifen-api-dotnet

# Construir las imágenes (primera vez tarda 5-10 minutos)
docker-compose -f docker-compose.local.yml build

# Iniciar todos los servicios
docker-compose -f docker-compose.local.yml up -d

# Ver los logs para verificar que todo arrancó bien
docker-compose -f docker-compose.local.yml logs -f
```

**Presiona Ctrl+C para salir de los logs cuando veas que todo está corriendo.**

### 2. Verificar que todo esté corriendo

```powershell
# Ver el estado de los contenedores
docker-compose -f docker-compose.local.yml ps

# Deberías ver algo como:
# NAME                     STATUS
# sifen-api-dotnet-sifenapi-1    Up
# sifen-api-dotnet-sqlserver-1   Up
# sifen-api-dotnet-redis-1       Up
```

### 3. Probar la API

Abre tu navegador y ve a:

```
http://localhost:8080/swagger
```

Deberías ver la documentación interactiva de la API (Swagger UI).

O prueba con PowerShell:

```powershell
# Test básico de health
curl http://localhost:8080/health

# Debería responder: "Healthy"
```

---

## Flujo completo de facturación electrónica

### Diagrama del flujo

```
┌─────────────────────────────────────────────────────────────────┐
│                    FLUJO DE FACTURACIÓN SIFEN                   │
└─────────────────────────────────────────────────────────────────┘

1. TU SISTEMA                    2. SIFEN API                3. SIFEN (SET)
   │                                  │                          │
   │  Crear Factura                   │                          │
   │──────────────────────────────────>│                          │
   │  POST /api/v1/facturas           │                          │
   │  {datos de la factura}           │                          │
   │                                  │                          │
   │                                  │  Generar XML             │
   │                                  │  (según formato SIFEN)   │
   │                                  │                          │
   │                                  │  Firmar XML              │
   │                                  │  (con certificado)       │
   │                                  │                          │
   │                                  │  Enviar a SIFEN          │
   │                                  │─────────────────────────>│
   │                                  │                          │
   │                                  │  Validar documento       │
   │                                  │  Asignar CDC             │
   │                                  │                          │
   │                                  │<─────────────────────────│
   │                                  │  Respuesta con CDC       │
   │                                  │                          │
   │  Generar PDF (KUDE)              │                          │
   │  Generar QR                      │                          │
   │                                  │                          │
   │<──────────────────────────────────│                          │
   │  Respuesta:                      │                          │
   │  - CDC                           │                          │
   │  - XML firmado                   │                          │
   │  - PDF                           │                          │
   │  - QR Code                       │                          │
   │                                  │                          │
   │  Entregar a cliente              │                          │
   │  (PDF + QR)                      │                          │
```

### Conceptos clave:

- **CDC (Código de Control)**: Identificador único de 44 dígitos que SIFEN asigna a cada documento
- **XML**: Formato en que se envía el documento a SIFEN
- **Firma Digital**: El XML debe estar firmado con tu certificado digital
- **KUDE**: Representación impresa del documento electrónico (el PDF)
- **QR**: Código para verificar la autenticidad del documento

---

## Ejemplos prácticos paso a paso

### Herramientas necesarias

Vamos a usar **Postman** o **curl** (PowerShell). Te recomiendo Postman porque es más visual.

**Descargar Postman**: https://www.postman.com/downloads/

### Paso 1: Autenticación (Opcional según configuración)

Si la API requiere autenticación, primero necesitas obtener un token:

```powershell
# Con PowerShell (curl)
curl -X POST http://localhost:8080/api/v1/auth/login `
  -H "Content-Type: application/json" `
  -d '{
    "username": "admin",
    "password": "Admin123!"
  }'

# Respuesta (ejemplo):
# {
#   "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
#   "expiration": "2024-01-15T10:30:00Z"
# }
```

O simplemente usa la API Key que configuraste en `appsettings.Test.json`:

```
X-API-Key: test-key-12345
```

### Paso 2: Crear una factura electrónica

Este es el paso principal. Aquí envías los datos de la factura y la API hace todo el proceso.

**Ejemplo completo de factura:**

```json
{
  "tipoDocumento": 1,
  "tipoEmision": 1,
  "tipoContribuyente": 1,
  "ambiente": 2,
  "establecimiento": "001",
  "puntoExpedicion": "001",
  "numero": "0000001",
  "fecha": "2024-01-15T10:30:00",
  "tipoTransaccion": 1,
  "tipoImpuesto": 1,
  "moneda": "PYG",

  "emisor": {
    "ruc": "80012345-6",
    "razonSocial": "MI EMPRESA S.A.",
    "nombreFantasia": "Mi Empresa",
    "direccion": "Av. Mariscal López 123",
    "ciudad": "Asunción",
    "telefono": "021-123456",
    "email": "facturacion@miempresa.com.py"
  },

  "receptor": {
    "tipoDocumento": 1,
    "numeroDocumento": "80087654-3",
    "razonSocial": "CLIENTE S.A.",
    "direccion": "Calle Palma 456",
    "ciudad": "Asunción",
    "telefono": "021-654321",
    "email": "cliente@cliente.com.py"
  },

  "items": [
    {
      "codigo": "PROD001",
      "descripcion": "Notebook HP 15-dy1036nr",
      "cantidad": 2,
      "unidadMedida": "UNI",
      "precioUnitario": 5000000,
      "montoTotal": 10000000,
      "afectacionTributaria": 1,
      "porcentajeIva": 10,
      "montoIva": 909091
    },
    {
      "codigo": "SERV001",
      "descripcion": "Servicio de instalación",
      "cantidad": 1,
      "unidadMedida": "UNI",
      "precioUnitario": 500000,
      "montoTotal": 500000,
      "afectacionTributaria": 1,
      "porcentajeIva": 10,
      "montoIva": 45455
    }
  ],

  "totales": {
    "subTotal": 10500000,
    "totalIva10": 954546,
    "totalIva5": 0,
    "totalExenta": 0,
    "total": 10500000
  },

  "condicionVenta": {
    "tipo": 1,
    "plazo": "Contado"
  }
}
```

**Hacer la petición:**

**Con Postman:**

1. Abre Postman
2. Crea nueva petición → POST
3. URL: `http://localhost:8080/api/v1/facturas`
4. Headers:
   - `Content-Type`: `application/json`
   - `X-API-Key`: `test-key-12345`
5. Body → Raw → JSON → Pega el JSON de arriba
6. Click en "Send"

**Con PowerShell:**

Guarda el JSON en un archivo `factura.json` y ejecuta:

```powershell
curl -X POST http://localhost:8080/api/v1/facturas `
  -H "Content-Type: application/json" `
  -H "X-API-Key: test-key-12345" `
  --data "@factura.json"
```

**Respuesta esperada:**

```json
{
  "success": true,
  "cdc": "01800123456001001000000120240115102345678901234567",
  "mensaje": "Documento generado y enviado exitosamente",
  "xmlUrl": "http://localhost:8080/api/v1/documentos/xml/01800123456...",
  "pdfUrl": "http://localhost:8080/api/v1/documentos/pdf/01800123456...",
  "qrCode": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA...",
  "estado": "Aprobado",
  "fechaAprobacion": "2024-01-15T10:30:45Z"
}
```

### Paso 3: Descargar el PDF (KUDE)

Una vez creada la factura, puedes descargar el PDF:

```powershell
# Opción 1: Abrir en el navegador
start http://localhost:8080/api/v1/documentos/pdf/01800123456001001000000120240115102345678901234567

# Opción 2: Descargar con curl
curl -o factura.pdf http://localhost:8080/api/v1/documentos/pdf/01800123456001001000000120240115102345678901234567
```

### Paso 4: Consultar el estado de un documento

Puedes consultar si SIFEN ya procesó el documento:

```powershell
# Consultar por CDC
curl -X GET http://localhost:8080/api/v1/consultas/cdc/01800123456001001000000120240115102345678901234567 `
  -H "X-API-Key: test-key-12345"
```

**Respuesta:**

```json
{
  "cdc": "01800123456001001000000120240115102345678901234567",
  "estado": "Aprobado",
  "fechaAprobacion": "2024-01-15T10:30:45Z",
  "numeroControl": "ABC123456",
  "observacion": null
}
```

### Paso 5: Cancelar un documento (si es necesario)

Si necesitas anular una factura:

```powershell
curl -X POST http://localhost:8080/api/v1/eventos/cancelacion `
  -H "Content-Type: application/json" `
  -H "X-API-Key: test-key-12345" `
  -d '{
    "cdc": "01800123456001001000000120240115102345678901234567",
    "motivo": "Error en el monto facturado"
  }'
```

---

## Campos importantes del JSON de factura

### Tipos de Documento (tipoDocumento)
- `1`: Factura electrónica
- `4`: Autofactura electrónica
- `5`: Nota de crédito electrónica
- `6`: Nota de débito electrónica
- `7`: Nota de remisión electrónica

### Ambiente (ambiente)
- `1`: Producción (REAL - cuidado!)
- `2`: Test (para pruebas)

**IMPORTANTE**: Siempre usa `"ambiente": 2` para testing.

### Tipo de Transacción (tipoTransaccion)
- `1`: Venta de mercadería
- `2`: Prestación de servicios
- `3`: Mixto (venta + servicios)
- `4`: Consignación
- etc.

### Condición de Venta
- `tipo: 1`: Contado
- `tipo: 2`: Crédito

### Moneda
- `"PYG"`: Guaraníes
- `"USD"`: Dólares
- `"EUR"`: Euros
- etc.

### IVA (porcentajeIva)
- `10`: IVA 10%
- `5`: IVA 5%
- `0`: Exento

---

## Integración con Nginx Proxy Manager

Una vez que tengas todo funcionando en local, para conectarlo con tu Nginx Proxy Manager:

### 1. En tu servidor con Nginx Proxy Manager

**Agregar Proxy Host:**

1. Abre Nginx Proxy Manager (http://tu-servidor:81)
2. Proxy Hosts → Add Proxy Host
3. Configuración:
   - **Domain Names**: `sifen-api.tu-dominio.com`
   - **Scheme**: `http`
   - **Forward Hostname/IP**: `IP_DE_TU_PC_WINDOWS` (ej: 192.168.1.100)
   - **Forward Port**: `8080`
   - **Cache Assets**: ✅ (activado)
   - **Block Common Exploits**: ✅ (activado)
   - **Websockets Support**: ✅ (activado)

4. SSL:
   - Request a new SSL Certificate
   - Let's Encrypt
   - Email: tu-email@dominio.com
   - ✅ Force SSL
   - ✅ HTTP/2 Support

5. Save

### 2. Configurar el firewall de Windows

Para que Nginx Proxy Manager pueda acceder a tu PC:

```powershell
# Abrir puerto 8080 en el Firewall de Windows
New-NetFirewallRule -DisplayName "SifenAPI Local" -Direction Inbound -LocalPort 8080 -Protocol TCP -Action Allow
```

### 3. Probar desde internet

```powershell
# Desde cualquier lugar
curl https://sifen-api.tu-dominio.com/health
```

---

## Troubleshooting

### Problema: "Cannot connect to Docker daemon"

**Solución:**
```powershell
# Asegúrate de que Docker Desktop está corriendo
# Busca el ícono de Docker en la bandeja del sistema (abajo a la derecha)
# Si no está, abre Docker Desktop desde el menú inicio
```

### Problema: "Port 8080 is already in use"

**Solución:**
```powershell
# Cambiar el puerto en docker-compose.local.yml
# Línea: - "8080:80"
# Cambiar a: - "8081:80" (o cualquier puerto libre)

# Ver qué está usando el puerto 8080
netstat -ano | findstr :8080
```

### Problema: "SQL Server container keeps restarting"

**Solución:**
```powershell
# Ver los logs
docker-compose -f docker-compose.local.yml logs sqlserver

# Común: Contraseña débil
# Cambia "SifenTest123!" a algo más fuerte como "SifenT3st!2024$Secure"
# en docker-compose.local.yml y appsettings.Test.json
```

### Problema: "Error al conectar con SIFEN"

**Verificaciones:**

```powershell
# 1. Verificar conectividad a SIFEN
ping sifen-test.set.gov.py

# 2. Verificar certificado
# Asegúrate de que el archivo existe:
dir certificados\certificado-test.pfx

# 3. Verificar que la contraseña sea correcta en appsettings.Test.json

# 4. Ver logs detallados
docker-compose -f docker-compose.local.yml logs sifenapi | Select-String -Pattern "sifen"
```

### Problema: "Swagger no carga"

**Solución:**
```powershell
# Espera 30-60 segundos después de iniciar los contenedores
# La primera vez, .NET necesita compilar y cargar todo

# Verifica que el contenedor esté healthy:
docker-compose -f docker-compose.local.yml ps

# Revisa los logs:
docker-compose -f docker-compose.local.yml logs sifenapi
```

---

## Comandos útiles para el día a día

```powershell
# Iniciar los servicios
docker-compose -f docker-compose.local.yml up -d

# Detener los servicios
docker-compose -f docker-compose.local.yml down

# Ver logs en tiempo real
docker-compose -f docker-compose.local.yml logs -f

# Ver solo logs de la API
docker-compose -f docker-compose.local.yml logs -f sifenapi

# Reiniciar solo la API (después de cambios)
docker-compose -f docker-compose.local.yml restart sifenapi

# Ver recursos que están usando
docker stats

# Limpiar todo (cuidado: borra la base de datos)
docker-compose -f docker-compose.local.yml down -v

# Reconstruir después de cambios en el código
docker-compose -f docker-compose.local.yml build
docker-compose -f docker-compose.local.yml up -d
```

---

## Próximos pasos

1. ✅ Levantar el sistema en local
2. ✅ Probar con facturas de prueba (ambiente = 2)
3. ✅ Verificar que SIFEN acepta los documentos
4. ✅ Revisar los PDFs generados
5. ⬜ Integrar con tu sistema de ventas
6. ⬜ Pasar a producción (ambiente = 1) cuando estés seguro

---

## Colección de Postman

Puedes importar esta colección en Postman para tener ejemplos listos:

**Archivo**: `postman-collection.json` (crear este archivo con los endpoints listos)

---

**¿Necesitas ayuda?** Revisa los logs:
```powershell
docker-compose -f docker-compose.local.yml logs -f
```

**Última actualización**: Diciembre 2025
