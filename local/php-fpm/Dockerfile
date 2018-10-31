FROM centos:7

COPY conf/aliyun-mirror.repo /etc/yum.repos.d/CentOS-Base.repo
COPY conf/aliyun-epel.repo /etc/yum.repos.d/epel.repo

COPY container-entrypoint check_fcgi /usr/sbin/
COPY fix-permissions /usr/local/bin/fix-permissions
COPY ep-linux /usr/local/bin/ep

RUN chmod u+x /usr/local/bin/{fix-permissions,ep,container-entrypoint}

RUN yum install -y epel-release \ 
        http://rpms.remirepo.net/enterprise/remi-release-7.rpm  \
        yum-utils && \
    yum-config-manager --enable remi-php70 && \
    yum install -y \
        php-bcmath \
        php-cli \
        php-fpm \
        php-mysqlnd \
        php-mssql \
        php-xml \
        php-pgsql \
        php-gd \
        php-mcrypt \
        php-ldap \
        php-imap \
        php-soap \
        php-tidy \
        php-mbstring \
        php-opcache \
        php-pdo \
        php-pecl-apcu \
        php-pecl-apcu-bc \
        php-pecl-geoip \
        php-pecl-igbinary \
        php-pecl-imagick \
        php-pecl-redis \
		php-pecl-swoole \
		php-pecl-xdebug \
		php-pecl-mongodb && \
    yum --enablerepo=epel install -y fcgi && \
    yum clean all

COPY conf/php-fpm.conf conf/php.ini /etc/
COPY conf/www.conf /etc/php-fpm.d/www.conf

RUN mkdir -p /app && \
    fix-permissions /etc/php.ini && \
    fix-permissions /etc/php-fpm.conf && \
    fix-permissions /etc/php-fpm.d/ && \
    fix-permissions /app && \
    fix-permissions /var/lib/php/session/

EXPOSE 9000

ENTRYPOINT ["container-entrypoint"]

CMD ["/usr/sbin/php-fpm", "-F", "-R"]