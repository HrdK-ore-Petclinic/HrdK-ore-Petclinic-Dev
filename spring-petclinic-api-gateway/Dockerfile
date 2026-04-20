# syntax=docker/dockerfile:1.7

FROM eclipse-temurin:17-jdk AS builder

ARG SERVICE=spring-petclinic-api-gateway
WORKDIR /build

COPY mvnw ./
COPY .mvn .mvn
COPY pom.xml ./

COPY spring-petclinic-api-gateway/pom.xml       spring-petclinic-admin-server/
COPY spring-petclinic-api-gateway/pom.xml        spring-petclinic-api-gateway/
COPY spring-petclinic-config-server/pom.xml      spring-petclinic-config-server/
COPY spring-petclinic-customers-service/pom.xml  spring-petclinic-customers-service/
COPY spring-petclinic-discovery-server/pom.xml   spring-petclinic-discovery-server/
COPY spring-petclinic-genai-service/pom.xml      spring-petclinic-genai-service/
COPY spring-petclinic-vets-service/pom.xml       spring-petclinic-vets-service/
COPY spring-petclinic-visits-service/pom.xml     spring-petclinic-visits-service/

RUN chmod +x mvnw

RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -B -pl ${SERVICE} -am dependency:go-offline -DskipTests

COPY ${SERVICE}/src ${SERVICE}/src

RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -B -pl ${SERVICE} -am package -DskipTests \
 && cp ${SERVICE}/target/*.jar /build/app.jar

FROM eclipse-temurin:17-jre AS extractor
WORKDIR /extract
COPY --from=builder /build/app.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:17-jre
WORKDIR /application

RUN groupadd --system spring && useradd --system --gid spring --uid 1001 spring
USER 1001:1001

ENV SPRING_PROFILES_ACTIVE=docker \
    JAVA_OPTS=""

COPY --from=extractor --chown=1001:1001 /extract/dependencies/          ./
COPY --from=extractor --chown=1001:1001 /extract/spring-boot-loader/    ./
COPY --from=extractor --chown=1001:1001 /extract/snapshot-dependencies/ ./
COPY --from=extractor --chown=1001:1001 /extract/application/           ./

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]