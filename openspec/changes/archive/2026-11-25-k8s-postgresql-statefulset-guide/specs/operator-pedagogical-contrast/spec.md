# Spec Delta: Operator Pedagogical Contrast

## Purpose

Define los requisitos observables del laboratorio de pruebas de fallo (caos y split-brain) y la matriz analítica comparativa entre la gestión manual con StatefulSets y los Operadores de Kubernetes.

## ADDED Requirements

### Requirement: Ausencia de failover automático en Kubernetes nativo
El sistema experimental SHALL evidenciar que la caída intencionada del Pod primario no desencadena la promoción automática de la réplica en un despliegue nativo sin Operador.

#### Scenario: Simulación de caída del nodo primario
- **WHEN** el Pod primario `postgresql-0` es detenido o eliminado forzosamente
- **THEN** las peticiones de escritura hacia la base de datos son rechazadas y el Pod `postgresql-1` permanece en estado de réplica de solo lectura (`standby.signal` activo)

### Requirement: Demostración del riesgo de split-brain por intervención manual
La guía SHALL guiar al usuario a través de una promoción manual no coordinada para evidenciar el peligro de corrupción por doble primario cuando no existe consenso distribuido ni fencing.

#### Scenario: Promoción manual sin aislamiento de red
- **WHEN** el administrador ejecuta `pg_ctl promote` en `postgresql-1` y posteriormente `postgresql-0` se reincorpora al clúster
- **THEN** ambos nodos intentan aceptar escrituras independientes provocando divergencia de timelines (split-brain)

### Requirement: Matriz de contraste técnico con Operadores de Kubernetes
La documentación SHALL incluir una matriz comparativa detallada entre la aproximación manual con StatefulSet y las soluciones basadas en Operadores (CloudNativePG, Zalando, Crunchy Data).

#### Scenario: Consulta de capacidades avanzadas de base de datos
- **WHEN** el lector evalúa los requisitos de producción (consenso Raft, switchover sin pérdida, backups PITR, connection pooling)
- **THEN** la guía expone qué componentes del Operador asumen cada responsabilidad y por qué no son cubiertas por las primitivas base de Kubernetes
