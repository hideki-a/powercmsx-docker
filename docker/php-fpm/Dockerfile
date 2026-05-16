# ================================================================
# Stage 1: builder
# Compiles QDBM, Hyper Estraier, MeCab, and all PHP extensions.
# Build tools stay in this stage and never reach the final image.
# ================================================================
ARG PHP_VERSION=8.3
FROM php:${PHP_VERSION}-fpm-bookworm AS builder

# ----------------------------------------------------------------
# Build dependencies (compiler, headers – builder stage only)
# ----------------------------------------------------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    autoconf \
    automake \
    libtool \
    pkg-config \
    cmake \
    git \
    wget \
    # Header packages required by libraries and PHP extensions
    libssl-dev \
    libcurl4-openssl-dev \
    libxml2-dev \
    libzip-dev \
    libpng-dev \
    libjpeg-dev \
    libwebp-dev \
    libfreetype6-dev \
    libonig-dev \
    libicu-dev \
    libgd-dev \
    zlib1g-dev \
    libbz2-dev \
    libzstd-dev \
    # MeCab headers (for Hyper Estraier --enable-mecab and php-mecab build)
    libmecab-dev \
    && rm -rf /var/lib/apt/lists/*

# ----------------------------------------------------------------
# QDBM (Alfasado patched – gcc patched tarball)
# ----------------------------------------------------------------
RUN wget -q https://www.powercms.jp/document/powercms/5/images/qdbm-1.8.78-gcc-patched.tar.gz \
        -O /tmp/qdbm.tar.gz \
    && tar -xzf /tmp/qdbm.tar.gz -C /tmp \
    && cd /tmp/qdbm-1.8.78-gcc-patched \
    && ./configure --prefix=/usr/local --enable-zlib \
    && make -j"$(nproc)" \
    && make install \
    && rm -rf /tmp/qdbm-1.8.78-gcc-patched /tmp/qdbm.tar.gz

# ----------------------------------------------------------------
# Hyper Estraier (depends on QDBM)
# ----------------------------------------------------------------
RUN wget -q https://dbmx.net/hyperestraier/hyperestraier-1.4.13.tar.gz \
        -O /tmp/hyperestraier.tar.gz \
    && tar -xzf /tmp/hyperestraier.tar.gz -C /tmp \
    && cd /tmp/hyperestraier-1.4.13 \
    && ./configure --prefix=/usr/local --enable-mecab \
    && make -j"$(nproc)" \
    && make install \
    && rm -rf /tmp/hyperestraier-1.4.13 /tmp/hyperestraier.tar.gz

# ----------------------------------------------------------------
# PHP extensions (compiled in builder so tools stay here)
# ----------------------------------------------------------------
RUN docker-php-ext-configure gd \
        --with-freetype \
        --with-jpeg \
        --with-webp \
    && docker-php-ext-install -j"$(nproc)" \
        bcmath \
        calendar \
        exif \
        gd \
        intl \
        mbstring \
        mysqli \
        opcache \
        pdo \
        pdo_mysql \
        soap \
        zip

RUN pecl install redis \
    && docker-php-ext-enable redis

# php-mecab を GitHub からビルド
RUN git clone https://github.com/ranvis/php-mecab.git /tmp/php-mecab \
    && cd /tmp/php-mecab/mecab \
    && phpize \
    && ./configure --with-php-config=/usr/local/bin/php-config \
    && make -j"$(nproc)" \
    && make install \
    && docker-php-ext-enable mecab \
    && rm -rf /tmp/php-mecab

# ================================================================
# Stage 2: final
# Clean php:${PHP_VERSION}-fpm base + only runtime artifacts copied from builder.
# No compiler, no *-dev headers.
# ================================================================
ARG PHP_VERSION=8.3
FROM php:${PHP_VERSION}-fpm-bookworm AS final

# ----------------------------------------------------------------
# Runtime libraries only (no -dev, no build tools)
# ----------------------------------------------------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
    libssl3 \
    libcurl4 \
    libxml2 \
    libzip4 \
    libpng16-16 \
    libjpeg62-turbo \
    libwebp7 \
    libfreetype6 \
    libonig5 \
    libicu72 \
    zlib1g \
    libbz2-1.0 \
    libzstd1 \
    # MeCab runtime
    mecab \
    libmecab2 \
    mecab-ipadic-utf8 \
    # Misc
    procps \
    curl \
    && rm -rf /var/lib/apt/lists/*

# ----------------------------------------------------------------
# Copy compiled artifacts from builder
# ----------------------------------------------------------------
COPY --from=builder /usr/local/lib/libqdbm.so*      /usr/local/lib/
COPY --from=builder /usr/local/lib/libestraier.so*  /usr/local/lib/
COPY --from=builder /usr/local/bin/estcmd           /usr/local/bin/
COPY --from=builder /usr/local/bin/estmaster        /usr/local/bin/
COPY --from=builder /usr/local/lib/php/extensions/  /usr/local/lib/php/extensions/
COPY --from=builder /usr/local/etc/php/conf.d/      /usr/local/etc/php/conf.d/

RUN ldconfig

# ----------------------------------------------------------------
# supercronic (cron コンテナ用)
# ----------------------------------------------------------------
ARG SUPERCRONIC_VERSION=0.2.33
ARG TARGETARCH
RUN curl -fsSLo /usr/local/bin/supercronic \
        "https://github.com/aptible/supercronic/releases/download/v${SUPERCRONIC_VERSION}/supercronic-linux-${TARGETARCH}" \
    && chmod +x /usr/local/bin/supercronic

# ----------------------------------------------------------------
# Composer
# ----------------------------------------------------------------
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

# ----------------------------------------------------------------
# PHP configuration
# ----------------------------------------------------------------
COPY php.ini      /usr/local/etc/php/conf.d/powercmsx.ini
COPY php-fpm.conf /usr/local/etc/php-fpm.d/zz-powercmsx.conf

# ----------------------------------------------------------------
# Application directories
# ----------------------------------------------------------------
RUN mkdir -p /var/www/powercmsx /var/www/html /var/www/tmp /var/www/work \
    && chown -R www-data:www-data /var/www

WORKDIR /var/www/powercmsx

VOLUME ["/var/www/powercmsx", "/var/www/html"]

EXPOSE 9000
