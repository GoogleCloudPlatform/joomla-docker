FROM marketplace.gcr.io/google/php7-apache2:7.3

RUN set -ex; \
  savedAptMark="$(apt-mark showmanual)"; \
  apt-get update; \
  apt-get install -y --no-install-recommends \
    libjpeg-dev \
    libpng-dev \
    libzip-dev \
    libbz2-dev \
    libpq-dev \
    libldap2-dev \
    libmemcached-dev \
  ; \
  \
  docker-php-ext-configure gd --with-png-dir=/usr --with-jpeg-dir=/usr; \
  debMultiarch="$(dpkg-architecture --query DEB_BUILD_MULTIARCH)"; \
  docker-php-ext-install -j "$(nproc)" \
    bz2 \
    gd \
    ldap \
    mysqli \
    pdo_mysql \
    pdo_pgsql \
    pgsql \
    zip \
    opcache \
  ; \
# pecl will claim success even if one install fails, so we need to perform each install separately
  pecl install APCu-5.1.17; \
  pecl install memcached-3.1.3; \
  pecl install redis-4.3.0; \
  \
  docker-php-ext-enable \
    apcu \
    memcached \
    redis \
  ; \
  \
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
  apt-mark auto '.*' > /dev/null; \
  apt-mark manual $savedAptMark; \
  ldd "$(php -r 'echo ini_get("extension_dir");')"/*.so \
    | awk '/=>/ { print $3 }' \
    | sort -u \
    | xargs -r dpkg-query -S \
    | cut -d: -f1 \
    | sort -u \
    | xargs -rt apt-mark manual; \
  \
  apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
  rm -rf /var/lib/apt/lists/*

# set recommended PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
RUN { \
    echo 'opcache.memory_consumption=128'; \
    echo 'opcache.interned_strings_buffer=8'; \
    echo 'opcache.max_accelerated_files=4000'; \
    echo 'opcache.revalidate_freq=2'; \
    echo 'opcache.fast_shutdown=1'; \
    echo 'opcache.enable_cli=1'; \
  } > /usr/local/etc/php/conf.d/opcache-recommended.ini
RUN a2enmod rewrite expires

RUN set -ex ;\
  apt-get update ;\
  apt-get install -y mariadb-client \
  && rm -rf /var/lib/apt/lists/*

VOLUME /var/www/html

ENV JOOMLA_VERSION 3.9.19
ENV C2D_RELEASE 3.9.19
ENV JOOMLA_SHA512 c1dfdc297b36531c71cc48ad0ffcd0a632a58b5402e105a8143514dbd17895ba3074291919db3ac4ddcb74cb3baf823acef6c6e8f805d8517ae33ffcb484be75

# Download package and extract to web volume
RUN set -ex; \
  curl -o joomla.tar.bz2 -SL https://github.com/joomla/joomla-cms/releases/download/${JOOMLA_VERSION}/Joomla_${JOOMLA_VERSION}-Stable-Full_Package.tar.bz2; \
  echo "$JOOMLA_SHA512 *joomla.tar.bz2" | sha512sum -c -; \
  mkdir /usr/src/joomla; \
  tar -xf joomla.tar.bz2 -C /usr/src/joomla; \
  rm joomla.tar.bz2; \
  chown -R www-data:www-data /usr/src/joomla

# Copy init scripts
COPY docker-entrypoint.sh /entrypoint.sh
COPY makedb.php /makedb.php

ENTRYPOINT ["/entrypoint.sh"]
CMD ["apache2-foreground"]
