# Ansible OpenShift Bare Metal Tasks

Este repositorio contiene un ejemplo completo para generar la configuración necesaria para instalar OpenShift 4 sobre bare metal, laboratorio con QEMU/KVM o masters en VMware + workers físicos. Incluye las tareas para construir el `install-config.yaml`, crear los manifiestos/ignitions, personalizar ISOs de CoreOS y validar conectividad con balanceadores.

## Contenido del repo
- `playbooks/main.yml`: playbook de entrada que orquesta todos los roles.
- `vars/cluster.yml`: única fuente de variables de ejemplo; aquí se selecciona el entorno, nodos, red, credenciales y rutas de salida.
- `roles/`:
  - `environment_prepare`: valida el entorno elegido, propaga plataforma, nodos y balanceadores.
  - `qemu_prepare`: normaliza nodos de laboratorio, asegura que el controlador corre en QEMU/KVM y calcula réplicas.
  - `vmware_prepare`: valida que se ejecute sobre VMware, recrea el disco de 125 GB en cada VM, lee la MAC real y la inyecta en la definición de nodos.
  - `load_balancer_check`: comprueba conectividad a los balanceadores definidos.
  - `install_config`: genera `install-config.yaml` y ejecuta `openshift-install create manifests/ignition`.
  - `coreos_iso`: descarga la ISO base, genera configuración de red estática, personaliza una ISO por nodo e intenta montarla por iLO cuando hay BMC.
- Plantillas:
  - `roles/install_config/templates/install-config.yaml.j2`: plantilla del `install-config.yaml` con soporte para bare metal, VMware/QEMU (modo none) y redes estáticas/bonding.
  - `roles/coreos_iso/templates/static-network.nmconnection.j2`: conexión de NetworkManager usada al personalizar la ISO.

## Flujo de ejecución
1. `environment_prepare` carga la sección del entorno seleccionado en `vars/cluster.yml` (`environment_type`) y fija variables comunes (`platform`, `load_balancers`, `nodes`, `vmware`, `qemu`).
2. Según la plataforma:
   - `qemu_prepare` valida que el controlador esté en QEMU/KVM, genera la lista de masters/workers y calcula réplicas.
   - `vmware_prepare` comprueba que se ejecuta sobre VMware, recrea el disco adicional y sustituye la MAC detectada en cada VM.
3. `load_balancer_check` confirma conectividad a los puertos definidos.
4. `install_config` crea el `install-config.yaml`, manifiestos e ignitions en `install_config_output_dir`.
5. `coreos_iso` descarga la ISO de RHCOS (una vez), genera archivos `*.nmconnection` por nodo y produce ISOs personalizadas en `iso_output_dir`. Si el nodo tiene `bmc`, monta la ISO vía iLO.

## Variables y configuración
Todo se controla desde `vars/cluster.yml`:
- `environment_type`: `lab` (QEMU/KVM) o `nopro` (VMware + físicos).
- `environments.*`: define nodos, red, balanceadores y configuración específica de la plataforma elegida.
- `cluster_name`, `base_domain`, `network`, `master_replicas`, `worker_replicas`: parámetros globales usados por la plantilla del install-config.
- `pull_secret`, `ssh_key`: se leen por defecto del home del usuario; ajusta las rutas si usas ubicaciones distintas.
- `coreos_iso_url`, `iso_output_dir`, `install_config_output_dir`: rutas de entrada/salida para ISOs y manifiestos.

### Dónde tocar según la necesidad
- **Cambiar nodos o red**: editar `vars/cluster.yml` en la sección del entorno y ajustar `network`/`nodes` (interfaces, bonding, gateway, DNS, BMC, etc.).
- **Plantilla del install-config**: modificar `roles/install_config/templates/install-config.yaml.j2` si necesitas nuevas opciones de plataforma o tweaks en `networkConfig`.
- **Plantilla de red estática para la ISO**: editar `roles/coreos_iso/templates/static-network.nmconnection.j2` para cambiar parámetros de NetworkManager.
- **Comandos o validaciones de cada rol**: actualizar los `tasks/main.yml` dentro de cada rol según corresponda (por ejemplo, cambiar el tamaño del disco en `roles/vmware_prepare/tasks/main.yml`).

## Requisitos previos
- Python 3 y Ansible.
- Binarios de `openshift-install` y `coreos-installer` disponibles en `PATH` en el host de automatización.
- Colecciones Ansible necesarias:
  ```bash
  pip install ansible
  ansible-galaxy collection install community.general community.vmware
  ```
- Acceso de red a vCenter (para el escenario VMware) y a los balanceadores declarados.

## Cómo ejecutar
1. Ajusta `vars/cluster.yml` al entorno deseado (al menos `environment_type`, nodos, red y credenciales).
2. Ejecuta el playbook principal desde la raíz del repo:
   ```bash
   ansible-playbook playbooks/main.yml
   ```
3. Resultados esperados:
   - Manifiestos e ignitions en `install_config_output_dir` (por defecto `./install`).
   - ISOs personalizadas por nodo en `iso_output_dir` (por defecto `./iso`).
   - Montaje remoto de las ISOs en nodos con BMC/iLO definido.

## Notas para troubleshooting rápido
- Si falta alguna variable requerida, los roles lanzan `assert` con mensajes en castellano.
- Para verificar conectividad a balanceadores antes de ejecutar todo, puedes lanzar solo ese rol:
  ```bash
  ansible-playbook playbooks/main.yml --tags load_balancer_check
  ```
- Los comandos `openshift-install` y `coreos-installer` deben estar instalados en el equipo que corre Ansible; de lo contrario las tareas fallarán.
