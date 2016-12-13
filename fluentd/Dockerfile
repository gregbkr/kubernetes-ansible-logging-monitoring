# docker run -it -v $PWD/log:/root/log -v $PWD/fluent.conf:/fluentd/etc/fluent.conf --name fluentd fluent
FROM fluent/fluentd
MAINTAINER gregbkr@outlook.com
WORKDIR /home/fluent
ENV PATH /home/fluent/.gem/ruby/2.3.0/bin:$PATH

USER root
RUN apk --no-cache --update add sudo build-base ruby-dev && \

    sudo -u fluent gem install fluent-plugin-secure-forward && \
    sudo -u fluent gem install fluent-plugin-elasticsearch && \

    rm -rf /home/fluent/.gem/ruby/2.3.0/cache/*.gem && sudo -u fluent gem sources -c && \
    apk del sudo build-base ruby-dev && rm -rf /var/cache/apk/*

EXPOSE 24284

USER fluent
CMD exec fluentd -c /fluentd/etc/$FLUENTD_CONF -p /fluentd/plugins $FLUENTD_OPT
