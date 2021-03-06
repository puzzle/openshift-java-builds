---
title: Java Builds on OpenShift
---

<!-- .slide: class="master01" -->

# Java Builds<br/>on<br/>OpenShift

---

<!-- .slide: class="master02" -->

# Overview

---

<!-- .slide: class="text-left" -->
## Docker Images

* Adhere to [OpenShift Image Creation Guidelines](https://docs.openshift.com/container-platform/3.9/creating_images/guidelines.html) / [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
* Use base images from trusted sources
* Use supported and up to date images

---

<!-- .slide: class="text-left" -->
## Docker Images

* Use versioned images/tags
* Rebuild your images if base image changes
* Monitor for outdated images

---

<!-- .slide: class="text-left" -->
## Java Base Images

* Use TCK/JCK certified JDK builds
* OpenJDK 8:
```sh
registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift
fabric8/s2i-java:latest  # Same image based on CentOS
azul/zulu-openjdk-alpine:8
```
<!-- .element style="margin-top: 1rem; margin-bottom: 1rem; line-height: 1.6em;" --->
* OpenJDK 11:
```sh
registry.access.redhat.com/openjdk/openjdk-11-rhel7
fabric8/s2i-java:latest-java11  # Same image based on CentOS
azul/zulu-openjdk-alpine:11
```
<!-- .element style="margin-top: 1rem; margin-bottom: 1rem; line-height: 1.6em;" --->

---

<!-- .slide: class="text-left" -->
## Java Base Images

* S2I images can be used for S2I and Docker builds
* S2I images contain Jolokia and Prometheus JMX exporter agents
* S2I images contain `run-java.sh`, tuning JVM memory and cpu parameters
* S2I images support Spring Boot automatic restarts
* Not all AdoptOpenJDK images are TCK certified yet

---

<!-- .slide: class="text-left" -->
## Build Stages

1. Build artifacts
2. Run tests
3. Build Docker images

Each stage can run on OpenShift or not.

---

<!-- .slide: class="text-left" -->
## Some Options for Java Builds

* Source-To-Image (S2I)
* Binary Source-To-Image (S2I)
* Docker Build
* Fabric8 Maven Plugin
* Jib Maven/Gradle Plugin

---

<!-- .slide: class="master02" -->

# Source-To-Image (S2I)

---

<!-- .slide: class="text-left" -->
## Source-To-Image (S2I)

```txt
oc new-app --name=spring-boot-s2i \
  registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift~\
  https://github.com/puzzle/spring-boot-app-monitoring#techkafi

oc create route edge spring-boot-s2i \
  --service=spring-boot-s2i --insecure-policy=Redirect
```

Creates an ImageStream for the base image if it's a Docker image,
giving you control over its import policy.
<!-- .element style="font-size: 80%" -->

---

<!-- .slide: class="text-left" -->
## S2I Incremental Build

Java S2I images can cache Maven artifacts:
<!-- .element style="margin-bottom: -1.5rem;" --->
```sh
oc patch bc spring-boot-s2i \
  -p '{"spec":{"strategy":{"sourceStrategy":{"incremental": true}}}}'

oc start-build spring-boot-s2i -F 
```

Artifacts are fetched from preceding build image.
<!-- .element style="font-size: 80%" -->

---

<!-- .slide: class="text-left" -->
## S2I Binary Build

Java S2I images can use prebuilt artifacts:
<!-- .element style="margin-bottom: -1.5rem;" --->
```txt
git clone -b techkafi https://github.com/puzzle/spring-boot-app-monitoring
cd spring-boot-app-monitoring

mvn clean package
oc start-build bc/spring-boot-s2i -F --from-file target/*.jar
```

Incremental builds slow down binary builds and should be disabled!
<!-- .element style="font-size: 80%" -->

---

<!-- .slide: class="text-left" -->
## Pipeline with S2I Binary Build

```txt
oc create svc externalname jenkins --external-name=jenkins-test.puzzle.ch

oc new-build --name=spring-boot-s2i-pipeline --context-dir=puzzle/s2i \
  https://github.com/puzzle/spring-boot-app-monitoring#techkafi
```

Creates a pipeline job in Jenkins.
<!-- .element style="font-size: 80%" -->

The `jenkins` service prevents auto-provisioning of a Jenkins instance in the current namespace.

<!-- .element style="font-size: 80%; margin-top: -1rem" -->

---

<!-- .slide: class="text-left" -->
puzzle/s2i/Jenkinsfile ...
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```groovy
pipeline {
  agent { label 'buildnode' }
  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    timeout(time: 10, unit: 'MINUTES')
  }
  environment {
    M2_HOME = tool('maven35')
    OC_HOME = tool('oc')
    PATH = "${M2_HOME}/bin:${OC_HOME}/bin:$PATH"
  }
  stages {
    stage('Build App') {
      steps {
        withMaven {
          sh "mvn clean package"
        }
      }
    }
```

---

<!-- .slide: class="text-left" -->
... puzzle/s2i/Jenkinsfile
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```groovy
    stage('Build Image') {
      steps {
        script {
          openshift.withCluster('OpenShiftPuzzleProduction','dtschan-jenkins'){
            openshift.withProject('dtschan') {
              def bc = openshift.selector("bc/spring-boot-s2i")
              bc.startBuild("--from-file=target/app.jar")
              bc.logs("-f")
            }
          }
        }
      }
    }
  }
}
```

---

<!-- .slide: class="master02" -->

# Docker Builds

---

<!-- .slide: class="text-left" -->
# Docker Builds

```txt
oc new-build --name=spring-boot-docker --strategy=docker \
  https://github.com/puzzle/spring-boot-app-monitoring#techkafi 
```

Docker Builds run as root!
<!-- .element style="font-size: 80%" -->

Give all the flexibility and responsibility of Dockerfiles.
<!-- .element style="font-size: 80%; margin-top: -1rem" -->

Are usually the slowest option on OpenShift.

<!-- .element style="font-size: 80%; margin-top: -1rem" -->
---

<!-- .slide: class="text-left" -->
Dockerfile
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```Dockerfile
FROM registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift

SHELL ["/usr/bin/scl", "enable", "rh-maven35"]

WORKDIR /tmp/src

COPY . /tmp/src

USER 0

RUN if [ ! -s /tmp/src/app.jar ]; then \
      mvn package; \
      cp target/app.jar /home/jboss; \
    else \
      cp /tmp/src/app.jar /home/jboss; \
    fi; \
    rm -rf /tmp/src

USER 1001

EXPOSE 8080

CMD java -jar /home/jboss/app.jar
```

---

<!-- .slide: class="text-left" -->
## Pipeline with Docker Build

```txt
oc create svc externalname jenkins --external-name=jenkins-test.puzzle.ch

oc new-build --name=spring-boot-docker-pipeline --context-dir=puzzle/docker \
  https://github.com/puzzle/spring-boot-app-monitoring#techkafi 
```

Creates a pipeline job in Jenkins.
<!-- .element style="font-size: 80%" -->

The `jenkins` service prevents auto-provisioning of a Jenkins instance in the current namespace.

<!-- .element style="font-size: 80%; margin-top: -1rem" -->

---

<!-- .slide: class="text-left" -->

puzzle/docker/Jenkinsfile ...
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```groovy
pipeline {
  agent { label 'buildnode' }
  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    timeout(time: 10, unit: 'MINUTES')
  }
  environment {
    M2_HOME = tool('maven35')
    OC_HOME = tool('oc')
    PATH = "${M2_HOME}/bin:${OC_HOME}/bin:$PATH"
  }
  stages {
    stage('Build App') {
      steps {
        withMaven {
          sh "mvn clean package"
        }
      }
    }
```

---

<!-- .slide: class="text-left" -->
... puzzle/docker/Jenkinsfile
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```
    stage('Build Image') {
      steps {
        script {
          openshift.withCluster('OpenShiftPuzzleProduction','dtschan-jenkins'){
            openshift.withProject('dtschan') {
              sh "mkdir -p input && cp Dockerfile target/*.jar input"
              def bc = openshift.selector("bc/spring-boot-docker")
              bc.startBuild("--from-dir=input")
              bc.logs("-f")
            }
          }
        }
      }
    }
  }
}
```

---

<!-- .slide: class="master02" -->

# Fabric8 Maven Plugin (FMP)

---

<!-- .slide: class="text-left" -->
## Fabric8 Maven Plugin (FMP)

* Red Hat Open Source Project
* Zero-Config setup with defaults based on POM
* Customization through YAML fragments
* Can deploy to OpenShift or Kubernetes

---

<!-- .slide: class="text-left" -->
## Fabric8 Maven Plugin (FMP)

* Can build on OpenShift and Docker
* Supports S2I and Docker build strategies
* Can read Docker Compose files
* Maven independent parts are being moved to [fabric8-kit](https://github.com/fabric8io/fabric8-kit), e.g. for Gradle plugin

---

<!-- .slide: class="text-left" -->
## Add FMP 

```xml
<plugin>
  <groupId>io.fabric8</groupId>
  <artifactId>fabric8-maven-plugin</artifactId>
  <version>4.0.0-M2</version>    
</plugin>
```

For versions see [OpenShift and Kubernetes compatibility](https://github.com/fabric8io/fabric8-maven-plugin#openshift-and-kubernetes-compatibility)

<!-- .element style="font-size: 80%;" -->

---

<!-- .slide: class="text-left" -->
## Build Image with FMP

```sh
mvn package fabric8:build
```
Builds in current OpenShift namespace, falls back to Docker

<!-- .element style="font-size: 80%;" -->

---

<!-- .slide: class="text-left" -->
## Deploy Application with FMP

```sh
mvn fabric8:resource fabric8:apply
```

`fabric8:resource` builds OpenShift and Kubernetes resources to `target/classes/META-INF/fabric8`.

<!-- .element style="font-size: 80%;" -->

`fabric8:apply` applies resources against current cluster and namespace.

<!-- .element style="font-size: 80%;" -->

---

<!-- .slide: class="text-left" -->
## K8s Resources Fragments 

src/main/fabric8/deployment.yml
<!-- .element style="margin-bottom: -1.5rem;" --->
```yaml
apiVersion: v1
kind: Deployment
metadata:
  name: ${project.artifactId}
spec:
  template:
    spec:
      containers:
      - env:
        - name: TZ
          value: Europe/Zurich
```

Maven profiles and properties can be used to support different settings in different stages.
<!-- .element style="font-size: 80%;" -->

---

<!-- .slide: class="text-left" -->
## Pipeline with Fabric8 build

```txt
oc create svc externalname jenkins --external-name=jenkins-test.puzzle.ch

oc new-build --name=spring-boot-fabric8-pipeline --context-dir=puzzle/fabric8 \
  https://github.com/puzzle/spring-boot-app-monitoring#techkafi
```

Creates a pipeline job in Jenkins.
<!-- .element style="font-size: 80%" -->

The `jenkins` service prevents auto-provisioning of a Jenkins instance in the current namespace.

<!-- .element style="font-size: 80%; margin-top: -1rem" -->

---

<!-- .slide: class="text-left" -->
puzzle/fabic8/Jenkinsfile ...
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```groovy
pipeline {
  agent { label 'buildnode' }
  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    timeout(time: 10, unit: 'MINUTES')
  }
  environment {
    M2_HOME = tool('maven35')
    OC_HOME = tool('oc')
    PATH = "${M2_HOME}/bin:${OC_HOME}/bin:$PATH"
    KUBECONFIG = "${WORKSPACE}/.kube/config"
    OPENSHIFT_LOGIN = credentials('dtschan-openshift')
  }
```

---

<!-- .slide: class="text-left" -->
... puzzle/fabic8/Jenkinsfile
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```groovy
  stages {
    stage('Build App') {
      steps {
        withMaven {
          sh "mvn clean package"
        }
      }
    }
    stage('Build Image') {
      steps {
        sh 'oc login ose3-master.puzzle.ch:8443 ' +
           '--namespace=dtschan --token=${OPENSHIFT_LOGIN_PSW}'
        sh 'mvn fabric8:build'
      }
    }
  }
}
```

---

<!-- .slide: class="master02" -->

# Jib Maven/Gradle Plugin 

---

<!-- .slide: class="text-left" -->
## Jib Maven/Gradle Plugin

* Google Open Source Project
* Optimized for speed
* Reproducible image builds
* Requires no Docker, builds directly to registry
* Maven and Gradle Plugin

---

<!-- .slide: class="text-left" -->
## Add Jib Maven Plugin 

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>1.0.2</version>
  <configuration>
    <from>
      <image>registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift</image>
    </from>
    <to>
      <image>${env.TO_IMAGE}</image>
    </to>
    <container>
      <ports>
        <port>8080</port>
      </ports>
    </container>
  </configuration>
</plugin>
```
<!-- .element style="font-size: 1.18rem" -->

Registry credentials are configured in `settings.xml`.

<!-- .element style="font-size: 80%;" -->

---

<!-- .slide: class="text-left" -->
## Build Image with Jib

Build directly to image registry:
<!-- .element style="font-size: 80%; margin-bottom: -1.5rem" -->
```sh
mvn compile jib:build
```

Build to Docker daemon:
<!-- .element style="font-size: 80%; margin-bottom: -1.5rem;" -->
```sh
mvn compile jib:dockerBuild
```

---

<!-- .slide: class="text-left" -->
## Pipeline with Jib build

```txt
oc create svc externalname jenkins --external-name=jenkins-test.puzzle.ch

oc new-build --name=spring-boot-jib-pipeline --context-dir=puzzle/jib \
  https://github.com/puzzle/spring-boot-app-monitoring#techkafi
```

Creates a pipeline job in Jenkins.
<!-- .element style="font-size: 80%" -->

The `jenkins` service prevents auto-provisioning of a Jenkins instance in the current namespace.

<!-- .element style="font-size: 80%; margin-top: -1rem" -->

---

<!-- .slide: class="text-left" -->

puzzle/jib/Jenkinsfile ...
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```groovy
pipeline {
  agent { label 'buildnode' }
  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    timeout(time: 10, unit: 'MINUTES')
  }
  environment {
    M2_HOME = tool('maven35')
    PATH = "${M2_HOME}/bin:$PATH"
    TO_IMAGE = "registry.ose3.puzzle.ch/dtschan/spring-boot-jib"
  }
  stages {
    stage('Build App') {
      steps {
        withMaven {
          sh "mvn clean test"
        }
      }
    }
```

---

<!-- .slide: class="text-left" -->

... puzzle/jib/Jenkinsfile
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```groovy
    stage('Build Image') {
      steps {
        withMaven(mavenSettingsConfig: 'openshift-registry') {
          sh "mvn jib:build"
        }
      }
    }
  }
}
```

`openshift-registry` refers to a `settings.xml` with registry credentials provided by Jenkins config file management.

<!-- .element style="font-size: 80%" -->

---

<!-- .slide: class="master02" -->

# Triggering and Chaining Builds

---

<!-- .slide: class="text-left" -->
##  Poll Remote Images

```sh
oc patch is openjdk18-openshift \
  -p '{"spec":{"tags":[{"name":"latest","importPolicy":{"scheduled":true}}]}}'
 ```

or use `oc tag --scheduled=true` when creating new tags.

<!-- .element style="font-size: 80%; margin-top: -1rem; margin-bottom: -1.5rem" -->

Will update image stream if remote image changes.<br>
Only enable on tags as needed as this creates load on the platform.
<!-- .element style="font-size: 80%" -->

---

<!-- .slide: class="text-left" -->
## Triggering Builds
Trigger build when image stream changes:
<!-- .element style="margin-bottom: -1.5rem" --->
```txt
oc set triggers bc/spring-boot-jib-pipeline \
  --from-image=openjdk18-openshift:latest
```

* `--from-image` must specify an image stream tag
* Build configs created from an image have a trigger preconfigured
* Pipeline build configs trigger the corresponding Jenkins pipeline

<!-- .element style="font-size: 80%" -->

---

<!-- .slide: class="text-left" -->
## Build Chaining ...

* OpenShift's equivalent of Docker multi-stage builds
* Two or more builds are chained together via triggers
* Artifacts are exchanged via image sources

---

<!-- .slide: class="text-left" -->
## ... Build Chaining

* Allows to reduce image size and attack surface
* Can be used to install packages into S2I images
* Any combination of build strategies can be used (custom strategy in first build only)

---

<!-- .slide: class="text-left" -->
## Create Chained Build

```sh
oc new-build \
  --name=spring-boot-chained \
  --source-image=spring-boot-s2i:latest \
  --source-image-path=/deployments/app.jar:. \
  --dockerfile=- <<EOF
FROM azul/zulu-openjdk-alpine:8
COPY app.jar /deployments/
USER 1001
CMD java -jar /deployments/app.jar
EOF

oc new-app spring-boot-chained 
```

Creates Docker build with minimal base image which copies `app.jar` from previous S2I build.

<!-- .element style="font-size: 80%; margin-top: -1rem" -->

---

<!-- .slide: class="text-left" -->
## Custom Jenkins Slave

```sh
oc create -f https://raw.githubusercontent.com\
/puzzle/openshift-java-builds/master/openshift/maven-persistent-slave.yaml
```

Creates volume claim and config map with Jenkins pod template.
<!-- .element style="font-size: 80%; margin-top: -1rem" -->

Kubernetes cloud name in Jenkins must be `openshift`.

<!-- .element style="font-size: 80%; margin-top: -1rem" -->

---

<!-- .slide: class="text-left" -->
## Pipeline with Custom Slave

```txt
oc create svc externalname jenkins --external-name=jenkins-test.puzzle.ch

oc new-build --name=spring-boot-slave-pipeline --context-dir=puzzle/slave \
  https://github.com/puzzle/spring-boot-app-monitoring#techkafi 
```

Creates a pipeline job in Jenkins.
<!-- .element style="font-size: 80%" -->

The `jenkins` service prevents auto-provisioning of a Jenkins instance in the current namespace.

<!-- .element style="font-size: 80%; margin-top: -1rem" -->

---

<!-- .slide: class="text-left" -->

puzzle/slave/Jenkinsfile ...
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```groovy
pipeline {
  agent { label 'maven-persistent' }
  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    timeout(time: 10, unit: 'MINUTES')
  }
  environment {
    TO_IMAGE = "registry.ose3.puzzle.ch/dtschan/spring-boot-jib"
  }
  stages {
    stage('Build App') {
      steps {
        withMaven {
          sh "mvn clean test"
        }
      }
    }
```

---

<!-- .slide: class="text-left" -->

... puzzle/slave/Jenkinsfile
<!-- .element style="margin-bottom: -1.5rem; margin-top: -2rem; color: #3B7BBE" --->
```groovy
    stage('Build Image') {
      steps {
        withMaven(mavenSettingsConfig: 'openshift-registry') {
          sh "mvn jib:build"
        }
      }
    }
  }
}
```

---

<!-- .slide: class="text-left" -->
## Summary ...

* Pipeline deals with test reports, notifications, ...
* Binary builds can pass artifacts to OpenShift
* S2I supports source and binary builds
* Fabric8 has zero-config mode, deployment support

---

<!-- .slide: class="text-left" -->
## ... Summary

* Jib requires no Docker daemon, has Gradle Plugin
* OpenShift can detect remote images changes
* OpenShift can trigger Jenkins pipelines
* Build chaining can be used to minimize images

---

<!-- .slide: class="text-left" -->
## What's Next

* [OpenShift Advanced Build Operations](https://docs.openshift.com/container-platform/3.9/dev_guide/builds/advanced_build_operations.html)
* [OpenShift Jenkins Plugins](https://docs.openshift.com/container-platform/3.9/using_images/other_images/jenkins.html#plug-ins)
* [Java S2I Image Documentation](https://github.com/fabric8io-images/s2i/tree/master/java/images)
* [Customizing S2I Images](https://docs.openshift.com/container-platform/3.9/using_images/s2i_images/customizing_s2i_images.html)
* [Fabric8 Maven Plugin Documentation](https://maven.fabric8.io/)
* [Jib Documentation](https://github.com/GoogleContainerTools/jib)
