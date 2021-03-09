FROM elyra/elyra:2.0.1
ENV NB_PREFIX /
USER root
COPY fix-permissions-ocp /usr/local/bin/fix-permissions-ocp
RUN fix-permissions-ocp /opt/conda
USER 1000
CMD ["sh","-c", "jupyter lab --notebook-dir=/home/jovyan --ip=0.0.0.0 --no-browser --allow-root --port=8888 --NotebookApp.token='' --NotebookApp.password='' --NotebookApp.allow_origin='*' --NotebookApp.base_url=${NB_PREFIX}"]