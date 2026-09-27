FROM nginx:alpine

RUN apt-get update && \
    apt-get install --only-upgrade -y libexpat1 && \
    rm -rf /var/lib/apt/lists/*

COPY docker/nginx/default.conf /etc/nginx/conf.d/default.conf

WORKDIR /var/www/html

COPY public ./public

EXPOSE 80