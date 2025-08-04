# Ansible OpenShift Bare Metal Tasks

Este repositorio contiene ejemplos de tareas de Ansible para:

1. Generar un `install-config.yaml` paramétrico para OpenShift 4.
2. Preparar imágenes ISO de CoreOS con configuración de red estática e inyectarlas en nodos HP a través de iLO.

## Variables principales

Las variables se definen en `vars/cluster.yml` y cubren:

- Datos básicos del clúster (`cluster_name`, `base_domain`, `pull_secret`, `ssh_key`).
- Parámetros de red estática y número de réplicas.
- Definición de nodos incluyendo interfaz, IP, gateway, DNS y credenciales de iLO.
- Tipo de plataforma (`platform`) para permitir futuros cambios (por ejemplo `none`).

## Uso

```bash
ansible-playbook playbooks/main.yml
```

El playbook:

- Crea `install-config.yaml` en `{{ install_config_output_dir }}`.
- Descarga y personaliza la ISO de CoreOS para cada nodo con red estática.
- Monta la ISO personalizada en cada host a través de iLO ignorando certificados autofirmados.

## Dependencias

Se requiere tener instalado Ansible y la colección `community.general` para usar el módulo `hpilo_boot`:

```bash
pip install ansible
ansible-galaxy collection install community.general
```
