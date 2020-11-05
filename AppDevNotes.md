# Application Development Notes and Best Practices

JARVICE users have the ability to either develop custom applications or derive custom workflows from existing catalog applications.  This document provides some high level notes and best practices.  It does not cover:

* The `Dockerfile` format itself - please see [Build and run your image](https://docs.docker.com/get-started/part2/) in the Docker documentation for a tutorial.
* Layer optimization - it's assumed the developer understands how Docker layers work and how to avoid creating excess layers.
* Kubernetes or the JARVICE platform itself, as this would be out of scope of this document.
* Any other application-specific details, as these are assumed to be understood by the developer packaging them.

## Original vs Derived

While most Docker containers are derived (using the `FROM` directive in the `Dockerfile` to base on an existing image in a registry), an "original" app refers to an application container that does not inherit from an existing JARVICE container (either Nimbix packaged or 3rd party vendor/user packaged).  Other than this, there is no real distinction when creating application containers from a JARVICE perspective, e.g.:

* JARVICE can run "unprepared" Docker containers directly from an upstream registry - unprepared meaning they do not have [image-common](https://github.com/nimbix/image-common) installed, etc.  JARVICE provides a default [AppDef](https://jarvice.readthedocs.io/en/latest/appdef/) with a *Server* endpoint that can be used to launch these containers with a web-shell into them for interactive use.
* JARVICE can run packaged applications either interactively or in batch mode based on the endpoints defined in the AppDef.
* JARVICE can run derived containers with custom scripts that inherit from full packaged applications - e.g. modifying workflows on an existing application image, changing the AppDef or other metadata to better present workflow choices, or any combination of this.

## Building locally versus using the *PushToCompute* builder

App developers on the JARVICE platform can use the build functionality in the app target from the *PushToCompute* view to create containers, but this may require additional setup at deployment-time.  An alternate method is to use a local Docker installation to build, tag, and push container images to a registry of choice.  This also affords developers more flexibility in revision control and source tree structures, and may be preferable if an SSH-accessible Git service is not available to the JARVICE system.  For illustrative purposes, the remainder of this document will focus on locally built and pushed images.

For the cloud-based *PushToCompute* flow please see the [PushToCompute Pipeline Overview](https://jarvice.readthedocs.io/en/latest/cicd/).

## Prerequisites for building and pushing JARVICE apps

1. Docker installed on the local system (see *Docker Desktop* in [Get Started with Docker](https://www.docker.com/get-started))
2. Docker registry account, either on a private registry server or a publicly hosted one such as [Docker Hub](https://hub.docker.com), [gcr.io](https://gcr.io), etc.
3. Revision control system to manage `Dockerfile` and associated objects (**highly recommended**)

#### Notes

1. Some Docker registries have limits on egress which may affect container pulls - for large scale app deployment, please consider storing containers either on a private registry server or in a Nimbix-provided "bucket" generally as part of your JARVICE license entitlement.
2. Some docker registries have time-limited authentication sessions, which will require periodic re-authentication in order to run containers after they're deployed; this may be achieved manually or with custom scripting to create and refresh authentication secrets on the cluster.

## Directory structure for container building

A `docker build` command requires pointing at a directory that contains a `Dockerfile` as well as all objects referred to in it; these may exist in subdirectories of the tree, but cannot exist outside the tree.  It's not recommended that large binaries be stored directly in the tree as this generally affects revision control adversely - an alternative is to use an object storage or some form of external storage that can be accessed via a `curl` command inside the `Dockerfile` to download objects.

## General Flow for building and deploying JARVICE application containers

1. Modify desired file(s) in the tree, commit to revision control as appropriate.
2. Use `docker build` to build the container - optionally use the `-t` argument to tag it, or use an explicit `docker tag` operation once it successfully completes.
3. Use `docker push` to push the tagged container to its respective registry.
4. In your JARVICE platform account, log in to the respective Docker registry (if the repository you pushed to is private) using the login widget in the *PushToCompute* view; if previously logged in and the credentials have not changed (or do not need to be re-autorhized) you can skip this step.
5. In your JARVICE platform account, click the context menu in the application target in the *PushToCompute* view and select the *Pull* option.
6. In your JARVICE platform account, click the context menu in the application target in the *PushToCompute* view and select the *History* option.
7. Once the pull completes, as shown in the pull history window, close the window and wait a few seconds for the application target to refresh (this will apply any AppDef changes you may have made)

At this point you should be able to run the application by clicking on it and selecting a workflow; if this application is team shared or already public, the same changes will apply for other users who have access to it.  Note that AppDef changes may take 30-60 seconds to refresh automatically.

### Troubleshooting

Problem|Likely Cause|Resolution
---|---|---|
`docker build` failure (step 2)|explicit error message from the failing layer should be self-explanatory - e.g. shell syntax error or command failure|correct the error in the `Dockerfile`, and repeat step 2
`docker push` failure (step 3)|generally caused by an authentication failure to the regsitry, but read the error message carefully|if authentication failure, use the `docker login` command to log in or refresh the authentication to the registry, then repeat step 3
JARVICE docker login failure (step 4)|generally caused either by a mistyped username/password, an invalid JSON key (if using the JSON upload method), or a communications error to the registry|ensure the username/password is correct, and if it is, contact your system administrator
JARVICE docker pull failure (step 5/6)|will be explained in the history window (from step 6)|ensure the container address is correct and that the login was successful from step 4; if this is the case, contact your system administrator
JARVICE docker pull does not complete (step 5/6)|the `Dockerfile` is not optimized properly; the pull process downloads layers in reverse order until it finds all the metadata for a JARVICE app; see [Best Practices](#best-practices) for details on adding this optimization at the end of the `Dockerfile`|optimized the `Dockerfile` and repeat step 5; the pull should be relatively instantaneous; if it still does not complete, contact your system administrator
Application metadata does not refresh (e.g. AppDef, screenshot, EULA, etc.) - step 7|either the metadata was not properly updated in the container, or it has not yet refreshed|check that the local container build has the correct metadata (see below), and **ensure that you pushed it to the registry after the build**; if it does, perform an explicit browser page reload and check again

#### Checking metadata files inside a built container

It is possible to run an ephemeral instance of a given container to check its metadata from any system that has Docker installed.  If it's the build system, it likely won't be pulled again as it will be cached locally.  For example, to dump the contents of `AppDef.json` from its expected location in the container tagged as `gcr.io/mybucket/mycontainer:v1`:
```sh
docker run --rm --entrypoint=/bin/cat gcr.io/mybucket/mycontainer:v1 /etc/NAE/AppDef.json
```
You can pipe the above command to a file for easy comparison or inspection of the AppDef, and you can also use the same command to inspect any files inside the container; to list files, change the entrypoint to `/bin/sh`, etc.

You can also enter the container interactively to inspect it - e.g.:
```
docker run --rm -it --entrypoint=/bin/sh gcr.io/mybucket/mycontainer:v1
```
Once you exit the interactive shell, the ephemeral layers will be discarded automatically (hence the `--rm` flag)

**If you are finding the metadata not updating in JARVICE even after checking in the local container, ensure the `docker push` worked correctly (step 3) after the last successful build (step 2).**

Note also that Docker uses checksums in layers to determine what to rebuild, and will rebuild all subsequent layers when it decides one must be rebuilt.  Checksums pertain to all lines in a `Dockerfile` - for `COPY` lines, Docker will actually checksum the objects being copied into the container.  If there is no change to a layer, it will be used from cache if possible.  If you are downloading objects from outside the local tree using for example the `curl` command, Docker may cache the entire layer even if the remote objects changed.  To force a layer build, either clear your local build cache or insert a meaningless space character in the `RUN` layer itself - e.g. change `curl https://abcdefg.io/x.txt` to `curl  https://abcdefg.io/x.txt`.

### Best Practices

1. Try to avoid deleting and creating new app targets in JARVICE for application updates; if the application update replaces a previous version, reuse the same target and simply change the container address to reflect the proper tag.  This reduces the risk of referential integrity errors in JARVICE.  Note that changing an app target while users are already consuming that app will not impact them until they launch the app again in the future.
2. Use revision control for the container build tree, and try to correlate source tags with container tags; this will make it easier to determine what source version a container was built from in the future; avoid building containers from source files that are not committed, as this will present inconsistencies.
3. Add the following layer at the very end of your `Dockerfile` which will substantially speed up pulls into JARVICE:
    ```sh
    RUN mkdir -p /etc/NAE && touch /etc/NAE/{screenshot.png,screenshot.txt,license.txt,AppDef.json}
    ```
    see *Best Practices* in [Docker Images on JARVICE](https://jarvice.readthedocs.io/en/latest/docker/) for more information.

## Links

* [Get Started with Docker](https://www.docker.com/get-started)
* [JARVICE Developer Reference](https://jarvice.readthedocs.io)
