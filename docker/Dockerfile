FROM httpd:2.4.41-alpine

ENV MOD_AUTH_OPENIDC_REPOSITORY https://github.com/zmartzone/mod_auth_openidc.git

ENV MOD_AUTH_OPENIDC_BRANCH master

ENV BUILD_DIR /tmp/mod_auth_openidc

ENV APACHE_DEFAULT_CONF /usr/local/apache2/conf/httpd.conf

# ADD source
RUN mkdir ${BUILD_DIR} 

# add dependencies, build and install mod_auth_openidc, need atomic operation for image size
RUN apk add --no-cache git \
  g++ \
  autoconf \
  automake \
  make \
  libtool \
  jansson-dev \
  openssl-dev \
  apr-util-dbm_db && \
  git clone https://github.com/cisco/cjose.git && \
  cd cjose && \
  ./configure && make && make install && \
  cd .. && rm -rf cjose && \
  apk add --no-cache \
  apache2 \
  apache2-proxy \
  wget \
  hiredis \
  hiredis-dev \
  build-base \
  curl \
  apache2-dev \
  curl-dev \
  pcre-dev \
  pcre2-dev \
  bash \
  && \
  cd ${BUILD_DIR} && \
  git clone -b ${MOD_AUTH_OPENIDC_BRANCH} ${MOD_AUTH_OPENIDC_REPOSITORY} && \
  cd mod_auth_openidc && \
  ./autogen.sh && \
  ./configure CFLAGS="-g -O0" LDFLAGS="-lrt" && \
  make test && \
  make install && \
  cd ../.. && \
  rm -fr ${BUILD_DIR} && \
  apk del git \
  apache2-dev \
  autoconf \
  automake \
  build-base \
  wget \
  hiredis-dev \
  pcre-dev \
  libtool

# configure apache 
RUN ln -s /usr/local/apache2/modules/mod_auth_openidc.so /usr/lib/apache2/mod_auth_openidc.so && \
  echo "LoadModule auth_openidc_module /usr/lib/apache2/mod_auth_openidc.so" >> "${APACHE_DEFAULT_CONF}"

# Copy in our configuration files.
COPY conf/ conf/

RUN set -ex; \
    # Create empty default DocumentRoot.
    mkdir -p "/var/www/html"; \
    # Create directories for Dav data and lock database.
    mkdir -p "/var/lib/dav/data"; \
    touch "/var/lib/dav/DavLock"; \
    chown -R www-data:www-data "/var/lib/dav"; \
    \
    # Enable DAV modules.
    for i in dav dav_fs; do \
        sed -i -e "/^#LoadModule ${i}_module.*/s/^#//" "conf/httpd.conf"; \
    done; \
    \
    # Make sure other modules are enabled.
    for i in alias headers mime setenvif; do \
        sed -i -e "/^#LoadModule ${i}_module.*/s/^#//" "conf/httpd.conf"; \
    done; \
    \
    # Run httpd as "www-data" (instead of "daemon").
    for i in User Group; do \
        sed -i -e "s|^$i .*|$i www-data|" "conf/httpd.conf"; \
    done; \
    \
    # Include enabled configs and sites.
    printf '%s\n' "Include conf/conf-enabled/*.conf" \
        >> "conf/httpd.conf"; \
    printf '%s\n' "Include conf/sites-enabled/*.conf" \
        >> "conf/httpd.conf"; \
    \
    # Enable dav and default site.
    mkdir -p "conf/conf-enabled"; \
    mkdir -p "conf/sites-enabled"; \
    ln -s ../conf-available/dav.conf "conf/conf-enabled"; \
    ln -s ../sites-available/default.conf "conf/sites-enabled"

COPY ./scripts/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
COPY ./scripts/wait-for-it.sh /scripts/wait-for-it.sh
EXPOSE 80/tcp 443/tcp
ENTRYPOINT [ "docker-entrypoint.sh" ]
CMD [ "httpd-foreground" ]
