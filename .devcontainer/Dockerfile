# AstraOS Yocto build container.
# Based on Ubuntu 26.04 (LTS).

FROM ubuntu:26.04

ARG USER_UID=1000
ARG USER_GID=1000
ARG USERNAME=builder

ENV DEBIAN_FRONTEND=noninteractive

# Yocto Wrynose host deps (from Yocto Project Reference Manual,
# "Required Packages for the Build Host"). Plus a few quality-of-life
# tools (vim, less, rsync, sudo) and ca-certificates for HTTPS fetches.
RUN apt-get update && apt-get install -y --no-install-recommends \
        gawk \
        wget \
        git \
        diffstat \
        unzip \
        texinfo \
        gcc \
        build-essential \
        chrpath \
        socat \
        cpio \
        python3 \
        python3-pip \
        python3-pexpect \
        xz-utils \
        debianutils \
        iputils-ping \
        python3-git \
        python3-jinja2 \
        python3-subunit \
        libegl1 \
        libsdl1.2-dev \
        mesa-common-dev \
        libacl1 \
        zstd \
        lz4 \
        file \
        locales \
        sudo \
        curl \
        ca-certificates \
        openssh-client \
        vim \
        less \
        rsync \
        bash-completion \
    && rm -rf /var/lib/apt/lists/*

# en_US.UTF-8 locale - bitbake refuses to run otherwise
RUN locale-gen en_US.UTF-8 && update-locale LANG=en_US.UTF-8
ENV LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8

# Google repo tool (used to manage the multi-layer source tree)
RUN curl -fsSL https://storage.googleapis.com/git-repo-downloads/repo \
        -o /usr/local/bin/repo \
    && chmod a+rx /usr/local/bin/repo

# Non-root build user. bitbake refuses to run as root; UID/GID match the
# host user (overridable via build args) so bind-mounted files have correct
# ownership. Passwordless sudo for one-off apt installs.
#
# Ubuntu 24.04+ ships a default 'ubuntu' user at UID 1000 - remove it first
# so our useradd --uid 1000 doesn't collide.
RUN userdel -r ubuntu 2>/dev/null || true \
    && groupadd --gid ${USER_GID} ${USERNAME} \
    && useradd --uid ${USER_UID} --gid ${USER_GID} --shell /bin/bash -m ${USERNAME} \
    && echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME} \
    && chmod 0440 /etc/sudoers.d/${USERNAME}

USER ${USERNAME}
WORKDIR /home/akothapalli/yocto/astraos

# Helpful shell history persistence in the workspace
RUN echo 'export HISTFILE=/home/akothapalli/yocto/.bash_history' >> ~/.bashrc \
    && echo 'export HISTSIZE=10000 HISTFILESIZE=20000' >> ~/.bashrc

CMD ["/bin/bash"]
