# BankOCR
Backend de onboarding digital de un banco, donde se procesa documentos de identidad y formularios de solicitud, con OCR (AWS Textract) y debe clasificar cada solicitud.

## Diseño y propuesta de solución 

### 1. Riesgos principales (en orden de ataque)



**Riesgo 1  La entrada no tiene hardening: el sistema confía en todo lo que recibe.** El borrador no valida ni autentica nada en la puerta: acepta cualquier PDF o imagen, y el texto que devuelve el OCR entra directo al prompt del LLM. Es grave porque la entrada es superficie de ataque activa: archivos maliciosos o malformados, documentos falsificados, solicitudes duplicadas o masivas (denegación de servicio con coste, porque cada envío paga Textract y LLM), y prompt injection, un documento puede contener texto diseñado para manipular al modelo que "clasifica y valida"

**Riesgo 2  El LLM decide solo, con OCR de baja confianza, y su respuesta se parsea por substring.** El modelo no debe decidir: si un caso no se resuelve con reglas deterministas, va a evaluación humana, no a un LLM. "El modelo suele arreglarlo" ignora que un LLM no recupera un dígito ilegible: genera uno plausible, y en identificaciones eso es corrupción silenciosa de datos. Buscar "natural" o "jurídica" en texto libre es frágil "no es persona jurídica" contiene "jurídica". Es grave porque el error no es reproducible mismo documento, distinta respuesta y todo termina en `aprobada` sin ningún estado de excepción.

**Riesgo 3 Sin trazabilidad ni registro claro de errores: solo se guarda la decisión final.** "Para no llenar la base de datos" se elimina la evidencia de qué pasó, cuándo y por qué: no hay OCR crudo, ni confianza por campo, ni reglas aplicadas, ni errores por etapa, ni versiones. Es grave porque ante un reclamo, una auditoría o un incidente no se puede reconstruir nada. La respuesta correcta no es "guardar menos": es un almacén de trazas separado del transaccional (evidencias cifradas con retención definida; la base operativa solo con estados).

**Riesgo 4  Sin resiliencia: cadena síncrona, reintento de la solicitud completa y un solo proveedor sin fallback.** Todo vive en una ejecución de Lambda (tope 15 min, cobra por espera) y si Claude falla se repite todo: doble OCR, registros duplicados y dos decisiones distintas para el mismo documento. No existe plan B si el proveedor de OCR o de LLM se degrada. Es grave porque a escala las cuotas de Textract y de la API se vuelven cuello de botella y el sistema se cae en los picos.

**Riesgo 5  "Cero intervención humana" como meta absoluta.** La intervención humana tiene un coste real y hay que minimizarla, pero eliminarla es el riesgo, no la eficiencia. Con plantillas de documento bien estructuradas y captura determinista de campos, un objetivo realista es que ~80 % o más de las solicitudes fluyan sin tocar un analista; el resto anomalías, contradicciones, plantillas desconocidas, baja confianza debe ir a revisión humana con evidencia. Es grave como está planteado porque a escala un 1 % de error sobre cien mil solicitudes mensuales son mil cuentas mal abiertas al mes reportadas como éxito.

## 2. Arquitectura corregida
 
```
Cliente → API Gateway (+ WAF, auth, rate limit)
      → Lambda recepción: valida archivo (tipo, tamaño, escaneo), genera application_id
      → S3 privado y cifrado (presigned URLs, acceso mínimo)
      → Step Functions (una etapa = un paso idempotente por application_id + etapa)
          1. OCR: Textract asíncrono (AnalyzeDocument + Queries por plantilla)
             └─ fallback: proveedor OCR alterno si falla o si la confianza es sistemáticamente baja
          2. Post-proceso OCR: normalización (mayúsculas, tildes, formatos de fecha/ID),
             confianza POR CAMPO, origen (documento/página), contraste documento ↔ formulario
             └─ [OPCIONAL] LLM (Claude vía Bedrock): SOLO si la plantilla es no estructurada
                (escrituras, actas, poderes) → extrae campos con schema estricto, cita evidencia,
                puede abstenerse; su salida entra al motor de reglas como un campo más, con
                confianza propia — nunca decide
          3. Motor de reglas versionado
          4. Router de estados
      → PUEDE_CONTINUAR | INCOMPLETA | REVISION_HUMANA | ERROR_TECNICO
             └─ [OPCIONAL] LLM: redacta el resumen del caso para el analista en REVISION_HUMANA
      Errores irrecuperables → DLQ con alerta (ninguna solicitud se pierde en silencio)
```

**Hardening de entrada (riesgo 1).** WAF y autenticación en API Gateway; rate limiting por cliente; validación estricta del archivo antes de procesarlo (tipo real por magic bytes, no por extensión; tamaño máximo; escaneo antimalware); subida por presigned URL para que el binario nunca pase por la Lambda; deduplicación por hash del documento. El texto del OCR se trata siempre como contenido no confiable: nunca se concatena a un prompt sin delimitar, y el LLM si interviene no tiene permisos de acción.
 
**Post-proceso del OCR (antes de cualquier regla).** La salida cruda de Textract se normaliza de forma determinista: unificación de mayúsculas y tildes, limpieza de caracteres de ruido, parseo de formatos de fecha e identificador, mapeo de sinónimos de tipo documental. Cada campo conserva tres cosas: valor crudo, valor normalizado y confianza propia  la confianza global no se usa para decidir. Si un campo crítico queda bajo umbral, el sistema pide mejor imagen o contrasta con el proveedor alterno; nunca lo adivina.
 
**Plantillas de documento y formulario.** Cada tipo de documento admitido (identidad personal, registro mercantil, formulario de solicitud) se define en configuración con: campos esperados, `Queries` de Textract, formato por campo, umbral de confianza por campo, campos obligatorios y qué tipo de persona implica. Un documento cuya plantilla no se reconoce no se "interpreta": va a `REVISION_HUMANA`. El formulario de solicitud es una plantilla más, con versión, para que un cambio de diseño no rompa la extracción en silencio.
 
**Reglas deterministas (resuelven la clasificación y la validación):**
- Tipo de documento → tipo de persona esperado (natural / jurídica).
- Formato y longitud del identificador coherente con el tipo de documento (parametrizado por país).
- Obligatorios por tipo de persona: jurídica exige representante legal y su documento de identidad.
- Consistencia: nombre ↔ tipo de documento (una razón social en un documento personal es contradicción), mismo nombre entre documento y formulario.
- Confianza de campo crítico bajo umbral → verificar o pedir mejor imagen, nunca adivinar.
- Vigencia y duplicidad del documento, si el banco lo requiere.
**Estados de salida:**
- `PUEDE_CONTINUAR`: todas las reglas pasan y la confianza está en rango → avanza a la validación definitiva, que realiza una persona.
- `INCOMPLETA`: falta un documento o campo claramente identificado → se pide al cliente, sin consumir analista.
- `REVISION_HUMANA`: contradicción entre documentos, plantilla desconocida o confianza baja en campo crítico que no se resolvió.
- `ERROR_TECNICO`: fallo de infraestructura. Es un estado aparte: un timeout no significa que el cliente sea inválido.
**LLM: opcional, fase 2, solo alto impacto.** Fase 1 se implementa sin LLM: Textract estructurado + reglas cubren los casos de la prueba y la mayoría de los reales. Se evaluaría un LLM únicamente para documentos no estructurados que Textract no mapea a campos (escrituras de constitución, actas de nombramiento, poderes) y para redactar el resumen del caso al analista. Condiciones: Bedrock en la región del banco (los datos no salen de la cuenta), campos mínimos en el prompt, salida con schema estricto, opción explícita de abstenerse, sin permisos de escritura, respuesta y versión del modelo guardadas en el expediente. Se incorpora solo si demuestra mejora medible frente a reglas y con autorización de Seguridad y Cumplimiento.
 
**Revisión humana.** El analista ve documento, campos extraídos, confianza y reglas fallidas. Su decisión y motivo quedan auditados. Se muestrean también casos que continuaron automáticamente, para detectar falsos positivos.
 
**Notificaciones activas.** El sistema avisa, no espera a que alguien mire: cuando una solicitud cae en `REVISION_HUMANA` o supera un tiempo máximo en cola, se notifica al equipo por el canal que el banco autorice (SNS → correo, chat corporativo; un bot de mensajería es viable solo para alertas técnicas). Regla estricta: la notificación **avisa, nunca transporta** — lleva `application_id`, motivo y enlace al panel interno autenticado; ningún nombre, identificador ni documento sale por el canal de mensajería. Lo mismo aplica a alertas operativas (DLQ con mensajes, coste por solicitud fuera de rango, cola de revisión creciendo).
 
**Trazabilidad y datos (almacén separado).** Las trazas no compiten con la base transaccional: van a un almacén propio (S3 cifrado + índice consultable) con retención definida por el banco. Por solicitud: OCR crudo y normalizado, confianza y origen por campo, versión de plantillas y reglas, resultado y errores de cada etapa con timestamp, reintentos, intervención humana con motivo. Logs operativos solo con `application_id`, tiempos y códigos de error; nunca nombres ni identificadores. Métricas y alertas por etapa: tasa de error, latencia, solicitudes atascadas, % de automatización, correcciones humanas.
 
**Aplicación a los ejemplos**
 
| Caso | Resultado | Motivo |
|------|-----------|--------|
| A | `REVISION_HUMANA` | Documento de identidad personal con nombre de empresa: contradicción entre tipo de documento y titular. Probable mezcla de datos del representante con la razón social. La confianza 0.94 no resuelve una contradicción semántica. |
| B | `INCOMPLETA` | Documento de registro empresarial coherente con el nombre, pero representante legal ausente. Se pide el documento del representante. La confianza 0.71 obliga a verificar el identificador, no a enviar a analista por sí sola. |
 
## 3. Líneas rojas (nunca el modelo solo)
 
- Aprobar o rechazar una solicitud.
- Completar, inventar o "corregir" un campo con baja confianza.
- Resolver una contradicción entre documentos.
- Ignorar un campo obligatorio o un umbral de confianza.
- Modificar reglas, plantillas o umbrales.
- Decidir retención o borrado de datos personales.
- Ejecutar cualquier acción sobre el cliente (bloquear, notificar, escalar).
## 4. Preguntas al banco y supuestos
 
**Preguntas:**
- ¿Qué documentos y plantillas documentales existen hoy? ¿Hay muestras reales de cada una (buenas y malas) para calibrar el OCR y las reglas?
- ¿Para qué países se quiere este proceso de OCR? Los formatos de identificador, los tipos documentales y las reglas cambian por país, y eso define la configuración desde el día uno.
- ¿Cuáles son los campos obligatorios por tipo de persona (natural / jurídica) y por documento? ¿Quién los aprueba y con qué frecuencia cambian?
- ¿Qué criterio y rango de umbral acepta el banco para la clasificación? Es decir: ¿con qué confianza por campo una solicitud puede continuar sola, en qué rango pasa a revisión y bajo qué valor se pide nuevo documento?
- ¿Qué volumen actual y proyectado se espera, con qué SLA de respuesta y ventanas pico? ¿Hay estacionalidad (campañas, cierres) que debamos dimensionar?
- ¿Qué no estamos viendo? ¿Existen restricciones internas (regulatorias, de proveedores, de infraestructura ya contratada) que condicionen el diseño?
- ¿Se esta autorizado a enviar datos personales a un proveedor de IA, en qué región y bajo qué contrato?
- ¿Qué capacidad tiene el equipo de revisión y cómo corrige o apela un cliente?

  
**Supuestos:**
- El proceso es asíncrono de extremo a extremo: el cliente recibe confirmación de recepción y el resultado llega después.
- Esto es una **pre-validación documental**: el sistema clasifica, valida completitud y filtra. La validación definitiva la realiza siempre una persona cuando el caso está dentro del rango de aceptación; el sistema solo prepara y prioriza ese trabajo, nunca aprueba.
- Las reglas, plantillas y umbrales los define y aprueba el banco; el sistema los aplica versionados.
- Existe un equipo de revisión con capacidad conocida, y los casos fuera de rango o contradictorios siempre llegan a él con evidencia.

**Recomendaciones de escalabilidad y bajo consumo:** el diseño es serverless de punta a punta (Lambda, Step Functions, S3, Textract), así que el coste en reposo es cercano a cero y escala solo con la demanda. Para mantenerlo óptimo: cada documento se procesa con OCR una sola vez (dedup por hash y resultados reutilizables), el LLM queda fuera del camino crítico (la mayoría de solicitudes no lo paga), las Lambdas nunca esperan a servicios lentos (Textract asíncrono con notificación), límites de concurrencia por etapa para respetar cuotas de Textract sin tormentas de reintentos, y alarmas de coste por solicitud como métrica de primera clase , si el coste unitario sube, es síntoma de reprocesamiento o abuso, no solo de crecimiento.
 
