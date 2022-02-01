FROM httpd
COPY index.html /usr/local/apache2/htdocs/
RUN mkdir -p /var/local/apache2/conf/sites
COPY site1.conf /var/local/apache2/conf/sites/
CMD ["httpd", "-D", "FOREGROUND"]