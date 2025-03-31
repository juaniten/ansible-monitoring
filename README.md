# Ansible Monitoring Deployment

Este repositorio recoge el trabajo de un DevOps challenge desarrollado durante un fin de semana.

## Planteamiento: diseño de solución de monitorización centralizada

Queremos construir una solución de **monitorización centralizada** para nuestra infraestructura, con las siguientes premisas:

1. Nuestra infraestructura está dispersa geográficamente, con servidores **on-premise** en diferentes localizaciones y también en **varios proveedores cloud**.
2. La comunicación entre la solución de monitorización y los diferentes servidores debe realizarse de manera **segura** a través de **Internet**, utilizando **direccionamiento IPv6**.
3. Necesitamos que la solución sea **escalable** y **automatizada**.
4. Todos los servidores utilizan **Ubuntu 24.04**, por lo que la solución debe ser compatible con este sistema operativo.

Basándonos en estos requisitos,

- ¿Qué **diseño y herramientas** propones para implementar esta solución?
- ¿Cómo gestionarías la **seguridad y autenticación** de los agentes de monitoreo?
- ¿Cómo harías el **despliegue automatizado** de la solución en todos los servidores?
- ¿Cómo manejarías la **retención de datos** y la **escalabilidad** de la plataforma?
- ¿Podrías construir una **pequeña prueba de concepto** para validar la solución propuesta?

## Diseño y herramientas de la solución

La solución propuesta se basa en herramientas open source conocidas y maduras, pensadas para ser desplegadas de manera automatizada, escalable y replicable.

Para la recopilación y centralización de información emplearemos **Prometheus** y para su visualización utilizaremos **Grafana** en un servidor dedicado a la monitorización, mientras que los nodos monitorizados utilizarán **Node Exporter** para exponer sus métricas.

Para la agregación y consulta de logs de las herramientas que conformen el sistema se propone integrar **Loki** en el servidor de monitorización y **Promtail** como agente en los nodos de la infraestructura.

## Gestión de la seguridad y autenticación

Se habilitará HTTPS en todos los servidores mediante **certificados TLS**:

- **Prometheus y Grafana** tendrán configurados certificados de manera directa
- **Node Exporter** no soporta de manera nativa autenticación, pero podemos exponer el servicio detrás de un **servidor nginx** que se encargue de gestionar los certificados.

La seguridad se basará en autenticación mutua (mTLS): tanto el servidor de monitorización como los nodos de los agentes tendrán certificados únicos.

En la prueba de concepto he utilizado certificados auto-firmados y, como consecuencia, algunas de las herramientas, como Grafana, han tenido que ser configuradas para desactivar la verificación de los certificados. En un sistema en producción, lógicamente, sería necesario emplear certificados emitidos por una Autoridad Certificadora.

Además, se configurará firewall en todos los nodos para restringir el tráfico a los puertos estrictamente necesarios.

## Despliegue automatizado

La solución se desplegará mediante **Ansible**, que permite que las tareas de despliegue sean replicables y automatizadas. Como toda la automatización se realiza a través de SSH, y como se asume que todos los nodos corren Ubuntu 24.04, basta con tener acceso a los nodos mediante una clave pública para que el despliegue se realice de manera homogénea, sin importar que los servidores sean on-premise o formen parte de un proveedor cloud.

Se elaborarán playbooks para el despliegue y configuración de todas las herramientas y proxies, así como para la gestión de los certificados TLS.

## Retención de datos y escalabilidad

La retención de datos en Prometheus es, por defecto, de 15 días, pero es posible configurarla en función de las necesidades del proyecto, así como limitar el tamaño total del almacenamiento.

Si el proyecto requiere dividir la carga de Prometheus entre varios nodos, es posible **federar varias instancias** entre sí, de manera que cada una de ellas recolecte datos de un subconjunto de los nodos monitorizados.

En caso de requerir una retención de datos a muy largo plazo y de gran capacidad, **Thanos** es una solución open source que se utiliza comúnmente para este propósito.

## Prueba de concepto de la solución propuesta

En la solución aportada al repositorio he considerado un conjunto más limitado de las herramientas propuestas. El objetivo ha sido poner en marcha un **sistema mínimo funcional** mediante **playbooks de Ansible** que ilustre lo fundamental, por lo que hay varios detalles de seguridad que pasamos por alto y la potencial complejidad del sistema de monitorización de métricas se ha reducido a tan solo una.

### Configurar el inventario de Ansible

En `inventory.ini` tenemos dos grupos de nodos:

- `monitoring_server` contendrá al servidor central de monitorización.
- `remote_servers` contendrá todos los servidores que se desean monitorizar.

> Es necesario sustituir las IP del inventario por las de los servidores donde se deseen realizar los despliegues.

### Preparar los servidores para el despliegue

Para esta pequeña prueba de concepto he utilizado máquinas virtuales en mi máquina local empleando **multipass**, pero es posible aprovisionar y emplear servidores en cualquier proveedor cloud en su lugar.

#### Desplegar máquinas virtuales con multipass (opcional)

Si ya se tienen servidores disponibles, se puede obviar este paso. Para instalar multipass en Ubuntu 24.04:

```bash
sudo snap install multipass
```

Creamos algunas máquinas virtuales: utilizaremos la primera para el servidor de monitorización y el resto serán los nodos a monitorizar.

```bash
multipass launch 24.04 -n vm1
multipass launch 24.04 -n vm2
multipass launch 24.04 -n vm3
```

Comprobamos que las VM se han lanzado correctamente

```bash
multipass list
```

Tomamos nota de las IPs para configurar el inventario de Ansible:

```
Name         State    IPv4           Image
vm1         Running  10.45.153.106   Ubuntu 24.04 LTS
vm2         Running  10.45.153.201   Ubuntu 24.04 LTS
vm3         Running  10.45.153.182   Ubuntu 24.04 LTS
```

#### Iniciar servicios de OpenSSH en todos los servidores

Es necesario que todos los servidores hayan sido aprovisionados con un servicio de OpenSSH en funcionamiento. Si no lo están, en Ubuntu 24.04:

```bash
sudo apt update && sudo apt install -y openssh-server
```

Activamos e iniciamos el servicio SSH:

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

Para comprobar que el servicio esté corriendo:

```bash
sudo systemctl status ssh
```

#### Crear un usuario para las operaciones de despliegue (opcional)

Nuevamente, este es un paso opcional. He utilizado un playbook llamado `bootstrap.yml` para realizar algunas labores previas en todos los servidores:

- Actualizar todos los paquetes.
- Crear un usuario y grupo para realizar las tareas de despliegue, asegurar que cuenta con permisos de root y añadir una clave pública para Ansible.

Si se prefiere utilizar cualquier otro usuario, puede obviarse este paso y configurarse la clave pública y el nombre de usuario de Ansible en `ansible.cfg`.

#### Ejecutar el playbook `site.yml`

Para desplegar toda la propuesta, solo hay que ejecutar:

```bash
ansible-playbook site.yml
```

Este playbook realizará las siguientes tareas:

- Actualizar paquetes en todos los hosts Ubuntu. Si más adelante se desea dar soporte a otros sistemas operativos, será sencillo añadir las tareas necesarias en el playbook.

- Para el servidor de monitorización, se aplican las tareas asociadas a los siguientes roles:

  - **tls_monitoring**: genera una clave privada, la petición de firma del certificado y un certificado auto-firmado.
  - **prometheus**: inicia el servicio de Prometheus y configura los certificados TLS. Además, copia una plantilla de configuración muy simple en que se encuentran las direcciones IP de los nodos que exportarán métricas: \(es necesario sustituirlas por las direcciones IP de los servidores en uso\).
  - **grafana**: agrega el repositorio de Grafana, instala el servicio, configura los certificados TLS, configura Prometheus como fuente de datos y carga una plantilla de dashboard mínima que monitorizará el uso de CPU de los nodos.

- Para los nodos monitorizados se aplican las tareas de los siguientes roles:
  - **tls_node_exporter:** de manera similar al caso del servidor de monitorización, genera una clave privada, la petición de firma del certificado y un certificado auto-firmado para cada nodo que expone métricas.
  - **nginx_remote_servers**: instala y crea un servicio de nginx, importa una plantilla de configuración e inicia el servicio como proxy inverso para Node Exporter.
  - **node_exporter**: instala e inicia Node Exporter para exponer un endpoint con las métricas para Prometheus.

## Conclusión

Esta prueba de concepto muestra la viabilidad de una solución de monitorización, basada en herramientas de código abierto como Prometheus, Grafana y Node Exporter. Mediante Ansible logramos un despliegue automatizado y reproducible en servidores distribuidos, garantizando una instalación replicable y facilitando su mantenimiento.

Si bien esta implementación inicial solo establece una base mínima funcional, hay varios aspectos a considerar para ampliar su funcionalidad y mejorar su seguridad, escalabilidad y automatización:

1. **Seguridad y autenticación:** actualmente, se emplean certificados auto-firmados para la autenticación mutua (mTLS), lo que en un entorno de producción debería ser reemplazado por certificados emitidos por una Autoridad Certificadora. También sería recomendable implementar mecanismos de rotación y revocación de certificados para reforzar la seguridad. En esta prueba utilizamos usuarios y contraseñas por defecto que, naturalmente, sería necesario cambiar. En su lugar, también podríamos utilizar OAuth para autenticar usuarios en Prometheus y Grafana.

2. **Escalabilidad:** en la prueba de concepto, Prometheus se ejecuta en una única instancia, lo que puede convertirse en un cuello de botella en entornos con una alta cantidad de métricas. Para abordar esto, podría valorarse la federación de varias instancias y distribuir la carga entre ellas, y se podría integrar Thanos para permitir almacenamiento a largo plazo. Del mismo modo, Grafana podría escalar horizontalmente mediante Kubernetes y un balanceador de carga.

3. **Automatización y mantenimiento:** podríamos incluir herramientas como Ansible Vault para manejar credenciales de forma segura. También sería recomendable desarrollar mecanismos de actualización y rollback de las herramientas de monitorización sin afectar la disponibilidad del sistema.

4. **Gestión de logs:** además de integrar Loki y Promtail, se podría configurar una estrategia de retención y almacenamiento distribuido para los logs, con reglas de alerta en Grafana y análisis centralizado.
