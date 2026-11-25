# Spec Delta: PostgreSQL StatefulSet Cluster

## Purpose

Define el comportamiento esperado del despliegue manual de un cluster PostgreSQL 16 primario/réplica mediante StatefulSet, Headless Service, PVCs dinámicos e initContainers en Kubernetes.

## ADDED Requirements

### Requirement: Inicialización y bootstrap según ordinal de Pod
El initContainer del Pod de PostgreSQL SHALL inspeccionar el ordinal en el hostname del Pod (`${HOSTNAME##*-}`) para determinar de forma determinista si asume el rol de nodo primario (ordinal 0) o de nodo réplica en espera (ordinal > 0).

#### Scenario: Inicialización de nodo primario (ordinal 0)
- **WHEN** el Pod `postgresql-0` inicia y detecta que su ordinal es 0
- **THEN** prepara la configuración base y permite que el contenedor principal inicialice el clúster con permisos de lectura y escritura

#### Scenario: Inicialización de nodo réplica (ordinal > 0) con clonación
- **WHEN** el Pod `postgresql-1` inicia con un volumen de datos vacío
- **THEN** espera a que `postgresql-0` esté accesible vía `pg_isready`, ejecuta `pg_basebackup` para clonar los datos del primario y genera el archivo `standby.signal` junto con `primary_conninfo`

### Requirement: Resolución de red punto a punto vía Headless Service
El sistema SHALL exponer los Pods de PostgreSQL mediante un Headless Service (`clusterIP: None`) que permita la resolución DNS unívoca para cada réplica en el clúster.

#### Scenario: Conexión de replicación dirigida al primario
- **WHEN** la réplica `postgresql-1` inicia su proceso de streaming WAL
- **THEN** resuelve y conecta directamente a la dirección FQDN `postgresql-0.postgresql-headless` sin balanceo de carga intermedio

### Requirement: Aislamiento y persistencia de datos por réplica
El StatefulSet SHALL asociar un PersistentVolumeClaim independiente a cada Pod mediante `volumeClaimTemplates` gestionado por una StorageClass dinámica.

#### Scenario: Conservación de volumen tras reinicio de Pod
- **WHEN** el Pod `postgresql-0` es reiniciado o reprogramado en otro nodo del clúster
- **THEN** se vuelve a vincular al mismo PVC `data-postgresql-0` preservando íntegramente las transacciones previas sin requerir re-clonado

### Requirement: Separación de servicios de lectura y escritura
La arquitectura SHALL exponer dos servicios diferenciados para el tráfico de las aplicaciones consumidoras: uno exclusivo para escritura hacia el primario y otro para consultas de lectura.

#### Scenario: Enrutamiento de transacciones de escritura
- **WHEN** una aplicación envía una sentencia `INSERT` al servicio `postgresql-primary`
- **THEN** el tráfico es dirigido exclusivamente a la instancia en modo lectura-escritura (`postgresql-0`)

#### Scenario: Enrutamiento de consultas de solo lectura
- **WHEN** una aplicación consulta el servicio `postgresql-replicas`
- **THEN** la consulta es servida por las instancias secundarias en modo standby
