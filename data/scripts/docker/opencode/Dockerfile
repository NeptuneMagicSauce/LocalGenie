FROM debian:bookworm-slim

ARG USER_ID=1000
ARG GROUP_ID=1000
RUN groupadd -g ${GROUP_ID} user && \
    useradd -l -u ${USER_ID} -g user -m user

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
    vim \
    ca-certificates

RUN rm -rf /var/lib/apt/lists/*

ENV PATH="/home/user/.opencode/bin:${PATH}"

ARG EDITOR=vim
ENV EDITOR=${EDITOR}
ENV COLORTERM=truecolor

COPY emacs.tgz /
RUN tar zxf /emacs.tgz -C /usr

USER user

WORKDIR /workspace

ENTRYPOINT ["opencode"]

ARG CACHE_BUST=1

# install OpenCode
RUN curl -fsSL https://opencode.ai/v2/install | bash
