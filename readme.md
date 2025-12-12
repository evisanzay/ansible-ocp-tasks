# Ansible OpenShift Bare Metal Tasks

Este repositorio contiene ejemplos de tareas de Ansible para:

1. Generar un `install-config.yaml` paramétrico para OpenShift 4.
2. Crear los manifiestos e ignition configs necesarios para la instalación.
3. Preparar imágenes ISO de CoreOS con configuración de red estática e ignition embebida e inyectarlas en nodos HP a través de iLO.
4. Gestionar máquinas virtuales en VMware recreando un disco de 125 GB, recogiendo la MAC real y reutilizando las tareas anteriores para generar manifiestos e ISOs.
5. Verificar la salud de los balanceadores externos antes de continuar con la instalación.

## Variables principales

Las variables se definen en `vars/cluster.yml` y cubren:

- Datos básicos del clúster (`cluster_name`, `base_domain`, `pull_secret`, `ssh_key`).
- Parámetros de red estática y número de réplicas.
- Definición de nodos incluyendo interfaz, IP, gateway, DNS, bonding y credenciales de iLO.
- Tipo de plataforma (`platform`) calculado automáticamente a partir del entorno elegido (`environment_type`).
- Conexión a vCenter, datastore y parámetros del disco adicional cuando se ejecuta sobre VMware.
- Lista de balanceadores a validar (dirección y puertos).

## Escenarios soportados

### Laboratorio (QEMU/KVM)

- Selecciona `environment_type: lab`.
- Usa `platform: qemu` y las listas `qemu.masters`, `qemu.workers` y `qemu.dns` para poblar los nodos.
- El rol `qemu_prepare` valida que el host de automatización se esté ejecutando sobre QEMU/KVM, genera la lista completa de nodos y actualiza los contadores de réplicas.
- Se validan los balanceadores definidos en `load_balancers` para confirmar conectividad (puertos 6443/22623 para API y 80/443 para apps en el ejemplo).

### Desarrollo/Preproducción (nopro)

- Selecciona `environment_type: nopro`.
- Define los masters en VMware y los workers físicos HP con iLO.
- El rol `environment_prepare` propaga la configuración de VMware, la lista completa de nodos (incluyendo los físicos) y los balanceadores externos.
- El rol `vmware_prepare` solo aplica las operaciones de disco y descubrimiento de MAC sobre los nodos que tienen sección `vmware`, permitiendo mezclar masters virtuales y workers baremetal.
- El archivo `static-network.nmconnection.j2` y el `install-config` soportan bonding activo/pasivo sobre dos interfaces.

## Uso

```bash
ansible-playbook playbooks/main.yml
```

El playbook:

- Selecciona el entorno (`lab` o `nopro`) y ajusta plataforma, nodos, balanceadores y configuración de VMware.
- (VMware) Valida la ejecución sobre VMware, recrea el disco adicional de 125 GB solo en los nodos virtuales, obtiene la MAC real y la integra en la definición de nodos.
- (QEMU/KVM) Verifica que el controlador se ejecuta sobre QEMU/KVM, divide los nodos entre masters y workers y recompone la lista general antes de continuar.
- Comprueba conectividad hacia los balanceadores externos declarados.
- Crea `install-config.yaml` y los manifiestos/ignition correspondientes en `{{ install_config_output_dir }}` con soporte de bonding.
- Descarga y personaliza la ISO de CoreOS para cada nodo con red estática e ignition.
- Monta la ISO personalizada en cada host a través de iLO ignorando certificados autofirmados.

## Dependencias

Se requiere tener instalado Ansible y las colecciones necesarias:

```bash
pip install ansible
ansible-galaxy collection install community.general community.vmware
```
