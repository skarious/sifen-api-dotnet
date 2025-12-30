# 🚀 Inicio Rápido - SifenAPI en Windows

## 3 Pasos para levantar el sistema en local

### Paso 1: Preparar certificado

```powershell
# Crear carpeta y copiar tu certificado SIFEN
mkdir certificados
copy "C:\ruta\a\tu\certificado.pfx" "certificados\certificado-test.pfx"
```

### Paso 2: Configurar contraseña del certificado

Edita el archivo: `src\SifenApi.WebApi\appsettings.Test.json`

Busca esta línea y cambia el password:
```json
"CertificatePassword": "TU_PASSWORD_REAL_AQUI"
```

### Paso 3: Levantar todo con Docker

```powershell
# Construir e iniciar
docker-compose -f docker-compose.local.yml up -d

# Ver logs
docker-compose -f docker-compose.local.yml logs -f
```

## ✅ Verificar que funciona

Abre tu navegador en: **http://localhost:8080/swagger**

Deberías ver la documentación de la API.

## 📚 Guías completas

- **[GUIA-INICIO-RAPIDO-WINDOWS.md](GUIA-INICIO-RAPIDO-WINDOWS.md)** - Guía completa paso a paso con ejemplos
- **[GUIA-PRODUCCION.md](GUIA-PRODUCCION.md)** - Para poner en producción
- **[SifenAPI-Postman-Collection.json](SifenAPI-Postman-Collection.json)** - Importar en Postman para probar

## 🔑 API Key de prueba

Para las peticiones usa:
```
X-API-Key: test-key-12345
```

## 📝 Ejemplo rápido: Crear una factura

**Con PowerShell:**

```powershell
$factura = @{
  tipoDocumento = 1
  ambiente = 2
  establecimiento = "001"
  puntoExpedicion = "001"
  numero = "0000001"
  emisor = @{
    ruc = "80012345-6"
    razonSocial = "MI EMPRESA S.A."
  }
  receptor = @{
    numeroDocumento = "80087654-3"
    razonSocial = "CLIENTE S.A."
  }
  items = @(
    @{
      codigo = "PROD001"
      descripcion = "Notebook HP"
      cantidad = 1
      precioUnitario = 5000000
      montoTotal = 5000000
    }
  )
  totales = @{
    total = 5000000
  }
} | ConvertTo-Json -Depth 10

Invoke-RestMethod -Uri "http://localhost:8080/api/v1/facturas" `
  -Method POST `
  -Headers @{"X-API-Key"="test-key-12345"; "Content-Type"="application/json"} `
  -Body $factura
```

**Con Postman:**

1. Importar: `SifenAPI-Postman-Collection.json`
2. Ejecutar: "Crear Factura - Ejemplo Básico"

## ⚠️ Importante

- **Ambiente = 2**: Siempre usa ambiente de TEST para pruebas
- **Certificado válido**: Debe ser emitido por SIFEN
- **Primeras pruebas**: Verifica en el portal de SIFEN que los documentos se reciban

## 🛑 Detener los servicios

```powershell
docker-compose -f docker-compose.local.yml down
```

## 🔄 Reiniciar después de cambios

```powershell
docker-compose -f docker-compose.local.yml restart sifenapi
```

## 🆘 Problemas comunes

### Docker Desktop no arranca
- Reinicia tu PC
- Asegúrate de tener WSL 2 instalado

### Puerto 8080 ocupado
Edita `docker-compose.local.yml` y cambia:
```yaml
ports:
  - "8081:80"  # Cambiar 8080 a 8081
```

### Error de certificado
- Verifica que el archivo exista: `dir certificados`
- Verifica la contraseña en `appsettings.Test.json`

## 📞 Siguiente paso

Lee la guía completa: **[GUIA-INICIO-RAPIDO-WINDOWS.md](GUIA-INICIO-RAPIDO-WINDOWS.md)**
