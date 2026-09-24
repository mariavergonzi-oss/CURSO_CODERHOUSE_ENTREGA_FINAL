# Sistema de Notificación de Vencimientos de Free Sale Certificates (FSC)

**Entrega Final – Ecosistema de Automatización IA Autónomo para Negocios (Coderhouse)**
Autora: María Victoria Vergonzi

---

## Resumen del proyecto

Las empresas que exportan dispositivos médicos necesitan un **Certificado de Venta Libre (Free Sale Certificate, FSC)** vigente para cada producto y país de destino. Si un certificado vence sin que nadie lo note, se frena la comercialización en ese mercado.

Este sistema automatiza el seguimiento de esos certificados de punta a punta:

- **Rama A – Aviso proactivo de vencimientos:** un cron diario lee los certificados vigentes en Notion, calcula los días que faltan para el vencimiento y detecta los que entran en ventana de aviso (180 o 30 días). La IA (Claude) redacta el mail de aviso y le asigna una prioridad según la criticidad del producto. Un humano aprueba el aviso antes de que el sistema registre la notificación.
- **Rama B – Extracción y confirmación de renovaciones:** la IA lee la respuesta del organismo regulador (texto libre pegado en Notion) y extrae la fecha de emisión, la vigencia y si el certificado fue confirmado. Con esos datos, el sistema calcula el nuevo vencimiento. Un humano aprueba o rechaza la actualización antes de que se escriba en la base de datos.

Cada acción del sistema (avisos, extracciones, aprobaciones, rechazos y errores) queda registrada en un **Historial de Eventos**, que alimenta el Dashboard de Control.

### Stack tecnológico

| Categoría | Herramienta | Uso |
|---|---|---|
| Orquestador | **n8n** | Flujo principal con Schedule Trigger (cron diario) y dos ramas de procesamiento |
| Base de datos | **Notion** | Tablas relacionadas *Certificados FSC* e *Historial de Eventos* |
| Procesamiento IA | **Anthropic – Claude Sonnet 5** | Redacción de avisos con prioridad + extracción estructurada (JSON) desde texto libre |
| Canal de salida | **Gmail** | *Send and Wait for Approval*: mail con botones Aprobar / Rechazar (HITL) |

### Características clave

- **Human-in-the-Loop:** dos puntos de aprobación obligatoria, antes de cada escritura crítica en la base de datos. El rechazo tiene poder real de veto (Prueba 5).
- **Filtro anti-bucle:** compara la ventana actual con la "Última ventana notificada" para no reenviar el mismo aviso todos los días (Prueba 2).
- **Rutas de error:** control de fechas faltantes (`error_fecha`), control de JSON inválido de la IA (`error_ia`) y confirmación explícita (`certificado_confirmado`) para no asumir renovaciones que el texto no confirma (Prueba 4).
- **Prompts dinámicos:** construidos con variables de cada registro (producto, país, criticidad, días restantes, ventana).
- **Test de estrés:** 5 ejecuciones documentadas, que incluyen caminos felices e infelices.

---

## Entregables

| # | Entregable | Archivo / Link |
|---|---|---|
| 1 | Diagrama de arquitectura | [architecture_diagram.pdf](architecture_diagram.pdf) |
| 2 | Manual operativo de datos (tablas + esquemas JSON) | [manual_operativo_datos (1).pdf](manual_operativo_datos%20%281%29.pdf) |
| 3 | Matriz de costos / optimización de modelos | [matriz_costos (1).pdf](matriz_costos%20%281%29.pdf) |
| 4 | Seguridad y resiliencia (minimización, error handlers, HITL) | [seguridad_resiliencia (1).pdf](seguridad_resiliencia%20%281%29.pdf) |
| 5 | **Dashboard de Control** (vista pública en Notion) | [Abrir Dashboard de Control](https://speckle-fireplace-048.notion.site/cfc2307ac57c497580a645ba6b60733f?v=3e4615dfa10281a1aee5000ce4771517&source=copy_link) |

## Archivos técnicos y evidencias

| Recurso | Archivo / Link |
|---|---|
| Documento principal: flujo, test de estrés (5 pruebas) y capturas de evidencia | [ENTREGA FINAL-VERGONZI MARIA.pdf](ENTREGA%20FINAL-VERGONZI%20MARIA.pdf) |
| Lógica del flujo (export de n8n) | [My workflow.json](My%20workflow.json) |
| Video demo (3 min) | [Ver video en Google Drive](https://drive.google.com/file/d/1gfqavBcCBDHk1SoDnpRT2xsGVbWxQf4x/view?usp=sharing) |
| Base de datos (solo lectura) – Certificados FSC | [Abrir en Notion](https://speckle-fireplace-048.notion.site/ccc02efe9b5749cf8fd916da1258d96f?v=a0539f7939cf4b4ab5f48bb545c535c4&source=copy_link) |
| Base de datos (solo lectura) – Historial de Eventos | [Abrir en Notion](https://speckle-fireplace-048.notion.site/cfc2307ac57c497580a645ba6b60733f?v=0fc8184e63e4464ebf80b87deeec2ed4&source=copy_link) |

### Contenido del repositorio

```
├── README.md
├── ENTREGA FINAL-VERGONZI MARIA.pdf     # Documento principal + test de estrés + capturas
├── My workflow.json                     # Flujo exportado de n8n
├── architecture_diagram.pdf             # Entregable 1
├── manual_operativo_datos (1).pdf       # Entregable 2
├── matriz_costos (1).pdf                # Entregable 3
└── seguridad_resiliencia (1).pdf        # Entregable 4
```

---

## Test de estrés (resumen)

| Prueba | Escenario | Resultado |
|---|---|---|
| 1 | Flujo completo – camino feliz (Rama A) | 2 certificados detectados en ventana, aviso aprobado y Notion actualizado |
| 2 | Re-ejecución inmediata (anti-bucle) | Ningún aviso duplicado, sin consumo de tokens de IA |
| 3 | Extracción de renovación desde texto libre (Rama B) | Datos extraídos correctamente, nuevo vencimiento calculado y aprobado |
| 4 | Camino infeliz: fecha faltante + mail ambiguo | `error_fecha: true` descartado antes de la IA; `certificado_confirmado: false` sin modificar el registro |
| 5 | Rechazo humano (Decline) | Registro sin cambios y rechazo registrado en el Historial |
