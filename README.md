# Asistente Inteligente con RAG y Automatización para la Validación de Recetas y Prestaciones Médicas (Discapacidad)

Proyecto Final: flujo de n8n que recibe consultas de prestadores de salud por correo institucional y WhatsApp, valida los datos del prestador, consulta normativas con RAG y decide si **aprueba**, **rechaza por formato inválido** o **deriva a un operador humano**.

## 1. Problema y objetivo

| Indicador | Situación actual | Objetivo |
|---|---|---|
| Casos diarios | 60 | 60 |
| Trabajo manual | 4,5 h/día | Reducción del 80 % |
| Ahorro estimado | — | Más de 22 h/semana |

El área de discapacidad valida a mano recetas, órdenes médicas y coberturas contra manuales, vademécums y normativas. El asistente automatiza la clasificación y deja a las personas solo los casos dudosos.

## 2. Arquitectura
```text
Correo institucional (Gmail) ─┐
                              ├─> Normalizar ─> Validar datos del prestador (CUIT mod-11)
WhatsApp (chat simulado) ─────┘                         │
                                          ¿CUIT inválido? ──sí──> Derivación humana
                                                        │no
                                        Agente de validación (RAG)
                                   (LLM + memoria + consultar_normativas)
                                                        │
                                        ¿Falló la consulta RAG? ──sí──> Derivación humana (fallback)
                                                        │no
                                        Interpretar decisión del agente
                                                        │
                                Ruta según decisión: APROBADO / RECHAZADO / DERIVAR
                                   │                 │                 │
                     Cargar en sistema        Responder al      Notificar operador (Gmail)
                     (Data Table)             prestador         + Registrar en planilla (Google Sheets)
                                                                + Responder al prestador

Formulario "Cargar normativas al RAG" ─> Leer documentos ─> Dividir en fragmentos ─> Embeddings ─> Vector Store

### Componentes

| Componente | Nodo n8n | Función |
|---|---|---|
| Entrada correo | Gmail Trigger | Lee correos nuevos del buzón institucional |
| Entrada WhatsApp | Chat Trigger (simulado) | Simula los mensajes de WhatsApp Business desde el chat de n8n |
| Validación | Code | Extrae CUIT y n° de afiliado, valida el dígito verificador del CUIT (módulo 11) |
| Agente | AI Agent | Aplica el System Prompt y decide la resolución |
| LLM | Ollama Chat Model (`llama3.1:8b`) | Modelo local, por privacidad de datos de salud |
| Memoria | Window Buffer Memory | Contexto por remitente (6 mensajes) |
| RAG | Simple Vector Store (retrieve-as-tool) + Ollama Embeddings (`nomic-embed-text`) | Busca fragmentos en manuales, vademécums y normativas (top 5) |
| Ingesta | Form Trigger + Default Data Loader + Recursive Text Splitter | Carga PDF/DOCX/TXT al RAG (fragmentos de 1000 caracteres, solapamiento 150) |
| Persistencia | Data Table `prestaciones_validadas` | Registra los casos aprobados |
| Derivación | Gmail + Google Sheets ("Derivaciones de SaludBOT") | Avisa al operador y deja el caso como `pendiente` |
```
## 3. System Prompt

El agente actúa como asistente oficial de validación de recetas y órdenes del área de discapacidad. Reglas clave:

- Usa **solo** la información recuperada con `consultar_normativas`.
- Protocolo para respuestas desconocidas, con este texto literal:
  > "No dispongo de esa información en los manuales prestacionales vigentes. Por favor, indíqueme su CUIT y el número de afiliado para derivar su consulta a un operador humano."
- Tono profesional, claro y conciso, pensado para WhatsApp.
- Última línea obligatoria: `DECISION: APROBADO`, `DECISION: RECHAZADO` o `DECISION: DERIVAR`. Si falta, el caso se deriva.

## 4. Manejo de errores

| Situación | Comportamiento |
|---|---|
| CUIT ausente o con dígito verificador inválido | Derivación humana; **no** se inserta en la base |
| Falla o timeout del Vector Store / LLM | El agente continúa con salida de error; el If "¿Falló la consulta RAG?" envía el caso a derivación (fallback) |
| Respuesta del agente sin etiqueta de decisión | Se trata como `DERIVAR` |
| Formato de receta corregible | `RECHAZADO`: se responde al prestador indicando qué corregir, sin insertar |

## 5. Datos

**Data Table `prestaciones_validadas`:** canal, remitente, cuit_prestador, numero_afiliado, mensaje_original, resolucion, respuesta_agente, procesado_en.

**Google Sheets "Derivaciones de SaludBOT":** fecha, nombre, matricula_o_cuit, telefono, motivo, resumen, estado.

## 6. Instalación

1. Importar `workflow-asistente-salud-rag.json` en n8n (menú **⋯ > Import from file**).
2. Configurar credenciales: Gmail OAuth2, Google Sheets OAuth2 y Ollama.
3. En el servidor Ollama: `ollama pull llama3.1:8b` y `ollama pull nomic-embed-text`.
4. Si n8n corre en la nube, exponer Ollama con un túnel (ngrok o Cloudflare) y usar esa URL como Base URL de la credencial.
5. Completar el correo del operador en **Notificar operador humano**.
6. Publicar el workflow y cargar los manuales desde el formulario `/form/cargar-normativas-salud`.
7. Probar con el botón **Open chat** del canvas.

## 7. Modo demostración (simulado)

Para esta entrega se ejecutó el flujo en **modo simulado**, porque no había un servidor Ollama accesible desde n8n Cloud (uso computadora del trabajo y tiene limitaciones):

- WhatsApp se simula con el Chat Trigger de n8n.
- La respuesta del agente es un ejemplo fijo (decisión `APROBADO`). La búsqueda vectorial `consultar_normativas` no se ejecutó.
- Los envíos (correo, chat) y la inserción en la base usaron datos de prueba.

La lógica propia del flujo sí se ejecutó con datos reales de entrada: normalización, validación de CUIT, detección de fallo RAG, interpretación de la decisión y ruteo por resultado.

| Prueba | Entrada | Resultado |
|---|---|---|
| CUIT inválido | `20-12345678-9` | Derivación humana, sin inserción |
| Ollama no disponible | CUIT válido, LLM caído | Fallback a derivación |
| Caso aprobado (simulado) | CUIT `20-12345678-6`, afiliado `123456/01`, kinesiología | Aprobado → carga en sistema → respuesta al prestador |

## 8. Decisiones técnicas y limitaciones

- **Llama 3.1 en lugar de Llama 3:** Llama 3 no soporta herramientas en Ollama, y el agente necesita llamar a `consultar_normativas`.
- **Privacidad:** el modelo local evita enviar datos de salud a terceros. Con n8n Cloud, Ollama debe quedar expuesto a internet; para producción se recomienda n8n self-hosted en la misma red que Ollama.
- **Vector Store en memoria:** se pierde al reiniciar n8n. Para producción conviene Qdrant o PGVector.
- **WhatsApp:** simulado. Para producción se reemplaza el Chat Trigger por el WhatsApp Trigger y el nodo de envío de WhatsApp Business Cloud.
o automatizado: Recepción, validación semántica y carga en sistemas institucionales de recetas médicas, órdenes de prácticas y documentación de prestadores de salud (discapacidad) ingresadas a través de canales digitales (correo electrónico institucional y WhatsApp Business).

<img width="1314" height="412" alt="image" src="https://github.com/user-attachments/assets/ae6400d2-d67d-4b3f-aee4-377209655055" />
´´´
