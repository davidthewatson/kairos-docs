---
title: "Development notes"
linkTitle: "Development"
weight: 1
date: 2022-11-13
description: >
  Guidelines when developing Kairos
---

Here you can find development notes intended for maintainers and guidance for new contributors.

## Repository structure

Kairos uses [earthly](https://earthly.dev/) as a build system instead of Makefiles. This ensures that despite the environment you should be able to build `Kairos` seamlessly. To track external packages (like kernels, additional binaries, and so on) which follow their own versioning [luet](https://luet.io) is used and there is a separate [repository](https://github.com/kairos-io/packages) with package building specifications.

- [The Kairos repository](https://github.com/kairos-io/kairos) - contains the OS definitions (`Dockerfile`s) and configuration. The releases generate ISOs without (Kairos core) and with (Kairos standard) Kubernetes engine.
- [The kairos-agent repository](https://github.com/kairos-io/kairos-agent/) contains the `kairos-agent` code
- [The packages repository](https://github.com/kairos-io/packages) contains package specifications used by `kairos` while building OS images.
- [The provider-kairos repository](https://github.com/kairos-io/provider-kairos) contains the kairos provider component which uses the SDK to bring up a Kubernetes cluster with `k3s`.

## Build Kairos

To build a Kairos OS you only need Docker. There is a convenience script in the root of the repository `earthly.sh` which wraps `earthly` inside Docker to avoid to install locally which can be used instead of `earthly` (e.g. `./earthly.sh +iso ...`). However, for daily development, it is strongly suggested to install Earthly it on your workstation. The `earthly.sh` script runs `earthly` in a container, and as such there are limitations on image caching between builds.

To build a Kairos ISO, you need to specify a few parameters. For example, to build Kairos Fedora with `earthly` installed locally:

```bash
earthly +iso \
  --FAMILY=rhel \
  --FLAVOR=fedora \
  --FLAVOR_RELEASE=38 \
  --BASE_IMAGE=fedora:38 \
  --MODEL=generic \
  --VARIANT=core
```

- **Flavor**: is the distribution (the name flavor is kept for backwards compatibility). In this case `fedora`
- **Family**: is a group of distributions that get built in a very similar way. In this case `rhel`, which can build `fedora`, `rockylinux` and `almalinux`.
- **Flavor release**: is the version of the distribution. In this case `38`
- **Base image**: is the base image used to build the distribution. In this case `fedora**:38`, but you could use your own image which itself is based on `fedora:38`.
- **Model**: is the hardware model. In this case all ISOs are `generic`
- **Variant**: is the Kairos variant, which can be `core` or `standard`. `core` is the minimal Kairos OS, while `standard` includes `k3s`.

This will build a container image from scratch and create an ISO which is ready to be booted.

Note earthly targets are prefixed with `+` while variables are passed as flags, and `ARGS` can be passed as parameters with `--`.

### Adding flavors

Every source image used as a flavor is inside the `images` folder in the top-level directory. Any Dockerfile have the extension corresponding to the family which can be used as an argument for earthly builds (you will find a `Dockerfile.rhel` that will be used by our command above).

To add a flavor is enough to create a Dockerfile corresponding to the flavor and check if any specific setting is required for it on its build target.

Generally to add a flavor the image needs to have installed:

- An init system (systemd or openRC are supported)
- Kernel
- GRUB
- rsync

If you are building a flavor without Earthly, be sure to consume the packages from our repository to convert it to a Kairos-based version.

### Bumping packages

Let's assume there is some change you introduce in a package consumed by kairos
(e.g. [kcrypt](https://github.com/kairos-io/kcrypt)). In order to build a kairos image
with the updated package, first tag the repository (`kcrypt` in our example).
Then trigger [the auto-bump pipeline](https://github.com/kairos-io/packages/actions/workflows/autobump.yaml)
on the packages repository. This should create at least on PR which bumps the desired package to the latest tag.
It may also create more PRs if other packages had new tags recently. When PR passes CI, merge it.
Next, in order to bump the packages on kairos, manually trigger [the bump-repos pipeline](https://github.com/kairos-io/kairos/actions/workflows/bump_repos.yml).
This will automatically open a PR on the kairos repository which can be merged when it passes CI.
After this, any images produced by the kairos repository, will have the latest version of the package(s).

## New controllers

Kairos-io adopts [operator-sdk](https://github.com/operator-framework/operator-sdk).

To install `operator-sdk` locally you can use the `kairos` repositories:

1. Install Luet:
   `curl https://luet.io/install.sh | sudo sh`
2. Enable the Kairos repository locally:
   `luet repo add kairos --url quay.io/kairos/packages --type docker`
3. Install operator-sdk:
   `luet install -y utils/operator-sdk`

### Create the controller

Create a directory and let's init our new project it with the operator-sdk:

```bash

$ mkdir kairos-controller-foo
$ cd kairos-controller-foo
$ operator-sdk init --domain kairos.io --repo github.com/kairos-io/kairos-controller-foo

```

### Create a resource

To create a resource boilerplate:

```
$ operator-sdk create api --group <groupname> --version v1alpha1 --kind <resource> --resource --controller
```

### Convert to a Helm chart

operator-sdk does not have direct support to render Helm charts (see [issue](https://github.com/operator-framework/operator-sdk/issues/4930)), we use [kubesplit](https://github.com/spectrocloud/kubesplit) to render Helm templates by piping kustomize manifests to it. `kubesplit` will split every resource and add a minimal `helm` templating logic, that will guide you into creating the Helm chart.

If you have already enabled the `kairos` repository locally, you can install `kubesplit` with:

```
$ luet install -y utils/kubesplit
```

### Test with Kind

Operator-sdk will generate a Makefile for the project. You can add the following and edit as needed to add kind targets:

```
CLUSTER_NAME?="kairos-controller-e2e"

kind-setup:
	kind create cluster --name ${CLUSTER_NAME} || true
	$(MAKE) kind-setup-image

kind-setup-image: docker-build
	kind load docker-image --name $(CLUSTER_NAME) ${IMG}

.PHONY: test_deps
test_deps:
	go install -mod=mod github.com/onsi/ginkgo/v2/ginkgo
	go install github.com/onsi/gomega/...

.PHONY: unit-tests
unit-tests: test_deps
	$(GINKGO) -r -v  --covermode=atomic --coverprofile=coverage.out -p -r ./pkg/...

e2e-tests:
	GINKGO=$(GINKGO) KUBE_VERSION=${KUBE_VERSION} $(ROOT_DIR)/script/test.sh

kind-e2e-tests: ginkgo kind-setup install undeploy deploy e2e-tests
```
