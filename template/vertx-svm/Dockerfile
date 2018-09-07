# Get a build environment
FROM ubuntu:16.04 as build-aot

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update && \
    apt-get -y install gcc build-essential zlib1g zlib1g-dev wget && \
    rm -rf /var/lib/apt/lists/*

ENV GRAALVM_VERSION=1.0.0-rc6

# Get GraalVM CE
RUN mkdir -p /opt/java
RUN wget -c https://github.com/oracle/graal/releases/download/vm-${GRAALVM_VERSION}/graalvm-ce-${GRAALVM_VERSION}-linux-amd64.tar.gz
RUN tar -zxf graalvm-ce-${GRAALVM_VERSION}-linux-amd64.tar.gz -C /opt/java

ENV GRAALVM_HOME=/opt/java/graalvm-ce-${GRAALVM_VERSION}
ENV JAVA_HOME=${GRAALVM_HOME}
ENV PATH=${PATH}:${JAVA_HOME}/bin

# Set working dir
RUN mkdir /app
WORKDIR /app

# Add maven wrapper
ADD mvnw /app/
ADD .mvn /app/.mvn/
# Add settings.xml to allow snapshots
ADD settings.xml /root/.m2/
ADD pom.xml /app/
# build all dependencies for offline use
RUN ./mvnw -Pnative-image dependency:go-offline -B
# Add the sources
ADD src /app/src/
# Build (java side)
RUN ./mvnw -Pnative-image

# At this point the native image is built

# Create new image from alpine
FROM frolvlad/alpine-glibc:alpine-3.8

RUN addgroup -S app && adduser -S -g app app

# Alternatively use ADD https:// (which will not be cached by Docker builder)
RUN apk --no-cache add ca-certificates curl \
    && echo "Pulling watchdog binary from Github." \
    && curl -sSLf https://github.com/openfaas-incubator/of-watchdog/releases/download/0.2.7/of-watchdog > /usr/bin/fwatchdog \
    && chmod +x /usr/bin/fwatchdog \
    && apk del curl --no-cache

WORKDIR /root/

RUN mkdir -p /home/app

# Wrapper/boot-strapper
WORKDIR /home/app
# Copy generated native executable from build-aot
COPY --from=build-aot /app/target/openfaas.vertx-0.0.1-SNAPSHOT-fat /usr/bin/vertx-fn

USER app

ENV cgi_headers="true"
ENV fprocess="vertx-fn"
ENV mode="http"
ENV upstream_url="http://127.0.0.1:8000"

ENV exec_timeout="20s"
ENV write_timeout="25s"
ENV read_timeout="25s"

HEALTHCHECK --interval=1s CMD [ -e /tmp/.lock ] || exit 1

CMD ["fwatchdog"]