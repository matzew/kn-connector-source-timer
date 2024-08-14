# Knative connector timer-source

Knative eventing connector based on [Apache Camel Kamelets](https://camel.apache.org/camel-kamelets/).
The connector project creates a container image that is pushed into a registry so the image can be referenced in a Kubernetes deployment.

## Build the container image

The project uses Quarkus in combination with Apache Camel, Kamelets and Maven as a build tool.

You can use the following Maven commands to build the container image.

```shell
./mvnw package -Dquarkus.container-image.build=true
```

The container image uses the project version and image group defined in the Maven POM.
You can customize the image group with `-Dquarkus.container-image.group=my-group`.

By default, the container image looks like this:

```text
quay.io/openshift-knative/kn-connector-source-timer:1.0-SNAPSHOT
```

The project leverages the Quarkus Kubernetes and container image extensions so you can use Quarkus properties and configurations to customize the resulting container image.

See these extensions for details:
* https://quarkus.io/guides/deploying-to-kubernetes
* https://quarkus.io/guides/container-image

## Push the container image

```shell
./mvnw package -Dquarkus.container-image.build=true -Dquarkus.container-image.push=true
```

Pushes the image to the image registry defined in the Maven POM. 
The default registry is [quay.io](https://quay.io/).

You can customize the registry with `-Dquarkus.container-image.registry=localhost:5001` (e.g. when connecting to local Kind cluster).

In case you want to connect with a local cluster like Kind or Minikube you may also need to set `-Dquarkus.container-image.insecure=true`.

## Kubernetes manifest

The build produces a Kubernetes manifest in (`target/kubernetes/kubernetes.yml`).
This manifest holds all resources required to run the application on your Kubernetes cluster.

The Kubernetes manifest includes:

* Service
* Deployment
* SinkBinding

You can customize the Kubernetes resources in [src/main/kubenretes/kubernetes.yml](src/main/kubernetes/kubernetes.yml).
This is being used as a basis and Quarkus will generate the final manifest in `target/kubernetes/kubernetes.yml` during the build.

## Deploy to Kubernetes

You can deploy the application to Kubernetes with:

```shell
./mvnw package -Dquarkus.kubernetes.deploy=true
```

This connects to the current Kubernetes cluster that you are connected with (e.g. via `kubectl config set-context --current --namespace $1`).

You may change the target namespace with `-Dquarkus.kubernetes.namespace=my-namespace`.

## Kamelet source pipe

The source produces events for the Knative broker.
It uses a Pipe resource as the central piece of code to define how the Knative events are produced.

The Pipe is a YAML file located in [src/main/resources/camel/kn-connector-source-timer.yaml](src/main/resources/camel/kn-connector-source-timer.yaml)

_kn-connector-source-timer.yaml_
```yaml
apiVersion: camel.apache.org/v1
kind: Pipe
metadata:
  name: kn-connector-source-timer
spec:
  source:
    ref:
      apiVersion: camel.apache.org/v1
      kind: Kamelet
      name: timer-source
  sink:
    dataTypes:
      in:
        format: http-application-cloudevents
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: default
```

This connector uses the [timer-source](https://camel.apache.org/camel-kamelets/timer-source.html) Kamelet that produces events for the Knative broker.

The Pipe references the Kamelet as a source and connects to the Knative broker as a sink.

The name of the broker is always `default` because the actual broker URL is injected into the application via `SinkBinding` resource.
The SinkBinding injects a `K_SINK` environment variable to the deployment and the application uses the injected broker URL to send events to it.

This way the same container image can be used with different brokers.
It is only a matter of configuring the SinkBinding resource that connects the application with the Knative broker.

You can find a sample SinkBinding in `src/main/kubernetes/kubernetes.yml`

```yaml
apiVersion: sources.knative.dev/v1
kind: SinkBinding
metadata:
  annotations:
    sources.knative.dev/creator: connectors.knative.dev
  finalizers:
    - sinkbindings.sources.knative.dev
  labels:
    eventing.knative.dev/connector: timer-source
  name: kn-connector-source-timer
spec:
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: default
  subject:
    apiVersion: apps/v1
    kind: Deployment
    name: kn-connector-source-timer
```

## Configuration

Each Kamelet defines a set of properties.
The user is able to customize these properties when running a connector deployment.

### Environment variables

You can customize the properties via environment variables on the deployment:

* CAMEL_KAMELET_TIMER_SOURCE_MESSAGE=Hello
* CAMEL_KAMELET_TIMER_SOURCE_PERIOD=1000

You can set the environment variable on the running deployment:

```shell
kubectl set env deployment/kn-connector-source-timer CAMEL_KAMELET_TIMER_SOURCE_MESSAGE="I updated it..."
```

The environment variables that overwrite properties on the Kamelet source follow a naming convention:

* CAMEL_KAMELET_{{KAMELET_NAME}}_{{PROPERTY_NAME}}={{PROPERTY_VALUE}}

The name represents the name of the Kamelet source as defined in the [Kamelet catalog](https://camel.apache.org/camel-kamelets/).

### ConfigMap and Secret

You may also mount a configmap/secret to overwrite Kamelet properties with values from the configmap/secret resource.

Given a configmap named `my-source-config` in Kubernetes that has two entries:

```properties
message = Knative rocks!
period = 3000
```

You can reference the values of the configmap in the environment variables like this:

* CAMEL_KAMELET_TIMER_SOURCE_MESSAGE={{configmap:my-source-config/message}}
* CAMEL_KAMELET_TIMER_SOURCE_PERIOD={{configmap:my-source-config/period}}

The configmap property values follow this general syntax:

```text
configmap:name/key[:defaultValue]
```

This means you can also set a default value in case the configmap should not be present.

```text
configmap:timer-source-config/period:5000
```

You can set the environment variables on the Kubernetes deployment for the connector:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: timer-source
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: timer-source
      app.kubernetes.io/version: 1.0-SNAPSHOT
  template:
    spec:
      containers:
        - image: localhost:5001/cdeppisc/timer-source:1.0-SNAPSHOT
          imagePullPolicy: Always
          name: timer-source
          env:
            - name: CAMEL_KAMELET_TIMER_SOURCE_MESSAGE
              value: "{{configmap:my-source-config/message}}"
            - name: CAMEL_KAMELET_TIMER_SOURCE_PERIOD
              value: "{{configmap:my-source-config/period:1000}}"
```

In order to use the values from the configmap you need to add a volume and a volume mount to the deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: timer-source
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: timer-source
      app.kubernetes.io/version: 1.0-SNAPSHOT
  template:
    spec:
      containers:
        - image: localhost:5001/cdeppisc/timer-source:1.0-SNAPSHOT
          imagePullPolicy: Always
          name: timer-source
          env:
            - name: CAMEL_KAMELET_TIMER_SOURCE_MESSAGE
              value: "{{configmap:my-source-config/message}}"
            - name: CAMEL_KAMELET_TIMER_SOURCE_PERIOD
              value: "{{configmap:my-source-config/period:1000}}"
          volumeMounts:
            - mountPath: /etc/camel/conf.d/_configmaps/my-source-config
              name: timer-source-config
              readOnly: true
      volumes:
        - configMap:
            name: my-source-config
          name: timer-source-config
```

Camel is able to resolve the configmap mount path given in the volume mount.
The mount path is configurable via `application.properties` in the connector project:

* camel.kubernetes-config.mount-path-configmaps=/etc/camel/conf.d/_configmaps/my-source-config
* camel.kubernetes-config.mount-path-secrets=/etc/camel/conf.d/_secrets/my-source-config

The mount path configured on the Kubernetes deployment should match the configuration in the `application.properties`.

Instead of settings the mount paths statically in the `application.properties` you can also set these via environment variables on the
Kubernetes deployment.

* CAMEL_K_MOUNT_PATH_CONFIGMAPS=/etc/camel/conf.d/_configmaps/my-source-config
* CAMEL_K_MOUNT_PATH_SECRETS=/etc/camel/conf.d/_secrets/my-source-config

The same mechanism applies to mounting and configuring Kubernetes secrets.
The syntax for referencing a secret value in an environment variable is as follows:

```text
secret:name/key[:defaultValue]
```

This means you can overwrite Kamelet properties with the values from the secret like this:

* CAMEL_KAMELET_TIMER_SOURCE_MESSAGE=secret:source-config/msg
* CAMEL_KAMELET_TIMER_SOURCE_PERIOD=secret:source-config/period

## CloudEvent attributes

Each connector source produces an event in CloudEvent data format. 
The connector uses a set of default values for the CloudEvent attributes:

* _ce-type_: dev.knative.connector.event.timer
* _ce-source_: dev.knative.eventing.timer-source
* _ce-subject_: timer-source

You can customize the CloudEvent attributes with setting environment variables on the deployment.

* KN_CONNECTOR_CE_OVERRIDE_TYPE=value
* KN_CONNECTOR_CE_OVERRIDE_SOURCE=value
* KN_CONNECTOR_CE_OVERRIDE_SUBJECT=value

You can set the CE_OVERRIDE attributes on a running deployment.

```shell
kubectl set env deployment/kn-connector-source-timer KN_CONNECTOR_CE_OVERRIDE_TYPE=custom-type
```

You may also use the SinkBinding `K_CE_OVERRIDES` environment variable set on the deployment.

## Dependencies

The required Camel dependencies need to be added to the Maven POM before building and deploying. 
You can use one of the Kamelets available in the [Kamelet catalog](https://camel.apache.org/camel-kamelets/) as a source or sink in this connector.

Typically, the Kamelet is backed by a Quarkus Camel extension component dependency that needs to be added to the Maven POM.
The Kamelets in use may list additional dependencies that we need to include in the Maven POM.

## Custom Kamelets

Creating a new kn-connector project is very straightforward.
You may copy one of the sample projects and adjust the reference to the Kamelets.

Also, you can use the Camel JBang kubernetes export functionality to generate a Maven project from a given Pipe YAML file.

```shell
camel kubernetes export my-pipe.yaml --runtime quarkus --dir target
```

This generates a Maven project that you can use as a starting point for the kn-connector project.

The connector is able to reference all Kamelets that are part of the [default Kamelet catalog](https://camel.apache.org/camel-kamelets/).

In case you want to use a custom Kamelet, place the `kamelet.yaml` file into `src/main/resources/kamelets`.
The Kamelet will become part of the built container image and you can just reference the Kamelet in the Pipe YAML file as a source or sink.

## More configuration options

For more information about Apache Camel Kamelets and their individual properties see https://camel.apache.org/camel-kamelets/.

For more detailed description of all container image configuration options please refer to the Quarkus Kubernetes extension and the container image guides:

* https://quarkus.io/guides/deploying-to-kubernetes
* https://quarkus.io/guides/container-image
