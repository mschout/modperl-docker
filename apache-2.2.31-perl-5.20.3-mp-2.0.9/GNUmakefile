# makefile to build apache2 stuff.

APACHE_VER ?=	2.2.31
APACHE_SRC = dist/httpd-$(APACHE_VER)

MODPERL_VER ?=	2.0.9
MODPERL_SRC = dist/mod_perl-$(MODPERL_VER)

LIBAPREQ_VER ?= 2.13
LIBAPREQ_SRC = dist/libapreq2-$(LIBAPREQ_VER)

TOPDIR := $(CURDIR)
PATCHDIR := $(TOPDIR)/patches

prefix ?= $(PERLBREW_ROOT)/perls/$(PERLBREW_PERL)

OSTYPE := $(shell uname)
CFLAGS = -std=gnu89 -fPIC

PATCH_MODPERL_TARGETS = extract-modperl

ifeq ($(OSTYPE), FreeBSD)
PATCH_MODPERL_TARGETS += patch-modperl-clang
endif

CPANM = cpanm -q --skip-satisfied

all: apache

apache: clean install-apache modperl prune-cruft git-commit
	@touch $@

extract-apache:
	tar xf $(APACHE_SRC).tar.bz2 -C dist
	@touch $@

patch-apache: extract-apache
	@touch $@

ifeq ($(OSTYPE), Darwin)
patch-apache:
	cd $(APACHE_SRC); patch -p1 < $(PATCHDIR)/httpd-2.0.53-no-sendfile.patch
	@touch $@
endif

build-apache: patch-apache
	cd $(APACHE_SRC); \
		CFLAGS="$(CFLAGS)" ./configure --prefix=$(prefix) \
		--with-included-apr \
		--enable-mods-shared="most dav proxy unique_id"
	$(MAKE) -C $(APACHE_SRC) 
	$(MAKE) -C $(APACHE_SRC) install
	@touch $@

install-apache: build-apache
	@touch $@

modperl: patch-modperl
	cd $(MODPERL_SRC); \
		perl Makefile.PL \
			MP_APXS=$(prefix)/bin/apxs \
			MP_COMPAT_1X=0
	make -C $(MODPERL_SRC)
	make -C $(MODPERL_SRC) install
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

#cd $(MODPERL_SRC); patch -p1 < $(PATCHDIR)/modperl-clang.patch
#	perl -pi -e 's/APR_INLINE//g' \
#		$(MODPERL_SRC)/src/modules/perl/modperl_common_util.h
patch-modperl-clang: patches/modperl-clang.patch
	@touch $@

extract-modperl: $(MODPERL_SRC).tar.gz
	@tar xzf $(MODPERL_SRC).tar.gz -C dist
	@touch $@

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

install-apreq:
	@CFLAGS="-fPIC" $(CPANM) -q --notest Apache2::Request
	@cd $(prefix); \
		git add -A . ; \
		git commit -m 'installed Apache2::Request'
	@touch $@

clean:
	@rm -rf $(APACHE_SRC) $(MODPERL_SRC) \
		$(LIBAPREQ_SRC) \
		$(PATCH_MODPERL_TARGETS) \
		apache extract-apache patch-apache build-apache install-apache \
		extract-modperl patch-modperl modperl install-apreq \
		extract-libapreq configure-libapreq patch-libapreq libapreq \
		libapreq-cpan-deps cpan cpan-minver git-commit
