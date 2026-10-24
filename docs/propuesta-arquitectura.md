# Propuesta de arquitectura para coordinar el equipo de agentes

Estado: diseño propuesto; no implementa un servicio ni modifica el funcionamiento actual del equipo.

## Brief y alcance

**Usuario:** quien solicita y supervisa trabajo de producto desde Control Room. **Problema supuesto:** convertir las instrucciones del equipo en ejecuciones acotadas, trazables y recuperables cuando una solicitud excede el tiempo disponible. **Resultado esperado:** un único supervisor integra entregas de especialistas y presenta evidencia con pendientes explícitos.

La hipótesis es que el pedido se refiere a este equipo de agentes; aún no hay un producto de negocio definido ni validación comercial. El repositorio actual contiene instrucciones en `AGENTS.md` y `EQUIPO.md`, no un orquestador desplegado. Vercel Hobby y el límite de 300 segundos se toman del encargo de esta ejecución, sin presuponer infraestructura adicional disponible.

Incluido en el diseño: coordinación, contratos de tareas, estado recuperable, autorización y verificación. Excluido: agentes permanentes, publicación automática, selección de un proveedor de almacenamiento y construcción de una nueva interfaz.

## Componentes y recorrido

1. **Usuario → Control Room:** presenta objetivo y restricciones; consulta progreso y solicita reanudación. Se reutiliza la interfaz existente.
2. **Control Room → supervisor único:** abre una ejecución autenticada, transmite permisos y un presupuesto de tiempo. No crea una capa adicional de supervisión.
3. **Supervisor → hasta dos especialistas efímeros:** encarga tareas acotadas cuando son independientes, con archivos de propiedad exclusiva. El número de roles disponibles no equivale al número de procesos activos.
4. **Especialistas → supervisor:** devuelven cambios, decisiones y evidencia; el supervisor espera sus resultados antes de integrarlos. Después de 90 segundos puede continuar con evidencia propia y registra la revisión como pendiente, según el encargo vigente.
5. **Supervisor ↔ estado de ejecución y artefactos:** registra plan, tareas, versiones y resultados antes de avanzar. El almacenamiento durable es un requisito futuro, no una capacidad demostrada del repositorio ni una garantía del disco temporal de Vercel.
6. **Supervisor → etapa de verificación y entrega → Control Room → usuario:** reúne evidencia, declara límites y devuelve el resultado. La revisión independiente utiliza uno de los dos cupos existentes o se realiza en una tanda posterior; no añade un tercer especialista.

Al acercarse al límite, el supervisor deja un punto de recuperación y devuelve estado pendiente. Una solicitud posterior de Control Room recupera ese estado y sigue el trabajo autorizado; no se supone que el proceso siga vivo después de responder. Se propone reservar 30 segundos de los 300 para persistir y entregar: es un presupuesto inicial a validar, no una medición.

## Datos y contratos mínimos propuestos

Una única versión canónica por proyecto contiene brief, backlog y contratos. Los resultados de cada ejecución referencian esa versión; los especialistas no mantienen copias divergentes.

| Registro | Campos mínimos | Regla |
| --- | --- | --- |
| Ejecución | `runId`, `projectId`, `actorId`, `status`, `revision`, `deadline`, `authorizationScope` | Estados: pendiente, ejecutando, pausada, completada, fallida. Solo una ejecución modifica una revisión a la vez. |
| Tarea | `taskId`, `runId`, `role`, `objective`, `inputs`, `ownedPaths`, `acceptance`, `status` | Propiedad de archivos sin solapamientos; máximo dos especialistas activos. |
| Resultado | `taskId`, `artifactRefs`, `decisions`, `verification`, `risks`, `pending` | Cada evidencia declara comando o procedimiento, resultado y fecha; ausencia de medición se expresa como tal. |
| Evento | `eventId`, `runId`, `taskId`, `actorId`, `type`, `timestamp`, `revision` | Identificador único para deduplicar; sin secretos ni contenido sensible innecesario. |

Operaciones conceptuales: iniciar ejecución con clave de idempotencia; consultar por `runId`; registrar resultado con revisión esperada; pausar con referencia al punto de recuperación; reanudar desde esa revisión. No se especifican rutas HTTP como si ya existieran. Un conflicto de revisión rechaza la escritura sin sobrescribir cambios y obliga a releer el estado.

## Fallos, autorización y operación

- La identidad de Control Room debe vincular cada solicitud con actor y proyecto. Cada lectura, cambio y reanudación verifica ese alcance; un identificador de ejecución no concede acceso.
- Un encargo de especialista no amplía permisos. Publicar, enviar mensajes externos o acceder a integraciones exige autorización vigente del usuario. En esta ejecución están prohibidos commits, push y PR.
- Los secretos se mantienen fuera de artefactos y registros, mediante variables del entorno autorizado. Los conectores externos son opcionales y no se incluyen en el camino mínimo.
- Datos inválidos, falta de acceso, recurso inexistente y conflicto se devuelven como errores accionables sin reintentos ciegos. Ante timeout o dependencia fallida se conserva estado; solo se reintentan operaciones idempotentes de forma acotada.
- Un resultado tardío conserva identidad de tarea y revisión; se integra únicamente si sigue siendo aplicable. Un timeout no demuestra que una operación externa haya fallado: se comprueba su estado antes de repetirla.
- La persistencia debe sobrevivir a la solicitud, disponer de control de concurrencia y recuperación. Su tecnología, retención, costo y respaldo requieren decidir primero volumen y sensibilidad de los datos. Sin ese mecanismo, la reanudación es manual desde artefactos disponibles y no se promete recuperación durable.
- Administración sin código propuesta en Control Room: consultar ejecuciones, pausar/reanudar y editar parámetros acotados de concurrencia y presupuesto, con permisos, validaciones e historial. No se afirma que esos controles estén implementados hoy.
- Instrumentación propuesta: inicio/fin, duración, pausas, reintentos, verificaciones y costo o tokens solo cuando el proveedor los entregue. Línea base, costo real y capacidad: **no medidos**.

## ADR-001: ejecución acotada con estado recuperable

**Decisión propuesta:** usar ejecuciones por solicitud con un supervisor y especialistas efímeros; persistir progreso y reanudar mediante una nueva solicitud explícita. No introducir una cola o procesos permanentes inicialmente.

| Alternativa | Ventaja | Consecuencia |
| --- | --- | --- |
| Todo síncrono en memoria | Menor esfuerzo inicial | Pierde progreso ante corte; insuficiente para recuperabilidad. |
| Solicitud acotada con estado durable | Encaja con el límite de ejecución y evita operar workers | Requiere persistencia, idempotencia y reanudación explícita; elegida como propuesta. |
| Cola y workers persistentes | Continuidad autónoma y trabajos largos | Añade operación, costos, reintentos y despliegue fuera del proceso web; pospuesta. |

**Desencadenantes propuestos para reconsiderar:** si más del 10% de las últimas 20 ejecuciones requieren otra solicitud por duración, evaluar cola/worker; si aparece una necesidad confirmada de continuar sin intervención, evaluar ejecución durable antes de ofrecerla; ante cualquier sobrescritura concurrente, detener escrituras incompatibles y revisar aislamiento. Estos umbrales son decisiones iniciales, no métricas alcanzadas ni requisitos comerciales validados.

**Evolución:** primero confirmar el uso y persistencia disponible; luego construir un corte iniciar → delegar → guardar → consultar; después verificar interrupción/reanudación e integración idempotente. Solo incorporar infraestructura adicional con evidencia de necesidad y presupuesto acordado.

## Verificación y pendientes

Este entregable es documental. Fuentes revisadas: `AGENTS.md`, `EQUIPO.md` y restricciones del encargo. No se ejecutaron pruebas funcionales ni de carga, no se midieron tokens y no se declara preparación para producción.

Para implementar: inventario versionado de casos y escenarios, pruebas de aislamiento, autorización, conflicto, timeout, resultado tardío y recuperación sin duplicados. La aceptación de software del equipo exige cobertura funcional estrictamente mayor al 90% y todos los escenarios críticos aprobados; hoy esa cobertura está **no medida**. Pendientes: confirmar hipótesis de producto, mecanismo de persistencia, presupuesto operativo, objetivos de recuperación y revisión independiente de seguridad.
