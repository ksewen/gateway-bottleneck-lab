FROM maven:3.9.3-sapmachine-17 as builder

COPY auth-service-for-test /auth-service-for-test

WORKDIR /auth-service-for-test

RUN mkdir out && \
    /auth-service-for-test/resources/scripts/build-app.sh

FROM openjdk:17-ea-slim

RUN mkdir /auth-service-for-test

COPY --from=builder /auth-service-for-test/target/auth-service-for-test-0.0.2.jar /auth-service-for-test
COPY --from=builder /auth-service-for-test/resources/scripts/bootstrap.sh  /

WORKDIR /auth-service-for-test

EXPOSE 8073

ENTRYPOINT [ "/bootstrap.sh" ]