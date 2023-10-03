The previous blog, [OpenShift App Observability with Dynatrace](https://cloud.redhat.com/blog/partner-showcase-openshift-app-observability-with-dynatrace-operator),
covered using the Dynatrace Operator to observe metrics from a java-based WebSphere app, deployed via the IBM OpenLiberty Operator.

The post walked through signing up for a [free Dynatrace trial account](https://www.dynatrace.com/trial),
creating a namespace (and labeling it for later monitoring) and deploying the OpenLiberty Operator.
The [Mod Resorts](https://github.com/IBM/appmod-resorts) demo app was then stood up by triggering the operator.

With the demo app deployed, the Dynatrace Operator was the next piece.
After logging into the Dynatrace website and generating an API key,
a Kubernetes secret was created to allow both the Dynatrace Operator and OneAgent to authenticate to the Dynatrace API.
The operator was then deployed and configured to monitor the namespace of the demo app.

Finally, the application monitoring capabilities were explored by reviewing the dashboard in the cloud-based Dynatrace live environment.
The blog showcased the `applicationMonitoring` mode of Dynatrace OneAgent, in lieu of the `cloudNativeFullStack` mode (for brevity),
which would have enabled host-level observability in addition to application monitoring.
