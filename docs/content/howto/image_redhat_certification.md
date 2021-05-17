# How to Red Hat certify a container image

This document covers how to prepare and Red Hat certify your container image for the goal of being added to [Red Hat Marketplace](https://marketplace.redhat.com/en-us).

## Image checklist

This is a base checklist of requirements for an image to be compatabile prior to certification.

1. The Base image must be (or must be based on) a supported Red Hat image, such as Red Hat Enterprise Linux or [Red Hat Universal Base Image](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/program-on-boarding/containers-with-red-hat-universal-base-image-ubi).

2. Confirm contents (images and packages) are from a trusted source and are maintained with CVE updates.

3. All contents (images and packages) should use the latest version unless a valid reason for not doing so.

4. Container image runs as non-root. Refer to [How to make a non-root based container](https://github.com/IBM/operator-guild/blob/main/docs/content/howto/how_to_make_a_nonroot_container.md) for more details. Don't use `USER nonroot:nonroot` with UBI base image as it doesn't exist for that image.

5. Uncompressed image should have less than 40 layers.

6. Valid licensing for image, refer to [Licenses Requirements](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/program-on-boarding/technical-prerequisites#licenses-requirements) for more details.

7. Product overview for the certified image, as well as other relevant documentation should be provided.

## Certification process

### Prerequisites

Follow the prerequisite steps as mentioned in the [Program Prerequisites](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/program-on-boarding/prerequisites). These prerequisites are part of [Certification Workflow](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/program-on-boarding/certification-workflow).

### Certification steps

After you satisfy all the [Prerequisites](#prerequisites), follow the steps below for certification:

1. Follow the steps of "Application Certification" from [Creating a container application project](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/certify-your-application/creating-a-container-application-project) section but stop before [Uploading your container images](https://redhat-connect.gitbook.io/partner-guide-for-red-hat-openshift-and-container/certify-your-application/image-upload) section.

2. Add labels for Red Hat build service and scanner:

- Add the following labels to `Dockerfile`:

```yaml
FROM registry.access.redhat.com/ubi8/ubi-minimal:latest

# Required Labels for Red Hat build service and scanner
LABEL name="Example App" \
  vendor="IBM" \
  version="v0.0.1" \
  release="1" \
  summary="This is an example of a container image." \
  description="This operator will deploy a example app cluster."

USER 1001
WORKDIR /
COPY --from=builder /workspace/manager .
ENTRYPOINT ["/manager"]
```

> Note: The use of `USER 1001` above means a UID (User Identifier) of 1001. This UID is arbitrary on OpenShift as it will launch the container with a random UID. No GID( Group Identifier) is specified above and OpenShift will use a GID of 0 (root). This doesn't give any special permissions like UID of 0 (root), but will allow the setting of static file/directory permissions within the container image since all containers launch as GID 0 in OpenShift.

3. Add license file:

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
FROM registry.access.redhat.com/ubi8/ubi-minimal:latest

# Required Labels for Red Hat build service and scanner
LABEL name="Example App" \
  vendor="IBM" \
  version="v0.0.1" \
  release="1" \
  summary="This is an example of app." \
  description="This operator will deploy a example app cluster."

# Required Licenses for Red Hat build service and scanner
COPY license /licenses

USER 1001
WORKDIR /
COPY --from=builder /workspace/manager .
ENTRYPOINT ["/manager"]
```

4. Build and push image to image registry:

```bash
$ docker build -t example-app:v0.0.1

$ docker push YOUR-USER-NAME/example-app:v0.0.1
```

5. Test image by deploying it to OpenShift

Test using your own Kubernetes manifest file or check out the blog [Getting any Docker image running in your own OpenShift cluster](https://www.openshift.com/blog/getting-any-docker-image-running-in-your-own-openshift-cluster) for more details.

6. Upload the image to the Red Hat project registry:

```bash
# Use Registry Key from https://connect.redhat.com/project/<project_id>/images/upload-image as the password when prompted from the 'docker login' command below.
# Note: On Mac, it doesn't seem to accept the password at the prompt. Use the --password-stdin flag instead.
# Refer to the Docker docs for more details:
# https://docs.docker.com/engine/reference/commandline/login/provide-a-password-using-stdin

# Login to the RH registry
$ docker login -u unused scan.connect.redhat.com

# Get the ID of the image
$ docker images

# Use ospid-id from https://connect.redhat.com/project/<project_id>/images/upload-image
$ docker tag <example_app_id> scan.connect.redhat.com/<ospid-id>/example-app:v0.0.1

# Push the iamge to the RH registry
$ docker push scan.connect.redhat.com/<ospid-id>/example-app:v0.0.1
```

7. Check image has been uploaded and scanned:

- Go to `https://connect.redhat.com/project/<project_id>/images`

- It might take a while for the image to appear. You then need to wait for the certification process to finish.

- If "certification test" passed then continue to next step. Otherwise, check the scan logs for errors and update the image accordingly.

8. Go back to "Checklist" (https://connect.redhat.com/project/<project_id>/checklist) and finish any items that are not complete. The image is now certified.
