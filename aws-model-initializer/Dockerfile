FROM amazon/aws-cli
USER root
RUN mkdir -p /.aws && \
    chmod 0777 /.aws
USER 1000    
ENTRYPOINT ["aws","s3","sync"]