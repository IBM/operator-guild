# How to Red Hat certify a Helm operator

This document covers how to prepare and Red Hat certify your Helm operator for the goal of being added to [Red Hat Marketplace](https://marketplace.redhat.com/en-us).

## Helm chart checklist

This is a base checklist of requirements for a Helm chart to be compatabile prior to certification.

1. Chart and dependencies are Helm 3 compatabile.

2. Chart and dependencies are actively maintained.

3. README which describes configuration, instructions etc.

4. The Base image must be (or must be based on) a supported Red Hat image, such as Red Hat Enterprise Linux or [Red Hat Universal Base Image](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/program-on-boarding/containers-with-red-hat-universal-base-image-ubi).

5. Confirm contents (images and packages) are from a trusted source and are maintained with CVE updates.

6. All contents (images and packages) should use the latest version unless a valid reason for not doing so.

7. Container images (parent and all dependent charts) run as non root. Refer to [How to make a non-root based container](https://github.com/IBM/operator-guild/blob/main/docs/content/howto/how_to_make_a_nonroot_container.md) for more details. Don't use `USER nonroot:nonroot` with UBI base image as it doesn't exist for that image.

8. Uncompressed container images should have less than 40 layers.

9. Chart can be installed on the latest OpenShift version using the most restrictive [Security Context Constraint (SCC)](https://www.openshift.com/blog/managing-sccs-in-openshift) where possible.

> Note: The UID is arbitrary on OpenShift as it will launch the container with a random UID. OpenShift will use a GID of 0 (root). This doesn't give any special permissions like UID of 0 (root), but will allow the setting of static file/directory permissions within the container image since all containers launch as GID 0 in OpenShift. May need to use `nonroot` SCC when UID/GID are specified.
  
10. Appropriate licensing for chart, dependencies and images. Refer to [Licenses Requirements](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/program-on-boarding/technical-prerequisites#licenses-requirements) for more details.

11. Product overview for the certified image, as well as other relevant documentation should be provided.

## Certification process

### Prerequisites

1. Follow the prerequisite steps as mentioned in the [Program Prerequisites](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/program-on-boarding/prerequisites). These prerequisites are part of [Certification Workflow](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/program-on-boarding/certification-workflow).
2. All container images used by the Helm operator must also be certified. Refer to [How to Red Hat certify a container image](https://github.com/IBM/operator-guild/blob/main/docs/content/howto/image_redhat_certification.md) for more details on how to do this.

### Certification steps

After you satisfy all the [Prerequisites](#prerequisites), follow the steps below for certification:

> Note: The operator example used is from [How to create an operator using an existing Helm chart](https://github.com/IBM/operator-guild/blob/main/docs/content/howto/existing_helm_chart_operator.md). This example uses the [Operator SDK](https://sdk.operatorframework.io/docs/building-operators/helm/quickstart/#olm-deployment) to simplify the process. This example needs to be tested on Redhat [OpenShift](https://www.openshift.com/) due to changes of the operator for certification.

1. Create a `Container Application` project - Follow the steps in [Certify your Operator Image](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/certify-your-operator/creating-an-operator-project) and stop before [Uploading your Operator Image](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/certify-your-operator/creating-an-operator-project/image-upload).

2. Update operator files for certification - Add labels for Red Hat build service and scanner:

- Add the following labels to `Dockerfile`:

```yaml
# Build the manager binary
FROM quay.io/operator-framework/helm-operator:v1.7.1

### Required Labels for Red Hat build service and scanner
LABEL name="Wordpress Operator" \
      maintainer="John Doe:john.doe@example.com" \
      vendor="Bitnami" \
      version="v0.0.1" \
      release="1" \
      summary="This is an example of a wordpress Helm operator." \
      description="Operator for deployment and management of Example WordPress instances based on the Helm chart."

ENV HOME=/opt/helm
COPY watches.yaml ${HOME}/watches.yaml
COPY helm-charts  ${HOME}/helm-charts
WORKDIR ${HOME}
```

> Note: It is recommended to use the latest version of the `helm-operator` image from the Operator SDK where possible. Updates to the image can include security fixes which maybe necessary for passing vulnerability tests during image scanning.

3. Update operator files for certification - Add license file:

- Add an example license file to the project:

```bash
$ mkdir licenses

$ cd licenses

$ cat <<EOF >license.txt
placeholder for license
EOF

$ cd ..
```

- Update `Dockerfile` to copy the license directory:

```yaml
# Build the manager binary
FROM quay.io/operator-framework/helm-operator:v1.7.1

### Required Labels for Red Hat build service and scanner
LABEL name="Wordpress Operator" \
      maintainer="John Doe:john.doe@example.com" \
      vendor="Bitnami" \
      version="v0.0.1" \
      release="1" \
      summary="This is an example of a wordpress Helm operator." \
      description="Operator for deployment and management of Example WordPress instances based on the Helm chart."

# Required Licenses for Red Hat build service and scanner
COPY licenses /licenses

ENV HOME=/opt/helm
COPY watches.yaml ${HOME}/watches.yaml
COPY helm-charts  ${HOME}/helm-charts
WORKDIR ${HOME}
```

4. Update operator files for certification - Use certified image for RBAC proxy:

> Note: This image is ONLY available on Redhat [OpenShift](https://www.openshift.com/) and therefore needs to be tested on it.

- Update the RBAC proxy (`config/default/manager_auth_proxy_patch.yaml`) to use the following image:

```yaml
# This patch inject a sidecar container which is a HTTP proxy for the
# controller manager, it performs RBAC authorization against the Kubernetes API using SubjectAccessReviews.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: controller-manager
  namespace: system
spec:
  template:
    spec:
      containers:
      - name: kube-rbac-proxy
        #image: gcr.io/kubebuilder/kube-rbac-proxy:v0.8.0
        # Use this certified image for certification
        image: registry.redhat.io/openshift4/ose-kube-rbac-proxy:latest
        args:
        - "--secure-listen-address=0.0.0.0:8443"
        - "--upstream=http://127.0.0.1:8080/"
        - "--logtostderr=true"
        - "--v=10"
        ports:
        - containerPort: 8443
          name: https
      - name: manager
        args:
        - "--metrics-addr=127.0.0.1:8080"
        - "--enable-leader-election"
        - "--leader-election-id=wordpress-operator"
```

5. Update operator files for certification - Enable single image variable

- Add a new single image variable for any images that are referenced with multiple image variables.

For example, in `helm-charts/wordpress/values.yaml`, you would comment or remove `registry`, `repository` and `tag` variables, and replace with one variable only as follows:

```yaml
[...]
image:
  #registry: docker.io
  #repository: bitnami/wordpress
  #tag: 5.7.0-debian-10-r8
  path: docker.io/bitnami/wordpress:5.7.0-debian-10-r8
[...]
```

> Note: You may need to retain the `tag` variable if it is referenced individually in the templates (i.e. not as part of 3 variables for image path)

- This new variable needa to be substituted everywhere the original variables occur in the templates. This may also necessitate changes in underlying common library charts.

For example, in `helm-charts/wordpress/templates/_helpers.tpl` a change would be as follows:

```yaml
[...]
{{- define "wordpress.image" -}}
{{- include "common.images.image" (dict "imageRoot" .Values.image.path "global" .Values.global) -}}
{{- end -}}
[...]
```

- Add an override image variable (e.g. `RELATED_IMAGE_WORDPRESS`) to the `watches.yaml` file as follows:

```yaml
[...]
chart: helm-charts/wordpress
  overrideValues:    
    image.image: $RELATED_IMAGE_WORDPRESS
# +kubebuilder:scaffold:watch
```

The variable is defined in `config/manager/manager.yaml` as follows:

```yaml
[...]
spec:
  containers:
  - image: controller:latest
    args:
    - "--enable-leader-election"
    - "--leader-election-id=wordpress-operator"
    env:
    - name: RELATED_IMAGE_WORDPRESS
      value: docker.io/bitnami/wordpress:5.7.0-debian-10-r8
[...]
```

- Making changes for the single image variable dependens on the chart and its dependencies. You will need to metliciously go through the image variables and where they are referenced in the templates. Refer to [Using a Single Image Variable](https://redhat-connect.gitbook.io/certified-operator-guide/helm-operators/building-a-helm-operator/using-a-single-image-variable) for more details.

> Note: Trying to show all changes for the WordPress chart is not feasible as an example in this doc. You can still perform the certification for this example without the single image variable.

6. Build and test the operator:

> Note: Need to test on RedHat [OpenShift](https://www.openshift.com/) due to certified image for RBAC proxy.

- Follow the steps in [Deploy the operator](https://github.com/IBM/operator-guild/blob/main/docs/content/howto/existing_helm_chart_operator.md#deploy-the-operator) and [Deploy WordPress](https://github.com/IBM/operator-guild/blob/main/docs/content/howto/existing_helm_chart_operator.md#deploy-wordpress).

7. Build and test the operator - Cleanup the operator:

- Follow the steps in [Clean up](https://github.com/IBM/operator-guild/blob/main/docs/content/howto/existing_helm_chart_operator.md#clean-up).

8. Upload the image to the Red Hat project registry:

```bash
# Use Registry Key from https://connect.redhat.com/project/<project_id>/images/upload-image as the password when prompted from the 'docker login' command below.
# Note: On Mac, it doesn't seem to accept the password at the prompt. Use the --password-stdin flag instead.
# Refer to the Docker docs for more details:
# https://docs.docker.com/engine/reference/commandline/login/provide-a-password-using-stdin

# Login to the RH registry
$ docker login -u unused scan.connect.redhat.com

# Get the ID of the WordPress image
$ docker images

# Use ospid-id from https://connect.redhat.com/project/<project_id>/images/upload-image
$ docker tag <wordpress-operator_image_id> scan.connect.redhat.com/<ospid-id>/wordpress-operator:v0.0.1

# Push the iamge to the RH registry
$ docker push scan.connect.redhat.com/<ospid-id>/wordpress-operator:v0.0.1
```

9. Check image has been uploaded and scanned:

- Go to `https://connect.redhat.com/project/<project_id>/images`

- It might take a while for the image to appear. You then need to wait for the certification process to finish.

- If "certification test" passed then continue to next step. Otherwise, check the scan logs for errors and update the image accordingly.

10. Go back to "Checklist" (https://connect.redhat.com/project/<project_id>/checklist) and finish any items that are not complete. The image is now certified.

11. Create an `Operator Bundle Image` project - Follow the steps in [Certify your Operator Bundle Image](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/certify-your-operator/certify-your-operator-bundle-image) until [Uploading your Operator Bundle Image](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/certify-your-operator/certify-your-operator-bundle-image/uploading-your-operator-bundle-image).

12. Create a bundle for your operator release:

```bash
$ export USERNAME=<docker-registry-userid>

$ export OPERATOR_IMG=docker.io/$USERNAME/wordpress-operator:v0.0.1
 
$ make bundle IMG=$OPERATOR_IMG

operator-sdk generate kustomize manifests -q

Display name for the operator (required): 
> Wordpress Helm

Description for the operator (required): 
> Operator for deployment and management of Example WordPress instances based on the Helm chart                                                  

Provider\'s name for the operator (required): 
> Example Inc.

Any relevant URL for the provider name (optional): 
> example.com

Comma-separated list of keywords for your operator (required): 
> WordPress,Helm,Example

Comma-separated list of maintainers and their emails (e.g. 'name1:email1, name2:email2') (required): 
> John Doe:john.doe@example.com
cd config/manager && /demo/wordpress-operator/bin/kustomize edit set image controller=controller:latest
demo/wordpress-operator/bin/kustomize build config/manifests | operator-sdk generate bundle -q --overwrite --version 0.0.1  
INFO[0000] Building annotations.yaml                    
INFO[0000] Writing annotations.yaml in demo/wordpress-operator/bundle/metadata 
INFO[0000] Building Dockerfile                          
INFO[0000] Writing bundle.Dockerfile in demo/wordpress-operator 
operator-sdk bundle validate ./bundle
INFO[0000] Found annotations file                        bundle-dir=bundle container-tool=docker
INFO[0000] Could not find optional dependencies file     bundle-dir=bundle container-tool=docker
INFO[0000] All validation tests have completed successfully 
```

The `make bundle` command used above, automates several tasks, including running the following `operator-sdk` subcommands in order:

- `generate kustomize manifests`

- `generate bundle`

- `bundle validate`

After running the command, you should see directory structure similar to the following:

```bash
$ tree config/manifests

config/manifests
├── bases
│   └── wordpress-operator.clusterserviceversion.yaml
└── kustomization.yaml

1 directory, 2 files

$ tree bundle

bundle
├── manifests
│   ├── helm-chart.example.com_wordpresses.yaml
│   ├── wordpress-operator-controller-manager-metrics-service_v1_service.yaml
│   ├── wordpress-operator-metrics-reader_rbac.authorization.k8s.io_v1_clusterrole.yaml
│   └── wordpress-operator.clusterserviceversion.yaml
├── metadata
│   └── annotations.yaml
└── tests
    └── scorecard
        └── config.yaml

4 directories, 6 files
```

13. Update bundle files for certification:

- Update `bundle/manifests/wordpress-operator.clusterserviceversion.yaml` by:

Replacing:
`mediatype: “ ”`

With:
`mediatype: "image/gif"`

- Add the following line to `bundle/metadata/annotations.yaml`:

`operators.operatorframework.io.bundle.channel.default.v1: alpha`

- Add the following labels to `bundle.Dockerfile`:

```bash
LABEL operators.operatorframework.io.bundle.channel.default.v1=alpha
LABEL com.redhat.openshift.versions="v4.6"
LABEL com.redhat.delivery.operator.bundle=true
```

14. Build the bundle image and push to registry:

- Build the bundle image:

```bash
$ export USERNAME=<docker-registry-userid>

$ export BUNDLE_IMG=docker.io/$USERNAME/wordpress-operator-bundle:v0.0.1 

$ make bundle-build IMG=$BUNDLE_IMG

docker build -f bundle.Dockerfile -t docker.io/<registry>/wordpress-operator-bundle:v0.0.1 .
Sending build context to Docker daemon  288.8kB
Step 1/14 : FROM scratch
 ---> 
Step 2/14 : LABEL operators.operatorframework.io.bundle.mediatype.v1=registry+v1
 ---> Running in 2fae061816c5
Removing intermediate container 2fae061816c5
 ---> 3ffa7ffca6c1
Step 3/14 : LABEL operators.operatorframework.io.bundle.manifests.v1=manifests/
 ---> Running in a9bdf55e153b
Removing intermediate container a9bdf55e153b
 ---> 28b278ba40da
Step 4/14 : LABEL operators.operatorframework.io.bundle.metadata.v1=metadata/
 ---> Running in be58738b86e1
Removing intermediate container be58738b86e1
 ---> fd02177a8e91
Step 5/14 : LABEL operators.operatorframework.io.bundle.package.v1=wordpress-operator
 ---> Running in 420cf49acb90
Removing intermediate container 420cf49acb90
 ---> a297a698baa7
Step 6/14 : LABEL operators.operatorframework.io.bundle.channels.v1=alpha
 ---> Running in 35e9e2c2afca
Removing intermediate container 35e9e2c2afca
 ---> 20f05b278bc4
Step 7/14 : LABEL operators.operatorframework.io.metrics.mediatype.v1=metrics+v1
 ---> Running in d81d0c33321f
Removing intermediate container d81d0c33321f
 ---> a451044b6c2a
Step 8/14 : LABEL operators.operatorframework.io.metrics.builder=operator-sdk-v1.4.0+git
 ---> Running in d6dd53de0595
Removing intermediate container d6dd53de0595
 ---> 4d09930ae00b
Step 9/14 : LABEL operators.operatorframework.io.metrics.project_layout=helm.sdk.operatorframework.io/v1
 ---> Running in cb0f013dd9b0
Removing intermediate container cb0f013dd9b0
 ---> b56417dd107b
Step 10/14 : LABEL operators.operatorframework.io.test.mediatype.v1=scorecard+v1
 ---> Running in 46e311ff13da
Removing intermediate container 46e311ff13da
 ---> cbb0f65464e9
Step 11/14 : LABEL operators.operatorframework.io.test.config.v1=tests/scorecard/
 ---> Running in 5cdcc6d6c673
Removing intermediate container 5cdcc6d6c673
 ---> 7d4ab15f25b5
Step 12/14 : COPY bundle/manifests /manifests/
 ---> 6dc3ae343ef7
Step 13/14 : COPY bundle/metadata /metadata/
 ---> 2ad3e942d54e
Step 14/14 : COPY bundle/tests/scorecard /tests/scorecard/
 ---> 8790f0177089
Successfully built 8790f0177089
Successfully tagged <registry>/wordpress-operator-bundle:v0.0.1
```

- Push to registry:

```bash
$ make docker-push IMG=$BUNDLE_IMG

The push refers to repository [docker.io/$USERNAME/wordpress-operator-bundle]
afe7ec4b10c1: Pushed 
df1b6956e025: Pushed 
42c16e689d32: Pushed 
v1.0.0: digest: sha256:33b97cd53a809d4e706c3b0987bcfa36b50bc48facbdbc6e981b0aaf8d07c34a size: 939
```

15. Deploy and test the operator:

> Note: For more details and manual steps on testing, check out [Testing your Operator with Operator Framework](https://github.com/operator-framework/community-operators/blob/master/docs/testing-operators.md#testing-your-operator-with-operator-framework)

- Check the status of [OLM](https://olm.operatorframework.io/docs/) on your cluster by using the following Operator SDK command:

```bash
$ operator-sdk olm status

INFO[0000] Fetching CRDs for version "0.15.1"           
INFO[0000] Using locally stored resource manifests      
INFO[0001] Successfully got OLM status for version "0.15.1" 

NAME                                            NAMESPACE    KIND                        STATUS
olm                                                          Namespace                   Installed
subscriptions.operators.coreos.com                           CustomResourceDefinition    Installed
operatorgroups.operators.coreos.com                          CustomResourceDefinition    Installed
installplans.operators.coreos.com                            CustomResourceDefinition    Installed
clusterserviceversions.operators.coreos.com                  CustomResourceDefinition    Installed
aggregate-olm-edit                                           ClusterRole                 Installed
catalog-operator                                olm          Deployment                  Installed
olm-operator                                    olm          Deployment                  Installed
operatorhubio-catalog                           olm          CatalogSource               Installed
olm-operators                                   olm          OperatorGroup               Installed
aggregate-olm-view                                           ClusterRole                 Installed
operators                                                    Namespace                   Installed
global-operators                                operators    OperatorGroup               Installed
olm-operator-serviceaccount                     olm          ServiceAccount              Installed
packageserver                                   olm          ClusterServiceVersion       Installed
system:controller:operator-lifecycle-manager                 ClusterRole                 Installed
catalogsources.operators.coreos.com                          CustomResourceDefinition    Installed
olm-operator-binding-olm                                     ClusterRoleBinding          Installed
```

If it is not installed or any of the services have issue then follow steps in [Enabling OLM](https://master.sdk.operatorframework.io/docs/olm-integration/quickstart-bundle/#enabling-olm).

If you get an error as follows

```bash
$ operator-sdk olm status

FATA[0002] Failed to get OLM status: error getting installed OLM version (set --version to override the default version): no existing installation found 
```

Then try installing a version as follows:

```bash
$ operator-sdk olm uninstall --version 0.16.1

$ operator-sdk olm install --version 0.16.1
```

- Deploy the Operator on your cluster using OLM and Operator SDK as follows:

```bash
$ kubectl create ns wordpress-operator-system
namespace/wordpress-operator-system created

$ operator-sdk run bundle $BUNDLE_IMG -n wordpress-operator-system

INFO[0006] Successfully created registry pod: docker-io-<registry>-wordpress-operator-bundle-v0-0-1 
INFO[0006] Created CatalogSource: wordpress-operator-catalog 
INFO[0006] OperatorGroup "operator-sdk-og" created      
INFO[0006] Created Subscription: wordpress-operator-v0-0-1-sub 
INFO[0053] Approved InstallPlan install-s2b9r for the Subscription: wordpress-operator-v0-0-1-sub 
INFO[0053] Waiting for ClusterServiceVersion "wordpress-operator-system/wordpress-operator.v0.0.1" to reach 'Succeeded' phase 
INFO[0054]   Waiting for ClusterServiceVersion "wordpress-operator-system/wordpress-operator.v0.0.1" to appear 
INFO[0057]   Found ClusterServiceVersion "wordpress-operator-system/wordpress-operator.v0.0.1" phase: Pending 
INFO[0059]   Found ClusterServiceVersion "wordpress-operator-system/wordpress-operator.v0.0.1" phase: Installing 
INFO[0075]   Found ClusterServiceVersion "wordpress-operator-system/wordpress-operator.v0.0.1" phase: InstallReady 
INFO[0076]   Found ClusterServiceVersion "wordpress-operator-system/wordpress-operator.v0.0.1" phase: Installing 
INFO[0092]   Found ClusterServiceVersion "wordpress-operator-system/wordpress-operator.v0.0.1" phase: Succeeded 
INFO[0092] OLM has successfully installed "wordpress-operator.v0.0.1" 
```

- Check status of the deployed operator:

```bash
$ kubectl get catalogsource -n wordpress-operator-system
NAME                         DISPLAY              TYPE   PUBLISHER      AGE
wordpress-operator-catalog   wordpress-operator   grpc   operator-sdk   9m5s

$ kubectl get pod -n wordpress-operator-system
NAME                                                              READY   STATUS      RESTARTS   AGE
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a947xjvx   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a948rcjg   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a9497csj   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a949smlb   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94cbq9b   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94fnlst   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94fsxxq   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94fwqvl   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94g8cpd   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94hphht   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94jtnjj   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94k86v8   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94k8psq   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94lgkcd   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94n6kz2   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94p9hqd   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94phw6z   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94qxszm   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94sdpbh   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94t5k49   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94tf85r   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94vrmc7   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94z4bjg   0/1     Completed   0          10m
c1c6f442e63fb61858ba2f9d875209c00e31f1fe9b742609c4a1bc7a94zsjz9   0/1     Completed   0          10m
docker-io-<registry>-wordpress-operator-bundle-v0-0-1             1/1     Running     0          10m
wordpress-operator-controller-manager-7f8b8b4d65-4ds57            2/2     Running     0          9m43s

$ kubectl get packagemanifests -n wordpress-operator-system | grep wordpress-operator
wordpress-operator   wordpress-operator   11m

$ kubectl get clusterserviceversion -n wordpress-operator-system
NAME                        DISPLAY          VERSION   REPLACES   PHASE
wordpress-operator.v0.0.1   Wordpress Helm   0.0.1                Succeeded

$ kubectl get deployment -n wordpress-operator-system
NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
wordpress-operator-controller-manager   1/1     1            1           13m
```

- [Deploy an instance of the WordPress chart](https://github.com/IBM/operator-guild/blob/main/docs/content/howto/existing_helm_chart_operator.md#deploy-wordpress) to validate that the operator is working as expected.

16. Test - Cleanup the operator:

First the deployed chart:

```bash
$ kubectl delete -f config/samples/wordpress-demo.yaml -n wordpress-demo
wordpress.helm-chart.example.com "wordpress-demo" deleted

$ kubectl delete -n wordpress-demo --all
persistentvolumeclaim "data-wordpress-demo-mariadb-0" deleted
```

Then the operator and its resources:

```bash
$ kubectl get subscription -n wordpress-operator-system 
NAME                            PACKAGE              SOURCE                       CHANNEL
wordpress-operator-v0-0-1-sub   wordpress-operator   wordpress-operator-catalog   alpha

$ kubectl delete subscription wordpress-operator-v0-0-1-sub -n wordpress-operator-system 
subscription.operators.coreos.com "wordpress-operator-v0-0-1-sub" deleted

$ kubectl get clusterserviceversion -n wordpress-operator-system
NAME                        DISPLAY          VERSION   REPLACES   PHASE
wordpress-operator.v0.0.1   Wordpress Helm   0.0.1                Succeeded

$ kubectl delete clusterserviceversion wordpress-operator.v0.0.1 -n wordpress-operator-system
clusterserviceversion.operators.coreos.com "wordpress-operator.v0.0.1" deleted

# When OLM uninstalls an operator it does not remove any of the operator’s owned CRDs, APIServices, or CRs in order to prevent data loss. This step cleans up such resources.
$ make undeploy
```

17. Upload the image to the Red Hat project registry:

```bash
# Use Registry Key from https://connect.redhat.com/project/<project_id>/images/upload-image as the password when prompted from the 'docker login' command below.
# Note: On Mac, it doesn't seem to accept the password at the prompt. Use the --password-stdin flag instead.
# Refer to the Docker docs for more details:
# https://docs.docker.com/engine/reference/commandline/login/provide-a-password-using-stdin

# Login to the RH registry
$ docker login -u unused scan.connect.redhat.com

# Get the ID of the WordPress bundle image
$ docker images

# Tag using your project ID
$ docker tag <wordpress-operator_image_id> scan.connect.redhat.com/<ospid-id>/wordpress-operator-bundle:v0.0.1

# Push the iamge to the RH registry
$ docker push scan.connect.redhat.com/<ospid-id>/wordpress-operator-bundle:v0.0.1
```

18. Check image has been uploaded and scanned:

- Go to `https://connect.redhat.com/project/<project_id>/images`

- It might take a while for the image to appear. You then need to wait for the certification process to finish.

- If "certification test" passed then continue to next step. Otherwise, check the scan logs for errors and update the image accordingly.

19. Go back to "Checklist" (https://connect.redhat.com/project/<project_id>/checklist) and finish any items that are not complete. The image is now certified.

## References

1. [Partner Guide for OpenShift Operator and Container Certification](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/)

2. [The canonical source for Kubernetes Operators that appear on OperatorHub.io, OpenShift Container Platform and OKD](https://operator-framework.github.io/community-operators/)

3. [Certified Operator Build Guide](https://redhat-connect.gitbook.io/certified-operator-guide/)

4. [Operator SDK](https://sdk.operatorframework.io/)
