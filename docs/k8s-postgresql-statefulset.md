# Guía Práctica de PostgreSQL en Kubernetes con StatefulSets
## Despliegue Manual de Alta Disponibilidad Primario-Réplica, Anatomía de Red y el Dilema del Failover sin Operadores

**Autor:** Equipo Docente de Kubernetes y Arquitectura de Datos  
**Documento Oficial del Curso:** Capítulo 2 - Gestión de Cargas con Estado y Bases de Datos  
**Entregable:** `k8s-postgresql-statefulset.pdf`

---

## Tabla de Contenidos

1. [Fundamentos y Arquitectura de Estado en Kubernetes](#1-fundamentos-y-arquitectura-de-estado-en-kubernetes)
   - [1.1 El Contrato de Estado: Deployment vs StatefulSet](#11-el-contrato-de-estado-deployment-vs-statefulset)
   - [1.2 Diagrama de Topología y Flujo de Replicación](#12-diagrama-de-topología-y-flujo-de-replicación)
   - [1.3 Matriz de Componentes del Clúster](#13-matriz-de-componentes-del-clúster)
2. [Preparación del Almacenamiento Dinámico (StorageClass)](#2-preparación-del-almacenamiento-dinámico-storageclass)
   - [2.1 El Problema del Almacenamiento en Clústeres Kubeadm / Bare-Metal](#21-el-problema-del-almacenamiento-en-clústeres-kubeadm--bare-metal)
   - [2.2 Despliegue de Rancher `local-path-provisioner`](#22-despliegue-de-rancher-local-path-provisioner)
   - [2.3 Verificación de la StorageClass por Defecto](#23-verificación-de-la-storageclass-por-defecto)
   - [2.4 Ajuste de Tolerancias del Nodo Master (Control Plane)](#24-ajuste-de-tolerancias-del-nodo-master-control-plane)
3. [Configuración Base y Gestión de Credenciales](#3-configuración-base-y-gestión-de-credenciales)
   - [3.1 Secret de Autenticación para Superusuario y Replicador](#31-secret-de-autenticación-para-superusuario-y-replicador)
   - [3.2 ConfigMap con Scripts de Inicialización y `pg_hba.conf`](#32-configmap-con-scripts-de-inicialización-y-pg_hbaconf)
4. [La Capa de Red: Headless Service y Servicios de Acceso](#4-la-capa-de-red-headless-service-y-servicios-de-acceso)
   - [4.1 Por qué un Servicio Estándar no Sirve: El Headless Service (`clusterIP: None`)](#41-por-qué-un-servicio-estándar-no-sirve-el-headless-service-clusterip-none)
   - [4.2 Resolución DNS Individual por Pod en CoreDNS](#42-resolución-dns-individual-por-pod-en-coredns)
   - [4.3 Separación de Tráfico: Servicio Primario (RW) y Servicio Réplicas (RO)](#43-separación-de-tráfico-servicio-primario-rw-y-servicio-réplicas-ro)
5. [Despliegue del StatefulSet y Mecánica del InitContainer](#5-despliegue-del-statefulset-y-mecánica-del-initcontainer)
   - [5.1 Anatomía del StatefulSet: `replicas`, `serviceName` y `OrderedReady`](#51-anatomía-del-statefulset-replicas-servicename-y-orderedready)
   - [5.2 Aislamiento de Volúmenes con `volumeClaimTemplates`](#52-aislamiento-de-volúmenes-con-volumeclaimtemplates)
   - [5.3 Desmontando la "Magia": Lógica Interna del InitContainer](#53-desmontando-la-magia-lógica-interna-del-initcontainer)
   - [5.4 Despliegue Secuencial y Seguimiento del Arranque](#54-despliegue-secuencial-y-seguimiento-del-arranque)
6. [Verificación Práctica de la Replicación Streaming](#6-verificación-práctica-de-la-replicación-streaming)
   - [6.1 Acceso Interactivo con `psql` al Pod Primario (`postgresql-0`)](#61-acceso-interactivo-con-psql-al-pod-primario-postgresql-0)
   - [6.2 Creación de Esquema y Datos de Prueba](#62-creación-de-esquema-y-datos-de-prueba)
   - [6.3 Inspección del Catálogo Interno de Replicación (`pg_stat_replication`)](#63-inspección-del-catálogo-interno-de-replicación-pg_stat_replication)
   - [6.4 Validación de Lectura y Protección contra Escritura en `postgresql-1`](#64-validación-de-lectura-y-protección-contra-escritura-en-postgresql-1)
7. [El Experimento del Caos: Limitaciones de Kubernetes Puro](#7-el-experimento-del-caos-limitaciones-de-kubernetes-puro)
   - [7.1 Simulación de Caída del Nodo Primario (`postgresql-0`)](#71-simulación-de-caída-del-nodo-primario-postgresql-0)
   - [7.2 El Comportamiento Real: Bloqueo de Escrituras y Réplica Estancada](#72-el-comportamiento-real-bloqueo-de-escrituras-y-réplica-estancada)
   - [7.3 El Peligro del Split-Brain por Promoción Manual no Coordinada](#73-el-peligro-del-split-brain-por-promoción-manual-no-coordinada)
   - [7.4 Por qué las Primitivas Nativas no Pueden Resolver el Failover](#74-por-qué-las-primitivas-nativas-no-pueden-resolver-el-failover)
8. [La Brecha hacia la Producción: Por Qué Usamos Operadores](#8-la-brecha-hacia-la-producción-por-qué-usamos-operadores)
   - [8.1 Matriz Comparativa: StatefulSet Manual vs. Operadores de Producción](#81-matriz-comparativa-statefulset-manual-vs-operadores-de-producción)
   - [8.2 Capacidades Esenciales de un Operador (CloudNativePG, Zalando, Crunchy)](#82-capacidades-esenciales-de-un-operador-cloudnativepg-zalando-crunchy)
9. [Resolución de Problemas Frecuentes (Troubleshooting)](#9-resolución-de-problemas-frecuentes-troubleshooting)
   - [9.1 Error 1: PVCs en Estado `Pending` por Ausencia de StorageClass](#91-error-1-pvcs-en-estado-pending-por-ausencia-de-storageclass)
   - [9.2 Error 2: `initdb: directory exists but is not empty` (`lost+found`)](#92-error-2-initdb-directory-exists-but-is-not-empty-lostfound)
   - [9.3 Error 3: Fallo en `pg_basebackup`: Autenticación Rechazada en `pg_hba.conf`](#93-error-3-fallo-en-pg_basebackup-autenticación-rechazada-en-pg_hbaconf)
   - [9.4 Error 4: Permisos Denegados en el Directorio de Datos (`chown 999:999`)](#94-error-4-permisos-denegados-en-el-directorio-de-datos-chown-999999)

---

## 1. Fundamentos y Arquitectura de Estado en Kubernetes

### 1.1 El Contrato de Estado: Deployment vs StatefulSet

En la computación nativa en la nube, la gran mayoría de las aplicaciones web y APIs son **apátridas** (*stateless*). Para este tipo de cargas de trabajo, el controlador por excelencia en Kubernetes es el `Deployment`. Un `Deployment` asume que:
* Todos los Pods son idénticos e intercambiables entre sí.
* Los Pods reciben nombres aleatorios con hashes efímeros (por ejemplo, `api-78f997df7-kq9pz`).
* El arranque, reinicio y escalado de Pods se ejecuta en paralelo sin un orden predeterminado.
* Si un Pod es destruido, el nuevo Pod resultante no tiene ninguna relación con el almacenamiento ni la identidad de red del Pod original.

Sin embargo, las bases de datos relacionales como **PostgreSQL** son sistemas con **fuerte acoplamiento de estado** (*stateful*). Cada nodo de una base de datos distribuida cumple un rol muy específico dentro del clúster (nodo Primario para lecturas y escrituras, o réplicas en espera *Standby* para solo lectura).

```
+-----------------------------------------------------------------------------------------+
|                  COMPARATIVA DE IDENTIDAD: DEPLOYMENT VS STATEFULSET                    |
+-----------------------------------------------------------------------------------------+
|  Característica         | Deployment (Stateless)         | StatefulSet (Stateful)       |
+-------------------------+--------------------------------+------------------------------+
|  Identidad de Red       | Efímera y aleatoria            | Determinista y Fija          |
|                         | (api-6bf9d7c8-4z9q)            | (postgresql-0, postgresql-1) |
+-------------------------+--------------------------------+------------------------------+
|  Orden de Despliegue    | Concurrente / Arbitrario       | Secuencial y Estricto        |
|                         | (Todos a la vez)               | (0 -> 1 -> 2 ...)            |
+-------------------------+--------------------------------+------------------------------+
|  Almacenamiento (PVC)   | Compartido o Efímero           | 1 PVC dedicado por Réplica   |
|                         | (Todos leen del mismo disco)   | (data-postgresql-0, ...)     |
+-------------------------+--------------------------------+------------------------------+
|  Enlace tras Reinicio   | Nuevo Pod = Volumen nuevo      | Nuevo Pod = Se re-vincula al |
|                         | o no determinista              | MISMO volumen persistente    |
+-------------------------+--------------------------------+------------------------------+
```

El objeto **`StatefulSet`** garantiza tres invariantes fundamentales:
1. **Identidad de Red Estable y Única:** Cada Pod recibe un ordinal numérico entero secuencial `[0, 1, ..., N-1]`. El primer Pod siempre será `postgresql-0`, el segundo `postgresql-1`, etc.
2. **Despliegue y Terminación Secuencial:** Por defecto (`podManagementPolicy: OrderedReady`), Kubernetes no arranca el Pod `postgresql-1` hasta que `postgresql-0` haya superado sus *Readiness Probes* y esté en estado `Running` y `Ready`.
3. **Almacenamiento Persistente Adherente:** Cada réplica dispone de su propio volumen dinámico creado a través de `volumeClaimTemplates`. Si el nodo físico donde corre `postgresql-0` se reinicia, el nuevo Pod `postgresql-0` volverá a reclamar exactamente el PVC `postgresql-data-postgresql-0`.

---

### 1.2 Diagrama de Topología y Flujo de Replicación

A continuación se detalla la arquitectura que implementaremos paso a paso en este laboratorio:

```
                                  APLICACIONES EXTERNAS / CLIENTES
                                                 |
                       +-------------------------+-------------------------+
                       |                                                   |
             Transacciones de Escritura                              Consultas de Solo Lectura
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
+-----------------------------------------------------------------------------------------+
|  STATEFULSET: postgresql (replicas: 2)                                                  |
|                                                                                         |
|  +-------------------------------------+       +-------------------------------------+  |
|  | Pod: postgresql-0 (PRIMARY / RW)    |       | Pod: postgresql-1 (REPLICA / RO)    |  |
|  | FQDN:                               |       | FQDN:                               |  |
|  | postgresql-0.postgresql-headless    |       | postgresql-1.postgresql-headless    |  |
|  |                                     |       |                                     |  |
|  |  +-------------------------------+  |       |  +-------------------------------+  |  |
|  |  | initContainer: init-postgres |  |       |  | initContainer: init-postgres |  |  |
|  |  | - Detecta Ordinal: 0          |  |       |  | - Detecta Ordinal: 1          |  |  |
|  |  | - Prepara permisos chown 700  |  |       |  | - Polling pg_isready a Pod 0  |  |  |
|  |  | - Omite clonación (es máster) |  |       |  | - Clona datos: pg_basebackup  |  |  |
|  |  +-------------------------------+  |       |  | - Genera standby.signal       |  |  |
|  |                 |                   |       |  | - Escribe primary_conninfo    |  |  |
|  |                 v                   |       |  +-------------------------------+  |  |
|  |  +-------------------------------+  |       |                 |                   |  |
|  |  | Container: postgresql         |  |       |                 v                   |  |
|  |  | - Modo Read-Write             |  |       |  +-------------------------------+  |  |
|  |  | - initdb crea 'replicator'    |  |<======|==| Container: postgresql         |  |  |
|  |  | - Emite WALs por streaming    |  |  WAL  |  | - Modo Hot-Standby (RO)       |  |  |
|  |  +-------------------------------+  | Stream|  | - Aplica WALs en tiempo real  |  |  |
|  |                 |                   |       |  +-------------------------------+  |  |
|  +-----------------+-------------------+       +-----------------+-------------------+  |
|                    |                                             |                      |
+--------------------+---------------------------------------------+----------------------+
                     |                                             |
       volumeClaimTemplates: 1Gi                     volumeClaimTemplates: 1Gi
                     |                                             |
                     v                                             v
        +-------------------------+                   +-------------------------+
        | PVC:                    |                   | PVC:                    |
        | postgresql-data-        |                   | postgresql-data-        |
        | postgresql-0            |                   | postgresql-1            |
        +-------------------------+                   +-------------------------+
                     |                                             |
                     v                                             v
        +-------------------------+                   +-------------------------+
        | PV: pvc-1a87b... (Ext4) |                   | PV: pvc-4b92c... (Ext4) |
        | /opt/local-path-...     |                   | /opt/local-path-...     |
        +-------------------------+                   +-------------------------+
```

---

### 1.3 Matriz de Componentes del Clúster

Para conseguir que este clúster opere sin software de terceros, coordinamos seis recursos nativos de Kubernetes:

| Recurso Kubernetes | Nombre | Namespace | Función Pedagógica Principal |
| :--- | :--- | :--- | :--- |
| **StorageClass** | `local-path` | Clúster | Aprovisionamiento dinámico local sobre el sistema de archivos del nodo anfitrión. |
| **Secret** | `postgres-credentials` | `default` | Almacenamiento seguro de contraseñas de superusuario (`postgres`) y usuario de replicación (`replicator`). |
| **ConfigMap** | `postgres-config` | `default` | Script de inicialización de usuarios SQL (`01-init-replication.sh`) y configuración de `pg_hba.conf`. |
| **Service (Headless)**| `postgresql-headless`| `default` | `clusterIP: None`. Provee nombres DNS directos y estables para cada Pod en CoreDNS. |
| **Service (ClusterIP)**| `postgresql-primary` | `default` | Expone un punto de acceso exclusivo de lectura/escritura enrutando tráfico únicamente al Pod 0. |
| **Service (ClusterIP)**| `postgresql-replicas`| `default` | Expone un punto de acceso balanceado para lecturas entre todas las instancias activas. |
| **StatefulSet** | `postgresql` | `default` | Orquesta las 2 réplicas, ejecuta el `initContainer` condicional y reclama volúmenes dedicados. |

---

## 2. Preparación del Almacenamiento Dinámico (StorageClass)

### 2.1 El Problema del Almacenamiento en Clústeres Kubeadm / Bare-Metal

En servicios gestionados en la nube pública (como Google Kubernetes Engine - GKE, AWS EKS o Azure AKS), el proveedor incluye de fábrica un controlador CSI (*Container Storage Interface*) que aprovisiona automáticamente discos virtuales (GCP Persistent Disks, AWS EBS, Azure Disks) cuando se solicita un PVC.

Sin embargo, en un clúster instalado desde cero mediante `kubeadm` (como el configurado en el Capítulo 1 sobre máquinas virtuales Ubuntu), **no existe ningún aprovisionador de almacenamiento configurado por defecto**. 

> [!WARNING]
> Si intentas aplicar un `StatefulSet` con `volumeClaimTemplates` en un clúster Kubeadm sin StorageClass, los PVCs quedarán congelados en estado `Pending` con el evento:
> `waiting for a volume to be created, either by dynamic provisioning or by manually creating a PersistentVolume`.

Para resolver esto en entornos de laboratorio y on-premise de manera ligera y confiable, utilizaremos **Rancher `local-path-provisioner`**. Este controlador escucha eventos de creación de PVCs y crea automáticamente carpetas dedicadas en el sistema de archivos del nodo físico (`/opt/local-path-provisioner`), vinculándolas de forma transparente como un `PersistentVolume` con modo de acceso `ReadWriteOnce`.

---

### 2.2 Despliegue de Rancher `local-path-provisioner`

El archivo [`manifests/00-local-path-storage.yaml`](file:///home/miguel/git/poc-training-k8s-cluster-postgresql/manifests/00-local-path-storage.yaml) contiene la definición completa: ServiceAccount, ClusterRoles con permisos de gestión de volumen, ConfigMap con scripts de montaje (`setup`/`teardown`), el Deployment del controlador y el objeto `StorageClass`.

Aplicamos el aprovisionador con el siguiente comando:

```bash
kubectl apply -f manifests/00-local-path-storage.yaml
```

**Salida esperada:**
```text
namespace/local-path-storage created
serviceaccount/local-path-provisioner-service-account created
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
configmap/local-path-config created
deployment.apps/local-path-provisioner created
storageclass.storage.k8s.io/local-path created
```

Esperamos unos segundos a que el pod del controlador esté en estado `Running`:

```bash
kubectl get pods -n local-path-storage -w
```

**Salida esperada:**
```text
NAME                                      READY   STATUS    RESTARTS   AGE
local-path-provisioner-57b6f6dc9c-s7wqp   1/1     Running   0          18s
```

---

### 2.3 Verificación de la StorageClass por Defecto

Comprobamos que Kubernetes reconoce `local-path` como la StorageClass predeterminada del clúster (marcada con el sufijo `(default)` gracias a la anotación `storageclass.kubernetes.io/is-default-class: "true"`):

```bash
kubectl get storageclass
```

**Salida esperada:**
```text
NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  45s
```

> [!NOTE]
> La directiva `volumeBindingMode: WaitForFirstConsumer` es una buena práctica crítica en Kubernetes: le indica al orquestador que **no cree el volumen físico en disco de inmediato**, sino que espere a que el Pod sea asignado a un nodo específico. Esto evita que el disco se cree en el nodo A mientras que el scheduler decide ubicar el Pod en el nodo B.

---

### 2.4 Ajuste de Tolerancias del Nodo Master (Control Plane)

En el clúster desplegado en la guía anterior disponemos de dos nodos:
* `k8s-master` (Control plane)
* `k8s-worker1` (Nodo de trabajo)

Por defecto, Kubeadm aplica un aislamiento (*taint*) al nodo maestro para evitar que se ejecuten cargas de trabajo de usuario:
`node-role.kubernetes.io/control-plane:NoSchedule`.

Para que nuestro clúster distribuya las réplicas de PostgreSQL entre ambas máquinas virtuales (permitiéndonos apagar o reiniciar una máquina física entera más adelante para probar la alta disponibilidad), retiramos el taint del nodo maestro:

```bash
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-
```

**Salida esperada:**
```text
node/k8s-master untainted
```

Si en el futuro deseas restaurar el aislamiento del nodo master, basta con ejecutar:
```bash
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule
```

---

## 3. Configuración Base y Gestión de Credenciales

### 3.1 Secret de Autenticación para Superusuario y Replicador

Para inicializar el motor de base de datos y permitir que el Pod réplica se conecte al primario, requerimos dos conjuntos de credenciales:
1. `POSTGRES_PASSWORD`: Contraseña del superusuario administrativo `postgres`.
2. `REPLICATION_USER` y `REPLICATION_PASSWORD`: Usuario dedicado con privilegios `REPLICATION` en el catálogo de PostgreSQL.

Inspeccionamos el archivo [`manifests/01-postgres-secret.yaml`](file:///home/miguel/git/poc-training-k8s-cluster-postgresql/manifests/01-postgres-secret.yaml):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
  namespace: default
type: Opaque
stringData:
  POSTGRES_PASSWORD: "SuperPostgres2026!"
  REPLICATION_USER: "replicator"
  REPLICATION_PASSWORD: "ReplicationSecret2026!"
```

Aplicamos el Secret en el clúster:

```bash
kubectl apply -f manifests/01-postgres-secret.yaml
```

**Salida esperada:**
```text
secret/postgres-credentials created
```

---

### 3.2 ConfigMap con Scripts de Inicialización y `pg_hba.conf`

La imagen oficial de PostgreSQL incluye un mecanismo de extensión: cualquier script `.sh` o `.sql` ubicado en el directorio `/docker-entrypoint-initdb.d/` se ejecutará **automáticamente una única vez durante el primer arranque**, cuando el directorio `$PGDATA` esté completamente vacío.

Aprovechamos esta característica en [`manifests/02-postgres-configmap.yaml`](file:///home/miguel/git/poc-training-k8s-cluster-postgresql/manifests/02-postgres-configmap.yaml) para inyectar el script de aprovisionamiento del usuario de replicación:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: default
data:
  01-init-replication.sh: |-
    #!/bin/bash
    set -e
    echo "==> [INITDB] Creando usuario de replicación '$REPLICATION_USER'..."
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
        DO \$\$
        BEGIN
            IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = '$REPLICATION_USER') THEN
                CREATE ROLE $REPLICATION_USER WITH REPLICATION LOGIN ENCRYPTED PASSWORD '$REPLICATION_PASSWORD';
            END IF;
        END
        \$\$;
    EOSQL
    echo "==> [INITDB] Usuario de replicación '$REPLICATION_USER' configurado exitosamente."
```

Aplicamos el ConfigMap en el clúster:

```bash
kubectl apply -f manifests/02-postgres-configmap.yaml
```

**Salida esperada:**
```text
configmap/postgres-config created
```

---

## 4. La Capa de Red: Headless Service y Servicios de Acceso

### 4.1 Por qué un Servicio Estándar no Sirve: El Headless Service (`clusterIP: None`)

Cuando creamos un servicio tradicional de tipo `ClusterIP`, Kubernetes realiza dos acciones:
1. Asigna una dirección IP virtual fija (VIP) dentro del rango del clúster (por ejemplo, `10.96.0.150`).
2. Configura reglas en `iptables` o `IPVS` en cada nodo para balancear el tráfico aleatoriamente (*round-robin*) entre todos los Pods seleccionados.

**¿Por qué este modelo destruye una replicación de base de datos?**  
Porque la réplica `postgresql-1` **no puede conectarse a una IP aleatoria**. Si la réplica se conectara al servicio balanceado, el 50% de sus intentos de streaming irían dirigidos a sí misma, provocando un fallo catastrófico de replicación en bucle. La réplica necesita una conexión TCP dirigida y persistente **exclusivamente hacia el Pod `postgresql-0`**.

Aquí es donde entra el **Headless Service**: al configurar `clusterIP: None`, le indicamos explícitamente a Kubernetes:
> *"No asignes ninguna IP virtual ni balancees paquetes. Limítate a crear registros DNS en CoreDNS que apunten directamente a las IPs de cada uno de los Pods."*

---

### 4.2 Resolución DNS Individual por Pod en CoreDNS

Al combinar un `StatefulSet` con un `Headless Service`, el plugin de DNS de Kubernetes (`CoreDNS`) genera automáticamente registros `A` con formato determinista:

$$\text{<nombre-del-pod>}.\text{<nombre-del-headless-service>}.\text{<namespace>}.\text{svc.cluster.local}$$

En nuestro escenario, esto nos proporciona nombres canónicos inmutables:
* **Primario:** `postgresql-0.postgresql-headless.default.svc.cluster.local`
* **Réplica:** `postgresql-1.postgresql-headless.default.svc.cluster.local`

Incluso si el Pod `postgresql-0` se elimina y se recrea con una IP interna de Flannel totalmente diferente (por ejemplo, cambiando de `10.244.1.42` a `10.244.2.89`), **el nombre DNS sigue siendo exactamente el mismo**.

---

### 4.3 Separación de Tráfico: Servicio Primario (RW) y Servicio Réplicas (RO)

Para que las aplicaciones conecten de forma transparente según la operación que deseen realizar, definimos dos servicios adicionales en [`manifests/03-postgres-services.yaml`](file:///home/miguel/git/poc-training-k8s-cluster-postgresql/manifests/03-postgres-services.yaml):

1. **`postgresql-primary` (Read-Write):**  
   Utiliza el label interno inyectado automáticamente por el controlador del StatefulSet:
   ```yaml
   selector:
     app: postgresql
     statefulset.kubernetes.io/pod-name: postgresql-0
   ```
   Cualquier petición a `postgresql-primary:5432` llegará exclusivamente al Pod maestro.

2. **`postgresql-replicas` (Read-Only):**  
   Utiliza el selector general `app: postgresql`. Permite distribuir consultas analíticas y de lectura entre cualquiera de las instancias del clúster sin sobrecargar el nodo maestro.

Aplicamos los servicios:

```bash
kubectl apply -f manifests/03-postgres-services.yaml
```

**Salida esperada:**
```text
service/postgresql-headless created
service/postgresql-primary created
service/postgresql-replicas created
```

Comprobamos los servicios creados:

```bash
kubectl get svc -l app=postgresql
```

**Salida esperada:**
```text
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
postgresql-headless   ClusterIP   None            <none>        5432/TCP   12s
postgresql-primary    ClusterIP   10.98.140.210   <none>        5432/TCP   12s
postgresql-replicas   ClusterIP   10.104.75.12    <none>        5432/TCP   12s
```

---

## 5. Despliegue del StatefulSet y Mecánica del InitContainer

### 5.1 Anatomía del StatefulSet: `replicas`, `serviceName` y `OrderedReady`

El manifiesto [`manifests/04-postgres-statefulset.yaml`](file:///home/miguel/git/poc-training-k8s-cluster-postgresql/manifests/04-postgres-statefulset.yaml) es el núcleo de este laboratorio.

Puntos clave de la cabecera:
* `serviceName: postgresql-headless`: Conecta el ciclo de vida de los Pods con el servicio Headless para el registro automático de nombres DNS.
* `replicas: 2`: Desplegará exactamente dos Pods (`postgresql-0` y `postgresql-1`).
* `podManagementPolicy: OrderedReady`: Garantiza que el Pod 0 se inicialice completamente antes de que el Pod 1 empiece siquiera a crearse.

---

### 5.2 Aislamiento de Volúmenes con `volumeClaimTemplates`

En un `Deployment`, todos los Pods montan el mismo PVC o bien se recurre a `emptyDir` (volátiles). En el `StatefulSet`, la sección `volumeClaimTemplates` actúa como una "fábrica de PVCs":

```yaml
  volumeClaimTemplates:
    - metadata:
        name: postgresql-data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: local-path
        resources:
          requests:
            storage: 1Gi
```

Cuando Kubernetes procese la plantilla, creará automáticamente:
1. `postgresql-data-postgresql-0`
2. `postgresql-data-postgresql-1`

Ambos PVCs son completamente independientes, garantizando que cada instancia tenga su propia copia física de las tablas y archivos WAL.

---

### 5.3 Desmontando la "Magia": Lógica Interna del InitContainer

Ambos Pods ejecutan exactamente la misma imagen de contenedor (`postgres:16-alpine`). ¿Cómo sabe el Pod secundario que debe comportarse como una réplica y clonar los datos del primero?

A través de un **`initContainer`** que corre un script en `/bin/sh` antes de que el proceso principal de PostgreSQL inicie:

```
+-----------------------------------------------------------------------------------------+
|                  ÁRBOL DE DECISIÓN DEL INITCONTAINER (init-postgres)                    |
+-----------------------------------------------------------------------------------------+
|                                                                                         |
|                         ¿Cuál es mi hostname y ordinal?                                 |
|                         ORDINAL=${HOSTNAME##*-}                                         |
|                                     |                                                   |
|                    +----------------+----------------+                                  |
|                    |                                 |                                  |
|              ORDINAL == 0                      ORDINAL > 0                              |
|                    |                                 |                                  |
|                    v                                 v                                  |
|         [ SOY EL PRIMARY ]                  [ SOY LA RÉPLICA ]                          |
|         - Asegurar permisos chmod 700       - Hacer loop de espera:                     |
|         - Si PGDATA está vacío, dejar         until pg_isready -h postgresql-0...       |
|           que el entrypoint ejecute         - Cuando el primario responda:              |
|           initdb y los scripts de             ¿Está vacío mi directorio PGDATA?         |
|           /docker-entrypoint-initdb.d/               |                                  |
|                    |                        +--------+--------+                         |
|                    |                        |                 |                         |
|                    |                       SÍ                 NO                        |
|                    |                        |                 |                         |
|                    |                        v                 v                         |
|                    |             [ CLONAR PRIMARIO ]   [ CONSERVAR ESTADO ]             |
|                    |             Ejecutar:             Ya tengo datos.                  |
|                    |             pg_basebackup -h      No sobreescribir.                |
|                    |               postgresql-0        (Arrancar standby                |
|                    |               -D $PGDATA          directamente)                    |
|                    |               -Fp -Xs -R                                           |
|                    |             (Genera standby.signal                                 |
|                    |              y primary_conninfo)                                   |
|                    |                        |                 |                         |
|                    +------------------------+-----------------+                         |
|                                             |                                           |
|                                             v                                           |
|                           InitContainer finaliza con éxito (exit 0)                     |
|                           El contenedor principal de PostgreSQL inicia                  |
+-----------------------------------------------------------------------------------------+
```

El parámetro **`-R`** de `pg_basebackup` es fundamental en PostgreSQL 16:
1. Crea el archivo marcador vacío `standby.signal` en el directorio de datos. Al detectarlo durante el arranque, PostgreSQL inicia en modo **Hot Standby** (solo lectura).
2. Escribe automáticamente en `$PGDATA/postgresql.auto.conf` la cadena de conexión con el primario:
   ```text
   primary_conninfo = 'user=replicator password=... host=postgresql-0.postgresql-headless port=5432 sslmode=prefer'
   ```

---

### 5.4 Despliegue Secuencial y Seguimiento del Arranque

Aplicamos el StatefulSet:

```bash
kubectl apply -f manifests/04-postgres-statefulset.yaml
```

**Salida esperada:**
```text
statefulset.apps/postgresql created
```

Monitoreamos en tiempo real el despliegue ordenado de los Pods:

```bash
kubectl get pods -l app=postgresql -w
```

**Secuencia real observada:**
```text
NAME           READY   STATUS     RESTARTS   AGE
postgresql-0   0/1     Pending    0          0s
postgresql-0   0/1     Init:0/1   0          2s
postgresql-0   0/1     PodInitializing   0   14s
postgresql-0   0/1     Running           0   16s
postgresql-0   1/1     Running           0   21s
postgresql-1   0/1     Pending           0   0s
postgresql-1   0/1     Init:0/1          0   3s
postgresql-1   0/1     PodInitializing   0   8s
postgresql-1   0/1     Running           0   10s
postgresql-1   1/1     Running           0   16s
```

Observemos los logs del `initContainer` en `postgresql-1` para confirmar la clonación:

```bash
kubectl logs postgresql-1 -c init-postgres
```

**Salida esperada:**
```text
==> [INIT] Ejecutando bootstrap en Pod 'postgresql-1' (Ordinal: 1)...
==> [INIT] Nodo Replica (postgresql-1). Esperando a que postgresql-0 esté listo...
==> [INIT] postgresql-0 está operativo.
==> [INIT] Clonando catálogo inicial desde postgresql-0 con pg_basebackup...
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/1600028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_24"
32540/32540 kB (100%), 1/1 tablespace
pg_basebackup: write-ahead log end point: 0/1600100
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: base backup completed
==> [INIT] Clonación finalizada con éxito. standby.signal y primary_conninfo generados.
```

Comprobamos que ambos PVCs han sido dinámicamente aprovisionados y enlazados (`Bound`):

```bash
kubectl get pvc -l app=postgresql
```

**Salida esperada:**
```text
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
postgresql-data-postgresql-0        Bound    pvc-8e2b9d28-3e44-4720-9189-913a48eefb20   1Gi        RWO            local-path     2m
postgresql-data-postgresql-1        Bound    pvc-1f48ba61-8cae-4f01-9491-9e23ba85e29a   1Gi        RWO            local-path     90s
```

---

## 6. Verificación Práctica de la Replicación Streaming

### 6.1 Acceso Interactivo con `psql` al Pod Primario (`postgresql-0`)

Accedemos mediante consola interactiva al Pod 0 utilizando la herramienta `psql` con el usuario `postgres`:

```bash
kubectl exec -it postgresql-0 -- psql -U postgres
```

**Salida esperada:**
```text
psql (16.2)
Type "help" for help.

postgres=#
```

---

### 6.2 Creación de Esquema y Datos de Prueba

Creamos una tabla de clientes e insertamos tres registros en el nodo maestro:

```sql
CREATE TABLE estudiantes_k8s (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    modulo VARCHAR(100) NOT NULL,
    creado_en TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO estudiantes_k8s (nombre, modulo) VALUES
    ('Miguel Serrano', 'Kubernetes StatefulSets'),
    ('Laura Gomez', 'Headless Services y DNS'),
    ('Carlos Fernandez', 'Arquitectura de Operadores');

SELECT * FROM estudiantes_k8s;
```

**Salida esperada:**
```text
 id |      nombre      |           modulo           |         creado_en          
----+------------------+----------------------------+----------------------------
  1 | Miguel Serrano   | Kubernetes StatefulSets    | 2026-11-25 15:10:20.123456
  2 | Laura Gomez      | Headless Services y DNS    | 2026-11-25 15:10:20.128765
  3 | Carlos Fernandez | Arquitectura de Operadores | 2026-11-25 15:10:20.131234
(3 rows)
```

---

### 6.3 Inspección del Catálogo Interno de Replicación (`pg_stat_replication`)

Sin salir de `psql` en `postgresql-0`, consultamos la vista del sistema `pg_stat_replication`:

```sql
SELECT 
    application_name, 
    client_addr, 
    state, 
    sync_state, 
    replay_lag 
FROM pg_stat_replication;
```

**Salida esperada:**
```text
 application_name | client_addr  |   state   | sync_state | replay_lag 
------------------+--------------+-----------+------------+------------
 walreceiver      | 10.244.1.65  | streaming | async      | 00:00:00
(1 row)
```

> [!NOTE]
> Observa cómo `application_name = walreceiver` y `state = streaming`. Esto certifica que el Pod `postgresql-1` está conectado de forma continua al socket TCP de `postgresql-0`, consumiendo transacciones en tiempo real con latencia `replay_lag` de 0 segundos.

Salimos de la sesión con `\q`.

---

### 6.4 Validación de Lectura y Protección contra Escritura en `postgresql-1`

Ahora conectamos a la réplica (`postgresql-1`) para verificar dos aspectos esenciales:
1. Que los datos insertados en el primario ya están disponibles para lectura.
2. Que la réplica rechaza tajantemente cualquier intento de modificación.

```bash
kubectl exec -it postgresql-1 -- psql -U postgres
```

Ejecutamos la consulta de lectura:

```sql
SELECT * FROM estudiantes_k8s;
```

**Salida esperada:**
```text
 id |      nombre      |           modulo           |         creado_en          
----+------------------+----------------------------+----------------------------
  1 | Miguel Serrano   | Kubernetes StatefulSets    | 2026-11-25 15:10:20.123456
  2 | Laura Gomez      | Headless Services y DNS    | 2026-11-25 15:10:20.128765
  3 | Carlos Fernandez | Arquitectura de Operadores | 2026-11-25 15:10:20.131234
(3 rows)
```

Ahora intentamos realizar una inserción de datos en la réplica:

```sql
INSERT INTO estudiantes_k8s (nombre, modulo) VALUES ('Intruso', 'Test Read Only');
```

**Salida esperada (Rechazo Inmediato):**
```text
ERROR: cannot execute INSERT in a read-only transaction
```

La réplica está protegida en modo solo lectura de manera estricta por el motor de PostgreSQL gracias a la presencia del archivo `standby.signal`.

Salimos con `\q`.

---

## 7. El Experimento del Caos: Limitaciones de Kubernetes Puro

Llegamos al punto culminante de la lección práctica. Si hemos logrado montar un clúster de base de datos primario/réplica con replicación activa usando solo Kubernetes estándar... **¿por qué todo el mundo insiste en que no se deben gestionar bases de datos en producción sin un Operador?**

Vamos a provocar una catástrofe controlada para observarlo.

---

### 7.1 Simulación de Caída del Nodo Primario (`postgresql-0`)

Imaginemos que el nodo primario sufre un pánico de kernel o se elimina forzosamente:

```bash
kubectl delete pod postgresql-0 --now
```

**Salida esperada:**
```text
pod "postgresql-0" deleted
```

---

### 7.2 El Comportamiento Real: Bloqueo de Escrituras y Réplica Estancada

Inmediatamente intentamos realizar una escritura a través del servicio primario (`postgresql-primary`):

```bash
kubectl run test-writer --rm -it --image=postgres:16-alpine --restart=Never -- \
  psql -h postgresql-primary -U postgres -c "INSERT INTO estudiantes_k8s (nombre, modulo) VALUES ('Test Caos', 'Caida Master');"
```

**Salida observada:**
```text
psql: error: connection to server at "postgresql-primary" (10.98.140.210), port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
pod "test-writer" deleted
pod default/test-writer terminated (Error)
```

**¿Y qué está haciendo la réplica `postgresql-1` mientras tanto?**
Comprobamos el estado de `postgresql-1`:

```bash
kubectl exec -it postgresql-1 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
```

**Salida:**
```text
 pg_is_in_recovery 
-------------------
 t
(1 row)
```

`pg_is_in_recovery` devuelve `t` (*true*). **`postgresql-1` sigue en modo standby de solo lectura.**  
Kubernetes ha detectado la muerte de `postgresql-0`, pero:
1. **Kubernetes NO tiene ni idea de qué es PostgreSQL.** Para Kubelet, `postgresql-0` es solo un contenedor con código de salida erróneo.
2. **Kubernetes NO asciende a la réplica.** Kubernetes intentará recrear `postgresql-0`. Durante todo el tiempo que tarde en levantarse, las transacciones de escritura de tu negocio estarán completamente caídas.

---

### 7.3 El Peligro del Split-Brain por Promoción Manual no Coordinada

Un administrador inexperto ante una caída prolongada podría verse tentado a conectarse a `postgresql-1` y forzar manualmente su promoción a primario mediante:

```bash
kubectl exec -it postgresql-1 -- psql -U postgres -c "SELECT pg_promote();"
```

**Salida:**
```text
 pg_promote 
------------
 t
(1 row)
```

En ese instante:
1. `postgresql-1` elimina `standby.signal` y se convierte en Primario (Read-Write).
2. La aplicación reconfigura sus credenciales y empieza a escribir en `postgresql-1`.
3. Pero el StatefulSet controller de Kubernetes, que seguía intentando resucitar a `postgresql-0`, **consigue levantar de nuevo a `postgresql-0`** re-vinculándolo a su volumen original `postgresql-data-postgresql-0`.
4. `postgresql-0` despierta creyendo que **él sigue siendo el Primario legítimo**.

**Resultado Catastrófico: SPLIT-BRAIN (Cerebro Dividido)**
* Tienes dos nodos en la misma red creyéndose primarios simultáneamente.
* Clientes A escriben en `postgresql-0`. Clientes B escriben en `postgresql-1`.
* Los historiales de transacciones (timelines) divergen de forma irreversible.
* La base de datos queda corrupta y la recuperación requiere días de intervención forense con pérdida inevitable de datos.

---

### 7.4 Por qué las Primitivas Nativas no Pueden Resolver el Failover

Este experimento demuestra una verdad fundamental de los sistemas distribuidos:

```
+-----------------------------------------------------------------------------------------+
|                  EL TRIÁNGULO IMPOSIBLE DEL FAILOVER EN KUBERNETES PURO                 |
+-----------------------------------------------------------------------------------------+
|                                                                                         |
|   1. DETECCIÓN DE FALLO            2. ELECCIÓN DE LÍDER             3. CERCADO (FENCING)|
|   ¿Murió el primario o es          ¿Quién tiene el WAL más          Asegurar al 100% que|
|   solo un corte de red entre       reciente para ascender           el primario viejo NO|
|   dos nodos del clúster?           sin perder transacciones?        pueda volver a escribir.
|                                                                                         |
|   [!] Kubernetes nativo carece de estos 3 mecanismos para software de terceros.         |
|       K8s solo monitoriza procesos a nivel de sistema operativo (cgroups/PID).          |
+-----------------------------------------------------------------------------------------+
```

---

## 8. La Brecha hacia la Producción: Por Qué Usamos Operadores

### 8.1 Matriz Comparativa: StatefulSet Manual vs. Operadores de Producción

| Capacidad Requerida | Despliegue Manual con StatefulSet | Operador K8s (CloudNativePG / Zalando / Crunchy) |
| :--- | :--- | :--- |
| **Inicialización y Bootstrap** | Manual con scripts bash en `initContainer`. | Declarativo nativo vía Custom Resource (CRD). |
| **Failover Automático** | **No existe.** Requiere intervención humana. | **Automático sub-segundo** con elección de líder por consenso. |
| **Prevención de Split-Brain** | **Ninguna.** Alto riesgo de corrupción de datos. | **Garantizado por Fencing** (aislamiento forzado del nodo caído). |
| **Elección del Mejor Candidato** | No disponible. | Analiza el LSN (*Log Sequence Number*) con menor lag de réplica. |
| **Backups y PITR** | Scripts manuales con CronJobs frágiles. | Integración nativa con Barman/WAL-G a S3/GCS/MinIO con recuperación a cualquier segundo (*Point-In-Time*). |
| **Actualizaciones Menores** | Caída de servicio durante reinicio de pods. | Rolling update inteligente con `switchover` previo planificado. |
| **Pool de Conexiones** | Requiere desplegar PgBouncer a mano. | PgBouncer integrado y autogestionado por el operador. |

---

### 8.2 Capacidades Esenciales de un Operador (CloudNativePG, Zalando, Crunchy)

Un **Operador de Kubernetes** es la materialización en código Go de todo el conocimiento y experiencia de un Administrador de Bases de Datos (DBA):

1. **Bucle de Reconciliación Inteligente:** No solo observa si el Pod existe, sino que ejecuta `SELECT pg_catalog` internamente para evaluar la topología del clúster cada segundo.
2. **Consenso Distribuido (Raft / Leases de K8s):** Para ascender a un nuevo primario, se requiere el voto de la mayoría del quórum, eliminando por completo la posibilidad de split-brain.
3. **Mecanismo de Fencing:** Si un primario deja de responder al quórum, el Operador utiliza la API de Kubernetes para degradar inmediatamente sus etiquetas de red y cortar sus credenciales antes de ascender a la réplica.

---

## 9. Resolución de Problemas Frecuentes (Troubleshooting)

### 9.1 Error 1: PVCs en Estado `Pending` por Ausencia de StorageClass
* **Síntoma:** Al ejecutar `kubectl get pvc`, los volúmenes muestran estado `Pending`.
* **Causa:** El clúster Kubeadm no tiene ningún proveedor de almacenamiento por defecto.
* **Solución:** Aplicar [`manifests/00-local-path-storage.yaml`](file:///home/miguel/git/poc-training-k8s-cluster-postgresql/manifests/00-local-path-storage.yaml) y verificar con `kubectl get sc` que `local-path` tiene la anotación `is-default-class: "true"`.

---

### 9.2 Error 2: `initdb: directory exists but is not empty` (`lost+found`)
* **Síntoma:** El Pod `postgresql-0` entra en `CrashLoopBackOff` con el mensaje:
  `initdb: error: directory "/var/lib/postgresql/data" exists but is not empty`.
* **Causa:** En sistemas de archivos Linux Ext4 montados por un aprovisionador de volumen, se crea automáticamente una carpeta de sistema llamada `lost+found`. PostgreSQL rechaza inicializar un clúster si el directorio base contiene cualquier archivo o carpeta.
* **Solución:** Configurar la variable de entorno `PGDATA` apuntando a un subdirectorio dedicado dentro del volumen:
  ```yaml
  env:
    - name: PGDATA
      value: /var/lib/postgresql/data/pgdata
  ```

---

### 9.3 Error 3: Fallo en `pg_basebackup`: Autenticación Rechazada en `pg_hba.conf`
* **Síntoma:** En los logs de `init-postgres` en `postgresql-1` aparece:
  `pg_basebackup: error: connection to server failed: FATAL: no pg_hba.conf entry for replication connection`.
* **Causa:** El archivo de control de acceso de PostgreSQL (`pg_hba.conf`) restringe por defecto las conexiones entrantes únicamente a localhost (`127.0.0.1`).
* **Solución:** Asegurar que `postgresql.conf` escuche en todas las interfaces (`listen_addresses = '*'`) y que las reglas de `pg_hba.conf` autoricen el tráfico de replicación desde la red de Pods del clúster (`10.244.0.0/16`).

---

### 9.4 Error 4: Permisos Denegados en el Directorio de Datos (`chown 999:999`)
* **Síntoma:** El contenedor de PostgreSQL no arranca indicando:
  `FATAL: data directory "/var/lib/postgresql/data/pgdata" has wrong ownership`.
* **Causa:** Los volúmenes montados por el sistema anfitrión pertenecen inicialmente al usuario `root` (`0:0`), mientras que el proceso de PostgreSQL corre bajo el usuario no privilegiado `postgres` (UID `999`).
* **Solución:** El `initContainer` debe ejecutar `chmod 700 "$TARGET_DIR"` y, si fuera necesario, delegar la creación de la carpeta al usuario `postgres` antes de la ejecución de `initdb` o `pg_basebackup`.

---

## Conclusión

A lo largo de este laboratorio has desmitificado por completo el funcionamiento interno de una base de datos en Kubernetes:
1. Comprendiste cómo **`StatefulSets`** otorgan ordinales e identidad predecible a cargas con estado.
2. Viste cómo un **`Headless Service`** permite la resolución de red punto a punto requerida para la replicación streaming.
3. Descubriste cómo los **`initContainers`** pueden orquestar el clonado físico de datos con `pg_basebackup`.
4. Y lo más importante: **experimentaste empíricamente los límites de Kubernetes puro**, entendiendo por qué la resiliencia en bases de datos relacionales requiere la lógica avanzada de consenso y fencing que proporcionan los **Operadores de Kubernetes**.
