# makefile to build apache2 stuff.

APACHE_VER ?=	2.4.4
APACHE_SRC = dist/httpd-$(APACHE_VER)

APR_SRC = dist/apr-1.4.6
APR_UTIL_SRC = dist/apr-util-1.5.2

MODPERL_VER ?=	2.0.9
MODPERL_SRC = dist/mod_perl-$(MODPERL_VER)

LIBAPREQ_VER ?= 2.13
LIBAPREQ_SRC = dist/libapreq2-$(LIBAPREQ_VER)

TOPDIR := $(CURDIR)
PATCHDIR := $(TOPDIR)/patches

prefix ?= $(PERLBREW_ROOT)/perls/$(PERLBREW_PERL)

OSTYPE := $(shell uname)
CFLAGS = -std=gnu89 -fPIC

PATCH_MODPERL_TARGETS = $(MODPER_SRC)

ifeq ($(OSTYPE), FreeBSD)
PATCH_MODPERL_TARGETS += patch-modperl-clang
endif

CPANM = cpanm -q --skip-satisfied

APR_CONFIG = $(prefix)/bin/apr-1-config
APU_CONFIG = $(prefix)/bin/apu-1-config

all: apr apr-util apache modperl prune-cruft git-commit

apr: $(APR_SRC)
	cd $(APR_SRC); ./configure --prefix=$(prefix)
	make -C $(APR_SRC) install
	@touch $@

apr-util: $(APR_UTIL_SRC)
	cd $(APR_UTIL_SRC); ./configure --prefix=$(prefix) --with-apr=$(APR_CONFIG)
	make -C $(APR_UTIL_SRC) install
	@touch $@

apache: $(APACHE_SRC)
	cd $(APACHE_SRC); ./configure \
		--with-apr=$(APR_CONFIG) \
		--with-apr-util=$(APU_CONFIG) \
		--with-mpm=prefork \
		--prefix=$(prefix)
	$(MAKE) -C $(APACHE_SRC)
	$(MAKE) -C $(APACHE_SRC) install
	@touch $@

modperl: $(MODPERL_SRC) patch-modperl
	cd $(MODPERL_SRC); \
		perl Makefile.PL MP_APR_CONFIG=$(prefix)/bin/apr-1-config MP_APXS=$(prefix)/bin/apxs MP_COMPAT_1X=0
	$(MAKE) -C $(MODPERL_SRC)
	$(MAKE) -C $(MODPERL_SRC) install
	@touch $@

cpan:
	@$(CPANM) --notest Test::More~0.94
	@$(CPANM) --notest URI~1.30
	@$(CPANM) --notest LWP::UserAgent~2.033
	@$(CPANM) --notest Apache::Test~1.28
	@rm -f $(prefix)/bin/head
	@touch $@

cpan-minver:
	@$(CPANM) --notest Test::More@0.94
	@$(CPANM) --notest URI@1.30
	@$(CPANM) --notest LWP::UserAgent@2.033
	@$(CPANM) --notest Apache::Test@1.28
	@rm -f $(prefix)/bin/head
	@touch $@

patch-modperl: $(PATCH_MODPERL_TARGETS)
	@touch $@

$(MODPERL_SRC):
	tar xf $(MODPERL_SRC).tar.gz -C dist

$(APR_UTIL_SRC): $(APR_UTIL_SRC).tar.gz
	tar xf $(APR_UTIL_SRC).tar.gz -C dist

$(APR_SRC): $(APR_SRC).tar.gz
	tar xf $(APR_SRC).tar.gz -C dist

$(APACHE_SRC):
	tar xf $(APACHE_SRC).tar.gz -C dist

libapreq: configure-libapreq
	@cd $(LIBAPREQ_SRC); \
		$(MAKE); \
		$(MAKE) install
	@cd $(prefix); git add -A .; git commit -m 'installed libapreq2 v$(LIBAPREQ_VER)'
	@touch $@

extract-libapreq: $(LIBAPREQ_SRC).tar.gz
	tar xf $(LIBAPREQ_SRC).tar.gz -C dist
	@touch $@

patch-libapreq: extract-libapreq
	@touch $@

configure-libapreq: patch-libapreq libapreq-cpan-deps
	@cd $(LIBAPREQ_SRC); CFLAGS="-fPIC" LDFLAGS="-pthread" ./configure \
		--with-apache2-apxs=$(prefix)/bin/apxs \
		--with-expat=/usr/local \
		--enable-perl-glue \
		--with-perl=$(prefix)/bin/perl
	@touch $@

libapreq-cpan-deps:
	@$(CPANM) --notest ExtUtils::XSBuilder::ParseSource Parse::RecDescent
	@touch $@

prune-cruft:
	@rm -rf $(prefix)/cgi-bin \
		$(prefix)/htdocs \
		$(prefix)/man \
		$(prefix)/manual

git-commit:
	@cd $(prefix); \
		git add -A . ; \
		git commit -m 'installed apache v$(APACHE_VER), modperl v$(MODPERL_VER)'
	@touch $@

clean:
	@rm -rf $(APACHE_SRC) $(MODPERL_SRC) \
		$(LIBAPREQ_SRC) $(APR_SRC) $(APR_UTIL_SRC) \
		$(PATCH_MODPERL_TARGETS) \
		apr apr-util apache extract-apache patch-apache build-apache install-apache \
		patch-modperl modperl install-apreq \
		extract-libapreq configure-libapreq patch-libapreq libapreq \
		libapreq-cpan-deps cpan cpan-minver git-commit
