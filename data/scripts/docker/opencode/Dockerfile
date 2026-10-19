FROM debian:bookworm-slim

# prevent interactive prompts for apt
ENV DEBIAN_FRONTEND=noninteractive

# install dependencies
RUN apt update && apt install -y --no-install-recommends \
    curl \
    git \
    grep \
    ripgrep \
    jq \
    git \
    ca-certificates

RUN rm -rf /var/lib/apt/lists/*

ENV PATH="/root/.opencode/bin:${PATH}"

WORKDIR /workspace

CMD ["opencode"]

ARG CACHE_BUST=1

# install OpenCode
RUN curl -fsSL https://opencode.ai/v2/install | bash