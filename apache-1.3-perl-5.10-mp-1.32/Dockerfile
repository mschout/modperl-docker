# Install perl 5.10
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

# Install perl
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    libexpat1-dev \
    libssl-dev \
  && rm -rf /var/cache/apt/* /var/lib/apt/lists/*

COPY patches/perl/* /usr/src/perl/
WORKDIR /usr/src/perl

RUN NPROC=$(nproc) \
    && curl -SL https://www.cpan.org/src/5.0/perl-5.10.1.tar.bz2 -o perl-5.10.1.tar.bz2 \
    && echo '9385f2c8c2ca8b1dc4a7c31903f1f8dc8f2ba867dc2a9e5c93012ed6b564e826 *perl-5.10.1.tar.bz2' | sha256sum -c - \
    && tar --strip-components=1 -xaf perl-5.10.1.tar.bz2 -C /usr/src/perl \
    && rm perl-5.10.1.tar.bz2 \
    && cat *.patch | patch -p1 \
    && ./Configure -Duse64bitall -Duseshrplib  -Accflags=-fPIC -des \
    && make -j$NPROC \
    && make install \
    && cd /usr/src \
    && curl -LO https://raw.githubusercontent.com/miyagawa/cpanminus/master/cpanm \
    && chmod +x cpanm \
    && ./cpanm -q --notest App::cpanminus \
    && rm -fr ./cpanm /root/.cpanm /usr/src/perl /tmp/*

# Apache + mod_perl
COPY patches/apache/* /usr/src/apache/
WORKDIR /usr/src/apache

# download and extract apache
RUN NPROC=$(nproc) \
  && curl -SL http://archive.apache.org/dist/httpd/apache_1.3.42.tar.gz -o apache_1.3.42.tar.gz \
  && echo '953ff952680efe0297bd925c842234bae2b313608200dcff8297b4590d134952 *apache_1.3.42.tar.gz' | sha256sum -c - \
  && tar --strip-components=1 -xvaf apache_1.3.42.tar.gz -C /usr/src/apache \
  && patch -p1 < apache-getline.patch \
  && patch -p1 < gcc-inline-fix.patch \
  && patch -p1 < alpine-rlim-t.patch

# download and build mod_perl
COPY patches/mod_perl/* /usr/src/mod_perl/
WORKDIR /usr/src/mod_perl
RUN NPROC=$(nproc) \
  && curl -SL http://archive.apache.org/dist/perl/mod_perl-1.31.tar.gz -o mod_perl-1.31.tar.gz \
  && echo '0bc7d2638139b7ac9742e27051513efb332504093eca84488ca15585b14860c1 *mod_perl-1.31.tar.gz' | sha256sum -c - \
  && tar --strip-components=1 -xaf mod_perl-1.31.tar.gz -C /usr/src/mod_perl \
  && patch -p1 < mod_perl-1.32.patch \
  && perl Makefile.PL \
    APACHE_SRC=/usr/src/apache/src \
    DO_HTTPD=1 \
    USE_APACI=1 \
    PREP_HTTPD=1 \
    EVERYTHING=1 \
  && make -j$NPROC

# build apache
WORKDIR /usr/src/apache
RUN NPROC=$(nproc) \
  && bash ./configure \
    --prefix=/usr/local \
    --server-uid=http \
    --sysconfdir=/usr/local/etc/apache \
    --with-layout=Apache \
    --enable-module=all \
    --disable-module=auth_db \
    --disable-module=auth_dbm \
    --enable-shared=max \
    --activate-module=src/modules/perl/libperl.a \
    --disable-shared=perl \
    --disable-rule=EXPAT \
    --disable-rule=WANTHSREGEX \
  && make -j$NPROC \
  && make install

# install mod_perl
WORKDIR /usr/src/mod_perl
RUN make install \
  && rm -rf /usr/local/man /usr/local/htdocs /usr/local/cgi-bin

# install CPAN deps
RUN cpanm -q --skip-satisfied --notest Test::More@0.94 \
  && cpanm -q --skip-satisfied --notest URI@1.30 \
  && cpanm -q --skip-satisfied --notest LWP::UserAgent@2.033 \
  && cpanm -q --skip-satisfied --notest Apache::Test@1.28 \
  && cpanm -q --skip-satisfied --notest Params::Util@1.07 \
  && rm -rf $HOME/.cpanm \
  && rm -f /usr/local/bin/bin/head

# final image, copy perl and apache builds into it
FROM base-image

COPY --from=modperl-build /usr/local /usr/local

RUN ldconfig

WORKDIR /app
