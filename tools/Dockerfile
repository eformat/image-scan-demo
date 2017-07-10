FROM docker.io/openshift/jenkins-2-centos7:latest
USER root
# php7
RUN yum-config-manager && \
    rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm && \
    rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm && \
    yum -y install php70w && \
    yum clean all
USER 1001
