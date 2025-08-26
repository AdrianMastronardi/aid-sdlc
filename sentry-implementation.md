# Sentry implementation guidelines

## Propósito

Proporcionar a los **desarrolladores** y **copilotos/asistentes de código** una estructura clara para
la implementación de Sentry como herramienta de monitoreo de errores y rendimiento en aplicaciones 
backend y frontend. Esta guía asegura una configuración consistente y efectiva para detectar, rastrear 
y solucionar problemas en producción.

## Configuración previa

Antes de implementar Sentry en cualquier aplicación, es necesario realizar la configuración de 
equipos y proyectos en el entorno de Sentry correspondiente a cada infraestructura.

### Creación de equipos

Un equipo corresponde a los grupos de trabajo de cada infraestructura:

1. Acceder a la cuenta de Sentry de la infraestructura
2. Seleccionar "Settings" y luego "Teams" 
3. Hacer clic en "Create Team" e ingresar el nombre siguiendo la nomenclatura

#### Nomenclatura para equipos

- Usar solo letras minúsculas (a-z)
- Para separación entre palabras, usar un guion medio "-" (una sola vez)
- Ejemplo: `my-team`

### Creación de proyectos

Un proyecto corresponde a una aplicación específica (frontend o backend) de un equipo:

1. Acceder a la cuenta de Sentry de la infraestructura
2. Seleccionar "Projects" y hacer clic en "Create Project"
3. Elegir la plataforma, ingresar el nombre y asociarlo al equipo correspondiente

#### Nomenclatura para proyectos

- Usar solo letras minúsculas (a-z)
- Para separación entre palabras, usar un guion medio "-" (una sola vez)
- Puede usar el nombre del repositorio como nombre de proyecto
- Ejemplo: `project-name`

### DSN (Data Source Name)

El DSN es la dirección de conexión para que Sentry capture los eventos en el proyecto:

1. Ingresar a la cuenta y dirigirse a "Projects"
2. Elegir el proyecto y hacer clic en el ícono de "Settings"
3. Ir a "Client Keys (DSN)" para obtener el DSN necesario para la implementación

### Entornos

Es necesario definir el entorno de ejecución para distinguir eventos generados en diferentes contextos:

- **dev**: Para entornos de desarrollo y pruebas
- **prod**: Para entornos de producción

## Implementación en Backend (Python)

### Requerimientos

Agregar la siguiente dependencia al archivo `requirements.txt`:

```text
sentry_sdk==1.18.0
```

> **Importante**: Esta implementación es compatible con Python 3.9 o superior.

### Configuración básica en Python

Para implementar Sentry en una aplicación Python estándar:

```python
import sentry_sdk

sentry_sdk.init(
    dsn="https://examplePublicKey@o0.ingest.sentry.io/0",
    environment="prod",  # o "dev" según corresponda
    
    # Establece el porcentaje de transacciones para monitoreo de rendimiento
    traces_sample_rate=0.2,  # 20% de las transacciones
)
```

### Implementación en Globack

Para usar Sentry con el framework Globack, se dispone de varias utilidades:

#### habi_middleware

Por defecto, Sentry se activa al definir `habi_middleware`, sin configuración adicional:

```python
from globack_utils.globack.middleware import habi_middleware

@habi_middleware
def lambda_handler(event, context):
    # Tu código aquí
    pass
```

#### sentry_middleware

Este decorador activa Sentry incluso cuando `habi_middleware` no está definido:

```python
from globack_utils.globack.logger.sentry import sentry_middleware
from globack_utils.globack.middleware import habi_middleware

@sentry_middleware
@habi_middleware
def lambda_handler(event, context):
    # Tu código aquí
    pass
```

#### sentry_event_middleware

Similar a `sentry_middleware`, pero permite recibir eventos personalizados diferentes a API Gateway:

```python
from globack_utils.globack.logger.sentry import sentry_event_middleware
from globack_utils.globack.middleware import habi_middleware

@sentry_event_middleware
@habi_middleware(is_custom_event=True)
def lambda_handler(event, context):
    # Tu código aquí
    pass
```

### Configuración de credenciales en Globack

Solicitar a DevOps la agregación de las credenciales del proyecto de Sentry (DSN) a Secret Manager:

```json
{
  "default": {...},
  "my-project-[dev|prod]": {
    "sentry": {
      "dsn": "https://examplePublicKey@o0.ingest.sentry.io/0",
      "config": {...}
    }
  }
}
```

## Implementación en Frontend (React)

### Configuración básica

La configuración para repositorios frontend debe incluir los siguientes atributos:

| **Atributo**            | **Req** | **Tipo**     | **Descripción**                                      |
|:------------------------|:--------|:-------------|:----------------------------------------------------|
| dsn                     | *       | string       | Identificador del proyecto                           |
| environment             | *       | string       | Ambiente del proyecto (dev, prod)                    |
| release                 | *       | string       | Versión del proyecto                                 |
| tracesSampleRate        |         | number(0-1)  | Porcentaje de muestras de performance                |
| replaysSessionSampleRate|         | number(0-1)  | Porcentaje de muestras de replay                     |
| replaysOnErrorSampleRate|         | number(0-1)  | Porcentaje de muestras de replay de errores          |
| integrations            |         | array        | Plugins opcionales                                   |
| tracesSampler           |         | function     | Función para filtrado de muestras                    |

> **Nota**: Es importante ajustar los porcentajes de muestreo según el entorno para optimizar cuotas.

### Implementación en CRA/Microfrontend

#### Instalación

```bash
# Usando yarn
yarn add @sentry/react

# Usando npm
npm install --save @sentry/react
```

#### Configuración

Inicializar Sentry lo antes posible en el ciclo de vida de la aplicación:

```javascript
import * as Sentry from "@sentry/react";

Sentry.init({
  dsn: process.env.REACT_APP_ENV_SENTRY_DSN,
  environment: process.env.REACT_APP_ENV_SENTRY_ENVIRONMENT,
  integrations: [
    new Sentry.BrowserTracing({
      tracePropagationTargets: ["localhost", /^https:\/\/yourserver\.io\/api/],
    }),
    new Sentry.Replay(),
  ],
  
  // Performance Monitoring
  tracesSampleRate: 1.0, // Reducir en producción
  
  // Session Replay
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});
```

### Implementación en Next.js

#### Instalación automática

```bash
npx @sentry/wizard@latest -i nextjs
```

El asistente creará automáticamente:
- `sentry.client.config.js` y `sentry.server.config.js` 
- Actualizará `next.config.js` con la configuración de Sentry
- Creará `.sentryclirc` y `sentry.properties` para la carga de mapas de origen

#### Variables de entorno

```
NEXT_PUBLIC_ENV_SENTRY_DSN=https://examplePublicKey@o0.ingest.sentry.io/0
NEXT_PUBLIC_ENV_SENTRY_ENVIRONMENT=prod
```

### Subida de mapas de origen (sourcemaps)

Para mejorar los stack traces en código minificado:

```bash
npx @sentry/wizard@latest -i sourcemaps
```

### Verificación de la instalación

Insertar código con un error intencional para verificar que Sentry captura correctamente:

```javascript
// Para React/Next.js
return <button onClick={() => methodDoesNotExist()}>Break the world</button>;

// Para JavaScript puro
myUndefinedFunction();
```

## Buenas prácticas

1. **Ambiente local**: Configurar para no enviar trazas en desarrollo local
2. **Muestreo**: Ajustar tasas de muestreo según el ambiente para optimizar costos
3. **Contexto**: Añadir información contextual a los eventos para facilitar diagnóstico
4. **PII**: Evitar enviar información personal identificable a través de Sentry
5. **Verificación**: Comprobar la instalación con errores controlados antes de desplegar
