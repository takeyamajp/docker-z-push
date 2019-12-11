FROM centos:centos7
MAINTAINER "Hiroki Takeyama"

# certificate
RUN mkdir /cert; \
    yum -y install openssl; \
    openssl genrsa -aes128 -passout pass:dummy -out "/cert/key.pass.pem" 2048; \
    openssl rsa -passin pass:dummy -in "/cert/key.pass.pem" -out "/cert/key.pem"; \
    rm -f /cert/key.pass.pem; \
    yum clean all;

# httpd
RUN yum -y install httpd mod_ssl; \
    sed -i 's/^#\(ServerName\) .*/\1 ${HOSTNAME}/' /etc/httpd/conf/httpd.conf; \
    sed -i 's/^\s*\(CustomLog\) .*/\1 \/dev\/stdout "%{X-Forwarded-For}i %h %l %u %t \\"%r\\" %>s %b \\"%{Referer}i\\" \\"%{User-Agent}i\\" %I %O"/' /etc/httpd/conf/httpd.conf; \
    sed -i 's/^\(ErrorLog\) .*/\1 \/dev\/stderr/' /etc/httpd/conf/httpd.conf; \
    sed -i 's/^\s*\(CustomLog\) .*/\1 \/dev\/stdout "%{X-Forwarded-For}i %h %l %u %t \\"%r\\" %>s %b \\"%{Referer}i\\" \\"%{User-Agent}i\\" %I %O"/' /etc/httpd/conf.d/ssl.conf; \
    sed -i 's/^\(ErrorLog\) .*/\1 \/dev\/stderr/' /etc/httpd/conf.d/ssl.conf; \
    sed -i 's/^\s*"%t %h %{SSL_PROTOCOL}x %{SSL_CIPHER}x \\"%r\\" %b"/CustomLog \/dev\/stdout "%{X-Forwarded-For}i %h %l %u %t %{SVN-ACTION}e %U" env=SVN-ACTION/' /etc/httpd/conf.d/ssl.conf; \
    sed -i 's/^\(LoadModule auth_digest_module .*\)/#\1/' /etc/httpd/conf.modules.d/00-base.conf; \
    rm -f /etc/httpd/conf.modules.d/00-proxy.conf; \
    rm -f /usr/sbin/suexec; \
    yum clean all;

# PHP, z-push
RUN yum -y install epel-release; \
    rpm -Uvh http://rpms.famillecollet.com/enterprise/remi-release-7.rpm; \
    { \
    echo '[z-push]'; \
    echo 'name=Z-Push noarch Enterprise Linux 7 - $basearch'; \
    echo 'baseurl=http://repo.z-hub.io/z-push:/final/RHEL_7'; \
    echo 'failovermethod=priority'; \
    echo 'enabled=1'; \
    echo 'gpgcheck=0'; \
    } > /etc/yum.repos.d/z-push.repo; \
    yum -y install --enablerepo=remi,remi-php72 z-push-common z-push-ipc-sharedmemory z-push-config-apache z-push-backend-combined z-push-backend-imap z-push-backend-carddav z-push-backend-caldav; \
    sed -i 's/^;\(error_log\) .*/\1 = \/dev\/stderr/' /etc/php.ini; \
    sed -i 's/\(define('\''IPC_PROVIDER'\'', '\''\).*\('\'');\)/\1IpcSharedMemoryProvider\2/' /etc/z-push/z-push.conf.php; \
    sed -i 's/\(define('\''BACKEND_PROVIDER'\'', '\''\).*\('\'');\)/\1BackendCalDAV\2/' /etc/z-push/z-push.conf.php; \
    sed -i 's/\(define('\''CALDAV_PATH'\'', '\''\).*\('\'');\)/\1/\%u\/\2/' /etc/z-push/caldav.conf.php; \
    yum clean all;

# entrypoint
RUN { \
    echo '#!/bin/bash -eu'; \
    echo 'rm -f /etc/localtime'; \
    echo 'ln -fs /usr/share/zoneinfo/${TIMEZONE} /etc/localtime'; \
    echo 'ESC_TIMEZONE=`echo ${TIMEZONE} | sed "s/\//\\\\\\\\\//g"`'; \
    echo 'sed -i "s/^;\?\(date\.timezone\) =.*/\1 =${ESC_TIMEZONE}/" /etc/php.ini'; \
    echo 'sed -i "s/\(define('\''TIMEZONE'\'', '\''\).*\('\'');\)/\1${ESC_TIMEZONE}\2/" /etc/z-push/z-push.conf.php;'; \
    echo 'openssl req -new -key "/cert/key.pem" -subj "/CN=${HOSTNAME}" -out "/cert/csr.pem"'; \
    echo 'openssl x509 -req -days 36500 -in "/cert/csr.pem" -signkey "/cert/key.pem" -out "/cert/cert.pem" &>/dev/null'; \
    echo 'sed -i "s/^\(SSLCertificateFile\) .*/\1 \/cert\/cert.pem/" /etc/httpd/conf.d/ssl.conf'; \
    echo 'sed -i "s/^\(SSLCertificateKeyFile\) .*/\1 \/cert\/key.pem/" /etc/httpd/conf.d/ssl.conf'; \
    echo 'if [ -e /svn/cert.pem ] && [ -e /svn/key.pem ]; then'; \
    echo '  sed -i "s/^\(SSLCertificateFile\) .*/\1 \/svn\/cert.pem/" /etc/httpd/conf.d/ssl.conf'; \
    echo '  sed -i "s/^\(SSLCertificateKeyFile\) .*/\1 \/svn\/key.pem/" /etc/httpd/conf.d/ssl.conf'; \
    echo 'fi'; \
    echo 'sed -i "s/^\(LogLevel\) .*/\1 ${HTTPD_LOG_LEVEL}/" /etc/httpd/conf/httpd.conf'; \
    echo 'sed -i "s/^\(LogLevel\) .*/\1 ${HTTPD_LOG_LEVEL}/" /etc/httpd/conf.d/ssl.conf'; \
    echo 'sed -i "s/^\(CustomLog .*\)/#\1/" /etc/httpd/conf/httpd.conf'; \
    echo 'sed -i "s/^\(ErrorLog .*\)/#\1/" /etc/httpd/conf/httpd.conf'; \
    echo 'sed -i "s/^\(CustomLog .*\)/#\1/" /etc/httpd/conf.d/ssl.conf'; \
    echo 'sed -i "s/^\(ErrorLog .*\)/#\1/" /etc/httpd/conf.d/ssl.conf'; \
    echo 'if [ ${HTTPD_LOG,,} = "true" ]; then'; \
    echo '  sed -i "s/^#\(CustomLog .*\)/\1/" /etc/httpd/conf/httpd.conf'; \
    echo '  sed -i "s/^#\(ErrorLog .*\)/\1/" /etc/httpd/conf/httpd.conf'; \
    echo '  sed -i "s/^#\(CustomLog .*\)/\1/" /etc/httpd/conf.d/ssl.conf'; \
    echo '  sed -i "s/^#\(ErrorLog .*\)/\1/" /etc/httpd/conf.d/ssl.conf'; \
    echo 'fi'; \
    echo 'if [ -e /etc/httpd/conf.d/requireSsl.conf ]; then'; \
    echo '  rm -f /etc/httpd/conf.d/requireSsl.conf'; \
    echo 'fi'; \
    echo 'if [ ${REQUIRE_SSL,,} = "true" ]; then'; \
    echo '  {'; \
    echo '  echo "<Location />"'; \
    echo '  echo "  SSLRequireSSL"'; \
    echo '  echo "</Location>"'; \
    echo '  } > /etc/httpd/conf.d/requireSsl.conf'; \
    echo 'fi'; \
    echo 'sed -i "s/\(define('\''CALDAV_SERVER'\'', '\''\).*\('\'');\)/\1${CALDAV_SERVER}\2/" /etc/z-push/carddav.conf.php;'; \
    echo 'exec "$@"'; \
    } > /usr/local/bin/entrypoint.sh; \
    chmod +x /usr/local/bin/entrypoint.sh;
ENTRYPOINT ["entrypoint.sh"]

ENV TIMEZONE Asia/Tokyo

ENV REQUIRE_SSL true

ENV HTTPD_LOG true
ENV HTTPD_LOG_LEVEL warn

ENV CALDAV_SERVER caldav.example.com

EXPOSE 80
EXPOSE 443

CMD ["httpd", "-DFOREGROUND"]
