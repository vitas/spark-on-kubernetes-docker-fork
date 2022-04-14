FROM vitass/livy-builder:0.3 as build

ARG aws_bundle_version=1.11.1001
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

ARG MINICONDA_VERSION=4.8.3
ARG MINICONDA_MD5=d63adf39f2c220950a063e0529d4ff74
ARG CONDA_VERSION=4.8.3
ARG PYTHON_VERSION=3.7.8

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
  ln -s /opt/spark-${SPARK_VERSION}-bin-hadoop3.2 ${SPARK_HOME}

COPY conf/* $SPARK_CONF_DIR/
# $SPARK_HOME/conf gets cleaned by Spark on Kubernetes internals, create and add to classpath another directory for logging and other configs
COPY conf/* $SPARK_HOME/cluster-conf/
COPY entrypoint.sh /opt/
COPY Dockerfile /my_docker/

WORKDIR $SPARK_HOME/work-dir
ENTRYPOINT [ "/opt/entrypoint.sh" ]

# Specify the User that the actual main process will run as
USER ${spark_uid}
