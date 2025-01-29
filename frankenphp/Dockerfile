# Base image for AlmaLinux 9 on any architecture
FROM almalinux:9 AS common
# Install EPEL and CBR repositories
RUN dnf -y update && \
    dnf clean all

RUN dnf install -y epel-release \
    && dnf config-manager --set-enabled crb \
    && dnf clean all

# Install dependencies
RUN dnf -y update && \
    dnf install -y \
    gcc \
    gcc-c++ \
    make \
    brotli-devel \
    tar \
    libticonv \
    bison \
    re2c \
    pkgconfig \
    autoconf \
    php-devel \
    php-pear \
    openssl-devel \
    libxml2-devel \
    sqlite-devel \
    oniguruma-devel \
    libpng-devel \
    libjpeg-devel \
    libzip-devel \
    mlocate \
    bzip2-devel \
    libcurl-devel \
    libavif-devel \
    libwebp-devel \
    libXpm-devel \
    gmp-devel \
    libicu-devel \
    readline-devel \
    libsodium-devel \
    git 


    
# Download and install Go manually
WORKDIR /usr/src
RUN curl -LO https://go.dev/dl/go1.22.0.linux-amd64.tar.gz && \
    tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz && \
    rm go1.22.0.linux-amd64.tar.gz

# Add Go to PATH
ENV PATH="/usr/local/go/bin:${PATH}"

# Download and extract PHP source
WORKDIR /usr/src/php
RUN curl -L https://www.php.net/distributions/php-8.3.10.tar.gz | tar xz --strip-components=1

# Configure PHP for FrankenPHP build
RUN ./configure \
    --enable-embed \
    --enable-zts \
    --disable-zend-signals \
    --enable-zend-max-execution-timers \
    --with-bz2 --enable-ctype --with-png --with-curl --enable-filter --enable-mbstring --enable-pcntl --enable-phar --enable-posix --with-readline --enable-sockets --with-sodium --with-zlib --enable-intl --with-openssl --with-zip --with-pear --enable-gd --with-avif --with-webp --with-jpeg --with-xpm --with-gmp --enable-embed --enable-cli --enable-phar --with-mysqli --with-pdo_mysql --enable-opcache  && \
    make -j"$(nproc)" && \
    make install

# Install Xdebug
RUN mkdir -p /etc/php.d && \
    pecl install xdebug && \
    echo "zend_extension=/usr/local/lib/php/extensions/no-debug-zts-20230831/xdebug.so" > /etc/php.d/xdebug.ini && \
    find / -name "xdebug.so" 2>/dev/null    # This will show us where the file is
    
# Set environment variables for cross-compiling on ARM64
ENV GOARCH=arm64 \
CGO_ENABLED=1

# Download and compile FrankenPHP with adjusted flags for ARM64
WORKDIR /usr/src
# Build FrankenPHP binary
RUN curl -L https://github.com/dunglas/frankenphp/archive/refs/heads/main.tar.gz | tar xz && \
    cd frankenphp-main/caddy/frankenphp && \
    CGO_CFLAGS="$(php-config --includes)" \
    CGO_LDFLAGS="$(php-config --ldflags | sed 's/-m64//') $(php-config --libs)" \
    go build -tags=nowatcher,nobrotli,nobadger,nomysql,nopgx

# Verify where frankenphp was placed after build
RUN find /usr/src/ -name "frankenphp" -type f

RUN echo "Checking /usr/src/" && ls /usr/src/ || true
RUN echo "Checking /usr/src/frankenphp-main" && ls /usr/src/frankenphp-main || true
RUN echo "Checking /usr/src/frankenphp-main/caddy" && ls /usr/src/frankenphp-main/caddy || true
RUN echo "Checking /usr/src/frankenphp-main/caddy/frankenphp" && ls /usr/src/frankenphp-main/caddy/frankenphp || true
RUN echo "Checking /usr/src/frankenphp-main/caddy/frankenphp/frankenphp" && ls /usr/src/frankenphp-main/caddy/frankenphp/frankenphp || true

# Using xcaddy for additional modules (uncomment and modify to include extra Caddy modules)
# RUN curl -fsSL https://github.com/caddyserver/xcaddy/releases/latest/download/xcaddy_linux_arm64 -o /usr/local/bin/xcaddy && \
#     chmod +x /usr/local/bin/xcaddy && \
#     XCADDY_GO_BUILD_FLAGS="-ldflags='-w -s -extldflags \'-Wl,-z,stack-size=0x80000\'" xcaddy build \
#         --output /usr/local/bin/frankenphp \
#         --with github.com/dunglas/frankenphp/caddy \
#         --with github.com/dunglas/mercure/caddy \
#         --with github.com/dunglas/vulcain/caddy

# Final stage to copy the binary to a clean image
FROM almalinux:9
RUN dnf install -y epel-release \
    && dnf config-manager --set-enabled crb \
    && dnf clean all

RUN dnf -y update && \
    dnf install -y \
    mlocate \
    git \
    zip

# Set environment variables
ENV LD_LIBRARY_PATH="/usr/local/lib:${LD_LIBRARY_PATH}"
ENV PHP_INI_SCAN_DIR=/etc/php.d

# Copy compiled PHP and FrankenPHP binary from builder stage
COPY --from=common /usr/local/bin/php /usr/local/bin/php
COPY --from=common /usr/src/frankenphp-main/caddy/frankenphp/frankenphp /usr/local/bin/frankenphp
COPY --from=common /usr/lib64/ /usr/lib64/
COPY --from=common /usr/local/lib/libphp.so /usr/local/lib/
COPY --from=common /usr/src/php/php.ini-development /usr/local/lib/php.ini

RUN mkdir -p /usr/local/lib/php/extensions/no-debug-zts-20230831
COPY --from=common /usr/local/lib/php/extensions/no-debug-zts-20230831/xdebug.so /usr/local/lib/php/extensions/no-debug-zts-20230831/

RUN echo "include_path = '/etc/php.d'" >> /etc/php.ini

# Create /etc/php.d if it doesn't exist and add dev.ini with specific configurations
RUN mkdir -p /etc/php.d && \
    echo "upload_max_filesize = 10M" > /etc/php.d/dev.ini && \
    echo "post_max_size = 10M" >> /etc/php.d/dev.ini && \
    echo "error_log = /var/log/frankenphp/error.log" >> /etc/php.d/dev.ini

# Create log directory and error log file, then set permissions
    RUN mkdir -p /var/log/frankenphp && \
    touch /var/log/frankenphp/error.log && \
    chmod 664 /var/log/frankenphp/error.log

# Install Composer
RUN EXPECTED_CHECKSUM="$(php -r 'copy("https://composer.github.io/installer.sig", "php://stdout");')" && \
    php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" && \
    ACTUAL_CHECKSUM="$(php -r "echo hash_file('sha384', 'composer-setup.php');")" && \
    if [ "$EXPECTED_CHECKSUM" != "$ACTUAL_CHECKSUM" ]; then \
        >&2 echo 'ERROR: Invalid installer checksum'; \
        rm composer-setup.php; \
        exit 1; \
    fi && \
    php composer-setup.php --quiet && \
    rm composer-setup.php && \
    mv composer.phar /usr/local/bin/composer

# Install Postfix and pypolicyd-spf
RUN dnf -y update && \
    dnf install -y postfix pypolicyd-spf \
    mutt

# Configure Postfix (SPF settings, generic maps, etc.)
RUN echo "smtp_generic_maps = hash:/etc/postfix/generic" >> /etc/postfix/main.cf && \
    echo "smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination, check_policy_service unix:private/policy-spf" >> /etc/postfix/main.cf && \
    echo "policy-spf  unix  -       n       n       -       0       spawn user=nobody argv=/usr/bin/pypolicyd-spf" >> /etc/postfix/master.cf && \
    echo "php@be.dev.localhost no-reply@be.dev.localhost" >> /etc/postfix/generic && \
    sed -i '/^myorigin =/s/.*/myorigin = $myhostname/' /etc/postfix/main.cf && \
    postmap /etc/postfix/generic

# Ensure Postfix logs to stdout for Docker
RUN echo "maillog_file = /dev/stdout" >> /etc/postfix/main.cf

# Install supervisord
RUN dnf install -y supervisor

# Configure supervisord to run both frankenphp and Postfix
RUN mkdir -p /etc/supervisor/conf.d && \
    echo "[supervisord]" > /etc/supervisord.conf && \
    echo "nodaemon=true" >> /etc/supervisord.conf && \
    echo "[inet_http_server]" >> /etc/supervisord.conf && \
    echo "port=127.0.0.1:9001" >> /etc/supervisord.conf && \
    echo "[supervisorctl]" >> /etc/supervisord.conf && \
    echo "[rpcinterface:supervisor]" >> /etc/supervisord.conf && \
    echo "supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface" >> /etc/supervisord.conf && \
    echo "[program:postfix]" >> /etc/supervisord.conf && \
    echo "command=/usr/sbin/postfix start-fg" >> /etc/supervisord.conf && \
    echo "[program:frankenphp]" >> /etc/supervisord.conf && \
    echo "command=/usr/local/bin/frankenphp run --config /etc/frankenphp/Caddyfile" >> /etc/supervisord.conf



# Expose necessary ports
EXPOSE 80 443

# Use supervisord to run both Postfix and FrankenPHP
CMD ["supervisord", "-c", "/etc/supervisord.conf"]




# Expose necessary ports
# EXPOSE 80 443

# # Default command
# CMD ["/usr/local/bin/frankenphp", "run", "--config", "/etc/frankenphp/Caddyfile"]