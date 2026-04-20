# syntax=docker/dockerfile:1.7
#
# Build context MUST be the repository root (the parent of this directory),
# because this multi-module Maven build needs the root pom.xml and sibling
# module poms to resolve the reactor.
#
#   docker build -f spring-petclinic-admin-server/Dockerfile -t <tag> .

# -----------------------------------------------------------------------------
# Stage 1: build & package the service with the Maven wrapper
# -----------------------------------------------------------------------------
FROM eclipse-temurin:17-jdk AS builder

ARG SERVICE=spring-petclinic-customers-service
WORKDIR /build

# Copy wrapper + parent pom first so dependency resolution is cached
# independently of source changes.
COPY mvnw ./
COPY .mvn .mvn
COPY pom.xml ./

# Copy every module pom so Maven can resolve the reactor. Source is copied
# later so a code change does not invalidate the dependency cache.
COPY spring-petclinic-admin-server/pom.xml       spring-petclinic-admin-server/
COPY spring-petclinic-api-gateway/pom.xml        spring-petclinic-api-gateway/
COPY spring-petclinic-config-server/pom.xml      spring-petclinic-config-server/
COPY spring-petclinic-customers-service/pom.xml  spring-petclinic-customers-service/
COPY spring-petclinic-discovery-server/pom.xml   spring-petclinic-discovery-server/
COPY spring-petclinic-genai-service/pom.xml      spring-petclinic-genai-service/
COPY spring-petclinic-vets-service/pom.xml       spring-petclinic-vets-service/
COPY spring-petclinic-visits-service/pom.xml     spring-petclinic-visits-service/

RUN chmod +x mvnw

# Pre-fetch dependencies into a BuildKit cache mount
RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -B -pl ${SERVICE} -am dependency:go-offline -DskipTests

# Now copy the service source and build
COPY ${SERVICE}/src ${SERVICE}/src

RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -B -pl ${SERVICE} -am package -DskipTests \
 && cp ${SERVICE}/target/*.jar /build/app.jar

# -----------------------------------------------------------------------------
# Stage 2: explode the Spring Boot layered JAR for better runtime caching
# -----------------------------------------------------------------------------
FROM eclipse-temurin:17-jre AS extractor
WORKDIR /extract
COPY --from=builder /build/app.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# -----------------------------------------------------------------------------
# Stage 3: minimal runtime image
# -----------------------------------------------------------------------------
FROM eclipse-temurin:17-jre
WORKDIR /application

# Run as a non-root user
RUN groupadd --system spring && useradd --system --gid spring --uid 1001 spring
USER 1001:1001

ENV SPRING_PROFILES_ACTIVE=docker \
    JAVA_OPTS=""

# Copy layers cheap-to-expensive so frequent changes sit at the top
COPY --from=extractor --chown=1001:1001 /extract/dependencies/          ./
COPY --from=extractor --chown=1001:1001 /extract/spring-boot-loader/    ./
COPY --from=extractor --chown=1001:1001 /extract/snapshot-dependencies/ ./
COPY --from=extractor --chown=1001:1001 /extract/application/           ./

EXPOSE 8081

# Liveness/readiness are enforced by the Kubernetes probes in k8s/*.yaml,
# so no HEALTHCHECK is declared here.
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
