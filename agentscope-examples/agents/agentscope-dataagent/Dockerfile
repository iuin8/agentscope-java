# syntax=docker/dockerfile:1
#
# Multi-stage build for agentscope-dataagent.
#
# IMPORTANT: the build context MUST be the repository root, because this module
# depends on sibling reactor modules (agentscope-core / agentscope-harness /
# agentscope-extensions / boms) that are not published to a remote repository.
# docker-compose.yml already sets `context: ../../..` for this reason; to
# build by hand run from the repo root (agentscope-java):
#
#   docker build -f agentscope-examples/agents/agentscope-dataagent/Dockerfile \
#     -t agentscope/dataagent:latest .
#
# ---------------------------------------------------------------------------
# Stage 1 — build the Spring Boot fat JAR (backend + React SPA bundled in)
# ---------------------------------------------------------------------------
FROM maven:3.9-eclipse-temurin-17 AS build

WORKDIR /build
COPY . .

# The frontend-maven-plugin downloads its own Node/npm during generate-resources
# and emits the SPA into src/main/resources/static, so no host Node is needed.
# A BuildKit cache mount keeps the Maven local repo warm across rebuilds.
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B -ntp -pl agentscope-examples/agents/agentscope-dataagent -am \
        package -DskipTests

# ---------------------------------------------------------------------------
# Stage 2 — slim runtime (glibc-based JRE; required by the bundled sqlite-jdbc
# native libs, which do not ship a musl build)
# ---------------------------------------------------------------------------
FROM eclipse-temurin:17-jre

# curl is only needed for the container HEALTHCHECK.
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Docker CLI (client only — no daemon). DataAgent runs every per-user workspace
# in a Docker sandbox; the app shells out to `docker` to manage those containers
# on the host daemon exposed via the bind-mounted /var/run/docker.sock
# (see docker-compose.yml). The CLI is a static binary copied from the official
# multi-arch `docker` image — no daemon/engine is installed.
COPY --from=docker:27-cli /usr/local/bin/docker /usr/local/bin/docker

# Run as an unprivileged user; /data holds all mutable state (workspace,
# auto-generated agentscope.json, embedded H2 DB) and is exposed as a volume.
RUN useradd --system --uid 10001 --create-home --home-dir /home/dataagent dataagent \
    && mkdir -p /app /data \
    && chown -R 10001:10001 /app /data

COPY --from=build --chown=10001:10001 \
    /build/agentscope-examples/agents/agentscope-dataagent/target/*-exec.jar \
    /app/app.jar

USER 10001
WORKDIR /app

# user.home=/data pins every default state path under the mounted volume:
#   workspace   -> /data/.agentscope/dataagent/workspace
#   config      -> /data/.agentscope/dataagent/agentscope.json
#   embedded H2 -> /data/.agentscope-dataagent/db.*
ENV HOME=/data \
    JAVA_OPTS=""

VOLUME ["/data"]
EXPOSE 8080

HEALTHCHECK --interval=15s --timeout=5s --retries=10 --start-period=90s \
    CMD curl -fsS http://localhost:8080/actuator/health || exit 1

# exec form via sh so $JAVA_OPTS expands while java still receives signals as PID 1.
ENTRYPOINT ["/bin/sh", "-c", "exec java -Duser.home=/data ${JAVA_OPTS} -jar /app/app.jar"]
