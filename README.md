# Knative connector timer-source

Knative eventing connector based on Apache Camel Kamelets

## Kamelet source pipe

The connector uses a Apache Camel Pipe to connect a Kamelet source with the Knative broker.
The pipe is a YAML file located in [src/main/resources/camel/kn-connector-source-timer.yaml](src/main/resources/camel/kn-connector-source-timer.yaml)

This connector uses the [timer-source Kamelet](https://camel.apache.org/camel-kamelets/timer-source.html) and produces events for the Knative broker. 

## Build the container image

```shell
./mvnw package -Dquarkus.container-image.build=true
```

The container image uses the project version and image group defined in the Maven POM.
You can customize the image group with `-Dquarkus.container-image.group=my-group`.

By default, the container image looks like this:

```text
quay.io/openshift-knative/kn-connector-source-timer:1.0-SNAPSHOT
```

## Kubernetes manifest

The build produces a Kubernetes manifest in (`target/kubernetes/kubernetes.yml`).
This manifest holds all resources required to run the application on your Kubenretes cluster.

The Kubernetes manifest includes:

* Service
* Deployment
* SinkBinding

You can customize the Kubernetes resources in [src/main/kubenretes/kubernetes.yml](src/main/kubernetes/kubernetes.yml).
This is being used as a basis and Quarkus will generate the final manifest in `target/kubernetes/kubernetes.yml` during the build.

## Push container image to registry

```shell
./mvnw package -Dquarkus.container-image.build=true -Dquarkus.container-image.push=true
```

Pushes the image to the image registry defined in the Maven POM. 
The default registry is [quay.io](https://quay.io/).

You can customize the registry with `-Dquarkus.container-image.registry=localhost:5001` (e.g. when connecting to local Kind cluster).

In case you want to connect with a local cluster like Kind or Minikube you may also need to set `-Dquarkus.container-image.insecure=true`.

## Deploy to Kubernetes

You can deploy the application to Kubernetes with:

```shell
./mvnw package -Dquarkus.kubernetes.deploy=true
```

This connects to the current Kubernetes cluster that you are connected with (e.g. via `kubectl config set-context --current --namespace $1`).

You may change the target namespace with `-Dquarkus.kubernetes.namespace=my-namesapce`.

## CloudEvent attributes

The connector produces a set of CloudEvent attributes with default values:

* ce-type: dev.knative.connector.event.timer
* ce-source: dev.knative.eventing.timer-source

## Kamelet source properties

The source Kamelet defines a set of properties.
You can customize the properties via environment variables on the deployment:

* KN_CONNECTOR_SOURCE_TIMER_MESSAGE
* KN_CONNECTOR_SOURCE_TIMER_PERIOD

As an alternative to that you may mount a configmap/secret with some `application.properties`:

_application.properties_
```properties
# connector defined source properties
kn.connector.source.timer.message=Knative rocks!
kn.connector.source.timer.period=2000

# any other Kamelet source property
camel.kamelet.timer-source.contentType=application/json
camel.kamelet.timer-source.repeatCount=5
camel.kamelet.timer-source.any-other-prop=value
```

## Dependencies

The required Camel dependencies need to be added to the Maven POM before building and deploying. 
You can use one of the Kamelets available in the [Kamelet catalog](https://camel.apache.org/camel-kamelets/) as a source or sink in this connector.

Typically the Kamelet is backed by a Quarkus Camel extension component dependency that needs to be added to the Maven POM.
The Kamelets in use may list additional dependencies that we need to include in the Maven POM.

## Custom Kamelets

The connector is able to use all source Kamelets that are part of the [default Kamelet catalog](https://camel.apache.org/camel-kamelets/).
In case you want to use your own Kamelet place the `kamelet.yaml` file into `src/main/resources/kamelets`.
The Kamelet will be part of the built container image for this connector then.

## More configuration options

For more information about Apache Camel Kamelets and their individual properties see https://camel.apache.org/camel-kamelets/.

For more detailed description of all container image configuration options please refer to the Quarkus Kubernetes extension and the container image guides:

* https://quarkus.io/guides/deploying-to-kubernetes
* https://quarkus.io/guides/container-image
