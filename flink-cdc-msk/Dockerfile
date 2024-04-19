ARG EMR_VERSION
FROM public.ecr.aws/emr-on-eks/flink/emr-${EMR_VERSION}-flink:latest
USER root
ARG FLINK_VERSION="1.17.1"
ENV FLINK_HOME="/usr/lib/flink/"
ENV FLINK_VERSION=${FLINK_VERSION}
ENV MAVEN_VERSION="3.9.6"
ENV MAVEN_URL="https://apache.osuosl.org/maven/maven-3/"${MAVEN_VERSION}"/binaries"

RUN mkdir -p /usr/share/maven
RUN curl -o /tmp/apache-maven.tar.gz ${MAVEN_URL}/apache-maven-${MAVEN_VERSION}-bin.tar.gz && \
    tar -xzf /tmp/apache-maven.tar.gz -C /usr/share/maven --strip-components=1 && \
    rm -f /tmp/apache-maven.tar.gz && \
    ln -s /usr/share/maven/bin/mvn /usr/bin/mvn

RUN mkdir -p $FLINK_HOME/usrlib
RUN rm -rf /usr/bin/python && ln -s /usr/bin/python2 /usr/bin/python
RUN yum -y install git && git clone https://github.com/yhyyz/flink-on-eks-workshop.git /tmp/flink-on-eks-workshop && \
    cd /tmp/flink-on-eks-workshop/ && \
    git pull && \
    cd /tmp/flink-on-eks-workshop/flink-cdc-msk && \
    mvn clean package -Dscope.type=provided

RUN rm -rf /usr/bin/python && ln -s /usr/bin/python3 /usr/bin/python

RUN cp /tmp/flink-on-eks-workshop/flink-cdc-msk/target/flink-cdc-msk-*.jar $FLINK_HOME/usrlib/

# Use hadoop user and group
USER hadoop:hadoop

