# Tasks: Implementación de la Guía Práctica de PostgreSQL en Kubernetes con StatefulSet

## 1. Manifiestos de Almacenamiento y Configuración Base

- [x] 1.1 Crear manifiesto de StorageClass dinámico local (`manifests/00-local-path-storage.yaml`) para aprovisionamiento automático y verificar sintaxis con `kubectl --dry-run=client`.
- [x] 1.2 Crear Secret para superusuario y usuario de replicación (`manifests/01-postgres-secret.yaml`) y verificar sintaxis y codificación.
- [x] 1.3 Crear ConfigMap con scripts de inicialización y configuración de replicación `pg_hba.conf` (`manifests/02-postgres-configmap.yaml`) y verificar sintaxis.

## 2. Manifiestos de Red y StatefulSet de PostgreSQL

- [x] 2.1 Crear manifiestos de red con Headless Service (`postgresql-headless`) y servicios de acceso RW y RO (`manifests/03-postgres-services.yaml`) y verificar selectores.
- [x] 2.2 Crear manifiesto del StatefulSet con `initContainer` de replicación condicional, `volumeClaimTemplates` y PostgreSQL 16 Alpine (`manifests/04-postgres-statefulset.yaml`) y verificar configuración de variables de entorno y montajes.

## 3. Redacción de la Guía Didáctica Exhaustiva en Markdown

- [x] 3.1 Redactar Capítulos 1 y 2 en `docs/k8s-postgresql-statefulset.md`: Arquitectura, conceptos StatefulSet vs Deployment, Headless Services, almacenamiento y preparación del clúster.
- [x] 3.2 Redactar Capítulos 3, 4 y 5 en `docs/k8s-postgresql-statefulset.md`: Despliegue guiado paso a paso, anatomía detallada del initContainer y verificación interactiva de replicación streaming con `psql`.
- [x] 3.3 Redactar Capítulos 6, 7 y 8 en `docs/k8s-postgresql-statefulset.md`: Experimento de caos (caída de nodo primario), demostración del peligro de split-brain por promoción manual, matriz comparativa con Operadores de producción (CloudNativePG, Zalando, Crunchy) y troubleshooting.

## 4. Pipeline de Compilación HTML/PDF y Validación Visual

- [x] 4.1 Desarrollar script `scripts/generate_html_pdf.py` con plantilla HTML5, maquetación CSS `@page` (A4, fuentes Inter/JetBrains Mono, badges, callouts, tablas) e integración de marcadores jerárquicos PDF con `pypdf`.
- [x] 4.2 Ejecutar el pipeline de compilación a través de Chrome headless y verificar la generación correcta de `k8s-postgresql-statefulset.pdf` con su tabla de contenido y marcadores navegables.
