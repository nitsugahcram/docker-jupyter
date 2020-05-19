ARG BASE_CONTAINER=ubuntu:bionic
# docker pull ubuntu:bionic
FROM $BASE_CONTAINER
## Base on https://github.com/jupyter/docker-stacks/blob/master/base-notebook/Dockerfile
LABEL maintainer="Nitsuga Chram <agustin.march@gmail.com>"
ARG NB_USER=analytics
ARG NB_UID=1000
ARG NB_GID=100
# default password for user
ARG PW=docker

USER root
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update && apt-get install --no-install-recommends -y locales && rm -rf /var/lib/apt/lists/*
RUN echo "LC_ALL=en_US.UTF-8" >> /etc/environment
RUN echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
RUN echo "LANG=en_US.UTF-8" > /etc/locale.conf

ENV LANG=en_US.UTF-8 \
    LANGUAGE=en_US:en \
    LC_ALL=en_US.UTF-8

# Option1: Using unencrypted password/ specifying password
RUN groupadd -g ${NB_GID} -o ${NB_GROUP:-${NB_USER}}
RUN useradd -m ${NB_USER} --uid=${NB_UID} -g ${NB_GID}  && echo "${NB_USER}:${PW}" | chpasswd
# Configure environment
ENV SHELL=/bin/bash \
    USER=$NB_USER \
    UID=$NB_UID \
    GID=$NB_GID

RUN export LC_ALL=en_US.UTF-8
RUN export LANG=en_US.UTF-8
RUN locale-gen en_US.UTF-8

ENV HOME=/home/${NB_USER} \
    PATH=/home/${NB_USER}/.local/bin:${PATH}

##SUDO USERS
RUN apt-get update && apt-get install -y sudo
RUN echo "$NB_USER ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/notebook

RUN apt-get update && apt-get install -y \
        software-properties-common
RUN add-apt-repository ppa:deadsnakes/ppa
RUN apt-get update && apt-get install -y \
    python3.6 \
    python3-pip
RUN python3.6 -m pip install pip
RUN apt-get update && apt-get install -y \
    python3-distutils \
    python3-setuptools \
    curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Copy a script that we will use to correct permissions after running certain commands
COPY fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

RUN fix-permissions ${HOME}
# Enable prompt color in the skeleton .bashrc before creating the default NB_USER
RUN sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc

# RUN apt-get install -y curl
RUN curl -sL https://deb.nodesource.com/setup_14.x | sudo bash -
RUN apt-get install --no-install-recommends -y nodejs && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN update-alternatives --install /usr/bin/python python /usr/bin/python3.6 1
USER $NB_UID
WORKDIR $HOME
# Setup work directory for backward-compatibility
RUN fix-permissions /home/${USER}

RUN pip3 install -U pip
RUN pip3 install -vU setuptools
RUN pip3 install jupyter jupyterlab && jupyter serverextension enable --py jupyterlab

## CODE FORMATTER
RUN jupyter labextension install @ryantam626/jupyterlab_code_formatter
RUN pip3 install yapf isort jupyterlab_code_formatter && jupyter serverextension enable --py jupyterlab_code_formatter
RUN pip3 install --upgrade jupyterlab-git nbdime && jupyter serverextension enable --py jupyterlab_git
RUN jupyter labextension install @jupyterlab/git @jupyterlab/toc @ijmbarr/jupyterlab_spellchecker \
    @telamonian/theme-darcula
RUN pip3 install jupyterlab_templates
RUN jupyter labextension install jupyterlab_templates
RUN jupyter serverextension enable --py jupyterlab_templates

RUN jupyter lab build -y && \
    jupyter lab clean -y && \
    npm cache clean --force && \
    rm -rf /home/$NB_USER/.cache/yarn && \
    rm -rf /home/$NB_USER/.node-gyp

COPY overrides.json /home/${USER}/.local/share/jupyter/lab/settings/
COPY jupyter_notebook_config.py /home/${USER}/.jupyter/

EXPOSE 8888
# Copy local files as late as possible to avoid cache busting
COPY start.sh start-notebook.sh start-singleuser.sh /usr/local/bin/

USER $NB_UID