# Knative Camel-K Example
This example shows how to create and deploy a basic Camel-K integration and how to run your in serverless mode using knative.

We will create a simple greeter and echoer integrations to demonstrate.

## Setup

### Install Minikube

minikube -p knativetutorial addons enable registry

### Install Apache Camel K
Download the latest Apache Camel K release from [here](https://github.com/apache/camel-k/releases/latest).
Extract the content and add binary to the PATH.

If you are having trouble you can check more detailed installation guide on the [offcial apache website](https://camel.apache.org/camel-k/2.0.x/installation/installation.html).

### Install Knative

Download and install Knative CLI from the [official knative website](https://knative.dev/docs/client/install-kn/).

Install the required custom resources by running the command:

```kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.11.1/serving-crds.yaml```

Install the core components of Knative Serving by running the command:

```kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.11.1/serving-core.yaml```

You can check your knative client version by running a command:

```kn version```

For this tutorial knative version 1.11.1 was used.

kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.11.2/kourier.yaml

kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'


kubectl apply \
  --filename https://projectcontour.io/quickstart/contour.yaml

### Setup a local cluster
For this example we will be using a local minikube cluster. You can install it using this [guide](https://minikube.sigs.k8s.io/docs/start/).

Once your minikube cluster is running, we can install kamel operator on your cluster using:
```
kamel install --olm=false --wait
```
This process may take a few minutes for the Apache Camel K pods to be up and running. You can monitor the process by watching the pods using a following command:
```
watch "kubectl get pods"
```

A successful Camel K setup should have the camel-k-operator pod running.

## Deploying Camel-K integration
Camel-K helps you in writing the Apache Camel integrations using Java, JS, XML and as YAML.

For all our integrations during this tutorial we will be using YAML DSL.

`Camel K integration - greeter:`
```
- from:
    uri: "timer:tick" 
    parameters: 
      # time in milliseconds (10 seconds)
      period: 10000
    steps: 
      - set-body: 
          constant: "Welcome to Apache Camel K"
      - set-header: 
          name: ContentType
          simple: text/plain
      - transform: 
          simple: "${body.toUpperCase()}"
      - to: 
          uri: "log:info?multiline=true&showAll=true"
```
To deploy this integration on our cluster, ```kamel``` CLI tool is used by running the following command:

```kamel run --dev greeter.yaml```

Camel K integration deployment process takes typically around 2-5 minutes as it involves multiple steps such as:

    1. Building an integration kit(camel-k-kit) which builds the container image with all the required Camel modules downloaded and added to classpath within the container image.
    2.If using Knative then deploying as a Knative Service
    3. Running the container image as a Kubernetes pod and start the Camel context

Once the integration is built, you sill see the terminal populated with logs from the integration like:

`greeter logs:`
```
[1] 2023-09-20 07:25:08,700 INFO  [info] (Camel (camel-1) thread #1 - timer://tick) Exchange[
[1]   Id: E2846846E347048-0000000000000007
[1]   ExchangePattern: InOnly
[1]   Properties: {CamelTimerCounter=8, CamelTimerFiredTime=Wed Sep 20 07:25:08 UTC 2023, CamelTimerName=tick, CamelTimerPeriod=10000, CamelToEndpoint=log://info?multiline=true&showAll=true}
[1]   Headers: {CamelMessageTimestamp=1695194708698, ContentType=text/plain, firedTime=Wed Sep 20 07:25:08 UTC 2023}
[1]   BodyType: String
[1]   Body: WELCOME TO APACHE CAMEL K
[1] ]
```
> [!WARNING]
> If you are not using maven repository manager or it takes long time to download maven artifacts, your earlier command kamel run --dev .. will report a failure. In those cases, run the command kamel get to see the status of the integration. Once you see the timed-greeter pod running, then use kamel log timed-greeter to see the logs as shown in the earlier listing.
>
> You can use Ctrl+C to stop the running Camel K integration and it automatically terminate its pods. If you had encountered the dev mode failure as described earlier, try to delete the integration using the command kamel delete timed-greeter.

The dev mode allows live reload of code. Update the timed-greeter.yaml body text and observe the automatic reloading of the context and the logs printing the new message.

## Deploying Camel-K Knative integration
Any Camel-K integration can be converted into a serverless service using Knative. For an integration to be deployed as a Knative Service you need to use the Camel-K's `knative` component.

The Camel-K Knative component provides two consumers: `knative:endpoint` and `knative:channel`. The former is used to deploy the integration as Knative Service, while the later is used to handle events from a Knative Event channel.

> [!NOTE]
> The Knative endpoints can be either a Camel producer or consumer depending on the context and need.

We wil now deploy a `knative:endpoint` consumer as part of your integration, that will add the serverless capabilities to your Camel-K integration using Knative.

The following listing shows a simple `echoer` Knative Camel-K integration, that will simply respond to your Knative Service call with same body that you sent into it in uppercase form. If there is no body received the service will respond with "no body received":

`Camel K integration - echoer`
```
- from:
    uri: "knative:endpoint/echoer" 
    steps:
      - log:
          message: "Got Message: ${body}"
      - convert-body-to: "java.lang.String" 
      - choice:
          when:
            - simple: "${body} != null && ${body.length} > 0"
              steps:
                - set-body:
                    simple: "${body.toUpperCase()}"
                - set-header:
                    name: ContentType
                    simple: text/plain
                - log:
                    message: "${body}"
          otherwise:
            steps:
              - set-body:
                  constant: "no body received"
              - set-header:
                  name: ContentType
                  simple: text/plain
              - log:
                  message: "Otherwise::${body}"
```

You can run this integration as shown in the following listing. If you notice that you are now deploying the integration in production mode i.e without --dev option.

```kamel run --wait echoer.yaml``` 

Since its the production mode and it takes some time for the integration to come up, you need to watch the integration's logs using the command `kamel log <integration name>` i.e. `kamel log echoer` and you can get the name of the integration using the command `kamel get`.

In the integration that you had deployed, you had applied the [Choice EIP](https://camel.apache.org/components/4.0.x/eips/choice-eip.html) when processing the exchange body. When the body has content then it simply converts the body to upper case otherwise it returns a canned response of "no body received". In either case, the content type header is set to `text/plain`.

Camel-K defines an `integration` via Custom Resource Definition (CRD) and you can view those CRDs and the actual `integrations` via the following commands:

```kubectl api-resources --api-group=camel.apache.org```

The command above shows the following output:
```
NAME                   SHORTNAMES   APIVERSION                  NAMESPACED   KIND
builds                 ikb          camel.apache.org/v1         true         Build
camelcatalogs          cc           camel.apache.org/v1         true         CamelCatalog
integrationkits        ik           camel.apache.org/v1         true         IntegrationKit
integrationplatforms   ip           camel.apache.org/v1         true         IntegrationPlatform
integrations           it           camel.apache.org/v1         true         Integration
kameletbindings        klb          camel.apache.org/v1alpha1   true         KameletBinding
kamelets               kl           camel.apache.org/v1         true         Kamelet
pipes                  pp           camel.apache.org/v1         true         Pipe

```

We can check is the integration was succesfull by running a command:

```kamel get```

A successfull integration should output the following:
```
NAME	PHASE	KIT
echoer	Running	default/kit-ck8kst3kdmqbrhmn0k20
```

Once the integration is started, you can check the Knative service using the command:

```kn service describe echoer```

When the service is in a ready state use the call script `call.sh` with parameter `echoer` and a request body of `"Hello World"`:

```./call.sh echoer 'Hello World'```