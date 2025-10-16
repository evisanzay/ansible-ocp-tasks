# Ansible OpenShift Bare Metal Tasks

Este repositorio contiene ejemplos de tareas de Ansible para:

1. Generar un `install-config.yaml` paramétrico para OpenShift 4.
2. Crear los manifiestos e ignition configs necesarios para la instalación.
3. Preparar imágenes ISO de CoreOS con configuración de red estática e ignition embebida e inyectarlas en nodos HP a través de iLO.
4. Gestionar máquinas virtuales en VMware recreando un disco de 125 GB, recogiendo la MAC real y reutilizando las tareas anteriores para generar manifiestos e ISOs.

## Variables principales

Las variables se definen en `vars/cluster.yml` y cubren:

- Datos básicos del clúster (`cluster_name`, `base_domain`, `pull_secret`, `ssh_key`).
- Parámetros de red estática y número de réplicas.
- Definición de nodos incluyendo interfaz, IP, gateway, DNS y credenciales de iLO.
- Tipo de plataforma (`platform`) para permitir elegir entre `baremetal`, `vmware` o `qemu`.
- Conexión a vCenter, datastore y parámetros del disco adicional cuando se ejecuta sobre VMware.

## Escenarios soportados

### Baremetal

No requiere pasos adicionales: establece `platform: baremetal` y define los nodos con su BMC y MAC preasignada.

### VMware

1. Establece `platform: vmware` y completa la sección `vmware` con las credenciales y ubicación de vCenter.
2. Opcionalmente define `vm_name` y ajustes particulares de disco/datastore por nodo.
3. El rol `vmware_prepare` detecta automáticamente si Ansible se ejecuta dentro de una VM de VMware, elimina el disco indicado y lo recrea con 125 GB.
4. Obtiene la MAC de la tarjeta de red desde vCenter y la integra en la definición de nodos antes de generar `install-config`, manifiestos e ISOs.

### QEMU/KVM

1. Establece `platform: qemu` y define las listas `qemu.masters` y `qemu.workers` con los nodos que formarán el clúster.
2. El rol `qemu_prepare` valida que el host de automatización se esté ejecutando sobre QEMU/KVM, genera la lista completa de nodos a partir de dichas listas y actualiza los contadores de réplicas.
3. Con la lista resultante se vuelven a ejecutar las tareas comunes para generar `install-config`, manifiestos e ISOs personalizadas.

## Uso

```bash
ansible-playbook playbooks/main.yml
```

El playbook:

- (VMware) Valida la ejecución sobre VMware, recrea el disco adicional, obtiene la MAC real y actualiza la definición de nodos.
- (QEMU/KVM) Verifica que el controlador se ejecuta sobre QEMU/KVM, divide los nodos entre masters y workers y recompone la lista general antes de continuar.
- Crea `install-config.yaml` y los manifiestos/ignition correspondientes en `{{ install_config_output_dir }}`.
- Descarga y personaliza la ISO de CoreOS para cada nodo con red estática e ignition.
- Monta la ISO personalizada en cada host a través de iLO ignorando certificados autofirmados.

## Dependencias

Se requiere tener instalado Ansible y las colecciones necesarias:

```bash
pip install ansible
ansible-galaxy collection install community.general community.vmware
```
