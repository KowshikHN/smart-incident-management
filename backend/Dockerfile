# syntax=docker/dockerfile:1

# ---------- Stage 1: build the jar ----------
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /build

COPY pom.xml .
COPY src ./src

# BuildKit cache keeps ~/.m2 between builds so dependencies are not re-downloaded.
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -DskipTests package

# ---------- Stage 2: small runtime image ----------
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

RUN addgroup -S app && adduser -S app -G app

# The app stores attachments in the relative path "uploads/issues" (and saves that
# relative path in the database), so WORKDIR must stay /app and /app/uploads must
# be a persistent volume (see docker-compose.yml).
RUN mkdir -p /app/uploads/issues && chown -R app:app /app

COPY --from=build /build/target/*.jar /app/app.jar

USER app
EXPOSE 8088

ENV JAVA_OPTS="-XX:MaxRAMPercentage=70"
ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]
