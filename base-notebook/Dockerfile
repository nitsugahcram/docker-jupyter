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
    python3.8 \
    python3-pip
RUN apt-get update && apt-get install -y \
    python3-distutils \
    python3-setuptools \
    inkscape pandoc texlive-xetex texlive-fonts-recommended \
    texlive-generic-recommended jq awscli \
    curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
RUN python3.8 -m pip install pip

# Copy a script that we will use to correct permissions after running certain commands
COPY fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

RUN fix-permissions ${HOME}
# Enable prompt color in the skeleton .bashrc before creating the default NB_USER
RUN sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc

# RUN apt-get install -y curl
# RUN curl -sL https://deb.nodesource.com/setup_14.x | sudo bash -
# RUN apt-get install --no-install-recommends -y nodejs && apt-get clean && rm -rf /var/lib/apt/lists/*

RUN update-alternatives --install /usr/bin/python python /usr/bin/python3.8 1
USER $NB_UID
WORKDIR $HOME
# Setup work directory for backward-compatibility
RUN fix-permissions /home/${USER}
RUN pip3 install -U pip
RUN pip3 install -vU setuptools
RUN pip3 install -U notebook 
RUN pip3 install yapf isort autopep8 RISE ipywidgets boto3 iplantuml ruamel-yaml pandas matplotlib seaborn pandoc
RUN pip3 install -U numpy pyarrow 
RUN pip3 install -U scipy statsmodels
RUN pip3 install jupyter_contrib_nbextensions && jupyter contrib nbextension install --user
RUN pip3 install -U psutil nbresuse[resources]

RUN jupyter nbextension enable codefolding/main
RUN jupyter nbextension enable spellchecker/main
RUN jupyter nbextension enable collapsible_headings/main
RUN jupyter nbextension enable snippets_menu/main
RUN jupyter nbextension enable code_prettify/autopep8
RUN jupyter nbextension enable code_prettify/isort
RUN jupyter nbextension enable execute_time/ExecuteTime
RUN jupyter nbextension enable zenmode/main
RUN jupyter nbextension enable toc2/main
RUN jupyter nbextension enable toggle_all_line_numbers/main
RUN jupyter nbextension enable splitcell/splitcell
RUN jupyter nbextension enable freeze/main
RUN jupyter nbextension enable scratchpad/main
RUN jupyter nbextension enable rise/main
RUN jupyter nbextension enable highlighter/highlighter
RUN jupyter nbextension enable hide_input/main
RUN jupyter nbextension enable move_selected_cells/main
RUN pip3 install -U jupyterthemes


# EXPOSE 8888
# Copy local files as late as possible to avoid cache busting
COPY start.sh start-notebook.sh start-singleuser.sh /usr/local/bin/

USER root
RUN jupyter serverextension enable --py nbresuse --sys-prefix
RUN jupyter nbextension install --py nbresuse --sys-prefix
RUN jupyter nbextension enable --py nbresuse --sys-prefix
RUN jupyter nbextension enable nbresuse --py --sys-prefix

# Spark dependencies
ENV APACHE_SPARK_VERSION=3.0.0 \
    HADOOP_VERSION=3.2
## V2.4.6-2.7 3A9F401EDA9B5749CDAFD246B1D14219229C26387017791C345A23A65782FB8B25A302BF4AC1ED7C16A1FE83108E94E55DAD9639A51C751D81C8C0534A4A9641
## V3.0.0-3.2 BFE45406C67CC4AE00411AD18CC438F51E7D4B6F14EB61E7BF6B5450897C2E8D3AB020152657C0239F253735C263512FFABF538AC5B9FFFA38B8295736A9C387
## Install Java RE 11 for compatibility with SPARK 3.0
RUN apt-get -y update && \
    apt-get install --no-install-recommends -y openjdk-11-jre-headless ca-certificates-java wget gnupg && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Using the preferred mirror to download the file
RUN cd /tmp && \
    wget -q $(wget -qO- https://www.apache.org/dyn/closer.lua/spark/spark-${APACHE_SPARK_VERSION}/spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz\?as_json | \
    python -c "import sys, json; content=json.load(sys.stdin); print(content['preferred']+content['path_info'])") && \
    echo "BFE45406C67CC4AE00411AD18CC438F51E7D4B6F14EB61E7BF6B5450897C2E8D3AB020152657C0239F253735C263512FFABF538AC5B9FFFA38B8295736A9C387 *spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz" | sha512sum -c - && \
    tar xzf spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz -C /usr/local --owner root --group root --no-same-owner && \
    rm spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz
RUN cd /usr/local && ln -s spark-${APACHE_SPARK_VERSION}-bin-hadoop${HADOOP_VERSION} spark

ENV SPARK_HOME=/usr/local/spark
ENV PYTHONPATH=$SPARK_HOME/python:$SPARK_HOME/python/lib/py4j-0.10.9-src.zip \
    # MESOS_NATIVE_LIBRARY=/usr/local/lib/libmesos.so \
    SPARK_OPTS="--driver-java-options=-Xms4096M --driver-java-options=-Xmx8192M --driver-java-options=-Dlog4j.logLevel=info" \
    PATH=$PATH:$SPARK_HOME/bin

# COPY mesos.key /tmp/
# RUN apt-get -y update && \
#     apt-get install --no-install-recommends -y gnupg libcurl3 && \
#     apt-key add /tmp/mesos.key && \ 
#     echo "deb http://repos.mesosphere.io/ubuntu xenial main" > /etc/apt/sources.list.d/mesosphere.list && \
#     apt-get -y update && \
#     apt-get --no-install-recommends -y install libcurl3 mesos=1.2\* && \
#     apt-get purge --auto-remove -y gnupg && \
#     rm -rf /var/lib/apt/lists/*

# COPY spark-2.4.6-bin-hadoop2.7.tgz.sha512 spark-2.4.6-bin-hadoop2.7.tgz.sha512
USER $NB_UID
## Set Dark Mode
# RUN jt -t chesterish -f anonymous -fs 12 -nf ptsans -tfs 11 -nfs 11 -N -kl -cursw 5 -cursc r -cellw 95% -T

EXPOSE 9001
EXPOSE 4040
EXPOSE 8000