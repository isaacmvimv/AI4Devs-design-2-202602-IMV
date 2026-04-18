# User stories generation from PRD
**Isaac Mani Valor**
_Exported on 4/10/2026 at 21:23:47 UTC from Cursor (3.0.16)_

---

**Prompt 01 - US**

Eres un "Product Owner" experto con conocimientos de "metodologías ágiles y ATS (Applicant-Tracking System)".
Quiero que analices el documento adjunto que es el PRD del proyecto (está en formato MarkDown), y que basándote en el mismo, me generes todas las User Stories correspondientes en un documento con formato MarkDown.

En el análisis debes de detectar si falta algún aspecto importante e imprescindible para llevar a cabo un primer MVP y considerarlo.

Genera un primer bloque con la definición de cada una de las User Stories, donde debes de aplicar buenas prácticas, entre ellas las siguientes:
- Crear cada US con un título conciso, claro y descriptivo.
- Formato "Como [rol], quiero [acción], para [beneficio]"
- Evaluación breve "INVEST" (Independent, Negotiable, Valuable, Estimable, Small, Testable).
- Listar los "Acceptance Criteria" (AC) en formato "Behavior-Driven Development" (BDD) (Gherkin): Given/When/Then.
- Los "Acceptance Criteria" (AC) deben de incluir el "happy path" y sugerir escenarios de "edge-cases".
- Detectar algunos escenarios de error habituales.
- Añadir una estimación de complejidad (S/M/L)
- Clasificar según su importancia para el MVP siguiendo el método de priorización MoSCoW (Must have, Should have, Could Have, Won't Have).

Una vez elaborada la lista de las US, quiero que valides la consistencia entre las que estén relacionadas.

---

**Prompt 02 - BackLog**

Ahora añade un segundo bloque donde se diseñe su Backlog, se listen las US sugiriendo su orden de priorización para el MVP (MoSCoW), y se tengan en cuenta criterios importantes como pueden ser los siguientes:
- Valor para el Negocio.
- Urgencia.
- Dependencias.
- Coste de implementación basado en esfuerzo y complejidad.
- Riesgos y obstáculos potenciales.
- Madurez tecnológica.

A modo de resumen, añade un tercer bloque con una matriz en formato tabla que resuma el backlog ordenado incluyendo las siguientes columnas: Item, Impacto usuario / negocio, Urgencia, Complejidad, Riesgos / dependencias, MoSCoW.

Añade un roadmap en formato Mermaid con el Backlog priorizado por Sprints. Cada conjunto de US debe aportar funcionalidades completas.

---

**Prompt 03 - Tickets**

Por último, quiero que actúes cómo un "Project Manager" experto con conocimientos de "metodologías ágiles y ATS (Applicant-Tracking System)".
Tu objetivo es analizar cada User Story del Backlog y dividirla en sus respectivos Tickets necesarios para su desarrollo y entrega del MVP.

Los tickets deben de estar bien estructurados y tener un nombre breve, conciso, claro y descriptivo. 
Cada ticket debe de contener toda la información necesaria para llevarse a cabo su desarrollo, así como incluir las partes relevantes de la US a la que pertenecen, como pueden ser los "Acceptance Criteria", "happy path", "edge-cases", etc.
También debe de quedar muy en cada ticket a que US pertenece.
Los tickets, al igual que se ha hecho con las User Stories, deben tener una estimación de complejidad (S/M/L) y unidades (puntos de historia) y clasificación MoSCoW.
