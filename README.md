# Docker support for GitLab CI
Utilities for building docker images using GitLab CI.

## Docker Template
[`docker-template.yml`](gitlab-ci/docker-template.yml) contains a pipeline to build a Docker image in GitLab CI.

### Minimal example
When the following configuration is saved as `.gitlab-ci.yml` in the same directory as a Dockerfile, GitLab CI will build an image for the platforms AMD64
and ARMv7 and publish them to the Docker registry:
```yaml
include:
  remote: "https://github.com/git-developer/docker-support/raw/v2.1.0/gitlab-ci/docker-template.yml"
```

### Pipeline Steps
The Docker template consists of the following steps:
|Step|Stage|Job|Description|Default|
|----|-----|---|-----------|-------|
|1|Setup|Collect Build Arguments|Read [build arguments](https://docs.docker.com/engine/reference/commandline/build/#set-build-time-variables---build-arg)|none|
|2a|Prepare|Collect Tags|Read [tags](https://docs.docker.com/engine/reference/commandline/tag/)|`latest`|
|2b|Prepare|Collect Labels|Read [labels](https://docs.docker.com/config/labels-custom-metadata/).|All annotations defined by the [Open Container Initiative](https://github.com/opencontainers/image-spec/blob/master/annotations.md#pre-defined-annotation-keys). The values are determined automatically from git repository metadata.|
|2c|Prepare|Build Image|Build the image for all [target platforms](https://docs.docker.com/buildx/working-with-buildx/#build-multi-platform-images).|`linux/amd64,linux/arm/v7`|
|3|Publish|Publish Image|Add all tags and labels to the image and publish it to a Docker registry.||

### Build for different target platforms
To specify the target platforms, set the variable `IMAGE_PLATFORMS`. Example:
```yaml
variables:
  IMAGE_PLATFORMS: 'linux/i386,linux/arm/v6'
```

### Publish to Docker Hub
To publish to Docker hub, set the following variables within the CI/CD settings in GitLab:
|Key|Value|
|---|-----|
|`CI_REGISTRY`|`docker.io`|
|`CI_REGISTRY_IMAGE`|`docker.io/$CI_PROJECT_PATH`|
|`CI_REGISTRY_USER`| _Username for Docker Hub_|
|`CI_REGISTRY_PASSWORD`| _Access Token for Docker Hub_|

It is highly recommended to
[generate an Access Token on Docker Hub](https://docs.docker.com/docker-hub/access-tokens/)
and use it in place of the password.

### Custom tags
- To replace the default tag `latest`, set the variable `IMAGE_VERSION`. Example:
  ```yaml
  variables:
    IMAGE_VERSION: "newest"
  ```
- Add tags:
    1. Create the directory `tags`
    1. For each tag, create a file within this directory and write the tag to it.  Example:
  ```yaml
  Add Some Tag:
    stage: prepare
    script:
    - mkdir -p tags
    - echo "${IMAGE_NAME}:fancy-tag" >"tags/fancy-tag"
    artifacts:
      paths:
      - tags
  ```

### Custom labels
- Add labels:
    1. Create the directory `labels`
    1. For each label, create a file within this directory and write the label to it. Example:
  ```yaml
  Add Some Label:
    stage: prepare
    script:
    - mkdir -p labels
    - echo "com.example.some-label=some-value" >"labels/some-label"
    artifacts:
      paths:
      - labels
  ```

### Custom build arguments
* When the build arguments do not contain spaces or special characters,
  simply put them into the variable `BUILD_ARGS`. Example:
  ```yaml
  variables:
    BUILD_ARGS: 'APP_NAME=foo-app APP_URL=http://example.org/foo'
  ```

* When the build arguments contain special characters:
    1. Create the directory `build-args`
    1. For each build argument, create a file within this directory and write the build argument to it. Example:
  ```yaml
  Add Some Build Arguments:
    stage: setup
    script:
    - mkdir -p build-args
    - printf "foo bar\nbaz" >build-args/some-build-arg
    artifacts:
      paths:
      - build-args
  ```

### Use values from the image for tags and labels
The job `.docker_support:.with_bare_image` may be used to analyze the built
image, or even start a container from it, before publishing to the registry.

The following example uses the busybox version as image tag:
```yaml
include:
  remote: "https://github.com/git-developer/docker-support/raw/v2.1.0/gitlab-ci/docker-template.yml"

stages:
  - setup
  - prepare
  - analyze
  - publish

Collect Additional Tags:
  extends: .docker_support:.with_bare_image
  stage: analyze
  artifacts:
    paths:
    - tags
  script:
  - echo >tags/version "${IMAGE_NAME}:busybox-$(docker run --rm "${IMAGE_NAME}:${BUILD_CACHE}" sh -c 'busybox | head -1 | cut -d " " -f 2')"
```

## Docker Support
[`docker-support.yml`](gitlab-ci/docker-support.yml) contains the building
blocks for `docker-template.yml`. It may be used to build custom build
configurations if `docker-template.yml` is not sufficient.

