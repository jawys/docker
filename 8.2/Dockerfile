ARG PHP_VERSION
ARG PROJECT_TYPE

FROM alpine:3.18 AS mozjpeg
WORKDIR /tmp
ENV MOZJPEG_BUILD_DEPS \
    autoconf \
    automake \
    build-base \
    libtool \
    nasm \
    pkgconf \
    tar
ENV MOZJPEG_VERSION="3.3.1"
ADD https://github.com/mozilla/mozjpeg/archive/v${MOZJPEG_VERSION}.tar.gz ./
RUN <<EOF
    set -eux
    apk add --no-cache --virtual .mozjpeg_build_deps $MOZJPEG_BUILD_DEPS
    tar -xzf v${MOZJPEG_VERSION}.tar.gz --strip-components=1
    autoreconf -fiv
    ./configure --with-jpeg8
    make -j$(nproc)
    make -j$(nproc) install
    apk del --no-network .mozjpeg_build_deps
    find /tmp -mindepth 1 -delete
EOF

FROM alpine:3.13 AS iconv-build
RUN apk upgrade --no-cache \
    && apk add --no-cache gnu-libiconv

FROM php:${PHP_VERSION}-${PROJECT_TYPE}-alpine3.18

# setup general options for environment variables
ARG PHP_MEMORY_LIMIT_ARG="256M"
ENV PHP_MEMORY_LIMIT=${PHP_MEMORY_LIMIT_ARG}
ARG PHP_MAX_EXECUTION_TIME_ARG="120"
ENV PHP_MAX_EXECUTION_TIME=${PHP_MAX_EXECUTION_TIME_ARG}
ARG PHP_UPLOAD_MAX_FILESIZE_ARG="20M"
ENV PHP_UPLOAD_MAX_FILESIZE=${PHP_UPLOAD_MAX_FILESIZE_ARG}
ARG PHP_MAX_INPUT_VARS_ARG="1000"
ENV PHP_MAX_INPUT_VARS=${PHP_MAX_INPUT_VARS_ARG}
ARG PHP_POST_MAX_SIZE_ARG="8M"
ENV PHP_POST_MAX_SIZE=${PHP_POST_MAX_SIZE_ARG}

# setup opcache for environment variables
ARG PHP_OPCACHE_ENABLE_ARG="1"
ARG PHP_OPCACHE_REVALIDATE_FREQ_ARG="0"
ARG PHP_OPCACHE_VALIDATE_TIMESTAMPS_ARG="0"
ARG PHP_OPCACHE_MAX_ACCELERATED_FILES_ARG="10000"
ARG PHP_OPCACHE_MEMORY_CONSUMPTION_ARG="128"
ARG PHP_OPCACHE_MAX_WASTED_PERCENTAGE_ARG="10"
ARG PHP_OPCACHE_INTERNED_STRINGS_BUFFER_ARG="16"
ARG PHP_OPCACHE_FAST_SHUTDOWN_ARG="1"
ENV PHP_OPCACHE_ENABLE=$PHP_OPCACHE_ENABLE_ARG
ENV PHP_OPCACHE_REVALIDATE_FREQ=$PHP_OPCACHE_REVALIDATE_FREQ_ARG
ENV PHP_OPCACHE_VALIDATE_TIMESTAMPS=$PHP_OPCACHE_VALIDATE_TIMESTAMPS_ARG
ENV PHP_OPCACHE_MAX_ACCELERATED_FILES=$PHP_OPCACHE_MAX_ACCELERATED_FILES_ARG
ENV PHP_OPCACHE_MEMORY_CONSUMPTION=$PHP_OPCACHE_MEMORY_CONSUMPTION_ARG
ENV PHP_OPCACHE_MAX_WASTED_PERCENTAGE=$PHP_OPCACHE_MAX_WASTED_PERCENTAGE_ARG
ENV PHP_OPCACHE_INTERNED_STRINGS_BUFFER=$PHP_OPCACHE_INTERNED_STRINGS_BUFFER_ARG
ENV PHP_OPCACHE_FAST_SHUTDOWN=$PHP_OPCACHE_FAST_SHUTDOWN_ARG

RUN <<EOF
    set -eux -o pipefail
    apk add --no-cache --virtual .build-deps \
        freetype-dev \
        icu-data-full \
        icu-dev \
        imagemagick-dev \
        libjpeg-turbo-dev \
        libpng-dev \
        libwebp-dev \
        libxml2-dev \
        libzip-dev \
        postgresql-dev \
    ;
    docker-php-ext-configure gd \
        --with-freetype \
        --with-jpeg \
        --with-webp \
    ;
    docker-php-ext-install -j$(nproc) \
        bcmath \
        gd \
        intl \
        opcache \
        pdo \
        pdo_mysql \
        pdo_pgsql \
        soap \
        zip \
    ;
    apk add --no-cache --virtual .phpize-deps-configure $PHPIZE_DEPS
    pecl install \
        imagick \
        redis \
    ;
    docker-php-ext-enable \
        imagick \
        redis \
    ;
    apk del --no-network .phpize-deps-configure
    runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )";
    apk add --no-cache $runDeps
    apk del --no-network .build-deps
    find /tmp -mindepth 1 -delete
EOF

# copy + link mozjpeg in single layer
RUN --mount=from=mozjpeg,target=/mnt <<EOF
    cp -a /mnt/opt/mozjpeg /opt
    ln -s /opt/mozjpeg/bin/jpegtran /usr/local/bin/mozjpegtran
    ln -s /opt/mozjpeg/bin/cjpeg /usr/local/bin/mozcjpeg
EOF

# https://github.com/craftcms/docker/issues/16
COPY --from=iconv-build /usr/lib/preloadable_libiconv.so /usr/lib/preloadable_libiconv.so
# https://github.com/docker-library/php/issues/1121
ENV LD_PRELOAD /usr/lib/preloadable_libiconv.so

# copy custom.ini settings
COPY craft-cms.ini /usr/local/etc/php/conf.d/

# run container as the www-data user
USER www-data

# set the working directory for convenience
WORKDIR /app
