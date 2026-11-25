# Design: Guía Práctica de PostgreSQL en Kubernetes con StatefulSet y Contraste de Operadores

## Context

Los entornos de formación en Kubernetes a menudo presentan soluciones basadas en Operadores (CRDs y controladores) como la forma estándar de desplegar bases de datos. Aunque esta es la mejor práctica para producción, genera una barrera pedagógica donde los ingenieros no comprenden cómo interactúan los StatefulSets, los servicios Headless y el almacenamiento dinámico, ni por qué el failover de una base de datos distribuida requiere mecanismos de consenso y cercado (fencing) que Kubernetes por sí solo no proporciona.

El proyecto de referencia previo (`poc-kubeadm-ubuntu`) proporciona un clúster Kubeadm de dos máquinas virtuales (Master y Worker 1) con Ubuntu 22.04 y Flannel CNI, pero carece de un aprovisionador dinámico de almacenamiento (`StorageClass`) por defecto.

## Goals / Non-Goals

**Goals:**
- Proporcionar una guía didáctica exhaustiva, visual e hiper-detallada que enseñe paso a paso la mecánica interna de Kubernetes desplegando un clúster PostgreSQL 16 (Primary/Replica) sin operadores.
- Explicar la teoría y práctica de:
  - `StatefulSets`: Identidad estable basada en ordinales e inicio secuencial.
  - `Headless Services`: Registros DNS FQDN por réplica para la comunicación de replicación punto a punto.
  - `volumeClaimTemplates`: Aislamiento dinámico de volumen y persistencia desacoplada del ciclo de vida del Pod.
  - `initContainers`: Lógica condicional de bootstrap según ordinal (`pg_basebackup`, `standby.signal`, `primary_conninfo`).
- Demostrar de forma reproducible por qué el failover no es automático en Kubernetes puro y los riesgos de split-brain ante intervenciones manuales desordenadas.
- Construir una matriz comparativa profunda entre la aproximación manual y los Operadores de producción (CloudNativePG, Zalando, Crunchy Data).
- Ofrecer un script Python que compile la guía a HTML editorial y genere un archivo PDF profesional (`k8s-postgresql-statefulset.pdf`) idéntico en calidad visual a `k8s-installation-steps.pdf`.

**Non-Goals:**
- No se busca crear un sistema de producción para entornos reales (en producción se debe utilizar un Operador o servicio gestionado).
- No se implementará un daemon de consenso propio en bash (el objetivo es justamente mostrar que implementar failover a mano es propenso a fallos).
- No se cubre integración de backups externos en la nube (S3/GCS) en este laboratorio, manteniéndolo autocontenido en el clúster local.

## Decisions

### Decisión 1: Imagen base PostgreSQL 16 Alpine (`postgres:16-alpine`)
- **Elección**: Utilizar `postgres:16-alpine`.
- **Razón**: Es una imagen extremadamente ligera (~140 MB y ~35 MB de consumo en reposo), ideal para máquinas virtuales locales con recursos ajustados (2 GB de RAM en el nodo worker). Además, utiliza la arquitectura de replicación moderna de PostgreSQL 12+ (`standby.signal` y configuración dinámica de conexión en `postgresql.auto.conf`), evitando la sintaxis deprecada de `recovery.conf`.
- **Alternativas consideradas**: `postgres:16-bookworm` (más pesada, ~400 MB) o imágenes personalizadas con scripts precompilados (restaría valor didáctico al ocultar la lógica de inicialización).

### Decisión 2: Aprovisionador de almacenamiento dinámico `local-path-provisioner`
- **Elección**: Incluir en la guía la instalación de Rancher `local-path-provisioner` al inicio del laboratorio.
- **Razón**: El clúster Kubeadm de la guía previa no tiene StorageClass dinámica. `local-path-provisioner` se instala con un único comando oficial (`kubectl apply`), mapea directorios locales en `/opt/local-path-provisioner` y permite que los `volumeClaimTemplates` de los StatefulSets creen PVs dinámicos de forma instantánea.
- **Alternativas consideradas**: Crear PVs estáticos a mano (resta dinamismo y no enseña cómo funcionan las StorageClasses) o desplegar NFS/Rook-Ceph (demasiada sobrecarga para un laboratorio didáctico).

### Decisión 3: Arquitectura del `initContainer` para inicialización y replicación
- **Elección**: Emplear la misma imagen `postgres:16-alpine` como `initContainer` con un script bash montado mediante ConfigMap o comando inline.
- **Lógica**:
  1. Extraer ordinal: `ORDINAL=${HOSTNAME##*-}`.
  2. Si `ORDINAL == 0`:
     - Asegurar permisos `chown 999:999` en el volumen montado.
     - Si el directorio de datos está vacío, permitir que el entrypoint estándar de PostgreSQL ejecute `initdb`.
  3. Si `ORDINAL > 0`:
     - Hacer polling con `pg_isready -h postgresql-0.postgresql-headless -p 5432` hasta que el primario esté disponible.
     - Si el directorio de datos está vacío, ejecutar:
       ```bash
       pg_basebackup -h postgresql-0.postgresql-headless -D "$PGDATA" -U replicator -v -P -R -X stream
       ```
     - El parámetro `-R` escribe automáticamente `standby.signal` y la directiva `primary_conninfo` en `postgresql.auto.conf`.
- **Razón**: Permite a los alumnos ver exactamente cómo interactúan las herramientas estándar de PostgreSQL con la topología de Kubernetes.

### Decisión 4: Estrategia de exposición con Servicios Dobles
- **Elección**: Desplegar tres servicios de Kubernetes:
  1. `postgresql-headless` (`clusterIP: None`): Identidad DNS fija por pod (`postgresql-0`, `postgresql-1`).
  2. `postgresql-primary` (`ClusterIP` apuntando a `statefulset.kubernetes.io/pod-name: postgresql-0`): Punto de entrada para operaciones de escritura (Read-Write).
  3. `postgresql-replicas` (`ClusterIP` apuntando a `app: postgresql`): Punto de entrada para operaciones de solo lectura (Read-Only).
- **Razón**: Demuestra claramente cómo desacoplar el enrutamiento de red según el tipo de carga de trabajo.

### Decisión 5: Diseño del laboratorio de simulación de fallos (Caos y Split-Brain)
- **Elección**: Estructurar un experimento guiado en tres fases:
  1. Comprobación de salud: Inserción en `postgresql-0` y consulta inmediata en `postgresql-1` vía `psql` para verificar el WAL streaming y `pg_stat_replication`.
  2. Caída del primario: Detener `postgresql-0` forzosamente (`kubectl delete pod postgresql-0 --now`). Observar que las escrituras se bloquean y que `postgresql-1` **no se promueve solo**.
  3. Demostración de split-brain: Forzar manualmente la promoción de `postgresql-1` (`pg_ctl promote`), escribir un dato, reactivar `postgresql-0` y observar la divergencia irreversible de datos.
- **Razón**: Es la demostración empírica definitiva que fundamenta por qué en producción se requiere un Operador con quórum y fencing.

### Decisión 6: Pipeline de generación documental con Python y Chrome Headless
- **Elección**: Replicar la arquitectura de `poc-kubeadm-ubuntu/scripts/generate_html_pdf.py`.
- **Razón**: Permite generar un entregable PDF con diseño editorial idéntico (fuente Inter, JetBrains Mono, márgenes A4, llamadas de atención, marcadores laterales jerárquicos) de forma automatizada y versionable en Git.

## Risks / Trade-offs

- **[Riesgo] Taint en nodo Master en clúster de 2 nodos** → **Mitigación**: Explicar en la guía cómo retirar el taint del nodo master (`kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-`) para poder ubicar una réplica en cada máquina y permitir la prueba de fallo de nodo completa, o bien documentar la simulación a nivel de Pod en un único worker.
- **[Riesgo] Errores de autenticación en `pg_basebackup`** → **Mitigación**: Proveer un Secret claro con las credenciales de `replicator` y configurar `pg_hba.conf` mediante un ConfigMap para autorizar conexiones de replicación desde la subred del clúster (`10.244.0.0/16`).
- **[Riesgo] Ausencia de Chromium/Chrome en la máquina del usuario** → **Mitigación**: El script de Python detectará automáticamente navegadores compatibles (Google Chrome, Chromium) e imprimirá instrucciones claras si falta el binario o la librería `pypdf`.

## Migration Plan

1. Fase de Autoría de Manifiestos: Crear y validar los archivos YAML en `manifests/` (storage, secrets, configmaps, services, statefulset).
2. Fase de Redacción de la Guía: Desarrollar `docs/k8s-postgresql-statefulset.md` con todos los capítulos, diagramas y capturas de terminal.
3. Fase de Pipeline Editorial: Desarrollar `scripts/generate_html_pdf.py` y validar la compilación a `k8s-postgresql-statefulset.pdf`.
4. Fase de Verificación: Validar la consistencia y reproducibilidad de los comandos en un clúster Kubeadm.
