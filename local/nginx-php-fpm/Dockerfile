FROM centos:7.4.1708

COPY conf/repo/aliyun-mirror.repo /etc/yum.repos.d/CentOS-Base.repo
COPY conf/repo/aliyun-epel.repo /etc/yum.repos.d/epel.repo

RUN useradd -r -s /sbin/nologin -d /home/wwwroot -m -k no www \
	&& mkdir -p /home/wwwroot/ \
	&& echo "<?php echo '123';" > /home/wwwroot/index.php \
	&& cp -rf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
	&& yum install -y nginx \
	&& yum install -y epel-release \ 
        http://rpms.remirepo.net/enterprise/remi-release-7.rpm  \
        yum-utils \
    && yum-config-manager --enable remi-php72 \
    && yum install -y \
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
        php-pecl-mongodb \
    && yum --enablerepo=epel install -y fcgi \
		supervisor \
    && yum clean all
	
COPY conf/nginx/nginx.conf /etc/nginx/nginx.conf
COPY conf/nginx/www.conf /etc/nginx/conf.d/default.conf

COPY conf/php-fpm/php.ini /etc/php.ini
COPY conf/php-fpm/php-fpm.conf /etc/php-fpm.conf
COPY conf/php-fpm/www.conf /etc/php-fpm.d/www.conf

COPY conf/supervisor/supervisord.conf /etc/supervisord.conf

EXPOSE 80

ENTRYPOINT ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisord.conf"]