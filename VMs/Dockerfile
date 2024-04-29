# This dockerfile builds a container that pulls down and runs the latest version of BenchmarkJava
FROM ubuntu:latest
MAINTAINER "Dave Wichers dave.wichers@owasp.org"

RUN apt-get update
RUN DEBIAN_FRONTEND="noninteractive" apt-get -y install tzdata
RUN apt-get install -q -y \
     openjdk-11-jre-headless \
     openjdk-11-jdk \
     git \
     maven \
     wget \
     iputils-ping \
     unzip \
     && apt-get clean

# Install CodeQL CLI
ARG CODEQL_VERSION=v2.17.1
ARG CODEQL_HOME=/tools/code-ql-home
RUN mkdir /tools
RUN mkdir /opt/codeql

RUN wget -q https://github.com/github/codeql-cli-binaries/releases/download/${CODEQL_VERSION}/codeql-linux64.zip -O /tmp/codeql_linux.zip
RUN unzip /tmp/codeql_linux.zip -d ${CODEQL_HOME}
RUN rm /tmp/codeql_linux.zip

ENV PATH="${CODEQL_HOME}/codeql:${PATH}" 

RUN codeql pack download codeql/java-queries
RUN codeql pack download codeql/javascript-queries

RUN mkdir /owasp
WORKDIR /owasp

# Download, build, install Benchmark Utilities required by crawler and scorecard generation
RUN git clone https://github.com/OWASP-Benchmark/BenchmarkUtils.git
WORKDIR /owasp/BenchmarkUtils
RUN mvn install

# Download, build BenchmarkJava
WORKDIR /owasp
RUN git clone https://github.com/OWASP-Benchmark/BenchmarkJava.git

# Workaround for security fix for CVE-2022-24765
RUN git config --global --add safe.directory /owasp/BenchmarkJava

WORKDIR /owasp/BenchmarkJava
RUN mvn clean package cargo:install
RUN codeql database create owasp-benchmark --db-cluster --no-run-unnecessary-builds --language=java,javascript

RUN useradd -d /home/bench -m -s /bin/bash bench 
RUN echo bench:bench | chpasswd

RUN chown -R bench /owasp/
RUN chown -R bench ${CODEQL_HOME}
ENV PATH /owasp/BenchmarkJava:$PATH

# start up Benchmark once, for 60 seconds, then kill it, so the additional dependencies required to run it are downloaded/cached in the image as well.
# exit 0 is required to return a 'success' code, otherwise the timeout returns a failure code, causing the Docker build to fail.
WORKDIR /owasp/BenchmarkJava
RUN codeql database analyze owasp-benchmark/javascript codeql/javascript-queries --format=sarifv2.1.0 --threads=0 --no-rerun --output=results/Benchmark-codeql_javascript_queries.sarif --sarif-category=javascript
RUN codeql database analyze owasp-benchmark/java codeql/java-queries --format=sarifv2.1.0 --threads=0 --no-rerun --output=results/Benchmark-codeql_java_queries.sarif --sarif-category=java
RUN timeout 60 ./runBenchmark.sh; exit 0

