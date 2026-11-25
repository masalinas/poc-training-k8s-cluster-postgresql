# Proposal: Guía Práctica de PostgreSQL en Kubernetes con StatefulSet y Análisis de Operadores

## Why

El uso de Operadores de Kubernetes (como CloudNativePG, Zalando Postgres Operator o Crunchy Data) oculta la complejidad interna de orquestar bases de datos relacionales distribuidas. Esto provoca que los estudiantes y administradores no comprendan las primitivas fundamentales de Kubernetes (StatefulSets, Headless Services, volumeClaimTemplates e initContainers), ni las razones fundamentales de por qué Kubernetes por sí solo no puede realizar un failover automático seguro con elección de líder y prevención de split-brain.

Esta propuesta crea una guía técnica paso a paso completa, práctica y visual para desplegar un cluster primario/réplica de PostgreSQL 16 montado manualmente sobre Kubernetes, demostrando de forma empírica las limitaciones de la plataforma nativa y la justificación arquitectónica de los Operadores. Además, incluye la automatización para compilar la guía a un documento PDF profesional con el mismo estándar visual que `k8s-installation-steps.pdf`.

## What Changes

- **Guía Técnica Exhaustiva en Markdown**: Creación de `docs/k8s-postgresql-statefulset.md` con explicación teórica, diagramas de arquitectura en texto/ASCII, comandos paso a paso con salida de consola esperada, pruebas de validación de replicación y laboratorio de simulación de fallos (caos / split-brain).
- **Manifiestos Kubernetes Nativos**:
  - Aprovisionamiento dinámico local (`local-path-provisioner`) para resolver la persistencia en clusters on-premise/Vagrant.
  - Secret y ConfigMap para inicialización, variables de entorno y configuración de replicación WAL (`pg_hba.conf`).
  - Headless Service (`clusterIP: None`) para resolución DNS individual por Pod.
  - Servicios de acceso para aplicaciones: Service Read-Write (`postgresql-primary`) y Service Read-Only (`postgresql-replicas`).
  - StatefulSet con 2 réplicas (`postgres:16-alpine`), `volumeClaimTemplates` y un `initContainer` que gestiona el bootstrap condicional según ordinal (`pg_basebackup`, `standby.signal` y `primary_conninfo`).
- **Laboratorio Demostrativo de Failover y Split-Brain**: Escenario guiado donde se simula la caída de `postgresql-0`, se observa la ausencia de promoción automática en `postgresql-1`, y se demuestra el riesgo de corrupción por doble primario si se promueve manualmente sin fencing.
- **Pipeline de Compilación HTML/PDF**: Creación de `scripts/generate_html_pdf.py` para generar una maquetación HTML/CSS imprimible (A4, Inter, JetBrains Mono, bloques de código, callouts) y compilación desatendida a PDF con marcadores jerárquicos vía Google Chrome headless.

## Capabilities

### New Capabilities

- `postgresql-statefulset-cluster`: Especificación del despliegue manual de un cluster PostgreSQL 16 (Primary/Replica) utilizando exclusivamente primitivas nativas de Kubernetes (StatefulSet, Headless Service, dynamic PVCs e initContainers).
- `operator-pedagogical-contrast`: Especificación del escenario de pruebas de fallo y comparación analítica que contrasta el comportamiento manual frente a la gestión autónoma provista por Operadores de Kubernetes.
- `technical-guide-pdf-pipeline`: Especificación del pipeline de documentación técnica y compilación a PDF profesional estructurado con soporte de marcadores e identidad visual homogénea.

### Modified Capabilities

*(Ninguna, proyecto nuevo sin especificaciones previas)*

## Impact

- **Código y Manifiestos**: Nuevos archivos en `docs/`, `manifests/` y `scripts/`.
- **Compatibilidad**: Compatible con clusters Kubeadm v1.31 (incluyendo el cluster de dos nodos definido en `poc-kubeadm-ubuntu`), Minikube, Kind o K3s.
- **Dependencias de generación**: Python 3, `pypdf` y Google Chrome o Chromium para la compilación a PDF.
