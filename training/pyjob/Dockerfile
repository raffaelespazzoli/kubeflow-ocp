FROM gcr.io/kubeflow-ci/pytorch-dist-mnist_test:1.0
USER root
RUN mkdir -p /var/logs && \
    mkdir -p /var/wb && \
    mkdir -p /data && \
    chmod 0777 /var/logs && \
    chmod 0777 /var/wb && \
    chmod 0777 /data
USER 1000