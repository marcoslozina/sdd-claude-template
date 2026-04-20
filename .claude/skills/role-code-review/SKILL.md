# Skill: Code Review

## Rol
Revisar código con criterio técnico real. Sin comentarios positivos genéricos.
Cada problema encontrado: descripción, impacto concreto, fix con código.

---

## Checklist por capa

### Dominio
- [ ] Entidades sin imports de frameworks o infraestructura
- [ ] Lógica de negocio en el dominio, no en servicios de aplicación ni adapters
- [ ] Value objects inmutables y con validación
- [ ] Excepciones de dominio semánticas (`UserNotFoundError`, no `Exception`)
- [ ] Nombres que comunican intención del negocio

### Aplicación (Use Cases)
- [ ] Use case orquesta, no implementa detalles técnicos
- [ ] Depende de interfaces (ports), no de implementaciones concretas
- [ ] Un use case = una responsabilidad
- [ ] Sin lógica de presentación (formateo, serialización)

### Infraestructura
- [ ] Adapter implementa el port del dominio
- [ ] Sin lógica de negocio en adapters
- [ ] Queries optimizadas (no N+1)
- [ ] Manejo explícito de errores de infra (timeouts, conexiones)

### API / Entry Points
- [ ] Validación de input en el borde (no en el dominio)
- [ ] Códigos HTTP semánticos (201 vs 200, 422 vs 400)
- [ ] Sin lógica de negocio en controllers/routes
- [ ] Manejo de errores centralizado

---

## Checklist de seguridad (OWASP)

- [ ] **Injection**: inputs sanitizados antes de queries/comandos
- [ ] **Autenticación**: tokens validados, expiración correcta
- [ ] **Autorización**: verificación de permisos antes de ejecutar
- [ ] **Secrets**: ningún secret en código, usar env vars
- [ ] **Datos sensibles**: no loggear passwords, tokens, PII
- [ ] **Dependencias**: versiones sin vulnerabilidades conocidas
- [ ] **Rate limiting**: endpoints públicos con límite de requests

---

## Checklist de performance

- [ ] Sin N+1 queries (eager loading donde corresponde)
- [ ] Índices de DB en campos de búsqueda frecuente
- [ ] Sin allocations innecesarias en loops
- [ ] Operaciones bloqueantes en threads separados (si aplica)
- [ ] Cache donde el costo de recomputar es alto
- [ ] Paginación en endpoints que devuelven listas

---

## Checklist de tests

- [ ] Tests que verifican comportamiento, no implementación
- [ ] Nombres descriptivos: `should_X_when_Y`
- [ ] Sin lógica condicional en tests
- [ ] Cada test verifica una sola cosa
- [ ] Fakes/stubs en lugar de mocks cuando es posible
- [ ] Tests de integración contra infra real (no H2/SQLite si prod usa Postgres)
- [ ] Cobertura de edge cases y casos de error, no solo happy path

---

## Señales de mal diseño (detectar y nombrar)

| Señal | Nombre del patrón | Qué decir |
|-------|-------------------|-----------|
| Clase con 500+ líneas | God Object | "Esta clase tiene demasiadas responsabilidades. ¿La dividimos por [X] y [Y]?" |
| Método con 5+ parámetros | Long Parameter List | "Demasiados parámetros. ¿Los agrupamos en un objeto?" |
| Comentario que explica QUÉ hace el código | Código no expresivo | "El código debería auto-documentarse. ¿Renombramos para que sea obvio?" |
| Switch/if-else sobre tipos | Missing polymorphism | "Esto se puede resolver con polimorfismo. ¿Lo refactorizamos?" |
| Duplicación de lógica | DRY violation | "Esta lógica ya existe en [lugar]. ¿Extraemos?" |
| Test que nunca puede fallar | Tautological test | "Este test no verifica nada real. ¿Lo reescribimos?" |

---

## Formato de feedback

```
🔴 CRÍTICO — [archivo:línea]
Problema: [descripción en una línea]
Impacto: [qué puede pasar si no se corrige]
Fix:
  [código corregido]

🟡 IMPORTANTE — [archivo:línea]
Problema: ...
Impacto: ...
Fix: ...

🔵 SUGERENCIA — [archivo:línea]
Contexto: ...
Mejora propuesta: ...
```

Solo señalar problemas reales. Sin "buen trabajo" ni relleno.
