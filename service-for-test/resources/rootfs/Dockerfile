FROM maven:3.9.3-sapmachine-17 as builder

COPY service-for-test /service-for-test

WORKDIR /service-for-test

RUN mkdir out && \
    /service-for-test/resources/scripts/build-app.sh

FROM openjdk:17-ea-slim

RUN mkdir /service-for-test

COPY --from=builder /service-for-test/target/service-for-test-0.0.2.jar /service-for-test
COPY --from=builder /service-for-test/resources/scripts/bootstrap.sh  /

WORKDIR /service-for-test

EXPOSE 8072

ENTRYPOINT [ "/bootstrap.sh" ]