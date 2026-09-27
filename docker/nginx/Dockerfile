FROM nginx:alpine

RUN apk update && \
    apk upgrade --no-cache libexpat && \
    rm -rf /var/cache/apk/*

COPY docker/nginx/default.conf /etc/nginx/conf.d/default.conf

WORKDIR /var/www/html

COPY public ./public

EXPOSE 80