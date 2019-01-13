FROM ubuntu:latest

RUN apt-get update && apt-get install -y \
apt-utils \
git \
emacs-nox \
vim \
curl \
wget \
psmisc \
tmux \
software-properties-common

RUN apt-get update && apt-get install -y \
python3 \
python3-pip \
python3-dev

RUN pip3 install --upgrade pip && \
hash -r pip pip3 && \
pip3 install pipenv
ENV SHELL /bin/bash

RUN apt-get update && apt-get install -y locales && rm -rf /var/lib/apt/lists/* \
&& localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LANG en_US.utf8