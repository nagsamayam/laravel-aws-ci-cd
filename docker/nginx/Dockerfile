FROM nginx:alpine

COPY docker/nginx/default.conf /etc/nginx/conf.d/default.conf

WORKDIR /var/www/html

COPY public ./public

EXPOSE 80