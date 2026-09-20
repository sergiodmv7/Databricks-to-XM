# ADR-001: Granularidad y horizonte del pronóstico

- **Estado:** aceptada
- **Fecha:** <2026-09-20>
- **Autor:** <Sergio Marquez>

## Contexto

<Qué sabemos del dato y del negocio que obliga a decidir: cuántos días de historia hay, cómo llega el dato, cuántas series, qué pide XM ("siguiente período"), qué necesita el curso.>

## Decisión

<La información es recibida mediante archivos con una latencia variable, observándose retrasos entre muy variables con un maximo de 205 días respecto a la fecha del dato. Debido a que la necesidad de negocio es generar un pronóstico una vez por día, no se justifica una arquitectura orientada a eventos o procesamiento en tiempo real, se escogeria batch.>

##Capa Bronce

Responsable de almacenar la información exactamente como es suministrada por la fuente, conservando historial, fechas de publicación y posibles republicaciones.

Grano
Un registro recibido desde el archivo fuente, sin transformaciones de negocio.

Responsabilidades 
Preservar trazabilidad completa.
Mantener todas las versiones recibidas.
Registrar metadatos de carga y publicación.


##Capa Plata

Responsable de depurar, estandarizar y consolidar la información operativa.

Una fila por serie y fecha.
Consolidación de demanda y pérdidas.
Aplicación de reglas de calidad.
Resolución de versiones vigentes.

Responsabilidades

Normalización de estructura.
Validaciones de calidad.
Gestión de vigencias y republicaciones.
Preparación de información para analítica y modelamiento.


##Capa Oro

Responsable de habilitar la analítica avanzada y el consumo por negocio.

Responsabilidades

Construcción de variables predictivas.
Ejecución de modelos de machine learning.
Publicación de resultados para consumo de usuarios y aplicaciones.
Identificación de series

##Serie
Cada serie se define mediante la combinación de:

codigo_sic_agente
mercado_comercializacion
tipo_mercado
clasificacion_industrial>

## Diagrama

DemandaPerdidas.xlsx
        │
Bronce (ingesta y preservación histórica)
        │
Plata (depuración, calidad y consolidación)
        
Generación de variables analíticas
        │
Entrenamiento e inferencia de modelos
        │
Pronóstico de demanda a 7 días
        │
Tableros, aplicaciones y consumidores de negocio>

## Alternativas consideradas

- <Procesamiento en tiempo real (streaming)>: <se descarta porque la información no se genera ni se consume en tiempo real. Los datos llegan mediante archivos con latencia variable y el pronóstico solo necesita actualizarse una vez al día, por lo que el costo y complejidad operativa de una arquitectura streaming no aportan valor>
- <Procesamiento manual o basado únicamente en reglas de negocio>: <se descarta porque el objetivo es estimar demanda futura para 355 series con comportamientos heterogéneos. Un enfoque basado solo en reglas estáticas tendría dificultades para capturar estacionalidades, tendencias y diferencias entre mercados regulados y no regulados, reduciendo la precisión del pronóstico>

## Consecuencias

<Qué se gana

Arquitectura simple y fácil de operar.
Menor costo de implementación y mantenimiento.
Trazabilidad completa de las publicaciones recibidas.

Qué se pierde

No se generan pronósticos inmediatamente después de recibir nueva información.
Existe una latencia de hasta un día entre la llegada de datos y la actualización del pronóstico.

Qué queda en el backlog

Evaluar actualización intradía de pronósticos si el negocio lo requiere.
Automatizar monitoreo de desempeño y reentrenamiento de modelos.

Qué habría que cambiar para revertirla

Sustituir el procesamiento batch por una arquitectura orientada a eventos o streaming.
Implementar mecanismos de ingestión continua y procesamiento en tiempo real.
Adaptar el pipeline de features y la inferencia para ejecución near real-time.>
