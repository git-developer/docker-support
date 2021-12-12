# Docker support for GitLab CI
Utilities for building docker images using GitLab CI.

## Docker Template
[`docker-template.yml`](gitlab-ci/docker-template.yml) contains two pipeline configurations:
* _Build_ builds a Docker image in GitLab CI (default).
* _Update-Check_ performs an update check and triggers _Build_ accordingly.

### Minimal example
When the following configuration is saved as `.gitlab-ci.yml` in the same directory as a Dockerfile, GitLab CI will build an image for the platforms AMD64
and ARMv7 and publish them to the Docker registry:
```yaml
include:
  remote: "https://github.com/git-developer/docker-support/raw/v3.2.0/gitlab-ci/docker-template.yml"
```

### Pipeline Steps

#### Build Pipeline
This pipeline is enabled by default or when the variable `PIPELINE_MODE` is set to `build`.
|Step|Stage|Job|Description|Default|
|----|-----|---|-----------|-------|
|1a|post_checkout|prepare_tags|Read [tags](https://docs.docker.com/engine/reference/commandline/tag/)|`latest`|
|1b|post_checkout|prepare_labels|Read [labels](https://docs.docker.com/config/labels-custom-metadata/).|All annotations defined by the [Open Container Initiative](https://github.com/opencontainers/image-spec/blob/master/annotations.md#pre-defined-annotation-keys). The values are determined automatically from git repository metadata.|
|2|pre_build|prepare_build_arguments|Read [build arguments](https://docs.docker.com/engine/reference/commandline/build/#set-build-time-variables---build-arg)|none|
|3|build|build_image|Build the image for all [target platforms](https://docs.docker.com/buildx/working-with-buildx/#build-multi-platform-images).|`linux/amd64,linux/arm/v7`|
|4|push|publish_image|Add all tags and labels to the image and publish it to a Docker registry.||

#### Update-Check Pipeline
This pipeline is enabled when the variable `PIPELINE_MODE` is set to `update-check`.
|Step|Stage|Job|Description|
|----|-----|---|-----------|
|1|post_checkout|trigger_on_update|Check if one of the `UPDATE_CHECK_URLS` is newer than the Docker image. If so, trigger the pipeline _Build_.|

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
  remote: "https://github.com/git-developer/docker-support/raw/v3.2.0/gitlab-ci/docker-template.yml"

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

### Update Check
The Pipeline _Update-Check_ is designed for projects that download and build an
application hosted externally.
#### Activation
To automatically build a project whenever an
external resource is updated, create a
[pipeline schedule](https://docs.gitlab.com/ee/ci/pipelines/schedules.html)
in GitLab with an interval of your choice (e.g. _Every day_) and set the variable
`PIPELINE_MODE` to `update-check`.

#### Execution
The variable `UPDATE_CHECK_URLS` is used to check the date of one or more
external resources. It may contain a space-separated list of URLs pointing to a
HTTP resource or a git repository. If the latest Docker image (multi-platform)
tagged with the value of variable `IMAGE_VERSION` is outdated,
a pipeline for project `DOWNSTREAM_PROJECT` on branch `DOWNSTREAM_REF` is
triggered. The variables for project and ref default to the current
environment, so that by default the current project is build.

#### Configuration
The variable `UPDATE_CHECK_URLS` may be defined where it is most appropriate,
e.g. in the pipeline schedule or in the project's `.gitlab-ci-yml`, e.g.
```yaml
variables:
  UPDATE_CHECK_URLS: 'https://www.example.org/ https://github.com/demo-group/demo-repo.git'
```

The date of the Docker images is read from a Docker registry,
defaulting to Docker Hub.
The following variables may be used to configure a custom registry
(see [`docker-support.yml`](gitlab-ci/docker-support.yml) for default values):
*  `REGISTRY_BASE_URL`
*  `REGISTRY_AUTH_BASE_URL`
*  `REGISTRY_AUTH_SERVICE`

This job supports
[registry authentication](https://docs.docker.com/registry/spec/auth/token/).
If the registry requires credentials, the pre-defined variables are used
(usually defined as GitLab project or group variable):
* `CI_REGISTRY_USER`
* `CI_REGISTRY_PASSWORD`

## Docker Support
[`docker-support.yml`](gitlab-ci/docker-support.yml) contains the building
blocks for `docker-template.yml`. It may be used to create custom build
configurations if `docker-template.yml` is not sufficient.
