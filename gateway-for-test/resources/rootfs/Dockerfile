FROM maven:3.9.3-sapmachine-17 AS builder

COPY gateway-for-test /gateway-for-test

WORKDIR /gateway-for-test

RUN mkdir out && \
    /gateway-for-test/resources/scripts/build-app.sh

FROM openjdk:17-ea-slim

RUN mkdir /gateway-for-test && \
    sed -i 's|deb.debian.org|archive.debian.org|g' /etc/apt/sources.list && \
    sed -i 's|security.debian.org|archive.debian.org|g' /etc/apt/sources.list && \
    echo 'Acquire::Check-Valid-Until "false";' > /etc/apt/apt.conf.d/99no-check-valid && \
    apt-get update --allow-releaseinfo-change && \
    apt-get update && apt-get install -y wget && \
    cd /opt && \
    wget https://download.ej-technologies.com/jprofiler/jprofiler_agent_linux-arm_13_0.tar.gz -O /opt/jprofiler_agent.tar.gz && \
    tar -zvxf jprofiler_agent.tar.gz

COPY --from=builder /gateway-for-test/target/gateway-for-test-0.0.1.jar /gateway-for-test
COPY --from=builder /gateway-for-test/resources/scripts/bootstrap.sh  /

WORKDIR /gateway-for-test

EXPOSE 8071

ENTRYPOINT [ "/bootstrap.sh" ]