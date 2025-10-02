# Ansible Role: **disk**
[![Style: ansible-lint](https://img.shields.io/badge/style-ansible--lint-green)](#lint)
[![Tests: molecule](https://img.shields.io/badge/tests-molecule-blue)](#testing)
[![License: MIT](https://img.shields.io/badge/license-MIT-informational)](LICENSE)

Rol de Ansible para **preparar y montar discos Linux** de forma segura e idempotente:
- Detecta etiqueta de particiones (`gpt` o `dos/msdos`) y la crea si hace falta.
- Crea particiones con `community.general.parted` usando rangos (`0%`, `25%`, `GiB`, etc.).
- Formatea particiones con el FS indicado (por defecto `ext4`).  
- Monta y **persiste** en `/etc/fstab` con `ansible.posix.mount`.
- Incluye **migración opcional** de datos desde una ruta existente (p. ej. `/var/log`) con `rsync`.
- Hooks para **parar/arrancar servicios** durante migraciones (journald, rsyslog, …).
- Soporta discos `sdX` y `nvmeXnY` (manejo automático del sufijo `p` para NVMe/mmcblk).

---

## Índice
- [Compatibilidad y requisitos](#compatibilidad-y-requisitos)
- [Instalación](#instalación)
- [Variables](#variables)
  - [Estructura de `disk_devices`](#estructura-de-disk_devices)
  - [Hints de servicios por ruta](#hints-de-servicios-por-ruta)
  - [Variables de control y defaults](#variables-de-control-y-defaults)
- [Ejemplos de uso](#ejemplos-de-uso)
  - [Layout DB (4 particiones)](#layout-db-4-particiones)
  - [Layout Web (2 particiones + migración /var/log)](#layout-web-2-particiones--migración-varlog)
- [Tags](#tags)
- [Idempotencia y seguridad](#idempotencia-y-seguridad)
- [Solución de problemas](#solución-de-problemas)
- [Testing](#testing)
- [Licencia y autoría](#licencia-y-autoría)

---

## Compatibilidad y requisitos

**Sistemas**: Debian/Ubuntu (y derivados).  
**Ansible Collections**:  
- `ansible.posix`
- `community.general`

**Paquetes en destino** (se instalan automáticamente si la familia OS es compatible):  
`parted`, `e2fsprogs`, `rsync` (este último sólo si hay migraciones).

Instala las colecciones con:

```yaml
# requirements.yml
collections:
  - name: ansible.posix
  - name: community.general
```

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Instalación

### A) Usando Git directamente (roles/requirements.yml)
```yaml
roles:
  - name: disk
    src: https://github.com/jhonnygo/ansible-rol-disk.git
    version: main
```
```bash
ansible-galaxy role install -r roles/requirements.yml
```

### B) Ansible Galaxy (si lo publicas)
```bash
ansible-galaxy role install jhonnygo.disk
```

---

## Variables

### Estructura de `disk_devices`

Define cada disco y sus particiones. El rol recorrerá esta lista y aplicará el layout de forma idempotente.

```yaml
disk_devices:
  - device: /dev/nvme1n1        # Disco a preparar
    label: gpt                  # (opcional) gpt | dos (msdos). Default: disk_label_default
    parts:
      - number: 1               # Número de partición
        start: "0%"             # Inicio (porcentaje o unidad: "1MiB", "10GiB", …)
        end:   "25%"            # Fin
        fs: ext4                # (opcional) Filesystem. Default: disk_fs_default
        mkfs_opts: ""           # (opcional) Flags para mkfs (p. ej. "-F")
        mount:
          path: /var/lib/mysql  # Punto de montaje
          opts: defaults        # Opciones fstab
          create_mode: "0755"   # Si crea el dir
          create_owner: root
          create_group: root
          migrate_from: ""      # Ruta a migrar si existe (ej: "/var/log")
```

> **Notas**  
> - El rol calcula automáticamente el **sufijo de partición** (`p`) para `nvme/mmcblk` y lo omite para `sdX` (ej: `/dev/nvme1n1p1` vs `/dev/sda1`).  
> - Tras particionar, el rol ejecuta `partprobe` y `udevadm settle` para forzar al kernel/udev a ver las nuevas particiones antes de continuar.

### Hints de servicios por ruta

Durante una migración (cuando `migrate_from` está presente), el rol puede parar/arrancar servicios asociados a esa ruta:

```yaml
disk_service_hints:
  "/var/log":
    stop:  [rsyslog, systemd-journald]
    start: [systemd-journald, rsyslog]
```

Puedes ampliar/overridear esta estructura en `group_vars/host_vars` para otras rutas (por ejemplo rutas de bases de datos).

### Variables de control y defaults

```yaml
# defaults/main.yml
disk_devices: []               # Lista de discos a preparar
disk_label_default: gpt        # Label por defecto si no se define d.label
disk_fs_default: ext4          # FS por defecto si no se define p.fs
disk_remove_lostfound: true    # Eliminar lost+found tras el montaje (estético)
disk_debug: false              # Habilitar mensajes debug (lsblk PTTYPE, etc)
```

---

## Ejemplos de uso

### Layout DB (4 particiones)

```yaml
- hosts: role_db
  become: true
  roles:
    - role: disk
      vars:
        disk_devices:
          - device: /dev/nvme1n1
            label: gpt
            parts:
              - { number: 1, start: "0%",  end: "25%", fs: ext4, mount: { path: /var/lib/mysql,       opts: defaults } }
              - { number: 2, start: "25%", end: "50%", fs: ext4, mount: { path: /var/lib/postgresql,  opts: defaults } }
              - { number: 3, start: "50%", end: "75%", fs: ext4, mount: { path: /var/lib/mongodb,     opts: defaults } }
              - { number: 4, start: "75%", end: "100%",fs: ext4, mount: { path: /var/log,             opts: defaults, migrate_from: "/var/log" } }
```

### Layout Web (2 particiones + migración `/var/log`)

```yaml
- hosts: role_app
  become: true
  roles:
    - role: disk
      vars:
        disk_devices:
          - device: /dev/nvme1n1
            parts:
              - { number: 1, start: "0%",  end: "50%",  fs: ext4, mount: { path: /var/www, opts: defaults } }
              - { number: 2, start: "50%", end: "100%", fs: ext4, mount: { path: /var/log, opts: defaults, migrate_from: "/var/log" } }
```

---

## Tags

- `disk` — tag general del rol  
- `disk:probe` — detecciones y avisos al kernel (`lsblk`, `partprobe`, `udevadm`)  
- `disk:mklabel` — creación de etiqueta de disco  
- `disk:parted` — creación de particiones  
- `disk:wait` — esperas por aparición de nodos de partición  
- `disk:partition` — formateo, montaje, fstab y migraciones  
- `debug` — salida adicional si `disk_debug: true`  

Ejecutar sólo particionado y montaje, por ejemplo:
```bash
ansible-playbook site.yml -l role_db -t "disk:parted,disk:partition"
```

---

## Idempotencia y seguridad

- No se forza `mkfs` si el FS ya existe en la partición (módulo `filesystem` es idempotente).  
- El montaje se realiza con `state: mounted` y queda persistido en `/etc/fstab`.  
- Si se define `migrate_from`, se usa `rsync -aXS --numeric-ids` contra un montaje temporal y se limpian recursos.  
- Se usan `partprobe` y `udevadm settle` para sincronizar con el kernel/udev tras cambios de particionado.  
- Los servicios definidos en `disk_service_hints` se paran antes de migrar y se arrancan al final.

---

## Solución de problemas

**Se detecta `PTTYPE=dos` (msdos) pero yo quiero GPT**  
Define `label: gpt` en el dispositivo o ajusta `disk_label_default: gpt`. El rol creará la etiqueta si difiere de la detectada:

```yaml
disk_devices:
  - device: /dev/nvme1n1
    label: gpt
    parts: ...
```

**Las particiones tardan en aparecer**  
El rol ya ejecuta `partprobe` y `udevadm settle`, y espera a que exista cada nodo `/dev/...`. Si aún así tu plataforma necesita más tiempo, amplía el `timeout` global mediante `ansible.cfg` (SSH timeouts) o añade una tarea de espera adicional.

**NVMe vs sdX**  
El rol calcula automáticamente el sufijo `p` para NVMe/mmcblk (`/dev/nvme1n1p1`) y no lo usa en `sdX` (`/dev/sda1`).

**`lost+found`**  
Se elimina si `disk_remove_lostfound: true` (puramente estético, no funcional).

**Modo check**  
Las tareas destructivas se condicionan con `not ansible_check_mode` cuando corresponde; en `--check` verás la intención pero no se alterará el disco.

---

## Testing

Incluye esqueleto compatible con **Molecule** (añade tu escenario Docker/Podman/EC2).

```bash
pip install molecule molecule-plugins[docker] ansible-lint
molecule test
```

---

## Licencia y autoría

**Licencia:** MIT  
**Autor:** Tu Nombre — <suport@jhoncytech.com>  
**GitHub:** https://github.com/jhonnygo/ansible-rol-disk.git


## Matriz de pruebas / Entornos verificados

> Este rol se valida de forma incremental. Aquí se listan los entornos probados con su estado.
> Si un entorno no aparece, significa "pendiente de prueba".

| Entorno / Plataforma                       | SO / AMI / Imagen                  | Ansible | Resultado | Notas |
|--------------------------------------------|------------------------------------|---------|-----------|-------|
| **Local – Vagrant (VirtualBox)**           | Ubuntu 22.04 LTS                   | 2.15+   | ✅ OK     | Pruebas de particionado `nvme` y `sda` simulados. |
| **Local – Docker**                          | ubuntu:22.04 (privileged)         | 2.15+   | ✅ OK     | Necesita `--privileged` para permitir `parted/udev`. |
| **AWS EC2**                                 | Ubuntu 22.04 LTS (t3/t3a)         | 2.15+   | ✅ OK     | Discos `nvme1n1`; ver udev/partprobe en notas. |
| **AWS EC2**                                 | Ubuntu 20.04 LTS                   | 2.15+   | 🟡 Pend.  |                                         |
| **AWS EC2**                                 | Amazon Linux 2 / 2023              | 2.15+   | 🟡 Pend.  | Ajustar gestor de paquetes (yum/dnf). |
| **GCP / Azure**                             | Ubuntu 22.04 LTS                   | 2.15+   | 🟡 Pend.  |                                         |
| **Bare-metal / VMware / Proxmox**           | Debian 12                          | 2.15+   | 🟡 Pend.  |                                         |
| **RHEL / Rocky / Alma**                     | 8.x / 9.x                          | 2.15+   | 🟡 Pend.  | Cambiar gestor de paquetes a `yum/dnf`. |

### Cómo reproducir las pruebas locales

**Vagrant (VirtualBox)**
```bash
vagrant init ubuntu/jammy64
# En el Vagrantfile, añade un disco extra o usa un box que exponga /dev/sdb
vagrant up
ansible-playbook -i inventory vagrant_storage.yml -l default
```

**Docker (privileged)**
```bash
docker run --rm -it --privileged -v $(pwd):/work -w /work ubuntu:22.04 bash
# dentro del contenedor:
apt-get update && apt-get install -y ansible parted util-linux udev
ansible-playbook -i inventory docker_storage.yml
```

> **Tip:** Si el entorno usa NVMe (ej. EC2), los nodos de partición serán `/dev/nvme1n1p1`.
> En SATA/virtio serán `/dev/sdb1`. El rol detecta ambos casos automáticamente.

---

