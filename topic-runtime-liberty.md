---

copyright:
  years: 2019
lastupdated: "2019-02-05"

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:codeblock: .codeblock}
{:pre: .pre}
{:tip: .tip}
{:note: .note}
{:important: .important}

# Open Liberty and WebSphere Liberty
{: #liberty}

[Open Liberty](https://openliberty.io/){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon") is a lightweight open-source Java runtime built using modular *features*. [WebSphere Liberty](https://developer.ibm.com/wasdev/){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon"). is a commercial version of Open Liberty. Liberty features support the following application frameworks:

* [MicroProfile](https://microprofile.io/){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon"), an open source project defining new standards and APIs to accelerate and simplify the creation of microservices.
* [Jakarta EE](https://jakarta.ee){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon") and [Java EE](https://www.oracle.com/technetwork/java/javaee/overview/index.html){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon"), including features for individual specifications, like JNDI or JAX-RS.
* [Spring Framework and Spring Boot](https://www.ibm.com/support/knowledgecenter/en/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/twlp_dep_springboot.html){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon"), including mechanisms to make compact containers from Spring Boot's fat jars.

There are a wide set of practical developer guides for creating microservices and cloud-native applications with Liberty at [https://openliberty.io/guides/](https://openliberty.io/guides/){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon").

## Optimized for Docker
{: #liberty-optimized}

When automated systems like Kubernetes are pushing container images around, image size starts to matter. The layers in Docker images are cached to help with this. Given its modular architecture, Liberty provides an efficient packaging pipeline for Docker containers, making it a great operational fit for cloud environments. One base image can be used to support many workloads, while allowing the resource footprint of running containers to vary based on the requirements of the service.

Liberty provides tools and optimized support for converting Spring Boot fat jars into compact, optimized Docker containers that take advantage of cached Docker image layers to improve cycle and publish times. By pushing the infrequently changing library dependencies down into a separate layer, and keeping only the application classes in the top layer, iterative rebuilds and re-deployments will be much faster. See more at [Creating Dual Layer Docker images for Spring Boot apps](https://openliberty.io/blog/2018/07/02/creating-dual-layer-docker-images-for-spring-boot-apps.html){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon").

## Liberty Docker Images
{: #liberty-images}

Both Open Liberty and WebSphere Liberty provide a variety of base images that you can use as starting points for your Java-based microservices. There are images that use small footprints or OpenJ9 VMs with different features pre-selected. For example:

1. open-liberty:microProfile2 or websphere-liberty:microProfile2
2. open-liberty:javaee8 or websphere-liberty:javaee8
3. open-liberty:springBoot2 or websphere-liberty:springBoot2

Check [websphere-liberty](https://hub.docker.com/_/websphere-liberty/){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon") or [open-liberty](https://hub.docker.com/_/open-liberty/){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon") on Docker Hub for the most up to date list of base images.

### Open Liberty and Docker
{: #openliberty-docker}

Choose the most appropriate image for your application from the [list of images on Docker Hub](https://hub.docker.com/_/open-liberty/){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon") (e.g. `open-liberty:microProfile`) and create a Dockerfile for your service that extends `FROM` it. Add the commands to copy your app and server configuration into the new image. As an example:

```docker
FROM open-liberty:microProfile\
COPY server.xml /config/\
ADD Sample1.war /config/dropins/
```
{: codeblock}

For more examples and working code, see the following Open Liberty guides:

* [Using Docker containers to develop microservices](https://openliberty.io/guides/docker.html){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon")
* [Running an application in a Docker container](https://openliberty.io/guides/getting-started.html#running-the-application-in-a-docker-container){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon")

### WebSphere Liberty and Docker
{: #wlp-docker}

There are some differences between Open Liberty and the commercial version, WebSphere Liberty. One of the most significant for creating Docker images is that the `installUtility` command is not available in Open Liberty.

WebSphere Liberty supports the same basic customization patterns that OpenLiberty does for Docker images, but the inherently modular design of Liberty makes it simple (and typical) to create a custom image that contains an application and the specific set of features it requires. WebSphere Liberty has an image for a feature-less kernel, [websphere-liberty:kernel](https://github.com/WASdev/ci.docker/blob/9d28dfba4d20596f89b393bc9e3ae8295feec469/ga/developer/kernel/Dockerfile){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon"), which makes a solid base for a truly customized image.

The [websphere-liberty image documentation](https://hub.docker.com/_/websphere-liberty/){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon") describes the following simple 3-line Dockerfile required to create a custom image:

```docker
FROM websphere-liberty:kernel
COPY server.xml /config/
RUN installUtility install --acceptLicense defaultServer
```
{: codeblock}

The `installUtility` tool looks for any features you require in your `server.xml` file that aren't already available in your Liberty image. It will then download those features and install them into your Docker image. This approach will create a minimal image, but the implicit feature list in `server.xml` does not work well with cached Docker layers. Any change to `server.xml` will invalidate the layer with the installed features, requiring the missing features to be installed again the next time the image is built.

A better way to create custom images is to invoke `installUtility` with an explicit list of features. This will still create a minimal image, but will behave much better at build time, as the layer of the image can be cached and reused regardless of server configuration changes. For example, the following will create a custom image using subset of features to support for an application that uses JAX-RS, JNDI and WebSockets:

```docker
FROM websphere-liberty:kernel
RUN /opt/ibm/wlp/bin/installUtility install --acceptLicense \
  cdi-1.2 \
  concurrent-1.0 \
  jaxrs-2.0 \
  jndi-1.0 \
  ssl-1.0 \
  websocket-1.1
COPY server.xml /config/
```
{: codeblock}

How you structure your Docker image depends on a few factors:

1. How reusable or customized do you want your base image to be?
    These steps produce the smallest possible image, potentially at the expense of reuse if each application's features are different. However, having very small custom images means that the application can be rapidly deployed across Docker hosts. If you're not sure, start with one of the pre-generated images from Docker Hub for greatest re-use.
2. How often do you update this image?
    If the application is updated very frequently, for example in a CI/CD pipeline, it is beneficial to structure the image so that the most frequently changed layer (often your application classes or app binary) is at the top. This reduces the number of layers that need to be re-built when the application changes and the image is rebuilt, speeding up build and deployment times.

When in doubt, follow the [Open Liberty Docker guide](https://openliberty.io/guides/docker.html){: new_window} ![External link icon](../icons/launch-glyph.svg "External link icon") to get started.
