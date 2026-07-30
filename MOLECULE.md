# Test de Molecule

## Escenarios

| Escenario     | Driver   | Propósito                                        |
|---------------|----------|--------------------------------------------------|
| `default`     | podman   | Single host, verifica instalación básica         |
| `cluster`     | docker   | 3 nodos (primary + 2 secondary), cluster completo |
| `backup-tag`  | podman   | 3 nodos + validación de `--tags backup`          |

## Prerrequisitos

```bash
# Python
pip install molecule molecule-podman ansible ansible-lint yamllint

# Podman (macOS)
brew install podman
podman machine set --rootful
podman machine start

# Podman (Linux)
sudo dnf install -y podman
```

## Imagen custom para Podman (rocky9-systemd)

Docker no soporta contenedores systemd en macOS. Se requiere una imagen custom con systemd:

```bash
cd roles/ansible_role_mongodb

cat > Dockerfile.rocky9-systemd << 'EOF'
FROM rockylinux:9
RUN dnf -y update && \
    dnf -y install systemd && \
    dnf clean all && \
    rm -rf /var/cache/dnf && \
    (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ "$i" = "systemd-tmpfiles-setup.service" ] || rm -f "$i"; done) && \
    rm -f /lib/systemd/system/multi-user.target.wants/* && \
    rm -f /etc/systemd/system/*.wants/* && \
    rm -f /lib/systemd/system/local-fs.target.wants/* && \
    rm -f /lib/systemd/system/sockets.target.wants/*initctl* && \
    rm -f /lib/systemd/system/sockets.target.wants/*udev* && \
    rm -f /lib/systemd/system/anaconda.target.wants/*
CMD ["/usr/sbin/init"]
EOF

podman build -t rocky9-systemd -f Dockerfile.rocky9-systemd .
podman tag localhost/rocky9-systemd:latest localhost/rocky9-systemd:latest
```

## Comandos

### Ejecutar escenario completo

```bash
# Single host
molecule converge -s default

# Cluster 3 nodos (usa Docker - ver notas abajo)
molecule converge -s cluster

# Backup tag (usa Podman)
molecule converge -s backup-tag
```

### Probar `--tags backup` específicamente

```bash
# Fase 1: despliegue completo del role
molecule converge -s backup-tag

# Fase 2: solo las tareas con tag "backup"
molecule converge -s backup-tag -- --tags backup
```

### Ejecutar verificaciones

```bash
molecule verify -s backup-tag
```

### Destroy + full cycle

```bash
molecule destroy -s backup-tag && molecule converge -s backup-tag && molecule verify -s backup-tag
```

## Escenario `cluster` (nota sobre Docker)

El escenario `cluster` usa el driver **docker** con una red custom (`test_network`, subnet `10.3.1.0/24`)
y direcciones IP estáticas. Esto porque molecule-docker sí soporta redes custom con IPs fijas,
mientras que molecule-podman no.

Para ejecutarlo necesitas Docker Engine funcionando:

```bash
# Asegurarse de que la imagen rockylinux:9-systemd existe
docker pull rockylinux:9-systemd

# Si no existe localmente, construirla:
# (misma receta que la imagen de podman pero con docker)
docker build -t rockylinux:9-systemd -f Dockerfile.rocky9-systemd .
```

## Escenario `backup-tag` (Podman + IPs dinámicas)

Podman asigna IPs dinámicas a los contenedores. Como `molecule-podman` no soporta redes custom
con IPs fijas, el playbook `molecule/backup-tag/prepare.yml` inyecta las IPs correctas en
`/etc/hosts` de cada contenedor automáticamente.

**Importante**: si los contenedores se destruyen y recrean, las IPs cambian.
El prepare playbook se encarga de actualizarlas.

## Debugging

### Ver logs de MongoDB

```bash
podman exec test-multi-01 journalctl -u mongod --no-pager
podman exec test-multi-01 cat /opt/somewhere/mongod.log
```

### Verificar conectividad entre nodos

```bash
podman exec test-multi-01 ping -c 1 test-multi-02
podman exec test-multi-01 mongosh --port 27018 --eval 'db.auth("admin", "something"); rs.status()'
```

### Ver configuración generada

```bash
podman exec test-multi-01 cat /etc/mongod.conf
podman exec test-multi-01 ls -la /etc/keyfile_mongo
podman exec test-multi-02 crontab -l
```

### Forzar recreate completo

```bash
molecule destroy -s backup-tag && molecule create -s backup-tag && molecule converge -s backup-tag
```

### Errores comunes

| Síntoma | Causa | Solución |
|---------|-------|----------|
| `No route to host` en cluster init | IPs en `/etc/hosts` incorrectas | Destroy + recreate (prepare.yml lo corrige) |
| `Unrecognized option: storage.journal.enabled` | Opción eliminada en MongoDB 7.0 | Quitar `journal.enabled` de `mongo_storage` en group_vars |
| `not primary` al crear backup user | La tarea se ejecuta en un secondary | Debe ejecutarse en el primary (fixed en `tasks/backup.yml`) |
| `Authentication failed` en verify | `db.auth()` contra db `test` en vez de `admin` | Usar `mongosh admin --port ...` |
