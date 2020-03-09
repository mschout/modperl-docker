# Install perl 5.18
FROM debian:stretch-slim AS base-image

# add an http user as early as possible
RUN useradd -m -s /bin/bash http \
  && mkdir /app \
  && chown http:http app

RUN apt-get update && apt-get install -y \
    curl \
    file \
    libpcre3 \
    libexpat1 \
    make \
    netbase \
  && rm -rf /var/cache/apt/* /var/lib/apt/lists/*

FROM base-image AS modperl-build

RUN apt-get update && apt-get install -y \
    build-essential \
    libexpat1-dev \
    libssl-dev \
    libpcre3-dev \
  && rm -rf /var/cache/apt/* /var/lib/apt/lists/*

# Install perl
COPY patches/perl/*.patch /usr/src/perl/
WORKDIR /usr/src/perl

RUN NPROC=$(nproc) \
    && curl -SL https://www.cpan.org/src/5.0/perl-5.20.3.tar.bz2 -o perl-5.20.3.tar.bz2 \
    && echo '1b40068166c242e34a536836286e70b78410602a80615143301e52aa2901493b *perl-5.20.3.tar.bz2' | sha256sum -c - \
    && tar --strip-components=1 -xaf perl-5.20.3.tar.bz2 -C /usr/src/perl \
    && rm perl-5.20.3.tar.bz2 \
    && cat *.patch | patch -p1 \
    && gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
    && archBits="$(dpkg-architecture --query DEB_BUILD_ARCH_BITS)" \
    && archFlag="$([ "$archBits" = '64' ] && echo '-Duse64bitall' || echo '-Duse64bitint')" \
    && ./Configure -Darchname="$gnuarch" "$archFlag" -Duseshrplib -Duseithreads -Dvendorprefix=/usr/local -Accflags=-fPIC -des \
    && make -j$NPROC \
    && make install \
    && rm -f /usr/bin/perl \
    && ln -s /usr/local/bin/perl /usr/bin/perl \
    && cd /usr/src \
    && curl -LO https://raw.githubusercontent.com/miyagawa/cpanminus/master/cpanm \
    && chmod +x cpanm \
    && ./cpanm -q --notest App::cpanminus \
    && rm -fr ./cpanm /root/.cpanm /usr/src/perl /tmp/*

# Install APR
WORKDIR /usr/src/apr
RUN NPROC=$(nproc) \
    && curl -SL http://mirrors.advancedhosters.com/apache/apr/apr-1.7.0.tar.gz -o apr-1.7.0.tar.gz \
    && tar --strip-components=1 -xaf apr-1.7.0.tar.gz -C /usr/src/apr \
    && rm -rf apr-1.7.0.tar.gz \
    && ./configure --prefix=/usr/local \
    && make -j$NPROC \
    && make install \
    && ldconfig \
    && rm -rf /usr/src/apr

# Install apr-util
WORKDIR /usr/src/apr-util
RUN NPROC=$(nproc) \
    && curl -SL http://mirrors.advancedhosters.com/apache/apr/apr-util-1.6.1.tar.gz -o apr-util-1.6.1.tar.gz \
    && tar --strip-components=1 -xaf apr-util-1.6.1.tar.gz -C /usr/src/apr-util \
    && rm -rf apr-util-1.6.1.tar.gz \
    && ./configure --prefix=/usr/local --with-apr=/usr/local/bin/apr-1-config \
    && make -j$NPROC \
    && make install \
    && ldconfig \
    && rm -rf /usr/src/apr-util

# Install apache
WORKDIR /usr/src/apache
RUN NPROC=$(nproc) \
    && curl -SL http://mirrors.advancedhosters.com/apache/httpd/httpd-2.4.38.tar.bz2 -o httpd-2.4.38.tar.bz2 \
    && tar --strip-components=1 -xaf httpd-2.4.38.tar.bz2 -C /usr/src/apache \
    && ./configure \
        --with-apr=/usr/local/bin/apr-1-config \
        --with-apr-util=/usr/local/bin/apu-1-config \
        --with-mpm=prefork \
        --prefix=/usr/local \
        --sysconfdir=/usr/local/etc/apache \
        --libexecdir=/usr/local/libexec \
    && make -j$NPROC \
    && make install \
    && ldconfig \
    && rm -rf /usr/src/apache /usr/local/cgi-bin /usr/local/man /usr/local/manual /usr/local/htdocs

# Install mod_perl
WORKDIR /usr/src/modperl
RUN NPROC=$(nproc) \
    && curl -SL http://mirrors.advancedhosters.com/apache/perl/mod_perl-2.0.11.tar.gz -o mod_perl-2.0.11.tar.gz \
    && tar --strip-components=1 -xaf mod_perl-2.0.11.tar.gz -C /usr/src/modperl \
    && perl Makefile.PL \
        MP_APR_CONFIG=/usr/local/bin/apr-1-config \
        MP_APXS=/usr/local/bin/apxs \
        MP_COMPAT_1X=0 \
    && make -j$NPROC \
    && make install \
    && echo 'LoadModule perl_module libexec/mod_perl.so' > /usr/local/etc/apache/extra/mod-perl.conf \
    && rm -rf /usr/src/modperl

# install CPAN deps
RUN cpanm -q --skip-satisfied --notest Test::More URI LWP::UserAgent Apache::Test \
  && rm -rf $HOME/.cpanm \
  && rm -f /usr/local/bin/head

# final image, copy perl and apache builds into it
FROM base-image

COPY --from=modperl-build /usr/local /usr/local

RUN ldconfig

WORKDIR /app
