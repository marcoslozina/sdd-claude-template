# Skill: Live Coding

## Propósito
Guiar una sesión de live coding donde el problema se descubre en tiempo real.
El objetivo dual es: resolver el problema + comunicar decisiones al stream/audiencia.

## Protocolo de inicio (SIEMPRE)

Cuando llega un problema nuevo:
1. **Repetí el problema** en tus propias palabras para confirmar entendimiento
2. **Hacé las 3 preguntas críticas** antes de cualquier código:
   - ¿Cuál es el input/output esperado?
   - ¿Hay restricciones de performance, escala, o tecnología?
   - ¿Es un prototipo o va a producción?
3. **Proponé el enfoque** con tradeoffs antes de implementar
4. **Confirmá con el usuario** antes de arrancar

## Durante el desarrollo

### Narración para la audiencia
Antes de cada decisión no obvia, explicá en voz alta:
- QUÉ vas a hacer
- POR QUÉ esta opción y no otra
- QUÉ tradeoffs estás aceptando

### Cuando te trabás
No adivinés. Decí exactamente:
- "Necesito verificar X antes de continuar"
- "Hay dos opciones acá, vamos a evaluar ambas"
- "Esto me huele a [patrón], vamos a explorar"

### Red flags que hay que nombrar en vivo
- Acoplamiento entre capas
- Lógica de negocio en la infra
- Tests que no testean nada real
- Over-engineering para el problema actual
- Under-engineering que va a doler en producción

## Estructura de respuestas durante el live coding

Para cada bloque de código que se escribe:
```
[DECISIÓN]: Por qué este approach
[CÓDIGO]: El código en sí
[TRADEOFF]: Qué estamos sacrificando
[ALTERNATIVA]: Qué haríamos diferente con más tiempo/escala
```

## Comandos de contexto rápido

- "explorá" → `/sdd-explore` del problema actual
- "decidamos arquitectura" → proponer 2-3 opciones con tradeoffs
- "documentá la decisión" → crear ADR en `docs/adr/`
- "mostrá la estructura" → generar árbol de carpetas del proyecto actual
- "resumí lo que hicimos" → bullet list de decisiones tomadas hasta ahora
