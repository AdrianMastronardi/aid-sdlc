# AWS Logging guidelines

## Propósito

Proporcionar a los **desarrolladores** y **copilotos/asistentes de código** una estructura clara 
para implementar y mantener la observabilidad de sus sistemas, con enfoque en logging, métricas,
alertas y dashboards. Estas prácticas aseguran que los sistemas sean monitoreables, diagnosticables 
y que proporcionen información valiosa para la toma de decisiones operativas y de negocio.

## Logging en AWS

### Niveles de Logging

Utilizamos los niveles estándar de la librería `logging` de Python, con el siguiente propósito:

- `DEBUG`
  - Información detallada y verbosa, útil únicamente durante la fase de depuración
  - Solo activado en entornos de desarrollo o temporalmente en producción
  - Ejemplos: valores de variables, cuerpos de peticiones/respuestas, flujo de ejecución detallado

- `INFO`
  - Confirma que la aplicación se ejecuta según lo esperado
  - El "pulso" normal de la aplicación en producción
  - Ejemplos: inicio de procesos, finalización exitosa, creación de recursos

- `WARNING`
  - Indica situaciones inesperadas que no afectan el flujo principal
  - No requiere acción inmediata pero merece revisión
  - Ejemplos: conexiones degradadas, uso de APIs obsoletas, fallos en operaciones no críticas

- `ERROR`
  - Problemas serios que impiden completar una operación
  - Siempre debe incluir el traceback completo para facilitar diagnóstico
  - Debe ser accionable y generar alertas
  - Ejemplos: excepciones no controladas, fallos de conexión a bases de datos

### Configuración del Nivel de Logging

#### Nivel Base por Entorno

- **Producción (prod):** `INFO` (por defecto)
- **Desarrollo (dev/uat):** `DEBUG`

Configuración en `serverless.yml`:

```yaml
provider:
  name: aws
  environment:
    LOG_LEVEL: "${sls:stage, 'dev' == 'dev' ? 'DEBUG' : 'INFO'}"
```

#### Cambio Dinámico del Nivel (Modo DEBUG Temporal)

Para depurar problemas en producción sin requerir un nuevo despliegue, utilizamos la API 
centralizada para gestión de niveles de logging:

**Endpoint:** `POST https://apiv2.habi.co/v1/logging/level`

```json
{
  "function_name": "mi-servicio-prod-miFuncion",
  "target_level": "DEBUG",
  "duration_minutes": 30
}
```

El sistema revierte automáticamente el nivel a `INFO` al expirar el tiempo, evitando costos 
innecesarios y problemas de seguridad.

### Buenas Prácticas de Implementación

#### Logs en Formato JSON

**Todos los logs deben emitirse en formato JSON estructurado** para facilitar consultas y análisis.
Recomendamos usar `python-json-logger`:

```python
# utils/logger.py
import logging
import os
import sys
from pythonjsonlogger import jsonlogger

def get_logger(name: str) -> logging.Logger:
    log_level = os.environ.get('LOG_LEVEL', 'INFO').upper()
    logger = logging.getLogger(name)
    
    if logger.hasHandlers():
        return logger
        
    logger.setLevel(log_level)
    handler = logging.StreamHandler(sys.stdout)

    # Formato que añade campos clave al log JSON
    formatter = jsonlogger.JsonFormatter(
        '%(asctime)s %(name)s %(levelname)s %(message)s',
        rename_fields={'levelname': 'level'}
    )
    
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    logger.propagate = False # Evita duplicación de logs
    
    return logger
```

#### Loguear Excepciones Correctamente

```python
from utils.logger import get_logger
logger = get_logger(__name__)

try:
    result = 10 / 0
except Exception:
    # Incluye el traceback completo en el log JSON
    logger.exception("Ocurrió un error al procesar la solicitud")
```

#### No Loguear Información Sensible

**NUNCA** incluir en logs:
- Contraseñas o tokens
- Datos personales (PII)
- Información financiera
- Secretos de API

#### Añadir Contexto Relevante

Enriquecer logs con identificadores para correlacionar eventos:

```python
logger.info(
    "Procesando orden",
    extra={"user_id": "user-123", "order_id": "order-abc"}
)
```

### Conservación y Consulta de Logs

#### Ciclo de Vida de los Logs

- **CloudWatch:** 5 días (por defecto)
- **S3 (archivado):** 1 año (requiere configuración adicional)

#### Habilitación del Archivado en S3

Para conservar logs más allá de 5 días, añadir en `serverless.yml`:

```yaml
provider:
  # ... resto de la configuración
  stackTags:
    ENABLE_LOG_EXPORT: "true"
```

#### Consulta de Logs Archivados con Amazon Athena

Para consultar logs históricos archivados en S3:

1. Acceder a Amazon Athena en la consola AWS
2. Configurar:
   - **Data source:** `AwsDataCatalog`
   - **Database:** `logs_cloudwatch_s3`
   - **Tabla:** `logs_us_east_2`

3. Ejecutar consultas SQL filtrando **siempre** por particiones:

```sql
-- Consulta básica con particiones (obligatorio)
SELECT
  from_unixtime(timestamp / 1000) AS event_time,
  message
FROM
  logs_us_east_2
WHERE
  service_name = 'lambda'
  AND resource_name = 'mi-funcion-lambda-prod'
  AND year = '2025'
  AND month = '06'
  AND day = '18'
  AND hour = '14'
ORDER BY
  timestamp DESC
LIMIT 10;
```

## Buenas Prácticas Generales

1. **Consistencia:** Usar el mismo formato y estructura de logs en todos los servicios
2. **Correlación:** Incluir IDs de traza para seguir peticiones a través de múltiples servicios
3. **Contextualización:** Agregar metadatos relevantes a cada log (IDs, timestamps, entorno)
4. **Economía:** Evitar logging excesivo, especialmente en bucles o código de alta frecuencia
5. **Seguridad:** Nunca loguear información sensible o credenciales
