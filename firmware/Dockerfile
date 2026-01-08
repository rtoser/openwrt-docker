FROM debian:bookworm

RUN apt-get update && apt-get install -y --no-install-recommends \
    binutils bison build-essential bzip2 ca-certificates clang coreutils diffutils \
    file findutils flex g++ g++-multilib gawk gcc-multilib gettext git grep gzip \
    libncurses5-dev libssl-dev patch perl python3 python3-distutils \
    python3-pyelftools python3-setuptools rsync subversion swig tar \
    unzip util-linux wget which zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /work
