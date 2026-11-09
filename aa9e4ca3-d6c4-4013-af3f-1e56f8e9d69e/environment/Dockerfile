# This plugin (jenkins.version 2.7.4, parent POM 2.5) targets Java 8: it imports
# javax.xml.bind (removed in JDK 11) and its JenkinsRule tests boot a real
# Jenkins 2.7.4, which does not run on modern JDKs. Build on Temurin 8.
FROM maven:3.9-eclipse-temurin-8

# Harbor/Terminus runtime tools (tmux, asciinema) + git. Installed early so the
# layer is cached across rebuilds. Verify both are on PATH.
RUN apt-get update && apt-get install -y --no-install-recommends \
        tmux asciinema git \
    && rm -rf /var/lib/apt/lists/* \
    && command -v tmux && command -v asciinema && command -v git

WORKDIR /app
COPY repo/ .

# The pom declares repo.jenkins-ci.org over http://, which Maven 3.8+ refuses to
# fetch from (maven-default-http-blocker). Mirror that repo id to its https URL
# so the Jenkins parent POM and plugin artifacts resolve during the online warm.
RUN mkdir -p /root/.m2 && printf '%s' \
'<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">' \
'<mirrors><mirror>' \
'<id>jenkins-ci-https</id>' \
'<mirrorOf>repo.jenkins-ci.org</mirrorOf>' \
'<url>https://repo.jenkins-ci.org/public/</url>' \
'</mirror></mirrors>' \
'</settings>' > /root/.m2/settings.xml

# Warm the FULL Maven cache at build time (online).
#
# This is a Jenkins plugin: `dependency:go-offline` alone is NOT enough because
# the surefire provider, jenkins.war, detached plugins and the test-harness
# artifacts are only resolved during the actual `test` phase (JenkinsRule spins
# up a real Jenkins per test and pulls those on demand). So we must run the real
# test phase once, online, to populate ~/.m2 for the later offline run.
#
# IMPORTANT: the goal of this step is ONLY to populate the dependency cache, not
# to gate on test results. Individual JenkinsRule tests are heavy and can be
# flaky under a fresh/constrained cloud builder (e.g. Daytona) — a single failed
# fork must NOT abort the image build. We therefore run with
# `-Dmaven.test.failure.ignore=true` so surefire still executes every test
# (triggering every on-demand download) but returns success regardless of test
# outcome. Real build errors (compilation, dependency resolution) still fail the
# build. The authoritative pass/fail check is the later OFFLINE `mvn test`.
#
# `dependency:go-offline` runs first to pull the bulk of compile/test-scope
# artifacts; the retry loop only guards against transient network hiccups on the
# remote builder.
RUN n=0; until mvn -B -e dependency:go-offline; do \
        n=$((n+1)); \
        if [ $n -ge 3 ]; then echo "go-offline failed" >&2; exit 1; fi; \
        echo "go-offline failed (attempt $n/3), retrying…" >&2; \
        sleep $((n*15)); \
    done; \
    mvn -B test-compile; \
    mvn -B -Dmaven.test.failure.ignore=true -DfailIfNoTests=false test

# Force Maven offline for all runtime invocations so `mvn test` never reaches
# out to (unavailable) remote repositories under --network=none.
RUN mkdir -p /root/.m2 && printf '%s' \
'<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0">' \
'<offline>true</offline>' \
'<mirrors><mirror>' \
'<id>jenkins-ci-https</id>' \
'<mirrorOf>repo.jenkins-ci.org</mirrorOf>' \
'<url>https://repo.jenkins-ci.org/public/</url>' \
'</mirror></mirrors>' \
'</settings>' > /root/.m2/settings.xml

CMD ["sh", "-c", "mvn test"]
RUN mkdir -p /installed-agent
RUN command -v patch >/dev/null 2>&1 || ( (apt-get update && apt-get install -y --no-install-recommends patch && rm -rf /var/lib/apt/lists/*) 2>/dev/null || apk add --no-cache patch 2>/dev/null || yum install -y patch 2>/dev/null || true )
