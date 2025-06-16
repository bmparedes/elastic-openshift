# Elasticsearch v2.5 en OpenShift

Esta guía detalla los pasos esenciales para configurar el Operador de Elasticsearch v2.5 y sus componentes relacionados en un clúster de OpenShift. Cubriremos la creación de proyectos, la configuración de **ServiceAccounts**, las **Security Context Constraints (SCCs)** y el **Control de Acceso Basado en Roles (RBAC)** para una operación correcta.

---

## Configuración Inicial y Componentes Principales

Comenzaremos preparando el entorno y desplegando los elementos fundamentales para tu stack de Elasticsearch.

### 1. Creación del Proyecto Dedicado

Primero, crea un nuevo proyecto de OpenShift para albergar todos tus componentes de Elasticsearch. Esto ayuda con la organización y el aislamiento.

```bash
oc new-project elastic
```

### 2. Estrategia de ServiceAccounts

Para mejorar la seguridad y tener un control más granular, crearemos **ServiceAccounts** dedicadas para cada componente principal de tu despliegue de Elastic. Esto te permite asignar permisos específicos a cada carga de trabajo.

* `kibana`: Para el despliegue de Kibana.
* `elastic`: Para el clúster de Elasticsearch en sí.
* `logging-forwarding`: Para los componentes encargados de reenviar logs (por ejemplo, desde el sistema de logging interno de OpenShift).
* `beats`: Para varios Elastic Beats (como Filebeat, Metricbeat).
* `apm-server`: Para el Servidor APM.

### 3. Configuración de Security Context Constraints (SCCs)

Las **SCCs** son cruciales en OpenShift para controlar los permisos de los pods. Asignaremos SCCs específicas a nuestros **ServiceAccounts** según sus necesidades.

| ServiceAccount    | SCC Asignada      | Propósito                                                                      |
| :---------------- | :---------------- | :----------------------------------------------------------------------------- |
| `elastic`         | `restricted-v2`   | SCC estándar para la mayoría de las aplicaciones, ofreciendo seguridad robusta. |
| `beats`           | `privileged`      | Requerida para los Beats que necesitan acceder a rutas del host (por ejemplo, `/var/log` para Filebeat). |
| `logging-fwd`     | `privileged`      | Necesaria si los pods de esta ServiceAccount requieren acceso al host para la recolección de logs. |
| `apm-server`      | `anyuid`          | Permite que el pod se ejecute con cualquier ID de usuario, a menudo necesario para imágenes preconstruidas. |

---

## Creación de ServiceAccounts y Aplicación de Permisos

Ahora, crearemos estas **ServiceAccounts** y aplicaremos las **SCCs** y roles **RBAC** necesarios.

### ServiceAccount y SCC para APM Server

```bash
oc create serviceaccount apm-server -n elastic
oc adm policy add-scc-to-user anyuid -z apm-server -n elastic
```

### ServiceAccount y SCC para Elastic Beats (por ejemplo, Filebeat)

```bash
oc create serviceaccount filebeat -n elastic
oc adm policy add-scc-to-user privileged -z filebeat -n elastic
```

### ServiceAccount y SCC para Reenvío de Logs

```bash
oc create serviceaccount logging-fwd -n elastic
oc adm policy add-scc-to-user privileged -z logging-fwd -n elastic
```


### Otorgar Rol `developer` al ServiceAccount del Operador de Elastic

Si el **ServiceAccount** del Operador de Elastic necesita permisos más amplios dentro del proyecto `elastic` (por ejemplo, para crear y gestionar recursos), puedes otorgarle el rol `developer`. Esto se suele hacer para el propio **ServiceAccount** del operador en su respectivo `namespace`, pero si tienes un **ServiceAccount** específico para el *operador dentro del proyecto `elastic`*, lo harías aquí.

```bash
oc adm policy add-role-to-user elastic-operator developer -n elastic
```

### Creación de RBAC para Monitoreo y Extracción

Para componentes como Beats y los reenviadores de logs internos, a menudo necesitarás roles **RBAC** específicos que les permitan monitorear recursos y extraer logs. Estos roles definen *qué* pueden hacer.

**Ejemplo de RBAC para Beats y Reenviadores de Logs:**

Necesitarás definir recursos `Role` y `RoleBinding` para estas **ServiceAccounts**, otorgándoles permisos como `get`, `list`, `watch` sobre Pods, Namespaces, Nodos y, potencialmente, acceso a Secrets o ConfigMaps para la configuración.

Un ejemplo básico para `logging-fwd` y `filebeat` implicaría roles que permitan la lectura de recursos del clúster relevantes para la recolección de logs:


Aplicarías estas definiciones YAML usando `oc apply -f <nombre_del_archivo.yaml>`.


## Verificación

Después de desplegar tus pods, puedes verificar que están utilizando la **SCC** correcta.

### Verificar el Uso de SCC en los Pods

Usa este comando para ver qué **SCC** está utilizando cada pod en el proyecto actual:

```bash
oc get pod -o go-template='{{range .items}}{{$scc := index .metadata.annotations "openshift.io/scc"}}{{.metadata.name}}{{" scc:"}}{{range .spec.containers}}{{$scc}}{{" "}}{{"\n"}}{{end}}{{end}}'
```

### Revision de SCC

```bash
oc adm policy who-can use scc anyuid | grep -i elastic
oc adm policy who-can use scc privileged | grep -i elastic
oc adm policy who-can use scc restricted-v2 | grep -i elastic
```

### Verificacion de rbac

```bash
oc get rolebinding -n <namespace> --output=json | jq '.items[] | select(.subjects[]?.name=="<nombre-del-sa>")'
```