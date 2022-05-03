FROM vitass/livy-builder:0.3 as build

ARG aws_bundle_version=1.12.210
RUN mvn dependency:copy -Dartifact=com.amazonaws:aws-java-sdk-bundle:${aws_bundle_version} -DoutputDirectory=/tmp

ARG java_image_tag=11-jre-slim

LABEL maintainer="Vitalis Green <vr@samebits.com>"

FROM openjdk:11-jre-slim

ENV BASE_IMAGE openjdk:${java_image_tag}

RUN set -ex && \
    sed -i 's/http:/https:/g' /etc/apt/sources.list && \
    apt-get update && \
    ln -s /lib /lib64 && \
    apt install -y bash tini libc6 libpam-modules krb5-user libnss3 wget bzip2 && \
    rm /bin/sh && \
    ln -sv /bin/bash /bin/sh && \
    echo "auth required pam_wheel.so use_uid" >> /etc/pam.d/su && \
    chgrp root /etc/passwd && chmod ug+rw /etc/passwd && \
    rm -rf /var/cache/apt/*

ENV SPARK_VERSION   3.1.3
ENV HADOOP_VERSION  hadoop-3.2.0-cloud
ENV SCALA_VERSION   2.12

ENV SPARK_HOME      /opt/spark
ENV SPARK_CONF_DIR  $SPARK_HOME/conf
ENV SPARK_CLASSPATH $SPARK_HOME/cluster-conf

ENV PYTHONHASHSEED  0
ENV CONDA_DIR       /opt/conda
ENV SHELL           /bin/bash

ENV PATH            $PATH:$SPARK_HOME/bin:$CONDA_DIR/bin


ARG PYTHON_VERSION=3.7
ARG spark_uid=185

COPY --from=build /tmp/aws-java-sdk-bundle-*.jar /
RUN  wget --content-disposition https://archive.apache.org/dist/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}-bin-hadoop3.2.tgz -O /tmp/spark-${SPARK_VERSION}-bin-hadoop3.2.tgz && \
  tar -xvzf /tmp/spark-${SPARK_VERSION}-bin-hadoop3.2.tgz -C /opt/ && \
  rm -f /tmp/spark-${SPARK_VERSION}-bin-hadoop3.2.tgz && \
  mkdir -p /opt/spark-${SPARK_VERSION}-bin-hadoop3.2/work-dir && \
  mkdir -p /opt/spark-${SPARK_VERSION}-bin-hadoop3.2/spark-warehouse && \
  mkdir -p /opt/spark-${SPARK_VERSION}-bin-hadoop3.2/cluster-conf && \
  chmod g+w /opt/spark-${SPARK_VERSION}-bin-hadoop3.2/work-dir && \
  cp /aws-java-sdk-bundle-*.jar /opt/spark-${SPARK_VERSION}-bin-hadoop3.2/jars && \
  wget -O /opt/spark-${SPARK_VERSION}-bin-hadoop3.2/jars/hadoop-aws-3.2.0.jar https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aws/3.2.0/hadoop-aws-3.2.0.jar && \
  ln -s /opt/spark-${SPARK_VERSION}-bin-hadoop3.2 ${SPARK_HOME}

ARG CONDA_MIRROR=https://github.com/conda-forge/miniforge/releases/latest/download

RUN mkdir -p $CONDA_DIR && set -x && \
    # Miniforge installer
    miniforge_arch=$(uname -m) && \
    miniforge_installer="Mambaforge-Linux-${miniforge_arch}.sh" && \
    wget --quiet "${CONDA_MIRROR}/${miniforge_installer}" && \
    /bin/bash "${miniforge_installer}" -f -b -p "${CONDA_DIR}" && \
    rm "${miniforge_installer}" && \
    # Conda configuration see https://conda.io/projects/conda/en/latest/configuration.html
    conda config --system --set auto_update_conda false && \
    conda config --system --set show_channel_urls true && \
    if [[ "${PYTHON_VERSION}" != "default" ]]; then mamba install --quiet --yes python="${PYTHON_VERSION}"; fi && \
    # Pin major.minor version of python
    mamba list python | grep '^python ' | tr -s ' ' | cut -d ' ' -f 1,2 >> "${CONDA_DIR}/conda-meta/pinned" && \
    # Using conda to update all packages: https://github.com/mamba-org/mamba/issues/1092
    conda update --all --quiet --yes && \
    conda clean --all -f -y 

# Using fixed version of mamba in arm, because the latest one has problems with arm under qemu
# See: https://github.com/jupyter/docker-stacks/issues/1539
RUN set -x && \
    arch=$(uname -m) && \
    if [ "${arch}" == "aarch64" ]; then \
        mamba install --quiet --yes \
            'mamba<0.18' && \
            mamba clean --all -f -y; \
    fi;

COPY conf/* $SPARK_CONF_DIR/
# $SPARK_HOME/conf gets cleaned by Spark on Kubernetes internals, create and add to classpath another directory for logging and other configs
COPY conf/* $SPARK_HOME/cluster-conf/
COPY entrypoint.sh /opt/
COPY Dockerfile /my_docker/

WORKDIR $SPARK_HOME/work-dir
ENTRYPOINT [ "/opt/entrypoint.sh" ]

# Specify the User that the actual main process will run as
USER ${spark_uid}
