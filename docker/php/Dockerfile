FROM php:8.4-fpm-alpine AS base

WORKDIR /var/www/html

COPY --from=mlocati/php-extension-installer:latest \
    /usr/bin/install-php-extensions \
    /usr/local/bin/

RUN install-php-extensions \
    bcmath \
    intl \
    mbstring \
    opcache \
    pdo_mysql \
    zip

RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"


# ============================================================
# Composer dependencies
# ============================================================

FROM base AS builder

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

COPY composer.json composer.lock ./

RUN composer install \
    --no-dev \
    --no-interaction \
    --no-plugins \
    --no-scripts \
    --prefer-dist \
    --optimize-autoloader


# ============================================================
# Common runtime
# ============================================================

FROM base AS runner

RUN adduser -D -u 1000 -G www-data laravel

COPY docker/php/conf.d/opcache-development.ini \
    "$PHP_INI_DIR/conf.d/opcache.ini"

RUN mkdir -p \
        bootstrap/cache \
        storage/framework/cache/data \
        storage/framework/sessions \
        storage/framework/views \
        storage/logs \
    && chown -R laravel:www-data \
        bootstrap/cache \
        storage \
    && chmod -R 775 \
        bootstrap/cache \
        storage

COPY --from=builder \
    --chown=laravel:www-data \
    /var/www/html/vendor \
    ./vendor

COPY --chown=laravel:www-data . .

USER laravel

EXPOSE 9000

CMD ["php-fpm", "-F"]

# ============================================================
# Production runtime
# ============================================================

FROM runner AS production

COPY docker/php/conf.d/opcache-production.ini \
    "$PHP_INI_DIR/conf.d/opcache.ini"