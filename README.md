# tpm-provider

## Overview
The `tpm-provider` is a golang libary that interfaces with a TPM device to support the capabilities of the ISecL TrustAgent.  The `tpm-provider` uses `tpm2-tss` and `'tpm2-abrmd` to access the TPM and supports targets RHEL 8.0.  Installing those packages on RHEL 8.0 will result in the following vesions of Tss2...

    tpm2-tss-2.0.0-4.el8.x86_64
    tpm2-abrmd-2.1.1-3.el8.x86_64

Due to the dependency on Tss2, the `tpm-provider` and any project that includes `tpm-provider` (ex. `go-trust-agent` and `workload-agent`) will need to be built in a Linux environment with those libraries and development tools installed.

## Building tpm-provider
`tpm-provider` uses 'Visual Studio Code Insiders' and docker containers to build and debug on Linux.

### Prerequisites
* Visual Studio Code - Insiders ('insiders' is used to support docker debuging)
* Docker
* golang

Building, debuging and ci/cd use the 'gta-devel' image defined in cicd/Dockerfile.  It currently uses Fedora 29 and includes tools for compiling go, c/c++, makeself, tpm2-tss, tpm2-abrmd, etc. The image also includes the tpm-simulator.

### Compiling tpm-provider
Typically, `tpm-provider` will be statically linked into an application (ex. `go-trust-agent`) vi `go.mod` files and does not need to be built independently.  However, the project does include a Makefile that compiles unit tests into `out/tpmprovider.test` (for convenience).  To compile `tp-provider`....

1. Create a 'gta-devel' docker image...
    1. `cd cicd`
    2. `docker build --tag=gta-devel --build-arg http_proxy=<proxy-if-needed> --build-arg https_proxy=<proxy-if-needed> .`
    3. `docker image ls` should show 'gta-devel'
2. Start a new instance of the container, mounting the root code directlry as `/docker_host` directory in the container
    1. `docker run -it -v $(pwd):/docker_host gta-devel -p 9443:1443 /bin/bash` (run this from the root directory of your development environment so that other ISecL projects are available in the container at '/docker_host')
    2. Configure git to access gitlab to resolve dependencies on other ISecL go libraries.
        1. `git config --global http.proxy http://proxy-us.intel.com:911`
        2. `git config --global https.proxy http://proxy-us.intel.com:911`
        3. `git config --global url."ssh://git@gitlab.devtools.intel.com:29418".insteadOf https://gitlab.devtools.intel.com`
        4. Create ssh keys in ~/.ssh (id_rsa and id_rsa.pub)
    3. `cd /docker_host/tpm-provider`
    3. `make`
    4. `out/tpmprovider-test` executable is compiled.  All unit tests can be invoked by running `out/tmpprovider.test` or individually by running `out/tpmprovider.test -test.run TestName`.

Note: The `gta-devel` docker contianer can be used in this fashion to build GTA, but cannot be used to run `tpm2-abrmd` because it must run as a service under `systemd`.  See `Unit Testing and TPM Simulator` below for instructions to run `systemd`, `tpm2-abrmd` and the TPM simulator in the `gta-devel` container.

## Unit Testing and Tpm Simulator
The `gta-devel` docker image also contains the IBM TPM simulator to support debugging and unit tests.  `tpm2-abrmd` has been configured to use the simulator (in the container) by adding the `--tcti=mssim` arguments to the .service file. 

In order to start the `tpm2-abrmd` service, the docker contianer must be running systemd.  The following instructions describe how to start `gta-devel` with `ssytemd` in order to run unit tests that use the TPM Simulator.

1. Start an container of `gta-devel` that runs `systemd`: `docker run --rm --privileged -ti -e 'container=docker' -v /sys/fs/cgroup:/sys/fs/cgroup:ro -v $(pwd):/docker_host -p 9443:1443 gta-devel /usr/sbin/init`
2. Use Docker/vscode to 'attach' to the container.
3. Run the unit tests by either...

    a. `make` and run `out/tpmprovider.test` or...

    b. Run `go test ./... -tags=unit_test`

### Starting/Stopping the TPM Simulator    

To start the simulator and tpm2-abrmd run...

    /simulator/src/tpm_server -rm&
    systemctl status tpm2-abrmd

The simulator and service can be reset/stopped by running...

    kill -9 `pgrep tpm_server`
    systemctl stop tpm2-abrmd

See `cicd/start-tpm-simulator.sh` for additional details.

