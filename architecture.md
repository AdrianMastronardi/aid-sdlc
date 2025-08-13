### 3. Patrones y estructuras arquitectónicas optimizadas para IA

La organización del código no solo afecta la mantenibilidad por humanos; en un entorno asistido por IA, también define la **calidad del contexto** con el que el asistente trabajará para proponer, modificar o extender funcionalidades.  

#### 3.1. Estructuras que sirven como “contexto vivo”
Los frameworks que ofrecen una arquitectura opinada y coherente facilitan que la IA entienda **dónde vive cada responsabilidad** y **cómo viajan los datos**. Por ejemplo:

- **FastAPI + SQLModel/Pydantic**:  
  - **DTOs unificados**: Definir modelos en SQLModel que sirvan a la vez para persistencia, validación de parámetros y transferencia interna de datos elimina duplicaciones y proporciona al asistente una referencia clara y única de las estructuras.  
  - **Convenciones de ubicación**: Endpoints en `app/api/routes`, dependencias en `app/api/deps.py`, modelos en `app/models`, esquemas en `app/schemas`. Esto permite a la IA localizar y editar código de forma confiable.  
  - **Pydantic como contrato**: Cada request/response validado por Pydantic actúa como documentación viva y verificable, reduciendo errores y ambigüedades.

Este tipo de arquitectura **inyecta contexto implícito**: los nombres de archivos, la separación de responsabilidades y el uso de modelos compartidos se convierten en “metadatos” que la IA aprovecha para inferir intención y mantener la coherencia.

#### 3.2. Layering vs. vertical slices
En un contexto de IA, el modelo tradicional en capas (controllers → services → repositories) sigue siendo válido, pero presenta limitaciones:  
- Para que la IA entienda un cambio, debe recorrer múltiples capas, dispersas en diferentes módulos.  
- La intención de una funcionalidad puede diluirse si no se documenta bien el flujo completo.

**Vertical slices** agrupan todo lo necesario para una funcionalidad (handler, lógica, modelos, validaciones) en un mismo módulo o carpeta.  
- **Ventaja para IA**: El asistente puede leer y modificar un slice completo con todo el contexto en un único recorrido, reduciendo riesgos de inconsistencias.  
- **Recomendación**: Incluso en proyectos con layering, considerar slices para dominios bien definidos o endpoints críticos, documentando cómo se conectan con el resto de la arquitectura.

#### 3.3. Diseñar para contar una historia
Diseñar el sistema en torno a slices claros y con propósito (el **why**) —en lugar de una colección desagregada de microservicios o lambdas— mejora radicalmente el trabajo con IA. Un slice bien definido actúa como capítulo completo de una historia:

- **Code should tell a story, not just compile**: El flujo desde la entrada de datos hasta el resultado final debe ser legible y coherente.  
- **Docs live beside the code, searchable and semantic**: La documentación relevante vive junto al código, enlazada y fácil de buscar por humanos y asistentes.  
- **APIs are boringly predictable**: Rutas, métodos y formatos de respuesta siguen patrones consistentes, evitando que la IA “adivine” comportamientos.  
- **Constraints are just tight enough to prevent chaos**: Reglas y convenciones claras limitan la dispersión de estilos y decisiones, sin ahogar la flexibilidad necesaria para evolucionar.

#### 3.4. Cuando no hay un framework opinado
Si el stack no cuenta con una estructura tan clara:
- Documentar explícitamente **qué archivos contienen qué tipo de lógica** (controladores, modelos, validaciones, utilidades).
- Definir **nombres y patrones consistentes** (por ejemplo, `*.model.ts` para modelos, `*.controller.js` para controladores).
- Incluir un **mapa de dependencias y flujo de datos** para que la IA pueda seguir el recorrido de la información sin ambigüedades.

#### 3.5. Recomendaciones clave para IA
- Mantener **archivos pequeños y funciones cortas**: facilitan que el asistente lea y entienda la unidad completa sin perder contexto.
- **Colocar funciones relacionadas físicamente cerca** en el árbol de archivos.
- Evitar “código mágico” o side-effects escondidos; documentar cualquier automatismo no obvio.



### 4. Contexto vivo como parte del ciclo de desarrollo

En un entorno asistido por IA, **el código y su contexto deben evolucionar juntos**. El contexto no es un anexo opcional: es un artefacto de primera clase que asegura que las decisiones, su propósito y sus consecuencias se mantengan accesibles y actualizadas.

#### 4.1. ¿Qué es “contexto vivo”?
Es toda la información necesaria para entender **por qué** el sistema es como es:  
- Requisitos y criterios de éxito.  
- Decisiones de diseño y arquitectura (ADRs).  
- Restricciones técnicas y de negocio.  
- Modelos de datos y sus relaciones.  
- Justificación de dependencias y librerías.  

Este contexto **vive junto al código**, versionado y editable como cualquier otro artefacto del repositorio.

#### 4.2. Integración en el flujo de desarrollo
El contexto vivo no se “escribe al final”; se produce y actualiza como parte de cada cambio:

1. **Nuevo feature o cambio**  
   - Crear/actualizar el ADR correspondiente.  
   - Definir contratos claros (interfaces, DTOs) en los modelos y endpoints.  
2. **Pull request**  
   - Verificar que el contexto asociado está completo y actualizado.  
   - Revisar si la IA podría entender el cambio con la información disponible.  
3. **Merge y despliegue**  
   - Contexto y código viajan juntos.  
   - Cambios relevantes quedan documentados de forma semántica y rastreable.

#### 4.3. ADRs como ancla del “why”
Los **Architecture Decision Records (ADRs)** son la forma más efectiva de capturar el *por qué* de las decisiones.  
- **Formato mínimo útil**: Contexto → Decisión → Alternativas → Consecuencias.  
- **Ubicación**: `/docs/adr` o `/architecture/decisions`.  
- **Beneficio para IA**: Cuando el asistente necesita modificar o extender código, puede consultar el ADR para entender la intención original y evitar cambios que rompan supuestos críticos.

#### 4.4. Metadatos y trazabilidad
Agregar metadatos estructurados al código y a los documentos de contexto facilita que la IA los consuma:
- Tags en comentarios (`@context`, `@adr-id`, `@owner`).
- Enlaces directos desde funciones/clases a ADRs relevantes.
- Referencias cruzadas entre slices, modelos y decisiones.

#### 4.5. Buenas prácticas para mantenerlo vivo
- **Automatizar recordatorios**: CI que advierte si hay cambios en código sin actualizaciones de ADR.  
- **Evitar la obsolescencia**: Revisar ADRs y metadatos en refactors o cambios de dominio.  
- **Visibilidad total**: El contexto debe ser fácilmente navegable y buscable, tanto por humanos como por herramientas.  

> **Principio clave**: *El contexto no es documentación que envejece; es parte del sistema en producción.*