# Integrating Anchore scanning with Codeship Pro

## Introduction

As Docker images and container technologies become more regular and integral pieces of the software development lifecycle, methods for securing these artifacts begin to become a necessity, and should be clearly defined steps in continuous integration pipelines.

Anchore is a service that conducts a deep image inspection and analysis of Docker images. Following this inspection, Anchore evaluates heavily customizable user-defined polices against the the analyzed image. Users are left with a detailed manifest of the contents of the image, and a final result of the policy evaluation. Most commonly, Anchore is run as a step in a CI pipeline. During which, images are analyzed and evaluted by Anchore. It is the final result of the policy evaluation that (depending on the steps defined) typically dictates whether the image should progress to the next stage in the pipeline. The Anchore results combined with a properly set up CI pipeline will ensure a high level of confidence when deploying container images into production environments. 

This will walkthrough integrating Anchore scanning into a Codeship pipeline. During the first step, a Docker image will be built from a Dockerfile. Following this, during the second step Anchore will scan the image, and depending on the result of the policy evaluation, proceed to the final step. During the final step the built image will be pushed to a Docker registry.

## Prerequisites

- Docker Installed
- Jet CLI: https://documentation.codeship.com/pro/jet-cli/installation/
- Codeship Pro Account: https://documentation.codeship.com/pro/quickstart/getting-started/
- Github Account
- Anchore Engine Service: https://github.com/anchore/anchore-engine

## Setup

Prior to setting up your Codeship build pipeline, an Anchore Engine service needs to be accessible from the pipeline. Typically this is on port 8228. In this example, I have an Anchore Engine service on AWS EC2 with standard configuration. I also have a Dockerfile in a Github repository that I will build an image from during the first step of the pipeline. In the final step, I will be pushing the built image to an image repository in my personal Dockerhub.

The Github repository can be referenced here: https://github.com/valancej/Codeship

Repository contents:

- codeship-services.yml (Contains all services needed to run your CI/CD builds)
- codeship-steps.yml (Contains all the steps for your CI/CD process)
- Dockerfile
- dockercfg.encrypted (Docker registry credentials)
- env.encrypted (Environment variables)

For more info on using encrypted files with Jet CLI: https://documentation.codeship.com/pro/jet-cli/encrypt/


Most typically, we advise on having a staging registry and production registry. Meaning, being able to push and pull images freely from the staging/dev registry, while maintaining more control over images being pushed to the production registry. In this example, I am using the same registry for both.

I've added the following environment variables via the `env`file: 

If `ANCHORE_FAIL_ON_POLICY` is set to true, the pipeline will fail, and the image will not be pushed to the registry. 

- `ANCHORE_CLI_URL`
- `ANCHORE_CLI_USER`
- `ANCHORE_CLI_PASS`
- `ANCHORE_CLI_IMAGE`
- `ANCHORE_RETRIES`
- `ANCHORE_FAIL_ON_POLICY`


The Docker registry has been configured with the `dockercfg` file:

```
{
	"auths": {
		"https://index.docker.io/v1/": {
			"auth": "anZhbGFuY2U6MjI2MTM3QGtLaw=="
		}
	},
	"HttpHeaders": {
		"User-Agent": "Docker-Client/17.10.0-ce (linux)"
	}
}
```

## Build Image

In the first step of the pipeline, we build a Docker image from a Dockerfile as definied in our `codeship-steps.yml`:

```
- name: imagebuildstep
  service: imagebuild
  type: push
  image_name: jvalance/sampledockerfiles
  encrypted_dockercfg_path: dockercfg.encrypted
```

and our `codeship-services.yml`:

```
imagebuild:
  build:
    dockerfile: Dockerfile
  cached: true
```

## Conduct Anchore Scan

In the second step of the pipeline, we scan the built image with Anchore as definied in our `codeship-steps.yml`:

```
- name: anchorestep
  service: anchorescan
  command: sh -c 'echo "Adding image to Anchore engine" && 
    anchore-cli image add $ANCHORE_IMAGE_SCAN &&
    echo "Waiting for image analysis to complete" &&
    anchore-cli image wait $ANCHORE_IMAGE_SCAN &&
    echo "Analysis complete" &&
    if [ "$ANCHORE_FAIL_ON_POLICY" == "true" ] ; then anchore-cli evaluate check $ANCHORE_IMAGE_SCAN  ; fi'
  encrypted_env_file: env.encrypted
```

and our `codeship-services.yml`:

```
anchorescan:
  image: anchore/engine-cli:latest
  encrypted_env_file: env.encrypted
```


Depending on the output of the policy evaluation, the pipeline may or may not fail. In this case, I have set `ANCHORE_FAIL_ON_POLICY` to true and exposed port 22. This is in violation of a policy rule, so the build will fail during this step.


## Push image

In the final step of the pipeline, we push the Docker image to a registry as defined in the `codeship-steps.yml`:

```
- name: imagepushstep
  service: imagebuild
  type: push
  image_name: jvalance/sampledockerfiles
  encrypted_dockercfg_path: dockercfg.encrypted
```

and our `codeship-services.yml`:

```
anchorescan:
  image: anchore/engine-cli:latest
  encrypted_env_file: env.encrypted
```

## Conclusion

By looking at the example above, we can see how simple Anchore scanning can be added to our Codeship configuration in order to gain a deeper insight into the contents of a Docker image, and deploy with a higher level of confidence based on the results of our Anchore policy evaluation.

As a reminder, we advise having separate Docker registries for images that are being scanned with Anchore, and images that have passed an Anchore scan. For example, a registry for dev/test images, and a registry to certified, trusted, production-ready images. 