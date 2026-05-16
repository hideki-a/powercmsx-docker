FROM httpd:2.4-bookworm

# ----------------------------------------------------------------
# Enable required Apache modules
# mod_proxy_fcgi  : forward PHP requests to php-fpm container
# mod_rewrite     : URL rewriting (PowerCMS X .htaccess rules)
# mod_headers     : response header control
# mod_expires     : cache-control for static assets
# mod_deflate     : gzip compression
# ----------------------------------------------------------------
RUN sed -i \
        -e 's/#LoadModule proxy_module/LoadModule proxy_module/' \
        -e 's/#LoadModule proxy_fcgi_module/LoadModule proxy_fcgi_module/' \
        -e 's/#LoadModule rewrite_module/LoadModule rewrite_module/' \
        -e 's/#LoadModule headers_module/LoadModule headers_module/' \
        -e 's/#LoadModule expires_module/LoadModule expires_module/' \
        -e 's/#LoadModule deflate_module/LoadModule deflate_module/' \
        -e 's/#LoadModule socache_shmcb_module/LoadModule socache_shmcb_module/' \
        /usr/local/apache2/conf/httpd.conf

# ----------------------------------------------------------------
# Configuration
# ----------------------------------------------------------------
COPY conf/httpd.conf /usr/local/apache2/conf/httpd.conf
COPY conf/vhost.conf /usr/local/apache2/conf/extra/vhost.conf

# ----------------------------------------------------------------
# Document root for static files (EFS mount point)
# ----------------------------------------------------------------
RUN mkdir -p /var/www/html /var/www/static \
    && chown -R www-data:www-data /var/www 2>/dev/null || true

WORKDIR /var/www/html

# EFS mount points (mounted at runtime via ECS volume)
VOLUME ["/var/www/html", "/var/www/static"]

EXPOSE 80

CMD ["httpd-foreground"]
