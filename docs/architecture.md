# Arquitectura de la solución

Pregunta de negocio: para cada combinación activa de operador de red, mercado de comercialización, tipo de mercado y sector CIIU, ¿cuál es la demanda de energía esperada para los próximos 7 días?

Patrón de solución: **batch diario + machine learning**. Los datos llegan por archivo con retraso variable (5 a 205 días) y el pronóstico se necesita una vez al día. No hay caso para streaming.

## Capas y contratos

Llena una fila por tabla. Reemplaza cada `<…>`. El dueño es un rol de XM, no una persona.

| Capa | Tabla | Grano (qué es una fila) | Dueño | Frescura | Garantías |
|---|---|---|---|---|---|
| Bronce | `bronze_energia.demanda_raw` | <un registro publicado, tal como llegó> | <Ingeniería de Datos> | <Cada que llega un nuevo archivo a la ruta o cuando cambie el catálogo> | <Conserva el dato original, historial de cargas y republicaciones sin transformaciones de negocio> |
| Plata | `silver_energia.demanda_diaria` | <Una fila por fecha y serie (codigo_sic_agente, mercado_comercializacion, tipo_mercado, clasificacion_industrial)> | <Ingeniería de Datos> | <Diario, posterior a la carga de Bronce> | <Datos normalizados, reglas de calidad aplicadas, versión vigente identificada y estructura consistente> |
| Plata | `silver_energia.dim_ciiu` | <Una fila por clasificación CIIU> | <Ingeniería de Datos> | <Bajo demanda cuando cambie el catálogo> | <Catálogo único, códigos válidos y atributos estandarizados> |
| Oro | `gold_energia.features_demanda_diaria` | <Una fila por fecha y serie (tipo_mercado, Variable)> | <Analítica Avanzada> | <Diario posterior a la actualización de Plata> | <Variables listas para entrenamiento e inferencia, definiciones consistentes y reproducibles> |
| Oro | `gold_energia.pronostico_demanda` | <Una fila por fecha de ejecución, serie y día pronosticado del horizonte de 7 días> | <Analítica Avanzada> | <Diario posterior a la actualización de Plata> | <Pronóstico oficial generado por el modelo vigente, con trazabilidad de ejecución y versión del modelo> |

## Llave de serie

`codigo_sic_agente`, `mercado_comercializacion`, `tipo_mercado`, `clasificacion_industrial`. 355 combinaciones activas.

## Decisiones de diseño derivadas de la exploración (lab 2)

| Hecho | Decisión | Dónde se implementa |
|---|---|---|
| Formato largo (2 filas por serie-día) | <Consolidar demanda y pérdidas en una única fila por serie y fecha, usando columnas separadas para cada métrica> | Plata (clase 6) |
| Publicación con retraso variable y republicaciones | <Conservar todas las publicaciones en Bronce y determinar en Plata la versión vigente mediante la fecha de publicación más reciente> | Bronce (clase 4) / Plata (clase 6) |
| Series incompletas (4 de 355) | <Mantener los registros incompletos, marcar faltantes y generar indicadores de completitud para el modelado> | Plata / features (clase 7) |
| Regulado vs. no regulado | <Implementar dos modelos por tipo de mercado para evitar problemas por la diferencia de patron de comportamiento de ambos mercados catálogo> | Modelo (clase 9) — ver ADR-001 |
| Ceros (299 serie-días) | <Conservar los ceros válidos y marcar como advertencia aquellos que resulten anómalos según reglas de calidad> | Reglas de calidad (clase 5) |
| Pérdidas ≤ demanda | <Validar que las pérdidas no superen la demanda; las violaciones se marcan para revisión de calidad> | Reglas de calidad (clase 5) |

## Diagrama

```
DemandaPerdidas.xlsx ──▶ [volumen raw] ──▶ bronze_energia.demanda_raw
                                                │
                                                ▼  (pipeline declarativo, reglas de calidad)
                                     silver_energia.demanda_diaria ◀── silver_energia.dim_ciiu
                                                │
                                                ▼  (job de features)
                                  gold_energia.features_demanda_diaria
                                                │
                                                ▼  (job de inferencia, modelo en UC)
                                     gold_energia.pronostico_demanda ──▶ tablero / app
```

En Free Edition todo vive en el catálogo `workspace`; desde la clase 3, en `dev`, `qa` y `prod`. En Azure Databricks cada capa se registra como external location sobre ADLS Gen2.

## ADRs relacionadas

- [ADR-001 — Granularidad y horizonte del pronóstico](adr/ADR-001-granularidad-horizonte.md)
