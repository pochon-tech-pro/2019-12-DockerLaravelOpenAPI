FROM php:7.3-apache

RUN apt-get update && apt-get install -y git libzip-dev libxml2-dev
RUN docker-php-ext-configure zip --with-libzip
RUN docker-php-ext-install pdo_mysql mbstring zip xml
RUN curl -sS https://getcomposer.org/installer | php
RUN mv composer.phar /usr/local/bin/composer
RUN a2enmod rewrite && a2enmod headers
RUN sed -ri -e 's!/var/www/html!/var/www/public!g' /etc/apache2/sites-available/*.conf
RUN sed -ri -e 's!/var/www/!/var/www/public!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf
RUN usermod -u 1000 www-data && groupmod -g 1000 www-data

ENV COMPOSER_ALLOW_SUPERUSER 1
ENV COMPOSER_HOME /composer
ENV PATH $PATH:/composer/vendor/bin

WORKDIR /var/www

RUN composer global require "laravel/installer"