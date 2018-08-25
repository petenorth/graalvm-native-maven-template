# graalvm-native-maven-template

This template will setup a Jenkins pipeline running in Openshift that builds a statically linked native image from a maven style project using the GraalVM technology. Note that the maven style project must be configured to build an uber jar i.e. something like Spring Boot. The `native-image` GraalVM executable does take a classpath option (just like javac etc.) so the template could be enhanced if this is needed.

The pipeline is based on https://zeit.co/zeit/java-spark-graal/dpfwezzbxs/source . Openshift does not support multi stage docker builds, so I followed https://blog.openshift.com/chaining-builds/ to work around this.

The netty,vertx and spring-boot repos used in the examples below are generally forks of pre-existing work!

## Prerequisites

An Openshift cluster, I use minishift.

To use this maven template checkout this project and

    oc process -f graalvm-native-maven-template.yaml \
      -p GIT_REPO=<GIT REPO URL> \
      -p GIT_BRANCH=<GIT_BRANCH> \
      -p APP_NAME=<APP_NAME> | oc create -f -

Or the template can be referenced via a URL. 

Start a pipeline build with 
  
    oc start-build test-pipeline

this will fail but will result in a Jenkins instance being started.

The Jenkins needs to be reconfigured so that the maven slave points to 

    docker.io/petenorth/graalvm-jenkins-slave:latest 

## Working Netty

    oc process -f graalvm-native-maven-template.yaml \
      -p GIT_REPO=https://github.com/petenorth/netty-native-demo.git \
      -p GIT_BRANCH=master \
      -p APP_NAME=netty \
      -p REFLECTION_CONFIGURATION_RESOURCES=netty_reflection_config.json | oc create -f -

    oc new-app netty-scratch
    oc expose dc/netty-scratch --port=8080
    oc expose svc/netty-scratch
    
## Working Vert.x

    oc process -f https://raw.githubusercontent.com/petenorth/graalvm-native-maven-template/master/graalvm-native-maven-template.yaml   \ 
      -p GIT_REPO=https://github.com/petenorth/vertx-graalvm.git \
      -p GIT_BRANCH=master \
      -p APP_NAME=vertx \
      -p REFLECTION_CONFIGURATION_RESOURCES=netty_reflection_config.json | oc create -f -
      
    oc new-app vertx-scratch
    oc expose dc/vertx-scratch --port=8080
    oc expose svc/vertx-scratch

## Spring Boot (not working)

    oc process -f graalvm-native-maven-template.yaml \
      -p GIT_REPO=https://github.com/petenorth/graalvm-spring-boot-native-pipeline.git \
      -p GIT_BRANCH=master \
      -p APP_NAME=test | oc create -f -

To create an application from the resulting image stream

    oc new-app <APP_NAME>-scratch

i.e.

    oc new-app test-scratch

Currently the Spring Boot application will fail to start with this is the log

    Exception in thread "main" com.oracle.svm.core.jdk.UnsupportedFeatureError: Unsupported constructor java.security.ProtectionDomain.<init>(CodeSource, PermissionCollection) is reachable: The declaring class of this element has been substituted, but this element is not present in the substitution class
	at java.lang.Throwable.<init>(Throwable.java:265)
	at java.lang.Error.<init>(Error.java:70)
	at com.oracle.svm.core.jdk.UnsupportedFeatureError.<init>(UnsupportedFeatureError.java:31)
	at com.oracle.svm.core.jdk.Target_com_oracle_svm_core_util_VMError.unsupportedFeature(VMErrorSubstitutions.java:109)
	at java.security.ProtectionDomain.<init>(ProtectionDomain.java:137)
	at com.oracle.svm.core.hub.DynamicHub.getProtectionDomain(DynamicHub.java:969)
	at org.springframework.boot.loader.Launcher.createArchive(Launcher.java:117)
	at org.springframework.boot.loader.ExecutableArchiveLauncher.<init>(ExecutableArchiveLauncher.java:38)
	at org.springframework.boot.loader.JarLauncher.<init>(JarLauncher.java:35)
	at org.springframework.boot.loader.JarLauncher.main(JarLauncher.java:51)
	at com.oracle.svm.core.JavaMainWrapper.run(JavaMainWrapper.java:177)



