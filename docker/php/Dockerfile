# =========================================================
# BinxinAdmin · PHP 8.3 FPM 自定义镜像
# 基于官方 php:8.3-fpm-bookworm (多架构: arm64 / amd64)
# 装入：pdo_mysql / redis / bcmath / gd / opcache / zip
#       intl / exif / pcntl / mbstring / sockets / swoole
# =========================================================
FROM php:8.3-fpm-bookworm

ARG TZ=Asia/Shanghai
ENV TZ=${TZ}

# ----- 系统依赖与时区 -----
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        git \
        unzip \
        vim \
        tzdata \
        procps \
        iputils-ping \
        net-tools; \
    ln -snf "/usr/share/zoneinfo/${TZ}" /etc/localtime; \
    echo "${TZ}" > /etc/timezone; \
    rm -rf /var/lib/apt/lists/*

# ----- PHP 扩展（用社区标准 install-php-extensions 一行装齐） -----
ADD --chmod=0755 https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/

RUN set -eux; \
    install-php-extensions \
        pdo_mysql \
        redis \
        bcmath \
        gd \
        opcache \
        zip \
        intl \
        exif \
        pcntl \
        mbstring \
        sockets \
        swoole

# ----- Composer 2（从官方镜像直接 COPY 二进制，最干净） -----
COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer

# ----- 工作目录 -----
WORKDIR /var/www

# ----- 健康检查（fpm 是否在监听 9000） -----
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD php-fpm -t || exit 1

# ----- 暴露 FPM 端口（仅容器内访问） -----
EXPOSE 9000

CMD ["php-fpm"]
