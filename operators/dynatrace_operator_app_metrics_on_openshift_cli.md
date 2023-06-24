# Partner Showcase: OpenShift App Observability with Dynatrace Operator
For a few years now, Red Hat has partnered with an ever-growing number of Independent Software Vendors (ISVs) to bring an expanded catalog of both operator and helm-driven product offerings to OpenShift.
As part of a new blog series highlighting these partner products, this post covers using Dynatrace Operator for App Observability on OpenShift.

A simple java websphere app will be used to demonstrate how Dynatrace Operator can be used to monitor apps in a cloud-native fashion and gain insights from App metrics, _without_ requiring modifying any source code or adding any source-based plugins.
Dynatrace Operator ensures a convenient and frictionless deployment at scale by utilizing cloud-native concepts like Webhooks and init-containers to instrument applications like the java websphere App.
While this blog purely focuses on App metrics, Dynatrace would provide a lot of additional value like end-to-end distributed tracing, real-time problem detection based on DAVIS AI, code-level visibility to troubleshoot issues and many more.

## Requirements
The following items are required if you'd like to duplicate this process in your own environment:
* An OpenShift cluster (OpenShift Local was used in this example)
* Cluster-admin privileges in OpenShift (to install operators)
* A Dynatrace [Free Trial](https://www.dynatrace.com/trial) Account
* Any app stack or framework [supported by OneAgent](https://dynatrace.com/hub/?tags=oa)
* A bash-compatible shell environment such as RHEL, macOS, or WSL (Fedora was used in this example)
* The [OpenShift Client](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/) (`oc`) CLI

Note: This guide focused on using the command-line, but the admin console can be used if you prefer a GUI.
This could also be used as a workaround in a non-supported client environment.

## Install Demo App
To start, let's begin with installing the demo app that will be monitored by OneAgent.
App Mod[ernization] Resorts (not to be confused with App Mon[itoring]) is a simple java app designed to run on an IBM Websphere application server.
The OpenLiberty Operator will assist here by creating and managing the OpenShift deployment, services and routes required to host the app.

Login to the OpenShift cluster as `kubeadmin` or another user with cluster-admin privileges:
```
$ oc login -u kubeadmin https://api.crc.testing:6443
```

For the sake of a clean environment, create a directory to house the yaml files that will be created.
You can keep these as a reference for a production deployment later, or remove them afterward.
```
$ mkdir appmon-files && cd appmon-files
```

Create the subscription yaml file which will be used to install the certified OpenLiberty Operator from OperatorHub.
[Download](https://github.com/jsm84/blogs/raw/assets/dynatrace-appmon/olo_subscription.yaml) or paste the yaml spec below into a new file named `olo_subscription.yaml`.
```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openliberty-operator
  namespace: openshift-operators
spec:
  channel: beta2
  installPlanApproval: Automatic
  name: open-liberty-certified
  source: certified-operators
  sourceNamespace: openshift-marketplace
```

Install OpenLiberty Operator by creating the `olo_subscription.yaml` in OpenShift.
This command will create the subscription in the `openshift-operators` namespace (redundantly set in the subscription.yaml):
```
$ oc apply -f olo_subscription.yaml -n openshift-operators
```

Next, create a new project/namespace to house the demo app.
This will also switch the current namespace to `ol-demo-app` (a failsafe in case `-n <namespace>` is omitted):
```
$ oc new-project ol-demo-app
```

Label the namespace with the key/value label `monitor: appMonitoring`.
Dynatrace operator will be configured later to only monitor namespaces with this particular label:
```
$ oc label namespace ol-demo-app monitor=appMonitoring
```

The next step is to create an `openlibertyapplication` Custom Resource (CR) yaml file.
[Download](https://github.com/jsm84/blogs/raw/assets/dynatrace-appmon/app-mod-withsslroute_cr.yaml) or paste the yaml spec below into a file named `app-mod-withsslroute_cr.yaml`:
```
apiVersion: apps.openliberty.io/v1beta2
kind: OpenLibertyApplication
metadata:
  name: appmod
spec:
  expose: true
  route:
    termination: reencrypt
    path: '/resorts'
    host: 'modresort.apps-crc.testing'
  applicationImage: quay.io/jmanning/ol-demo-app:latest
  pullPolicy: Always
  service:
    annotations:
      service.beta.openshift.io/serving-cert-secret-name: appmod-svc-tls
    certificateSecretRef: appmod-svc-tls
    port: 9443
```

Edit the above yaml file to adjust the URL set in `spec.route.host` to your cluster domain (the hard-coded URL will work for OpenShift Local).

Create the openlibertyapplication CR in the `ol-demo-app` namespace on the cluster:
```
$ oc apply -f app-mod-withsslroute_cr.yaml -n ol-demo-app
```

The application pod should now be running and showing a ready status:
```
$ oc get pods -n ol-demo-app
NAME                      READY   STATUS    RESTARTS   AGE
appmon-64548758f4-hhlpb   1/1     Running   0          6s
```

You can retrieve the route used to publish the demo app, and follow the URL to test the app (the following URL applies when using OpenShift local):
```
$ oc get route -n ol-demo-app
NAME     HOST/PORT                    PATH       SERVICES   PORT       TERMINATION   WILDCARD
appmon   modresort.apps-crc.testing   /resorts   appmon     9443-tcp   reencrypt     None
```

Paste the URL obtained from the route into your browser (**don't forget** to append the `/resorts` path to the end of the URL).
Proceed through the self-signed certificate warning page, and you should be greeted with the Mod Resorts web app.

![mod-resorts.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/mod-resorts.png)

### Demo App Resources:
* The container image is accessible from `quay.io/jmanning/ol-demo-app:latest`
* The Dockerfile, `.war` file, and `cr.yaml` files are found at https://github.com/jsm84/openliberty-operator-ocpz under `ol-app-install/`.
* The source code for the Mod Resorts app is located at https://github.com/IBM/appmod-resorts


## Install Dynatrace Operator
The next step is to install the operator, which will leverage Dynatrace OneAgent in the ol-demo-app pod to make app metrics, traces and metadata available to the Dynatrace platform.

Login to [Dynatrace](https://sso.dynatrace.com) with a free trial account.

Once you've accessed your live environment, select **Kubernetes** from the left pane, and then select **Connect automatically via Dynatrace Operator** in the top bar.

Fill out the web form as follows:
* Name the cluster / Dynakube (Custom Resource). `dynakube-appmon` is used in this example.
* Click **Create token** at the bottom of the page to create a Dynatrace Operator token.

![dynatrace-generate-token.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-generate-token.png)

The Operator access token will be displayed in a masked manner.
**Copy and paste** the token into a password manager or secure text document.
The token is only available upon generation, and can not be accessed at a later time.

While we could follow the instructions in the Dynatrace UI to deploy the Dynatrace Operator on OpenShift, we will take an alternative approach in this sample and use OperatorHub in order to benefit from automatic Operator updates.

**Make note** of the environment id for your Dynatrace instance.
This is located in the web page URL as `https://<environment-id>.live.dynatrace.com/`.

Switch to the `openshift-operators` namespace, which will hold all resources related to the Dynatrace Operator. This helps to ensure that things end up in the desired place:
```
$ oc project openshift-operators
```

Create the subscription yaml file that will deploy the certified Dynatrace Operator from OperatorHub.
[Download](https://github.com/jsm84/blogs/raw/assets/dynatrace-appmon/dto_subscription.yaml) or paste the yaml spec below into a file named `dto_subscription.yaml`:
```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/dynatrace-operator.openshift-operators: ""
  name: dynatrace-operator
  namespace: openshift-operators
spec:
  channel: alpha
  installPlanApproval: Automatic
  name: dynatrace-operator
  source: certified-operators
  sourceNamespace: openshift-marketplace
```

Install Dynatrace Operator by creating the subscription on OpenShift.
This command will create the `dto_subscription.yaml` object in the `openshift-operators` namespace (also set in the subscription):
```
$ oc apply -f dto_subscription.yaml -n openshift-operators
```

Confirm that the Dynatrace-operator pods are running:
```
$ oc get pods -n openshift-operators
```

Export a new shell variable containing the API token value recorded previously:
```
$ export APIKEY=<apiTokenValue>
```

Create a secret named `dynakube-appmon` in the `openshift-operators` namespace.
This will contain the token value `$APIKEY` in secret fields named `apiToken` and `paasToken`.
```
$ oc create secret generic dynakube-appmon --from-literal=apiToken=$APIKEY \
  --from-literal=paasToken=$APIKEY \
  -n openshift-operators
```

Note: The secret name in the previous command _does_ matter, as it must match the name of the CR that gets created in the next step.

Create the `dynakube` CR which will be used to trigger the operator.
[Download](https://github.com/jsm84/blogs/raw/assets/dynatrace-appmon/dynakube-appmon_cr.yaml) or paste the following yaml spec into a file named `dynakube-appmon_cr.yaml`.
**Replace** `<environment-id>` in the yaml file with _your_ previously noted Dynatrace environment ID.
```
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube-appmon
spec:
  apiUrl: https://<environment-id>.live.dynatrace.com/api
  namespaceSelector:
    matchLabels:
      monitor: appMonitoring
  oneAgent:
    applicationMonitoring:
      useCSIDriver: false
  activeGate:
    capabilities:
    - routing
    - kubernetes-monitoring
```

Trigger the Dynatrace operator by creating the CR on the cluster.
This will create the `dynakube-appmon` custom resource in `openshift-operators` namespace:
```
$ oc apply -f dynakube_cr.yaml -n openshift-operators
```

Check the `dynakube-appmon` custom resource for healthiness (to ensure there are no errors):
```
$ oc get dynakube -n openshift-operators
```

Finally, restart the `ol-demo-app` pod so that the Dynatrace webhook can inject the initContainer config for OneAgent.
As a note, `--all` is used below despite there being only one pod in the namespace (as the pod name can't be predetermined):
```
$ oc delete pod --all -n ol-demo-app
```

Check the pod's yaml output to see if OneAgent was installed in the pod.
The following command looks for the initContainer named `install-oneagent`, which gets injected by the Dynatrace webhook:
```
$ oc describe pods -n ol-demo-app | grep install-oneagent
      name: install-oneagent
      name: install-oneagent
```

## App Observability Walkthrough

### OpenShift (Kubernetes) Observability

With the demo App being monitored and having OneAgent injected, metrics will be sent to the Dynatrace API.
Switching context back to the Dynatrace web session, click to expand **Application & Microservices** in the left pane, and then select **Kubernetes workloads**.
There you will find an overview of all your deployed workloads. By entering "appmod" in the filter bar on top of the page,
all other apps will be filtered out and info will be visible that gives you an instant overview of key information like the Status,
or the number of Pods the app runs on.

![dynatrace-kubernetes-overview.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-kubernetes-overview.png)

Click the blue text link under **Name** to view all details about the workload app stack.
Here you get an overview of the resource utilization, Pods and also the health from a service perspective.

In the Service section you can see how the response time, failure rate and throughput of our sample app evolves over time.
Keep in mind that you need to generate some requests to your app by opening the page again in your browser,
_after_ Dynatrace was deployed, to generate some datapoints to display here.

![dynatrace-appmod-observation.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-appmod-observation.png)

Click on the blue link below Pods to see all the details for a pod, then find the "**Process**" section and click on the blue **appmod** entry there.

![dynatrace-pod-details.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-pod-details.png)

### Java Observability

Next, scroll down and click the blue text link in the **Process** list that guides you through all the detailed app metrics for OpenLiberty.
Here you can see that Dynatrace also automatically detected the underlying technology used by OpenLiberty.

By clicking on **JVM metrics** right below the info graphic, you'll see detailed metrics showing how Garbage collection suspension,
heap memory and threads evolve over time.

By opening **Further details** you can find more detailed insights to Java managed memory.

![dynatrace-openliberty-jvm-metrics.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-openliberty-jvm-metrics.png)

By clicking on **Analyze suspension** you can even dive into Dynatrace memory profiling capabilities, that analyze which objects contribute most
to memory allocation or to understand which objects often survive garbage collection (GC) cycles and therefore contribute the most to GC suspension.

![dynatrace-memory-profiling.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-memory-profiling.png)

### Web App Monitoring

Dynatrace Application Monitoring starts with the end user device, so you can see how the application performs on the client side, within the web browser.
To investigate metrics from there, just open the **Frontend** entry in the left side menu. There you will find an entry for **My web application** with
key info like the Apdex rating, the number of user actions and the **Visual complete** time (meaning how long it took to render the visual part of the web
page in the browser).

From this section, you can also see how key metrics evolve over time - and if your demo app is available from the internet, you can even set up
a synthetic availability check for it.

![dynatrace-client-side.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-client-side.png)

## Wrap Up

Dynatrace Operator supports cloud-native full stack injection for x86_64 OpenShift clusters, which encompasses Host Observability as well as App Observability.

OpenShift cluster metrics are provided with a separate add-on called ActiveGate, which runs exclusively on x86_64 as a kube client in order to gather metrics.
ActiveGate can also be used as an API gateway for the communication of all metrics back to the Dynatrace API.

All shown metrics can also be directly found and plotted via the Dynatrace Data explorer, put on any dashboard, accessed via API, or used in anomaly detection settings.

That being said, Dynatrace Operator is one of the most popular operators available in OperatorHub, and for good reason.
Stay tuned for more partner highlights from the Red Hat OpenShift Ecosystem.
