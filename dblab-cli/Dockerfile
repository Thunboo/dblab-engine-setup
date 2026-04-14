FROM ubuntu:22.04
LABEL org.opencontainers.image.title="dblab-cli" \
      org.opencontainers.image.description="Образ для использования dblab cli tool в GitLab CI pipelines" \
      org.opencontainers.image.authors="Gleb Broydo <gleb.broydo@x5.com>"

RUN apt-get update && \
    apt-get install -y \
        curl && \
    curl -sSL dblab.sh | bash && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

CMD ["sleep", "infinity"]