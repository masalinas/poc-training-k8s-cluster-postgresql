# PoC Training: PostgreSQL en Kubernetes con StatefulSets

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.31-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![StorageClass](https://img.shields.io/badge/StorageClass-local--path-0891b2)](#2-preparación-del-almacenamiento-dinámico-storageclass)
[![Curso](https://img.shields.io/badge/Curso-Cap%C3%ADtulo_2-0969da)](#)
[![PDF](https://img.shields.io/badge/PDF-k8s--postgresql--statefulset.pdf-dc2626)](./k8s-postgresql-statefulset.pdf)

> **Laboratorio Práctico y Guía Oficial:** Despliegue manual de un clúster PostgreSQL 16 (Primario/Réplica), anatomía interna de red en Kubernetes y demostración empírica de las limitaciones de la plataforma sin Operadores.

---

## 🎯 Objetivo Pedagógico

El propósito fundamental de este repositorio es **desmitificar la gestión de estado en Kubernetes**. 

En la mayoría de cursos y tutoriales modernos se recurre directamente a herramientas de alto nivel como los **Operadores de Kubernetes** (CloudNativePG, Zalando Postgres Operator o Crunchy Data). Si bien esta es la mejor práctica indiscutible para entornos productivos, introduce una barrera didáctica que oculta qué problemas resuelve realmente la plataforma y cómo se comportan las primitivas nativas.

A través de este proyecto aprenderás:
1. **El Contrato de Estado (`StatefulSets`):** Identidad determinista por ordinal (`postgresql-0`, `postgresql-1`), despliegue secuencial estricto (`OrderedReady`) y adherencia inmutable de almacenamiento tras reinicios.
2. **Red Dirigida (`Headless Services`):** Por qué un servicio balanceado tradicional destruye la replicación y cómo `clusterIP: None` provee nombres DNS individuales en CoreDNS (`<pod>.<servicio>.<ns>.svc.cluster.local`).
3. **Persistencia Dinámica Aislada (`volumeClaimTemplates`):** Creación y enlace automático de discos independientes por cada réplica, integrando un aprovisionador dinámico local (`local-path-provisioner`).
4. **Bootstrap Condicional (`initContainers`):** Cómo un único contenedor de inicialización evalúa su ordinal en tiempo de arranque, sincroniza el inicio de réplicas con `pg_isready` y clona físicamente el catálogo con `pg_basebackup -R` (generando `standby.signal` y `primary_conninfo`).
5. **La Contrapartida (El Dilema del Failover y el Peligro del Split-Brain):** Demostración práctica de que **Kubernetes puro no realiza failover automático de bases de datos relacionales**. Comprobaremos cómo la caída del primario bloquea las escrituras y cómo una promoción manual no coordinada desemboca en corrupción catastrófica por *Split-Brain*, evidenciando exactamente por qué existen los Operadores.

---

## 🏗️ Diagrama de Arquitectura y Topología

```
                                  APLICACIONES / CLIENTES
                                             |
                   +-------------------------+-------------------------+
                   |                                                   |
         Transacciones de Escritura                              Consultas de Lectura
         (INSERT, UPDATE, DELETE)                                (SELECT)
                   |                                                   |
                   v                                                   v
       +-----------------------+                           +-----------------------+
       |  postgresql-primary   |                           |  postgresql-replicas  |
       |  (ClusterIP Service)  |                           |  (ClusterIP Service)  |
       +-----------------------+                           +-----------------------+
                   |                                                   |
  Selector: pod-name=postgresql-0                             Selector: app=postgresql
                   |                                                   |
                   v                                                   v
+-------------------------------------------------------------------------------------+
| STATEFULSET: postgresql (replicas: 2, image: postgres:16-alpine)                    |
|                                                                                     |
| [ Pod: postgresql-0 ] (PRIMARY / RW)             [ Pod: postgresql-1 ] (REPLICA/RO) |
| FQDN: postgresql-0.postgresql-headless           FQDN: postgresql-1.postgresql-hl   |
|                                                                                     |
| +----------------------------------+             +--------------------------------+ |
| | initContainer: init-postgres     |             | initContainer: init-postgres   | |
| | - Ordinal: 0                     |             | - Ordinal: 1                   | |
| | - Permisos chmod 700             |             | - Polling pg_isready a Pod 0   | |
| | - Omite clonación (initdb estándar)            | - Clona datos: pg_basebackup   | |
| +----------------------------------+             | - Genera standby.signal        | |
|                 |                                +--------------------------------+ |
|                 v                                                 |                 |
| +----------------------------------+                              v                 |
| | Container: postgresql            |             +--------------------------------+ |
| | - Modo Read-Write                |             | Container: postgresql          | |
| | - Crea rol 'replicator'          |<=== WAL ====| - Modo Hot Standby (Read-Only) | |
| | - Emite WALs por streaming       |  Streaming  | - Consume WALs en tiempo real  | |
| +----------------------------------+             +--------------------------------+ |
+-----------------+-------------------------------------------------+-----------------+
                  |                                                 |
    volumeClaimTemplates: 1Gi                         volumeClaimTemplates: 1Gi
                  |                                                 |
                  v                                                 v
     +-------------------------+                       +-------------------------+
     | PVC:                    |                       | PVC:                    |
     | postgresql-data-        |                       | postgresql-data-        |
     | postgresql-0            |                       | postgresql-1            |
     +-------------------------+                       +-------------------------+
                  |                                                 |
                  v                                                 v
     +-------------------------+                       +-------------------------+
     | PV: (Storage: local-path|                       | PV: (Storage: local-path|
     | /opt/local-path-...     |                       | /opt/local-path-...     |
     +-------------------------+                       +-------------------------+
```

---

## 📂 Estructura del Repositorio

```text
├── README.md                                # Este documento
├── k8s-postgresql-statefulset.pdf          # Entregable PDF oficial maquetado con marcadores
├── docs/
│   ├── k8s-postgresql-statefulset.md       # Guía técnica completa (9 capítulos didácticos)
│   └── k8s-postgresql-statefulset.html     # Maquetación HTML5 intermedia para impresión
├── manifests/
│   ├── 00-local-path-storage.yaml          # Rancher local-path-provisioner (StorageClass)
│   ├── 01-postgres-secret.yaml             # Secret con credenciales de superusuario y replicador
│   ├── 02-postgres-configmap.yaml          # ConfigMap con script SQL de réplica y pg_hba.conf
│   ├── 03-postgres-services.yaml           # Headless Service, postgresql-primary y postgresql-replicas
│   └── 04-postgres-statefulset.yaml        # StatefulSet PostgreSQL 16 Alpine con initContainer
├── scripts/
│   └── generate_html_pdf.py                # Pipeline Python para compilar la guía a PDF profesional
└── openspec/                               # Especificaciones duraderas de arquitectura (OpenSpec)
```

---

## 🚀 Despliegue Rápido Paso a Paso

### 1. Requisitos Previos
* Un clúster de Kubernetes en ejecución (v1.28 a v1.31), como el clúster Kubeadm de Ubuntu (`poc-kubeadm-ubuntu`), Minikube, Kind o K3s.
* `kubectl` configurado con permisos de administrador en el clúster.

### 2. Aprovisionamiento de Almacenamiento Dinámico
Si ejecutas en un clúster Kubeadm o bare-metal sin StorageClass por defecto, despliega el aprovisionador local:
```bash
kubectl apply -f manifests/00-local-path-storage.yaml
```
Verifica que la clase `local-path (default)` esté disponible:
```bash
kubectl get storageclass
```

*(Opcional para clústeres de 2 nodos con Master y Worker)*: Retira el taint del nodo master para permitir que las réplicas se repartan físicamente entre ambos nodos:
```bash
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-
```

### 3. Configuración y Credenciales
Aplica el Secret con las contraseñas y el ConfigMap con la configuración de replicación:
```bash
kubectl apply -f manifests/01-postgres-secret.yaml
kubectl apply -f manifests/02-postgres-configmap.yaml
```

### 4. Capa de Red (Headless y Servicios de Acceso)
Despliega el servicio Headless y los puntos de acceso para la aplicación:
```bash
kubectl apply -f manifests/03-postgres-services.yaml
```

### 5. Despliegue del StatefulSet
Lanza las instancias de PostgreSQL:
```bash
kubectl apply -f manifests/04-postgres-statefulset.yaml
```

Sigue en tiempo real la creación secuencial y el bootstrap de las réplicas:
```bash
kubectl get pods -l app=postgresql -w
```

---

## 🧪 Verificación de la Replicación Streaming

### 1. Insertar Datos en el Primario (`postgresql-0`)
Conéctate por terminal interactiva al pod maestro:
```bash
kubectl exec -it postgresql-0 -- psql -U postgres
```
Ejecuta la creación de una tabla y registros de prueba:
```sql
CREATE TABLE cursos_k8s (id SERIAL PRIMARY KEY, nombre TEXT);
INSERT INTO cursos_k8s (nombre) VALUES ('Kubernetes StatefulSets'), ('Arquitectura de Operadores');
SELECT * FROM cursos_k8s;
```

Inspecciona el catálogo interno para verificar el canal de streaming WAL:
```sql
SELECT application_name, client_addr, state, sync_state FROM pg_stat_replication;
```
*(Debe devolver una fila con `application_name = walreceiver` y `state = streaming`)*. Sal con `\q`.

### 2. Comprobar Lectura y Bloqueo en la Réplica (`postgresql-1`)
Conéctate a la réplica para constatar la replicación y su modo de solo lectura:
```bash
kubectl exec -it postgresql-1 -- psql -U postgres -c "SELECT * FROM cursos_k8s;"
```
Intenta realizar una escritura:
```bash
kubectl exec -it postgresql-1 -- psql -U postgres -c "INSERT INTO cursos_k8s (nombre) VALUES ('Intruso');"
```
*(PostgreSQL devolverá inmediatamente el error: `ERROR: cannot execute INSERT in a read-only transaction`)*.

---

## 💥 El Experimento del Caos: Demostrando las Limitaciones de K8s

Simula la caída forzosa del nodo primario:
```bash
kubectl delete pod postgresql-0 --now
```

1. **Intento de escritura:** Al enviar peticiones a `postgresql-primary`, la conexión falla con `Connection refused`.
2. **Estado de la réplica:** Ejecuta en `postgresql-1`:
   ```bash
   kubectl exec -it postgresql-1 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
   ```
   Devuelve `t` (*true*). **Kubernetes no promueve a la réplica automáticamente**.
3. **Peligro de Split-Brain:** Si un administrador ejecuta manualmente `SELECT pg_promote();` en la réplica, esta empezará a aceptar escrituras. Cuando Kubernetes logre resucitar a `postgresql-0` con su volumen original, **tendrás dos primarios escribiendo a la vez en la misma red**, corrompiendo irreversiblemente el historial de transacciones.

---

## ⚖️ StatefulSet Manual vs. Operadores de Producción

| Funcionalidad | StatefulSet Manual (Este Lab) | Operadores (CloudNativePG / Zalando / Crunchy) |
| :--- | :--- | :--- |
| **Failover Automático** | ❌ Inexistente (requiere intervención humana) | ✅ Sub-segundo por elección de líder y consenso |
| **Prevención de Split-Brain** | ❌ Ninguna (alto riesgo de corrupción) | ✅ Garantizado mediante *Fencing* forzado |
| **Backups continuos y PITR** | ⚠️ Scripts manuales y frágiles | ✅ Integración nativa con Barman / WAL-G a S3/GCS |
| **Actualizaciones sin Caída** | ❌ Downtime durante rolling updates | ✅ Switchover planificado previo con 0 downtime |
| **Connection Pooling** | ⚠️ Despliegue manual de PgBouncer | ✅ PgBouncer gestionado e integrado nativamente |

---

## 📄 Compilación del Entregable PDF

El repositorio incluye un script en Python que compila la guía técnica completa a un documento PDF profesional con marcadores laterales jerárquicos:

```bash
# Requisitos: Python 3, pypdf y Google Chrome / Chromium instalado
python3 scripts/generate_html_pdf.py
```

El script generará el archivo `k8s-postgresql-statefulset.pdf` en la raíz del proyecto.
