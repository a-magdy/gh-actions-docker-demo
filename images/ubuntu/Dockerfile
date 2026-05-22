FROM ubuntu:24.04

RUN apt-get update && apt-get install -y --no-install-recommends \
    bash curl ca-certificates \
  && rm -rf /var/lib/apt/lists/*

CMD ["echo", "Hello from ubuntu image"]
