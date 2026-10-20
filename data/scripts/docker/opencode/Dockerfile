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
    zile \
    ca-certificates

RUN rm -rf /var/lib/apt/lists/*

RUN update-alternatives --install /usr/bin/emacs emacs /usr/bin/zile 100

USER user

ENV PATH="/home/user/.opencode/bin:${PATH}"

ENV EDITOR=emacs

ENV COLORTERM=truecolor

WORKDIR /workspace

ENTRYPOINT ["opencode"]

ARG CACHE_BUST=1

# install OpenCode
RUN curl -fsSL https://opencode.ai/v2/install | bash
