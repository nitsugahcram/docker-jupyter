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
RUN add-apt-repository ppa:inkscape.dev/stable
RUN apt-get update && apt-get install -y inkscape

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
RUN pip3 install -U setuptools
RUN pip3 install jupyter jupyterlab ipywidgets boto3 iplantuml ruamel-yaml pandas matplotlib seaborn bokeh 
RUN pip3 install -U numpy pyarrow
RUN pip3 install -U RISE

## INSTALL
## CODE FORMATTER
# RUN jupyter labextension install @ryantam626/jupyterlab_code_formatter
RUN pip3 install boto3 iplantuml ruamel-yaml pandas \
    matplotlib plotly jupyter jupyterlab ipywidgets \
    pyarrow yapf isort jupyterlab_code_formatter \
    jupyterlab_templates psutil nbresuse==0.3.3 \
    --upgrade nbdime --upgrade jupyterlab-git && \
    jupyter serverextension enable --py jupyterlab --py jupyterlab_code_formatter \
    --py jupyterlab_templates --py jupyterlab_git --py nbdime --py nbresuse
## INSTALL GIT
## TOC version that display arrows
RUN jupyter labextension install @ryantam626/jupyterlab_code_formatter @jupyterlab/git @ijmbarr/jupyterlab_spellchecker \
    @jupyterlab/toc @jupyter-widgets/jupyterlab-manager \
    jupyterlab_templates --no-build

RUN jupyter labextension install jupyterlab-topbar-extension \
    jupyterlab-system-monitor \
    jupyterlab-topbar-text \
    jupyterlab-logout \
    jupyterlab-theme-toggle --no-build


COPY overrides.json /home/${USER}/.local/share/jupyter/lab/settings/
COPY jupyter_notebook_config.py /home/${USER}/.jupyter/

# Copy local files as late as possible to avoid cache busting
COPY start.sh start-notebook.sh start-singleuser.sh /usr/local/bin/

USER root

# Spark dependencies
ENV APACHE_SPARK_VERSION=2.4.5 \
    HADOOP_VERSION=2.7

## Install Java RE 8 for compatibility with SPARK 2.4
RUN apt-get -y update && \
    apt-get install --no-install-recommends -y openjdk-8-jre-headless ca-certificates-java wget && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Using the preferred mirror to download the file
RUN cd /tmp && \
    wget -q $(wget -qO- https://www.apache.org/dyn/closer.lua/spark/spark-${APACHE_SPARK_VERSION}/spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz\?as_json | \
    python -c "import sys, json; content=json.load(sys.stdin); print(content['preferred']+content['path_info'])") && \
    echo "2426a20c548bdfc07df288cd1d18d1da6b3189d0b78dee76fa034c52a4e02895f0ad460720c526f163ba63a17efae4764c46a1cd8f9b04c60f9937a554db85d2 *spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz" | sha512sum -c - && \
    tar xzf spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz -C /usr/local --owner root --group root --no-same-owner && \
    rm spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz
RUN cd /usr/local && ln -s spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION} spark

# Mesos dependencies
# Install from the Xenial Mesosphere repository since there does not (yet)
# exist a Bionic repository and the dependencies seem to be compatible for now.
COPY mesos.key /tmp/
RUN apt-get -y update && \
    apt-get install --no-install-recommends -y gnupg libcurl3 && \
    apt-key add /tmp/mesos.key && \
    echo "deb http://repos.mesosphere.io/ubuntu xenial main" > /etc/apt/sources.list.d/mesosphere.list && \
    apt-get -y update && \
    apt-get --no-install-recommends -y install mesos=1.2\* && \
    apt-get purge --auto-remove -y gnupg && \
    rm -rf /var/lib/apt/lists/*

# Spark and Mesos config
# ENV SPARK_HOME=/usr/local/spark
# ENV PATH=$PATH:${SPARK_HOME}/bin
# Spark and Mesos config
ENV SPARK_HOME=/usr/local/spark
ENV PYTHONPATH=$SPARK_HOME/python:$SPARK_HOME/python/lib/py4j-0.10.7-src.zip \
    MESOS_NATIVE_LIBRARY=/usr/local/lib/libmesos.so \
    SPARK_OPTS="--driver-java-options=-Xms4096M --driver-java-options=-Xmx8192M --driver-java-options=-Dlog4j.logLevel=info" \
    PATH=$PATH:$SPARK_HOME/bin
# copy kernel.json .local/share/jupyter/kernels/

USER $NB_UID
# COPY kernel.json /home/${USER}/.local/share/jupyter/kernels/pyspark/
# RUN jupyter labextension install jupyterlab_spark
RUN jupyter lab build -y && \
    jupyter lab clean -y && \
    npm cache clean --force && \
    rm -rf /home/$NB_USER/.cache/yarn && \
    rm -rf /home/$NB_USER/.node-gyp 


EXPOSE 9001
EXPOSE 4040
EXPOSE 8000
