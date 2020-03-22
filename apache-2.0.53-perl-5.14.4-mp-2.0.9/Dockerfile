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
    libexpat1-dev \
    libssl-dev \
  && rm -rf /var/cache/apt/* /var/lib/apt/lists/*

COPY patches/perl/*.patch /usr/src/perl/
WORKDIR /usr/src/perl

RUN NPROC=$(nproc) \
    && curl -SL https://www.cpan.org/src/5.0/perl-5.14.4.tar.bz2 -o perl-5.14.4.tar.bz2 \
    && echo 'eece8c2b0d491bf6f746bd1f4f1bb7ce26f6b98e91c54690c617d7af38964745 *perl-5.14.4.tar.bz2' | sha256sum -c - \
    && tar --strip-components=1 -xaf perl-5.14.4.tar.bz2 -C /usr/src/perl \
    && rm perl-5.14.4.tar.bz2 \
    && cat *.patch | patch -p1 \
    && ./Configure -Duse64bitall -Duseshrplib  -Accflags=-fPIC -des \
    && make -j$NPROC \
    && make install \
    && cd /usr/src \
    && curl -LO https://raw.githubusercontent.com/miyagawa/cpanminus/master/cpanm \
    && chmod +x cpanm \
    && ./cpanm -q --notest App::cpanminus \
    && rm -fr ./cpanm /root/.cpanm /usr/src/perl /tmp/*

# install apache
RUN NPROC=$(nproc) \
  && mkdir /usr/src/apache \
  && cd /usr/src/apache \
  && curl -SL http://archive.apache.org/dist/httpd/httpd-2.0.53.tar.bz2 -o httpd-2.0.53.tar.bz2 \
  && echo 'cb2352d68af460194ba3f60f86252cd15d8d9c164960cbad3428b42edd7e8dcd *httpd-2.0.53.tar.bz2' | sha256sum -c - \
  && tar --strip-components=1 -xaf httpd-2.0.53.tar.bz2 -C /usr/src/apache \
  && CFLAGS="-std=gnu89 -fPIC" ./configure \
    --prefix=/usr/local \
    --sysconfdir=/usr/local/etc/apache \
    --with-included-apr \
    --enable-mods-shared="most dav proxy unique_id" \
  && make -j$NPROC \
  && make install \
  && rm -rf /usr/local/cgi-bin /usr/local/htdocs /usr/local/man /usr/local/manual

# install mod_perl
COPY patches/mod_perl/* /usr/src/mod_perl/
WORKDIR /usr/src/mod_perl

RUN NPROC=$(nproc) \
  && curl -SL http://archive.apache.org/dist/perl/mod_perl-2.0.9.tar.gz -o mod_perl-2.0.9.tar.gz \
  && echo '0260f26ab771c1c95fadc67544b3400e420bb30b8a77565d2d02ad05498ef52b *mod_perl-2.0.9.tar.gz' | sha256sum -c - \
  && tar --strip-components=1 -xaf mod_perl-2.0.9.tar.gz -C /usr/src/mod_perl \
  && patch -p1 < modperl-clang.patch \
  && perl Makefile.PL \
    MP_APXS=/usr/local/bin/apxs \
    MP_COMPAT_1X=0 \
  && make -j$NPROC \
  && make install

# install CPAN deps
RUN cpanm -q --skip-satisfied --notest Test::More~0.94 \
  && cpanm -q --skip-satisfied --notest URI~1.30 \
  && cpanm -q --skip-satisfied --notest LWP::UserAgent~2.033 \
  && cpanm -q --skip-satisfied --notest Apache::Test~1.28 \
  && rm -rf $HOME/.cpanm \
  && rm -f /usr/local/bin/head

# final image, copy perl and apache builds into it
FROM base-image

COPY --from=modperl-build /usr/local /usr/local

WORKDIR /app
