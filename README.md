# Manual del equipo de agentes

Sección estática «Cómo funciona», en español, para consultar el recorrido del equipo, sus responsabilidades y la propuesta de arquitectura. Puede integrarse posteriormente en una aplicación; no implementa un servicio de orquestación.

## Consultar localmente

Abrí `index.html` en el navegador, o serví el repositorio desde su raíz:

```sh
python3 -m http.server 8000
```

Visitá `http://localhost:8000`. No requiere instalación, JavaScript ni conexión externa. El servidor debe incluir `.archify/` para que el enlace al diagrama funcione; algunos servicios excluyen carpetas ocultas.

## Archivos

- `index.html`: contenido del manual y navegación por secciones.
- `manual.css`: estilos adaptables, foco de teclado y preferencia de movimiento reducido.
- `EQUIPO.md`: responsabilidades y criterios canónicos del equipo.
- `docs/propuesta-arquitectura.md`: decisiones, alternativas y límites del diseño.
- `.archify/architecture-equipo-20261024/equipo.html`: diagrama existente; su validación completa y revisión en navegador están pendientes por un bloqueo del entorno.

## Alcance

El manual es consultable. Persistencia durable, pausa/reanudación y ejecución de agentes son capacidades propuestas, pendientes de implementación. No se incluyen formularios ni controles que simulen esas capacidades. Costos, tokens y capacidad del servicio no están medidos.
