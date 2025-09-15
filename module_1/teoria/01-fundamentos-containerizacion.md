# Fundamentos de Containerización y Linux Kernel
## Módulo 1: Teoría Avanzada para Doctorado

---

## 1. INTRODUCCIÓN A LA CONTAINERIZACIÓN

### 1.1 Definición y Conceptos Fundamentales

La **containerización** es una tecnología de virtualización a nivel del sistema operativo que permite empaquetar aplicaciones junto con todas sus dependencias, bibliotecas, archivos de configuración y entorno de ejecución en una unidad portable llamada **container**.

**Definición formal:**
> Un container es una abstracción en la capa de aplicación que empaqueta código y dependencias juntos. Múltiples containers pueden ejecutarse en la misma máquina y compartir el kernel del OS con otros containers, cada uno ejecutándose como procesos aislados en el espacio de usuario.

### 1.2 Evolución Histórica

#### Timeline de Virtualización:
- **1960s**: IBM VM/370 - Primera virtualización de hardware
- **1999**: VMware - Virtualización para x86
- **2000**: FreeBSD Jails - Primeros containers primitivos
- **2005**: OpenVZ/Virtuozzo - Virtualización a nivel OS
- **2006**: Process Containers (Google) → cgroups
- **2008**: LXC (Linux Containers)
- **2013**: Docker - Revolución de containers
- **2014**: Kubernetes - Orquestación de containers
- **2016**: containerd, CRI-O - Runtimes especializados

### 1.3 Paradigmas de Virtualización

```
┌─────────────────────────────────────────────────────────┐
│                    APLICACIONES                         │
├─────────────────────────────────────────────────────────┤
│  Máquinas Virtuales    │      Containers                │
│  ┌─────────────────┐    │    ┌─────────────────────┐    │
│  │   App A    │App B│    │    │App A│App B│App C│App D│    │
│  │   Bins/Libs     │    │    │   Bins/Libs          │    │
│  │   Guest OS      │    │    │                     │    │
│  └─────────────────┘    │    └─────────────────────┘    │
│  ┌─────────────────┐    │    ┌─────────────────────┐    │
│  │   Hypervisor    │    │    │  Container Runtime  │    │
│  └─────────────────┘    │    └─────────────────────┘    │
├─────────────────────────────────────────────────────────┤
│                    HOST OPERATING SYSTEM                │
├─────────────────────────────────────────────────────────┤
│                        INFRASTRUCTURE                   │
└─────────────────────────────────────────────────────────┘
```

---

## 2. ARQUITECTURA DEL LINUX KERNEL

### 2.1 Subsistemas Clave para Containerización

El kernel de Linux proporciona los fundamentos tecnológicos que hacen posible la containerización a través de varios subsistemas:

#### 2.1.1 Namespaces (Espacios de Nombres)

Los **namespaces** proporcionan aislamiento de recursos del sistema, creando vistas separadas de recursos globales del sistema.

**Tipos de Namespaces:**

1. **PID Namespace** (`CLONE_NEWPID`)
   - Aísla el espacio de IDs de proceso
   - Cada namespace tiene su propio conjunto de PIDs
   - El proceso init tiene PID 1 dentro del namespace

```bash
# Ejemplo: Crear nuevo PID namespace
unshare --pid --fork --mount-proc /bin/bash
ps aux  # Solo muestra procesos del namespace
```

2. **Network Namespace** (`CLONE_NEWNET`)
   - Aísla interfaces de red, tablas de routing, firewall
   - Cada namespace tiene su propia pila de red

```bash
# Crear network namespace
ip netns add container1
ip netns exec container1 ip link list
```

3. **Mount Namespace** (`CLONE_NEWNS`)
   - Aísla puntos de montaje del filesystem
   - Permite diferentes vistas del filesystem

4. **UTS Namespace** (`CLONE_NEWUTS`)
   - Aísla hostname y domainname
   - Cada container puede tener su propio hostname

5. **IPC Namespace** (`CLONE_NEWIPC`)
   - Aísla System V IPC, POSIX message queues
   - Semáforos, memoria compartida

6. **User Namespace** (`CLONE_NEWUSER`)
   - Aísla user y group IDs
   - Mapeo de UIDs entre namespaces

7. **Cgroup Namespace** (`CLONE_NEWCGROUP`)
   - Aísla la vista del árbol cgroup

#### 2.1.2 Control Groups (cgroups)

**Cgroups** controlan y limitan el uso de recursos del sistema por parte de procesos.

**Subsistemas principales:**

1. **CPU Controller**
   - `cpu.cfs_quota_us`: Cuota de CPU en microsegundos
   - `cpu.cfs_period_us`: Período de programación
   - `cpu.shares`: Peso relativo de CPU

2. **Memory Controller**
   - `memory.limit_in_bytes`: Límite máximo de memoria
   - `memory.memsw.limit_in_bytes`: Memoria + swap
   - `memory.oom_control`: Control del OOM killer

3. **Block I/O Controller**
   - `blkio.throttle.read_bps_device`: Límite de lectura
   - `blkio.throttle.write_bps_device`: Límite de escritura

```bash
# Ejemplo: Crear cgroup y limitar memoria
mkdir /sys/fs/cgroup/memory/mycontainer
echo 128M > /sys/fs/cgroup/memory/mycontainer/memory.limit_in_bytes
echo $$ > /sys/fs/cgroup/memory/mycontainer/cgroup.procs
```

#### 2.1.3 Capabilities

Linux **capabilities** dividen los privilegios del root en unidades discretas.

**Capabilities relevantes para containers:**
- `CAP_NET_ADMIN`: Administración de red
- `CAP_SYS_ADMIN`: Operaciones administrativas
- `CAP_SETUID/SETGID`: Cambiar UIDs/GIDs
- `CAP_DAC_OVERRIDE`: Saltarse permisos de archivos
- `CAP_KILL`: Enviar señales a procesos

```bash
# Ver capabilities de un proceso
getpcaps PID

# Ejecutar con capabilities limitadas
capsh --drop=cap_net_admin --chroot=/container/root
```

### 2.2 Union Filesystems

Los **Union Filesystems** permiten superponer múltiples directorios, creando una vista unificada.

#### 2.2.1 OverlayFS

**OverlayFS** es el driver de storage preferido para Docker:

```
┌─────────────────────────────────────┐
│            MERGED VIEW              │  ← Vista final del container
├─────────────────────────────────────┤
│          UPPER DIR                  │  ← Capa escribible
├─────────────────────────────────────┤
│         LOWER DIR(s)                │  ← Capas read-only (imagen)
├─────────────────────────────────────┤
│          WORK DIR                   │  ← Directorio de trabajo
└─────────────────────────────────────┘
```

**Comandos OverlayFS:**
```bash
# Montar overlay
mount -t overlay overlay \
  -o lowerdir=/lower1:/lower2,upperdir=/upper,workdir=/work \
  /merged
```

#### 2.2.2 Copy-on-Write (CoW)

Mecanismo que optimiza el uso de storage:
- Las capas son read-only hasta que se modifican
- Al modificar, se crea una copia en la capa superior
- Ahorra espacio y mejora rendimiento

---

## 3. ARQUITECTURA DE DOCKER

### 3.1 Componentes Principales

```
┌─────────────────────────────────────────────────────────┐
│                    DOCKER ARCHITECTURE                  │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │Docker Client│    │Docker CLI   │    │Docker API   │  │
│  └─────────────┘    └─────────────┘    └─────────────┘  │
├─────────────────────────────────────────────────────────┤
│                    DOCKER DAEMON                        │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │ Images      │    │ Containers  │    │ Networks    │  │
│  └─────────────┘    └─────────────┘    └─────────────┘  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │ Volumes     │    │ Plugins     │    │ Swarm Mode  │  │
│  └─────────────┘    └─────────────┘    └─────────────┘  │
├─────────────────────────────────────────────────────────┤
│                      CONTAINERD                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │    runC     │    │Container    │    │ Image Store │  │
│  │  (Runtime)  │    │ Supervisor  │    │             │  │
│  └─────────────┘    └─────────────┘    └─────────────┘  │
├─────────────────────────────────────────────────────────┤
│                    LINUX KERNEL                         │
│        Namespaces │ Cgroups │ Capabilities │ SELinux    │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Docker Daemon

El **dockerd** es el proceso central que:
- Gestiona objetos Docker (imágenes, containers, redes, volúmenes)
- Expone REST API
- Maneja autenticación y autorización
- Interactúa con containerd

### 3.3 Container Runtime Interface (CRI)

Estándar que define la interfaz entre orquestadores y runtimes:

```
Kubernetes ─────► CRI-O ─────► runC
     │              │
     └────► dockerd ─┴─────► containerd ─────► runC
```

---

## 4. PROCESO DE CONTAINERIZACIÓN

### 4.1 Ciclo de Vida de un Container

1. **Image Pull**: Descarga de imagen desde registry
2. **Container Create**: Creación del container
3. **Container Start**: Inicio del container
4. **Container Running**: Ejecución activa
5. **Container Stop**: Detención del container
6. **Container Remove**: Eliminación del container

### 4.2 Anatomía de una Imagen Docker

Una imagen Docker consiste en:

1. **Manifest**: Metadatos de la imagen
2. **Config**: Configuración del container
3. **Layers**: Capas del filesystem

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 1234,
    "digest": "sha256:abc123..."
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 5678,
      "digest": "sha256:def456..."
    }
  ]
}
```

### 4.3 Container Runtime Spec (OCI)

La **Open Container Initiative** define:

1. **Runtime Specification**: Cómo ejecutar containers
2. **Image Specification**: Formato de imágenes
3. **Distribution Specification**: Cómo distribuir imágenes

---

## 5. SEGURIDAD EN CONTAINERS

### 5.1 Modelo de Amenazas

**Vectores de ataque:**
1. Container breakout
2. Privilege escalation
3. Resource exhaustion
4. Image vulnerabilities
5. Network attacks

### 5.2 Mecanismos de Seguridad

#### 5.2.1 SELinux/AppArmor
Mandatory Access Control (MAC) systems:

```bash
# SELinux para containers
setsebool -P container_manage_cgroup true
docker run --security-opt label=type:svirt_sandbox_file_t nginx
```

#### 5.2.2 Seccomp
Filtros de system calls:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": ["read", "write", "open"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

#### 5.2.3 User Namespaces
Mapeo de usuarios:

```bash
# Mapear root del container a usuario sin privilegios
echo '0 1000 1' > /proc/self/uid_map
echo '0 1000 1' > /proc/self/gid_map
```

---

## 6. PERFORMANCE Y OPTIMIZACIÓN

### 6.1 Métricas de Rendimiento

**Métricas clave:**
- CPU utilization y throttling
- Memory usage y pressure
- Disk I/O y latencia
- Network throughput y latencia

### 6.2 Tuning de Kernel

```bash
# Optimizaciones para containers
echo 'net.core.somaxconn = 65535' >> /etc/sysctl.conf
echo 'vm.max_map_count = 262144' >> /etc/sysctl.conf
echo 'fs.file-max = 2097152' >> /etc/sysctl.conf
```

### 6.3 Container Density

Factores que afectan la densidad:
- Memory overhead del runtime
- Namespace overhead
- Network stack por container
- Storage driver efficiency

---

## 7. CASOS DE USO AVANZADOS

### 7.1 Microservicios
- Service discovery
- Load balancing
- Circuit breakers
- Distributed tracing

### 7.2 CI/CD Pipelines
- Build containers
- Test isolation
- Deployment automation
- Blue-green deployments

### 7.3 Edge Computing
- Resource constraints
- Offline operations
- Update mechanisms
- Security considerations

---

## 8. FUTURO DE LA CONTAINERIZACIÓN

### 8.1 Tecnologías Emergentes

1. **WebAssembly (WASM)**
   - Lightweight alternative
   - Near-native performance
   - Language agnostic

2. **Unikernels**
   - Library operating systems
   - Minimal attack surface
   - Fast boot times

3. **Serverless Containers**
   - Event-driven execution
   - Auto-scaling
   - Pay-per-use

### 8.2 Investigación Actual

**Áreas de investigación:**
- Container security formal verification
- Performance isolation guarantees
- Energy efficiency optimization
- Container placement algorithms
- Network function virtualization

---

## CONCLUSIONES

La containerización representa una evolución fundamental en la forma de desplegar y gestionar aplicaciones. Su éxito se basa en la integración elegante de tecnologías del kernel Linux que proporcionan aislamiento, control de recursos y portabilidad.

**Puntos clave:**
1. Los namespaces proporcionan aislamiento fundamental
2. Los cgroups permiten control granular de recursos
3. Los union filesystems optimizan el storage
4. La seguridad requiere múltiples capas de protección
5. El rendimiento depende del tuning del kernel y runtime

Para investigadores de doctorado, la containerización ofrece oportunidades en áreas como optimización de rendimiento, seguridad formal, y nuevos modelos de ejecución.

---

## REFERENCIAS ACADÉMICAS

1. Bernstein, D. (2014). "Containers and Cloud: From LXC to Docker to Kubernetes". IEEE Cloud Computing, 1(3), 81-84.

2. Felter, W., et al. (2015). "An Updated Performance Comparison of Virtual Machines and Linux Containers". IBM Research Report.

3. Combe, T., et al. (2016). "To Docker or Not to Docker: A Security Perspective". IEEE Cloud Computing, 3(5), 54-62.

4. Sharma, P., et al. (2016). "Containers and Virtual Machines at Scale: A Comparative Study". ACM/IFIP/USENIX International Conference on Distributed Systems Platforms and Open Distributed Processing.

5. Zhang, Q., et al. (2018). "A Survey on Container Security: Threats and Countermeasures". Future Internet, 10(4), 36.