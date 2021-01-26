# AMQ BROKER WITH DISK PERSISTENCE

This chart handles the deployment of an AMQ broker with persistence and optional TLS encryption.
The broker is created with one multiplexed acceptor (configurable via values), TLS is an optional feature.

|NAME                              | DESCRIPTION                                              | DEFAULT VALUE |
|----------------------------------|----------------------------------------------------------|----------------|
| kind                             | Deploy broker as Deployment or StatefulSet               | `Deployment` |
| application.name                 | The name for the application.                            | `amq-broker-persistence` |
| application.amq_broker_version   | Broker Image tag                                         | `7.7` |
| application.amq_broker_image     | Broker Image name                                        | `registry.redhat.io/amq7/amq-broker` |
| application.pullPolicy           | Pull policy                                              | `IfNotPresent` |
| application.volume_capacity      | Size of persistent volume                                | `1G` |
| service.console                  | Jolokia console port and configuration | See values.yaml |
| service.acceptors                | Array of acceptors. Only the multiplex is exposed by default | See values.yaml |
| tls.keystore                     | Name of the keystore file                                | See values.yaml |
| tls.truststore                   | Name of the truststoreile                                | See values.yaml |
| tls.keystore_password            | Password to unlock the keystore on container boot        | See values.yaml |
| tls.truststore_password          | Password to unlock the truststore on container boot      | See values.yaml |
| nodeport.enabled                 | Create node port to expose AMQ to clients outside of the cluster | `30002` |
| nodeport.port                    | Node port number used when enabled | `30002` |
| passthrough_route.enabled        | Create a passthrough route to allow inbound TCP/SNI connections to a TLS-enabled broker | `False` |
| parameters.tls_enabled           | Enable or disable TLS support for acceptors | `false` |
| parameters.amq_protocols         | Protocols to configure, separated by commas. Allowed values are: `openwire`, `amqp`, `stomp`, `mqtt` and `hornetq`. | `openwire,amqp,stomp,mqtt,hornetq` |
| parameters.amq_broker_name       | Broker name (TODO is this used? Same as application.name ) | `broker` |
| parameters.amq_admin_role        | Admin role | `admin` |
| parameters.amq_global_max_size   | Maximum amount of memory which message data may consume ( TODO: 100 gb as default is a bit high for most systems) | `"100 gb"` |
| parameters.amq_require_login     | Determines whether or not the broker will allow anonymous access, or require login | `False` |
| parameters.amq_extra_args        | Extra arguments for broker creation  | `` |
| parameters.amq_anycast_prefix    | Anycast prefix applied to the multiplexed protocol port 61616   | `jmx.queue.` |
| parameters.amq_multicast_prefix  | Multicast prefix applied to the multiplexed protocol port 61616   | `jmx.topic.` |
| parameters.amq_enable_metrics_plugin | Whether to enable artemis metrics plugin | `False` |
| parameters.amq_journal_type      | Journal type to use; aio or nio supported | `nio` |
| parameters.amq_data_dir          | Directory for storing data | `/opt/amq/data` |
| templates.service                | Template for service name | See values.yaml |
| templates.deployment             | Template for deployment name | See values.yaml |
| templates.route                  | Template for route name | See values.yaml |
| templates.broker_image           | Template for image name | See values.yaml |
| templates.override_cm            | Template for ConfigMap name containing overrides | See values.yaml |
| templates.config_cm              | Template for ConfigMap nggame | See values.yaml |
| templates.app_secret             | Template for name of a secret containing credential data such as users and passwords | See values.yaml |
| templates.pvc_name               | Template for persistent volume name | See values.yaml |
| security.enabled                 | Enabled security | `true` |
| security.secrets                 | Array of names of additional secrets to mount into /opt/amq/conf  | [] |
| security.createSecret            | Create secret with users and passwords. Disable when secrets is created outside of this chart. For example by ExternalSecret | `true` |
| security.jaasUsers.key           | Specify the key (filename) of the user/password file in the secret | `artemis-users.properties` |
| admin.user                       | Admin user. Mandatory even if security.createSecret is `false`) | `admin` |
| admin.password                   | Admin password. Optional. Only used if security.createSecret is `true` | `password` |
| admin.roles                      | Array of role names to assign to admin | `[ admin ]` |
| users                            | Array of additional users. Only used if security.createSecret is `true` else users are expected to be defined in secret. | [] |
| queue.defaults                   | Default values for queues parameters | [] |
| queue.addresses                  | Array of queues to create. | [] |
| metrics.enabled                  | Enable metrics in AMQ and let Prometheus collect metrics using ServiceMonitor | `false` |
| metrics.jvm_memory               | Enable JVM memory metrics | `true` |
| metrics.jvm_gc                   | Enable JVM garbage collection statistics in metrics | `false` |
| metrics.jvm_threads              | Enable JVM Thread statistics | `false` |
| metrics.servicemonitor.port      | Collect metrics from this port. Default is the management port.  | `8161` |
| metrics.servicemonitor.interval  | Metrics are collected with fixed interval.  | `20s` |
| resources                        | Kubernetes limits and resources to attach to pod templates | See values.yaml |

## INSTALLATION

The most basic deployment can be performed by following these steps:

### Non-TLS AMQ Brokers

- Customize the application name in `values.yaml`:

```
application:
  name: amq-broker-persistence-ssl
  [...]
  volume_capacity: "1G"
```

```
parameters:
  [...]
  amq_data_dir: "/opt/amq/data"
  tls_enabled: false
  [...]
```

If needed, the broker can be consumed by clients running outside OCP by deploying a NodePort resource:

```
nodeport:
  [...]
  enabled: true
```

Since no TLS passthrough is possible without proper tls support, the passthrough_route should be disabled:

```
passthrough_route:
  enabled: false
  [...]
```

### TLS-enabled AMQ Brokers

- Create (or import) a keystore/truststore pair for this broker: look [here](https://github.com/mcaimi/amq-custom-templates-openshift/blob/master/README.md) for an howto. Put the files under `tls/` and update the tls section in `values.yaml`:

```
tls:
  keystore: keystore.ks
  truststore: keystore.ts
  keystore_password: kspwd
  truststore_password: tspwd
```

- Customize the application name in `values.yaml`:

```
application:
  name: amq-broker-persistence-ssl
  [...]
  volume_capacity: "1G"
```

```
parameters:
  [...]
  amq_data_dir: "/opt/amq/data"
  [...]
```

For TLS-enabled brokers, both the NodePort and the Passthrough route options are working. Both can be enabled at the same time.

### Common Setup

The application name will be used as a prefix for most of the objects deployed by the Chart itself.

- Update the Admin user name and password in `values.yaml`

```
admin:
  user: admin
  password: password
  role: admin
``` 

- If needed, enable and choose a node port TCP value and corresponding service for the external service in `values.yaml`:

```
nodeport:
  port: 30003
  service: multiplex-ssl
  enabled: true
```
this port needs to be in the allowed NodePort range set up in the kubelet (typically in the range 30000-32768)

- Install the Chart under your preferred project

```
$ oc new-project amq-demo-persistence-ssl
$ helm install amq-persistence-ssl .
```

After a while, the broker should be up and running:

```
$ oc get all
NAME                               READY   STATUS      RESTARTS   AGE
pod/amq-broker-basic-dc-1-deploy   0/1     Completed   0          13m
pod/amq-broker-basic-dc-1-trrsw    1/1     Running     0          13m

NAME                                          DESIRED   CURRENT   READY   AGE
replicationcontroller/amq-broker-basic-dc-1   1         1         1       13m

NAME                                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
service/amq-broker-basic-svc-amqp      ClusterIP   172.25.221.5    <none>        5672/TCP          13m
service/amq-broker-basic-svc-jolokia   ClusterIP   172.25.51.89    <none>        8161/TCP          13m
service/amq-broker-basic-svc-mqtt      ClusterIP   172.25.14.64    <none>        1883/TCP          13m
service/amq-broker-basic-svc-stomp     ClusterIP   172.25.196.97   <none>        61613/TCP         13m
service/amq-broker-basic-svc-tcp       ClusterIP   172.25.56.105   <none>        61616/TCP         13m
service/artemis-nodeport-svc           NodePort    172.25.159.17   <none>        61616:30000/TCP   13m

NAME                                                     REVISION   DESIRED   CURRENT   TRIGGERED BY
deploymentconfig.apps.openshift.io/amq-broker-basic-dc   1          1         1         config

NAME                                                      HOST/PORT                                                  PATH   SERVICES                       PORT    TERMINATION   WILDCARD
route.route.openshift.io/amq-broker-basic-route-console   amq-broker-basic-route-console-amq-demo.apps-crc.testing          amq-broker-basic-svc-jolokia   <all>                 None
```

## ADDING QUEUES, USERS AND ROLES

To add multiple users to the broker edit the `users` section in `values.yaml`. For example, this setup here:

```
users:
  - name: demouser
    password: "demo"
    role: user
  - name: anotheruser
    password: "demo1"
    role: user
```

would be rendered by the Helm Chart into these two files:

- `artemis-users.properties`

```
    ## CUSTOMCONFIG
    
    # ADMIN USER
    admin = password
    
    # ADDITIONAL USERS
    demouser = demo
    anotheruser = demo1
```

- `artemis-roles.properties`

```
    ## CUSTOMCONFIG
    # ADMIN ROLE MAPPING
    admin = admin

    # ADDITIONAL ROLE MAPPING
    user = demouser
    user = anotheruser
```

Users and passwords may be stored in an existing secret instead of as clear text in the values.yaml: disable the creation of the built-in user secret and specify the name of an existing secret.

Set the `jaasUsers.key` to the filename used in the secret. Note that the filename have to be something different from `artemis-users.properties` as the default file will be mounted in the same directory in the container.  

For example: 
```
security:
  secrets:
    - broker-external-secret
  createSecret: false
  jaasUsers:
    key: my-secured-artemis-users.properties

```

*Note*, that the AMQ_USER and AMQ_PASSWORD *must* be set, as the broker still uses these environment parameters:

```
stringData:
  AMQ_USER: broker-admin
  AMQ_PASSWORD: mySecretPassword
  my-secured-artimis-users.properties: |
    # ADMIN USER
    broker-admin = mySecretPassword
    # ADDITIONAL USERS
    consumer-user = otherSecretPassword
type: Opaque
```

The `queues` section in `values.yaml` allows to add custom queues to the broker at install time. For example, this setup:

```
queues:
  defaults:
    [...]
  addresses:
    - name: demoQueue
      permissions:
        - grant: consume
          roles:
            - admin
            - user
        - grant: browse
          roles:
            - admin
            - user
        - grant: send
          roles:
            - admin
            - user
        - grant: manage
          roles:
            - admin
```

would result in this rendered section inside `broker.xml`:

```
           <security-setting match="demoQueue">
             <permission type="consume" roles="admin,user," />
             <permission type="browse" roles="admin,user," />
             <permission type="send" roles="admin,user," />
             <permission type="manage" roles="admin," />
           </security-setting>
```

the defaults section under the queues stanza contains the values set for every queue if not overridden on a per queue basis.

## Metering

An optional prometheus ServiceMonitor is shipped with the chart. See values.yaml (metering stanza) for configuration.
