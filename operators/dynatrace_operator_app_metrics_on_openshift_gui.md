# Partner Showcase: OpenShift App Observability with Dynatrace Operator
For a few years now, Red Hat has partnered with an ever-growing number of Independent Software Vendors (ISVs)
to bring an expanded catalog of both operator and helm-driven product offerings to OpenShift.
As part of a new blog series highlighting these partner products,
this post covers using Dynatrace Operator for App Observability on OpenShift.

A simple java WebSphere app will be used to demonstrate how Dynatrace Operator can be used to monitor apps
in a cloud-native fashion and gain insights from app metrics,
_without_ requiring modifying any source code or adding any source-based plugins.

Dynatrace Operator ensures a convenient and frictionless deployment at scale by utilizing
cloud-native concepts like Webhooks and init-containers to instrument applications like the java WebSphere app.
While this blog purely focuses on app metrics, Dynatrace would provide a lot of additional value like end-to-end distributed tracing,
real-time problem detection based on DAVIS AI, code-level visibility to troubleshoot issues and many more.

## Requirements
The following items are required if you'd like to duplicate this process in your own environment:
* An OpenShift cluster (OpenShift Local was used in this example)
* Cluster-admin privileges in OpenShift (to install operators)
* A Dynatrace [Free Trial](https://www.dynatrace.com/trial) Account
* Any app stack or framework [supported by OneAgent](https://dynatrace.com/hub/?tags=oa)

## Install Demo App
To start, let's begin with installing the demo app that will be monitored by OneAgent.
App Mod[ernization] Resorts (not to be confused with App Mon[itoring])
is a simple java app designed to run on an IBM WebSphere Application Server.
The OpenLiberty Operator will assist here by creating and managing the OpenShift deployment,
services and routes required to host the app.

Login to the OpenShift admin console as `kubeadmin` or any user with cluster-admin privileges.

Create a namespace to install the OpenLiberty application into.
In the left-side menu, expand **Administration** and then select **Namespaces**.

Click the blue **Create Namespace** button in the upper right corner,
then type `ol-demo-app` when prompted for the Name.

In the Labels field, type `monitor=appMonitoring`.
Then click **Create**.

![openliberty-create-namespace.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-create-namespace.png)

Click to expand the **Operators** menu on the left side, then select **OperatorHub**.

In the search field, type `openliberty` to filter the results.

![operatorhub-openliberty-filter.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/operatorhub-openliberty-filter.png)

Select the OpenLiberty Operator (it should be the only visible result)
and you should see an installation overview.

![openliberty-install-prompt.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-install-prompt.png)

Click **Install** and be sure that both **All namespaces on the cluster (default)** and **beta2** are selected.

Then click **Install** once again.

![openliberty-install-options.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-install-options.png)

You should see a simple install status page. Once the operator is installed and ready for use,
select **Installed Operators** from the left menu (skipping View Operator actually saves a step).

![openliberty-install-status.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-install-status.png)

From the Installed Operators list, expand the pull down menu (above) for **Project:**
and choose the **ol-demo-app** namespace created previously.

Then click **Open Liberty** to view the operator.

![openliberty-install-list.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-install-list.png)

From the Operator Details page, select **Create instance** under **OLA OpenLibertyApplication**.

![openliberty-operator-details.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-operator-details.png)

Select **YAML view** (instead of the default _Form view_), which will provide a text editor window.
Replace the existing contents with that of the provided
[Custom Resource (CR) yaml file](https://github.com/jsm84/blogs/raw/assets/dynatrace-appmon/app-mod-withsslroute_cr.yaml).

Edit the yaml contents to adjust the URL in `spec.route.host` to match your cluster's domain
(the default URL _will_ work for OpenShift Local).

Once finished, click **Create**.

![openliberty-application-yaml.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-application-yaml.png)

In the OpenLibertyApplications list, select **appmod** to view the app instance.

![openliberty-applications-list.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-applications-list.png)

Click the **Resources** tab and then click **RT appmod**.

![openliberty-appmod-resources.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-appmod-resources.png)

Finally, to view the application web page, click the link shown under **Location** towards the upper right.

![openliberty-appmod-route.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-appmod-route.png)

Proceed through the self-signed certificate warning page, and you should be greeted with the Mod Resorts web app.

It may take a few moments for the application pod to start, so you may see the "application unavailable" page at first.
Just keep refreshing (F5 in most browsers) until the application comes up.

![mod-resorts.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/mod-resorts.png)

### Demo App Resources:
* The container image is accessible from `quay.io/jmanning/ol-demo-app:latest`
* The Dockerfile, `.war` file, and `cr.yaml` files are found at https://github.com/jsm84/openliberty-operator-ocpz under `ol-app-install/`.
* The source code for the Mod Resorts app is located at https://github.com/IBM/appmod-resorts


## Install Dynatrace Operator
The next step is to install the operator, which will leverage Dynatrace OneAgent in the `ol-demo-app` pod to make app metrics,
traces and metadata available to the Dynatrace platform.

Login to [Dynatrace](https://sso.dynatrace.com) with a free trial account.

![dynatrace-latest-version](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-latest-version.png)

Once you've accessed your live environment, select and open **Kubernetes App** within the infrastructure section or hit 'CTRL-K' and search for "Kubernetes".

Then, select **Connect automatically via Dynatrace Operator** in the top bar.

Fill out the web form as follows:
* Name the cluster / Dynakube (Custom Resource). `dynakube-appmon` is used in this example.
* Click **Create token** at the bottom of the page to create a Dynatrace Operator token.

![dynatrace-generate-token.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-generate-token.png)

The Operator access token will be displayed in a masked manner.
_Copy and paste_ the token into a password manager or secure text document.

The token is _only_ available upon generation, and can not be accessed at a later time.

While we could follow the instructions in the Dynatrace UI to deploy the Dynatrace Operator on OpenShift,
we will take an alternative approach in this example and use OperatorHub to benefit from automatic Operator updates.

_Make note_ of the environment id for your Dynatrace instance.
This is located in the web page URL as `https://<environment-id>.live.dynatrace.com/`.

Switch back to the the OpenShift admin console, expand the **Operators** menu (if it's not already) and select **OperatorHub**.
In the filter checklist (middle left), select **Certified** and type `dynatrace` into the search filter above.

![operatorhub-dynatrace-filter.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/operatorhub-dynatrace-filter.png)

Click on the Dynatrace Operator tile, and then click **Install**.

![dynatrace-install-prompt.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-install-prompt.png)

Accept all defaults, which will install Dynatrace Operator into a new project (kubernetes namespace) called `dynatrace`.
Then click **Install** again.

![dynatrace-install-options.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-install-options.png)

At the install status page, wait a few moments for the install to complete, and then click **View Operator**.

![dynatrace-install-status.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-install-status.png)

The next step is to create the secret, which will contain the API token created earlier.

From the Operator details page (which should now be in the `dynatrace` project by checking above),
expand **Workloads** in the left menu, and then click **Secret**.

![dynatrace-operator-details.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-operator-details.png)

From the secrets page, click **Create** which will expand a pull down menu. Then, select **Key/value secret**.

![dynatrace-secret-create.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-secret-create.png)

Fill out the web form by typing in the **Name** `dynakube-appmon`.

Enter the **Key** name `apiToken` and paste the token value (recorded previously) into the **Value** field.

Click to **Add key/value** and enter the **Key** name `paasToken`.

Paste the same token value into the corresponding **Value** field.

![dynatrace-secret-form.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-secret-form.png)

Note: The secret name and the key names from the previous step are important,
and must match the name of the CR that gets created in the next step.

Go back to the Operator details page again. Expand **Operators** in the left menu,
select **Installed Operators**.

Click on **Dynatrace Operator** in the center frame.

![dynatrace-secrets-to-operator-details.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-secrets-to-operator-details.png)

This time, select **Create Instance** in the **DK Dynatrace Dynakube** tile.

![dynatrace-operator-details.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-operator-details.png)

Select **YAML view**, then replace the default yaml contents with that of the provided
[Dynakube CR yaml file](https://github.com/jsm84/blogs/raw/assets/dynatrace-appmon/dynakube-appmon_cr.yaml).

_Replace_ `ENVIRONMENTID` in the yaml file with _your_ previously noted Dynatrace environment ID (highlighted in the screenshot below).

![dynatrace-dynakube-cr.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-dynakube-cr.png)

Click **Create** to instantiate the CR which will trigger the operator.

The `dynakube-appmon` will show up in a list of `dynakube` resources, along with the current Status.
Within a few minutes, you should see the Status Phase change from `Deploy` to `Running`:

![dynatrace-cr-deployed.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-cr-deployed.png)

The last step is to restart the OpenLiberty pod so that the Dynatrace webhook can inject the initContainer for OneAgent.
From the Dynakubes page, click the pull down menu above to change the **Project** from `dynatrace` to `ol-demo-app`.

Next, expand **Workloads** in the left menu and select **Pods**.

For the `appmod` pod shown, click the dots on the far right and select **Delete**.

![openliberty-pod-destroy.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/openliberty-pod-destroy.png)

If everthing was successful, a new pod will get created that contains the OneAgent initContainer.
A Status of `Init:0/1` is a clue that the initContainer was injected.
Within a few moments, the Status should change to `Running`.

The application can now be observed using Dynatrace.


## App Observability Walkthrough

### OpenShift (Kubernetes) Observability

With the demo app being monitored and having OneAgent injected, metrics will be sent to the Dynatrace API.
Switching context back to the Dynatrace web session, click to expand **Application & Microservices** in the left pane,
and then select **Kubernetes workloads**. There you will find an overview of all your deployed workloads.

By entering `appmod` in the filter bar on top of the page, all other apps will be filtered out and info will be visible
that gives an instant overview of key info like the Status, or the number of pods the app runs on.

![dynatrace-kubernetes-overview.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-kubernetes-overview.png)

Click the blue text link under **Name** to view all details about the workload app stack.
Here you get an overview of the resource utilization, pods and also the health from a service perspective.

In the Service section you can see how the response time, failure rate and throughput of our sample app evolves over time.
Keep in mind that you need to generate some requests to your app by opening the page again in your browser,
_after_ Dynatrace was deployed, to generate some datapoints to display here.

![dynatrace-appmod-observation.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-appmod-observation.png)

Click on the blue link below Pods to see all the details for a pod,
then find the "**Process**" section and click on the blue **appmod** entry there.

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

Dynatrace Application Monitoring starts with the end user device,
so you can see how the application performs on the client side, within the web browser.

To investigate metrics from there, just open the **Frontend** entry in the left side menu.

There you will find an entry for **My web application** with key info like the Apdex rating, 
the number of user actions, and the **Visual complete** time
(meaning how long it took to render the visual part of the web page in the browser).

From this section, you can also see how key metrics evolve over time -
and if your demo app is available from the internet, you can even set up
a synthetic availability check for it.

![dynatrace-client-side.png](https://github.com/jsm84/blogs/blob/assets/dynatrace-appmon/dynatrace-client-side.png)

## Wrap Up

Dynatrace Operator supports cloud-native full stack injection for x86_64 OpenShift clusters,
which encompasses Host Observability as well as App Observability.

OpenShift cluster metrics are provided with a separate add-on called ActiveGate,
which runs exclusively on x86_64 as a kube client in order to gather metrics.
ActiveGate can also be used as an API gateway for the communication of all metrics back to the Dynatrace API.

All shown metrics can also be directly found and plotted via the Dynatrace Data explorer,
put on any dashboard, accessed via API, or used in anomaly detection settings.

That being said, Dynatrace Operator is one of the most popular operators available in OperatorHub, and for obvious reasons.
Stay tuned for more partner highlights from the Red Hat OpenShift Ecosystem.
