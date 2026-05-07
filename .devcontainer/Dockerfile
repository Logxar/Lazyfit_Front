FROM ubuntu:24.04

ARG DEBIAN_FRONTEND=noninteractive
ARG FLUTTER_VERSION=3.32.0

RUN userdel -r ubuntu

RUN apt-get update && apt-get install -y --no-install-recommends \
    bash \
    ca-certificates \
    curl \
    git \
    gnupg \
    jq \
    openssh-client \
    unzip \
    xz-utils \
    zip \
    libglu1-mesa \
    libgtk-3-0 \
    libnss3 \
    libxss1 \
    libasound2 \
    libstdc++6 \
  && rm -rf /var/lib/apt/lists/*

# Node.js (for Flutter web + optional JS tooling)
RUN curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - \
  && apt-get update && apt-get install -y --no-install-recommends nodejs \
  && rm -rf /var/lib/apt/lists/*

# Create a non-root user for AI isolation
RUN useradd -m vscode -s /bin/bash

USER vscode
WORKDIR /home/vscode

# Flutter SDK
RUN git clone --depth 1 --branch "${FLUTTER_VERSION}" https://github.com/flutter/flutter.git /home/vscode/flutter
ENV PATH="/home/vscode/flutter/bin:/home/vscode/flutter/bin/cache/dart-sdk/bin:${PATH}"

RUN flutter --version \
  && flutter config --enable-web \
  && flutter doctor -v || true
