# Devops Engineer Challenge

## Diseño de solución de monitorización centralizada

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

> Nota: Sube tu solución a este repositorio, incluyendo la explicación en Markdown para facilitar su lectura.

## Diseño y herramientas de la solución

La solución propuesta se basa en herramientas open source conocidas y maduras, pensadas para ser desplegadas de manera automatizada, escalable y replicable.

Para la recopilación y centralización de de información emplearemos **Prometheus** y para su visualización utilizaremos **Grafana** en un servidor dedicado a la monitorización, mientras que los nodos monitorizados utilizarán **Node Exporter** para exponer sus métricas.

Para la agregación y consulta de logs de las herramientas que conformen el sistema se propone integrar **Loki** en el servidor de monitorización y **Promtail** como agente en nodo de la infraestructura.

## Gestión de la seguridad y autenticación

Se habilitará HTTPS en todos los servidores mediante **certificados TLS**:

- **Prometheus y Grafana** tendrán configurados certificados de manera directa
- **Node Exporter** no soporta de manera nativa autenticación, pero podemos exponer el servicio detrás de un **servidor nginx** que se encargue de gestionar los certificados.

La seguridad se basará en autenticación mutua (mTLS): tanto el servidor de monitorización como los nodos de los agentes tendrán certificados únicos.

Además, se configurará firewall en todos los nodos para restringir el tráfico a los puertos estrictamente necesarios.

## Despliegue automatizado

La solución se desplegará mediante **Ansible**, que permite que las tareas de despliegue sean replicables y automatizadas. Como toda la automatización se realiza a través de SSH, y como se asume que todos los nodos corren Ubuntu 24.04, basta con tener acceso a los nodos mediante una clave pública para que el despliegue se realice de manera homogénea, con independencia de que los servidores sean on-premise o formen parte de un proveedor cloud.

Se elaborarán playbooks para el despliegue y configuración de todas las herramientas y proxies, así como para la gestión de los certificados TLS.

## Retención de datos y escalabilidad

La retención de datos en Prometheus es, por defecto, de 15 días, pero es posible configurarla en función de las necesidades del proyecto, así como limitar el tamaño total del almacenamiento.

Si el proyecto requiere dividir la carga de Prometheus entre varios nodos, es posible **federar varias instancias** entre sí, de manera que cada una de ellas recolecte datos de un subconjunto de los nodos monitorizados.

En caso de requerir una retención de datos a muy largo plazo y de gran capacidad, **Thanos** es una solución open source que se utiliza comúnmente para este propósito.

En cuanto a Grafana, es posible escalar horizontalmente y utilizar balanceadores de carga entre nodos, aunque no es un requerimiento que consideramos en esta prueba de concepto.

## Prueba de concepto de la solución propuesta
