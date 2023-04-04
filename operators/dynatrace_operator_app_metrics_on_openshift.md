# Partner Showcase: OpenShift App Metrics with Dynatrace Operator
For a few years now, Red Hat has partnered with an ever-growing number of Independent Software Vendors (ISVs) to bring an expanded catalog of both operator and helm-driven product offerings to OpenShift.
As part of a new blog series highlighting these partner products, this post covers using Dynatrace Operator for monitoring app metrics on OpenShift.

A simple java websphere app will be used to demonstrate how Dynatrace Operator can be used to monitor app metrics in a cloud-native fashion, _without_ requiring source-based plugins.

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

## Install Java App
To start, lets begin with installing the demo app that will be monitored by OneAgent.
In this case let's use a simple java app deployed on IBM Websphere (there are operators for this!).

Login to the OpenShift cluster as `kubeadmin` or another user with cluster-admin privileges:
```
$ oc login -u kubeadmin https://api.crc.testing:6443
```

For the sake of a clean environment, create a directory to house the yaml files that will be created.
You can keep these as a reference for a production deployment later, or remove them afterward.
```
$ mkdir appmon-files && cd appmon-files
```

Create the subscription yaml file which will be used to install the certified OpenLiberty Operator from the OperatorHub:
```
$ cat > olo_subscription.yaml << EOF
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
  startingCSV: open-liberty-operator.v0.8.1
EOF
```

Install OpenLiberty Operator by creating the olo-subscription.yaml in the (hard-coded) `openshift-operators` namespace on the cluster:
```
$ oc create -f olo_subscription.yaml
```

Next, create a new project/namespace to house the demo app.
This will also switch your current (working) namespace to `ol-demo-app`:
```
$ oc new-project ol-demo-app
```

Label the namespace with the key/value label `monitor: appMonitoring`.
Dynatrace operator will be configured later to monitor namespaces with this particular label:
```
$ oc label namespace ol-demo-app monitor=appMonitoring
``` 

The next step is to create an `openlibertyapplication` Custom Resource (CR):
```
$ cat > app-mon-withsslroute_cr.yaml << EOF
apiVersion: apps.openliberty.io/v1beta2
kind: OpenLibertyApplication
metadata:
  name: appmon
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
      service.beta.openshift.io/serving-cert-secret-name: appmon-svc-tls
    certificateSecretRef: appmon-svc-tls
    port: 9443
EOF
```

Edit the above yaml file to adjust the URL set in `spec.route.host` to your cluster domain (the hard-coded URL will work for OpenShift Local).

Create the openlibertyapplication CR in the `ol-demo-app` namespace (the current context namespace) on the cluster:
```
$ oc create -f app-mon-withsslroute_cr.yaml
```

The application pod should now be running and showing a ready status:
```
$ oc get pods
NAME                      READY   STATUS    RESTARTS   AGE
appmon-64548758f4-hhlpb   1/1     Running   0          6s
```

A couple of notes on the demo app:
* The container image is accessible from quay.io/jmanning/ol-demo-app:latest
* The Dockerfile, `.war` file, and CR files are found at github.com/jsm84/openliberty-operator-ocpz under `ol-app-install/`.

You can retrieve the route used to publish the demo app, and follow the URL to test the app:
```
$ oc get route -n ol-demo-app
NAME     HOST/PORT                    PATH       SERVICES   PORT       TERMINATION   WILDCARD
appmon   modresort.apps-crc.testing   /resorts   appmon     9443-tcp   reencrypt     None
```

On a RHEL or Fedora (or other Linux based) host, you can open the link using the `xdg-open` command:
```
$ xdg-open https://modresort.apps-crc.testing/resorts
```

Proceed through the self-signed certificate warning page, and you should be greeted with the Mod Resorts web app.


 

## Install Dynatrace Operator
The next step is to install the operator, which will leverage Dynatrace OneAgent in the ol-demo-app container to make app metrics available to the Dynatrace dashboard.
There is a preliminary step to create API keys and environment ID for use by the operator.

Login to [Dynatrace](https://sso.dynatrace.com) with a free trial account.

Once you've accessed your live environment, select **Access Tokens** from the left pane, and then select **Generate New Token**.

Fill out the web form as follows:
* Name the token. `openliberty-demo` is used in this example.
* Select **Kubernetes: Dynatrace Operator** from the pull down menu.
* Click **Generate token** at the bottom of the page.




The API access token will be displayed. 
**Copy and paste** the token into a password manager or secure text document.
The token is only available upon generation, and can not be accessed at a later time.

Generate a second token for data ingest:
* From the **Access Tokens** page, click **Generate New Token**
* Name the token `Ingest` or similar
* In the **Search scopes...** field, type _metrics.ingest_.
* Check the box to select the `Ingest metrics` scope item
* Click **Generate Token**




Once again, **copy and paste** the token value for safe keeping, as it can not be obtained again.




**Make note** of the environment id for your Dynatrace instance.
This is located in the web page URL as https://<environment-id>.live.dynatrace.com/.

Create the `dto_subscription.yaml` file which will deploy the certified Dynatrace Operator from OperatorHub:
```
$ cat > dto_subscription.yaml << EOF
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
  startingCSV: dynatrace-operator.v0.10.3
```

Install Dynatrace Operator by creating the subscription in the `openshift-operators` namespace:
```
$ oc create -f dto_subscription.yaml
```

Confirm that the Dynatrace-operator pods are running:
```
$ oc get pods -n openshift-operators
```

Export a new shell variable containing the API token value recorded previously:
```
$ export APIKEY=<apiTokenValue>
```

Create a second shell variable containing the data-ingest token value:
```
$ export INGKEY=<ingestTokenValue>
```

Create a secret named `dynakube-appmon` in the `openshift-operators` namespace.
This will contain the token value `$APIKEY` in secret fields named `apiToken` and `paasToken`.
The `dataIngestToken` field will be populated by the value contained in `$INGKEY`. 

```
$ oc create secret generic dynakube-appmon --from-literal=apiToken=$APIKEY \
  --from-literal=paasToken=$APIKEY \
  --from-literal=dataIngestToken=$INGKEY \
  -n openshift-operators
```

Note: The secret name in the previous command _does_ matter, as it must match the name of the CR that gets created in the next step.

Create the `dynakube` CR which will be used to trigger the operator.
**Replace <environment-id>** below with your applicable host id noted earlier.
```
$ cat > dynakube_cr.yaml << EOF
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube-appmon
  namespace: openshift-operators
spec:
  apiUrl: https://<environment-id>.live.dynatrace.com/api
  namespaceSelector:
    matchLabels:
      monitor: appMonitoring
  oneAgent:
    applicationMonitoring:
      useCSIDriver: false
EOF
```

Trigger the Dynatrace operator by creating the CR in the (hard-coded) `openshift-operators` namespace:
```
$ oc create -f dynakube_cr.yaml
```

Check the `dynakube-appmon` custom resource for healthiness (to ensure there are no errors):
```
$ oc get dynakube -n openshift-operators
```

Finally, restart the `ol-demo-app` pod so that the Dynatrace webhook can inject the initContainer config for OneAgent (there should only be one pod in the namespace):
```
$ oc delete pod --all -n ol-demo-app
```

Check the pod's yaml output to see if OneAgent was installed in the pod.
This command looks for the initContainer named `install-oneagent`, which gets injected by the Dynatrace webhook:
```
$ oc get pods -n ol-demo-app | grep install-oneagent
      name: install-oneagent
      name: install-oneagent
```

## Review App Metrics

With the demo pod being monitored and having OneAgent injected, metrics will be sent to the Dynatrace API.
Switching context back to the Dynatrace web session, click to expand **Infrastructure** in the left pane, and then select **Technologies and Processes**.
The following info should be visible:




Click the blue text link under the **Group** list to see more info about the app stack:




Review the visible information, then scroll down and click the blue text link in the **Process** list.
This is where the more interesting app metrics can be seen:




## Wrap Up

You may have noticed some metrics appeared to be missing (services, host, and k8s cluster/API metrics), and that is due to the limited scope of this article.
Dynatrace Operator supports cloud-native full stack injection for x86_64 OpenShift clusters, which encompasses host metrics as well as app metrics.

OpenShift cluster metrics are provided with a separate add-on called ActiveGate, which runs exclusively on x86_64 as a kube client in order to gather metrics.
ActiveGate can also be used as an API gateway for the communication of all metrics back to the Dynatrace API.

That being said, Dynatrace Operator is one of the most popular operators available in OperatorHub, and for good reason.
Stay tuned for more partner highlights from the Red Hat OpenShift Ecosystem.
