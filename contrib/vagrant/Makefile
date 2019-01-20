#
# Copyright © 2011-2018 The Freicoin developers.
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of version 3 of the GNU Affero General Public
# License as published by the Free Software Foundation.
#
# This program is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
# Affero General Public License for more details.
#
# You should have received a copy of the GNU Affero General Public
# License along with this program.  If not, see
# <https://www.gnu.org/licenses/>.
#

MAKEFILE := $(word $(words $(MAKEFILE_LIST)),$(MAKEFILE_LIST))

PROJECT  := bitcoin

SHELL    := $(shell bash --help >/dev/null 2>&1 && echo ba)sh

AWK      := $(shell gawk --help >/dev/null 2>&1 && echo g)awk
GREP     := $(shell egrep --help >/dev/null 2>&1 && echo e)grep
PATCH    := $(shell gpatch --help >/dev/null 2>&1 && echo g)patch
SED      := $(shell gsed --help >/dev/null 2>&1 && echo g)sed
WGET     := wget --no-check-certificate \
                 --user-agent=$(shell wget --version | \
            $(SED) -n 's,GNU \(Wget\) \([0-9.]*\).*,\1/\2,p')

REQUIREMENTS := $(AWK) $(SHELL) diff find $(GREP) gzip $(MAKE) openssl	\
                packer $(PATCH) $(SED) sort tar vagrant xargs xz	\
                $(word 1,$(WGET))

BUILD_MANIFEST = cd '$(shell dirname '$(MAKEFILE)')'/../.. && \
	find $(1) -print0 | \
	xargs -0 -n1 openssl sha512 2>/dev/null | \
	openssl sha512 | \
	$(SED) -n 's,^.*\([0-9a-f]\{128\}\),\1,p'

VALIDATE_MANIFEST = \
	if [ -f "$(1)" ] && echo "$(2)" | diff "$(1)" - 2>&1 >/dev/null; then \
	    rm -f "$(1)"; \
	fi

# ===----------------------------------------------------------------------===

PKGS            :=
CACHE_DIR       := cache
HOST_CACHE_DIR  := $(CACHE_DIR)
GUEST_CACHE_DIR := /vagrant/$(CACHE_DIR)

SOURCEFORGE_MIRROR := downloads.sourceforge.net

PKG             := MacOSX10.9.sdk.tar.gz
$(PKG)_CHECKSUM := c34568856e0520b1459a304577b94a507f3a9932a6883af34cabfc6cbaca48305d2dc36c1954746d2dd69d7ee42023839bd941bfd5f6881161fbc176406f0672
$(PKG)_URL      := https://github.com/maaku/dependencies/raw/2178f4560ecb630260945d027f85d42682b62c06/$(PKG)
PKGS            += $(PKG)

# ===----------------------------------------------------------------------===

VAGRANT_UP = \
	vagrant up

VAGRANT_HALT = \
	vagrant halt

VAGRANT_RELOAD = \
	vagrant reload

VAGRANT_DESTROY = \
	vagrant destroy --force

# ===----------------------------------------------------------------------===

.PHONY: all
all: linux osx win

.PHONY: mostlyclean
mostlyclean:

.PHONY: clean
clean: mostlyclean
	-rm -rf box
	-rm -rf output
	-rm -f .stamp-*
	-$(call VAGRANT_DESTROY)
	-rm -rf .vagrant

.PHONY: distclean
distclean: clean
	-rm -rf cache
	-rm -rf output-gitian-host-xenial64
	-rm -rf packer_cache

.PHONY: maintainer-clean
maintainer-clean: distclean
	@echo 'This command is intended for maintainers to use; it'
	@echo 'deletes files that may need special tools to rebuild.'

.PHONY: dist
dist: all

.PHONY: shell
shell: gitian-host
	$(call VAGRANT_UP)
	vagrant ssh
ifdef VAGRANT_CYCLE
	-$(call VAGRANT_HALT)
endif

.PHONY: pass
pass: check-requirements pass-gitian-host pass-lxc-base

# ===----------------------------------------------------------------------===

.PHONY: check-requirements
define CHECK_REQUIREMENT
	@if ! $(1) --help &>/dev/null; then \
	    if ! which $(1) &>/dev/null; then \
	        echo; \
	        echo 'Missing requirement: $(1)'; \
	        echo; \
	        echo 'Please have a look at "README" to ensure'; \
	        echo 'that your system meets all requirements.'; \
	        echo; \
	        exit 1; \
	    fi; \
	fi

endef
define CHECK_REQUIREMENT_VERSION
	@if ! $(1) --version | head -1 | grep ' \($(2)\)$$' >/dev/null; then \
	    echo; \
	    echo 'Wrong version of requirement: $(1)'; \
	    echo; \
	    echo 'Please have a look at "README" to ensure'; \
	    echo 'that your system meets all requirements.'; \
	    echo; \
	    exit 1; \
	fi

endef
check-requirements: .stamp-check-requirements.h
.stamp-check-requirements.h: $(MAKEFILE)
	@echo '[check requirements]'
	$(foreach REQUIREMENT,$(REQUIREMENTS),$(call CHECK_REQUIREMENT,$(REQUIREMENT)))
	$(call CHECK_REQUIREMENT_VERSION,vagrant,[2-9]\.[0-9]\.[0-9])
	@touch "$@"

# ===----------------------------------------------------------------------===

.PHONY: download
download: $(addprefix download-,$(PKGS)) download-depends

.PHONY: download-depends
download-depends: .stamp-download-depends.h
.stamp-download-depends.h: .stamp-gitian-host.manifest $(HOST_CACHE_DIR)/src-local.manifest
	mkdir -p cache/gitian/cache/common
	$(call VAGRANT_UP)
	$(call DEPLOY_SOURCE_BUNDLE,$(PROJECT))
	vagrant ssh --command "\
	    make -C '$(PROJECT)'/depends download SOURCES_PATH='$(GUEST_CACHE_DIR)'/gitian/cache/common"
ifdef VAGRANT_CYCLE
	-$(call VAGRANT_HALT)
endif
	@touch "$@"

PKG_CHECKSUM = \
	openssl sha512 '$(HOST_CACHE_DIR)/$(1)' 2>/dev/null | $(SED) -n 's,^.*\([0-9a-f]\{128\}\)$$,\1,p'

CHECK_PKG_ARCHIVE = \
	[ x'$($(1)_CHECKSUM)' == x"`$$(call PKG_CHECKSUM,$(1))`" ]

DOWNLOAD_PKG_ARCHIVE = \
	mkdir -p '$(HOST_CACHE_DIR)' && \
	$(if $($(1)_URL_2), \
	    ( $(WGET) -T 30 -t 3 -O- '$($(1)_URL)' || $(WGET) -O- '$($(1)_URL_2)' ), \
	    $(WGET) -O- '$($(1)_URL)') \
	$(if $($(1)_FIX_GZIP), \
	    | gzip -d | gzip -9n, \
	    ) \
	> '$(HOST_CACHE_DIR)/$(1)'

define PKG_RULE
.PHONY: download-$(1)
download-$(1): $(HOST_CACHE_DIR)/$(1)
$(HOST_CACHE_DIR)/$(1): .stamp-check-requirements.h
	@echo -n "Checking cached package $(1)... "; \
	if ! $(call CHECK_PKG_ARCHIVE,$(1)); then \
		echo "failed"; \
	     $(call DOWNLOAD_PKG_ARCHIVE,$(1)); \
	     $(call CHECK_PKG_ARCHIVE,$(1)) \
	        || { echo 'Wrong checksum!'; exit 1; }; \
	else \
		echo "ok"; \
	fi

endef
$(foreach PKG,$(PKGS),$(eval $(call PKG_RULE,$(PKG))))

# ===----------------------------------------------------------------------===

GITIAN_HOST_FILES    := contrib/vagrant/Makefile \
                        contrib/vagrant/gitian-host.json \
                        contrib/vagrant/config/preseed.cfg \
                        contrib/vagrant/config/vagrant.sh \
                        contrib/vagrant/config/sshd.sh \
                        contrib/vagrant/config/packages.sh \
                        contrib/vagrant/config/cleanup.sh \
                        contrib/vagrant/Vagrantfile
GITIAN_HOST_FILES    := $(sort $(GITIAN_HOST_FILES))
GITIAN_HOST_MANIFEST := $(shell $(call BUILD_MANIFEST,$(GITIAN_HOST_FILES)))

_ := $(shell $(call \
	VALIDATE_MANIFEST, \
	.stamp-gitian-host.manifest, \
	$(GITIAN_HOST_MANIFEST) \
))

.PHONY: gitian-host
gitian-host: .stamp-gitian-host.manifest
.stamp-gitian-host.manifest: \
	.stamp-check-requirements.h \
	$(foreach FILE,$(GITIAN_HOST_FILES),$(shell echo $(FILE) | $(SED) 's:^contrib/vagrant/::g'))
	
	-$(call VAGRANT_DESTROY)
	
	# Remove any temporary files from interupted builds that would
	# interfere if present.
	if [ -e output-gitian-host-xenial64 ]; then \
	    rm -rf output-gitian-host-xenial64; \
	fi
	
	packer build gitian-host.json
	vagrant box add --name gitian-host-xenial64 --force box/virtualbox/gitian-host-xenial64-0.box
	-rm -rf box
	
	$(call VAGRANT_UP)
	
	-$(call VAGRANT_HALT)
	
	rm -f .stamp-lxc-base.manifest
	
	echo $(GITIAN_HOST_MANIFEST) > "$@"

.PHONY: pass-gitian-host
pass-gitian-host: check-requirements
	echo $(GITIAN_HOST_MANIFEST) > .stamp-gitian-host.manifest

# ===----------------------------------------------------------------------===

LXC_BASE_FILES    := contrib/vagrant/Makefile
LXC_BASE_FILES    := $(sort $(LXC_BASE_FILES))
LXC_BASE_MANIFEST := $(shell $(call BUILD_MANIFEST,$(LXC_BASE_FILES)))

_ := $(shell $(call \
	VALIDATE_MANIFEST, \
	.stamp-lxc-base.manifest, \
	$(LXC_BASE_MANIFEST) \
))

.PHONY: lxc-base
lxc-base: .stamp-lxc-base.manifest
.stamp-lxc-base.manifest: \
	.stamp-gitian-host.manifest \
	$(foreach FILE,$(LXC_BASE_FILES),$(shell echo $(FILE) | $(SED) 's:^contrib/vagrant/::g'))
	
	$(call VAGRANT_UP)
	
	vagrant ssh --command "\
	    if [ -d ~/gitian-builder ]; then \
	        sudo rm -rf ~/gitian-builder; \
	    fi; \
	    git clone 'https://github.com/devrandom/gitian-builder' ~/gitian-builder || exit 1; \
	    cd ~/gitian-builder; \
	    git checkout ee1806672b7b45fa6149d0163de5828ade4f8659 || exit 1; \
	    echo 'lxc.autodev = 0' >> etc/lxc.config.in; \
	    echo 'lxc.cgroup.devices.allow = c 108:0 rwm' >> etc/lxc.config.in; \
	    echo 'lxc.cgroup.devices.allow = b 7:0 rwm' >> etc/lxc.config.in; \
	    echo 'lxc.cgroup.devices.allow = c 10:200 rwm' >> etc/lxc.config.in; \
	    ln -s '$(GUEST_CACHE_DIR)' inputs; \
	    env LXC_BRIDGE=lxcbr0 LXC_GUEST_IP=10.0.3.2 GITIAN_HOST_IP=10.0.3.1 \
	        bin/make-base-vm --lxc --suite bionic --arch amd64"
	
ifdef VAGRANT_CYCLE
	-$(call VAGRANT_HALT)
endif
	
	echo $(LXC_BASE_MANIFEST) > "$@"

.PHONY: pass-lxc-base
pass-lxc-base: pass-gitian-host
	echo $(LXC_BASE_MANIFEST) > .stamp-lxc-base.manifest

# ===----------------------------------------------------------------------===

ifeq ($(shell git log -1 2>&1 >/dev/null && echo true),true)
SOURCE_FILES := $(shell \
	git ls-tree --name-only --full-tree -r `git log -1 --format="%H"` \
	    | $(GREP) -v '^contrib/vagrant/' \
	    | sort)
else
SOURCE_FILES := $(shell cd ../.. && \
	find -L . -type f -print \
	    | $(SED) 's:^\./::g' \
	    | $(GREP) -v '^\.git/' \
	    | $(GREP) -v '^contrib/vagrant/' \
	    | sort)
endif

SOURCE_MANIFEST := $(shell $(call BUILD_MANIFEST,$(SOURCE_FILES)))

_ := $(shell $(call \
	VALIDATE_MANIFEST, \
	$(HOST_CACHE_DIR)/src-local.manifest, \
	$(SOURCE_MANIFEST) \
))

.PHONY: bundle-source
bundle-source: $(HOST_CACHE_DIR)/src-local.manifest
$(HOST_CACHE_DIR)/src-local.manifest: $(HOST_CACHE_DIR)/src-local.tar.xz
	@echo $(SOURCE_MANIFEST) >'$@'
$(HOST_CACHE_DIR)/src-local.tar.xz: \
	$(foreach FILE,$(SOURCE_FILES),$(shell echo $(FILE) | $(SED) 's:^:../../:g'))
	
	mkdir -p '$(HOST_CACHE_DIR)'
	rm -f '$(HOST_CACHE_DIR)'/src-local.{tar,tar.xz}
	if git log -1 2>&1 >> /dev/null && [ -z "$(shell git status --untracked-files=no --porcelain)" ]; then \
	    pushd ../../; \
	    git archive HEAD --format tar --output contrib/vagrant/'$(HOST_CACHE_DIR)'/src-local.tar; \
	    popd; \
	else \
	    COPYFILE_DISABLE=1 tar -C ../.. -rf '$(HOST_CACHE_DIR)'/src-local.tar $(SOURCE_FILES); \
	fi
	xz '$(HOST_CACHE_DIR)'/src-local.tar
	echo "Gitian Builder" > '$(HOST_CACHE_DIR)'/src-local.committer-name
	echo "vagrant@vagrant" > '$(HOST_CACHE_DIR)'/src-local.committer-email
	echo "Gitian Builder <vagrant@vagrant>" > '$(HOST_CACHE_DIR)'/src-local.author
	echo `date` > '$(HOST_CACHE_DIR)'/src-local.committer-date
	cp '$(HOST_CACHE_DIR)'/src-local.committer-date '$(HOST_CACHE_DIR)'/src-local.author-date
	if git log -1 2>&1 >> /dev/null; then \
	    if [ -z "$(shell git status --untracked-files=no --porcelain)" ]; then \
	        git show --format=format:"%cn" | head -n 1 | tee '$(HOST_CACHE_DIR)'/src-local.committer-name; \
	        git show --format=format:"%ce" | head -n 1 | tee '$(HOST_CACHE_DIR)'/src-local.committer-email; \
	        git show --format=format:"%cd" | head -n 1 | tee '$(HOST_CACHE_DIR)'/src-local.committer-date; \
	        git show --format=format:"%an, <%ae>" | head -n 1 | tee '$(HOST_CACHE_DIR)'/src-local.author; \
	        git show --format=format:"%ad" | head -n 1 | tee '$(HOST_CACHE_DIR)'/src-local.author-date; \
	    else \
	        echo `git config user.name`, '<'`git config user.email`'>' > '$(HOST_CACHE_DIR)'/src-local.author; \
	    fi; \
	fi

# ===----------------------------------------------------------------------===

rightparen=)
CLIENT_VERSION_MAJOR := $(shell $(GREP) _CLIENT_VERSION_MAJOR ../../configure.ac | head -n 1 | $(AWK) -F'[ $(rightparen)]' '{print $$2}')
CLIENT_VERSION_MINOR := $(shell $(GREP) _CLIENT_VERSION_MINOR ../../configure.ac | head -n 1 | $(AWK) -F'[ $(rightparen)]' '{print $$2}')
CLIENT_VERSION_REVISION := $(shell $(GREP) _CLIENT_VERSION_REVISION ../../configure.ac | head -n 1 | $(AWK) -F'[ $(rightparen)]' '{print $$2}')
ifeq ($(CLIENT_VERSION_REVISION), 99)
SERIES := $(CLIENT_VERSION_MAJOR).$(shell expr 1 + $(CLIENT_VERSION_MINOR))
VERSION := $(SERIES)
else
SERIES := $(CLIENT_VERSION_MAJOR).$(CLIENT_VERSION_MINOR)
VERSION := $(SERIES).$(CLIENT_VERSION_REVISION)
endif

OUTPUT_DIR         := output
HOST_OUTPUT_DIR    := $(OUTPUT_DIR)
GUEST_OUTPUT_DIR   := /vagrant/$(OUTPUT_DIR)

TARGETS            :=

PLATFORM           := linux
TARGET             := $(PLATFORM)
$(TARGET)_NAME     := $(TARGET)-$(VERSION)
$(TARGET)_VERSION  := $(VERSION)
$(TARGET)_OUTPUTS  := $(PROJECT)-$($(TARGET)_VERSION)-linux32.tar.gz \
                      $(PROJECT)-$($(TARGET)_VERSION)-linux64.tar.gz
$(TARGET)_SCRIPT   := contrib/gitian-descriptors/gitian-$(TARGET).yml
$(TARGET)_PKGS     :=
$(TARGET)_DEPS     := .stamp-download-depends.h
$(TARGET)_FILES    := $(SOURCE_FILES)
$(TARGET)_FILES    := $(sort $($(TARGET)_FILES))
$(TARGET)_MANIFEST := $(call BUILD_MANIFEST,$($(TARGET)_FILES))
TARGETS            += $(TARGET)

PLATFORM           := osx
TARGET             := $(PLATFORM)
$(TARGET)_NAME     := $(TARGET)-$(VERSION)
$(TARGET)_VERSION  := $(VERSION)
$(TARGET)_OUTPUTS  := $(PROJECT)-$($(TARGET)_VERSION)-osx64.tar.gz
$(TARGET)_SCRIPT   := contrib/gitian-descriptors/gitian-$(TARGET).yml
$(TARGET)_PKGS     := MacOSX10.9.sdk.tar.gz
$(TARGET)_DEPS     := .stamp-download-depends.h
$(TARGET)_FILES    := $(SOURCE_FILES)
$(TARGET)_FILES    := $(sort $($(TARGET)_FILES))
$(TARGET)_MANIFEST := $(call BUILD_MANIFEST,$($(TARGET)_FILES))
TARGETS            += $(TARGET)

PLATFORM           := win
TARGET             := $(PLATFORM)
$(TARGET)_NAME     := $(TARGET)-$(VERSION)
$(TARGET)_VERSION  := $(VERSION)
$(TARGET)_OUTPUTS  := $(PROJECT)-$($(TARGET)_VERSION)-win64.zip \
                      $(PROJECT)-$($(TARGET)_VERSION)-win32.zip
$(TARGET)_SCRIPT   := contrib/gitian-descriptors/gitian-$(TARGET).yml
$(TARGET)_PKGS     :=
$(TARGET)_DEPS     := .stamp-download-depends.h
$(TARGET)_FILES    := $(SOURCE_FILES)
$(TARGET)_FILES    := $(sort $($(TARGET)_FILES))
$(TARGET)_MANIFEST := $(call BUILD_MANIFEST,$($(TARGET)_FILES))
TARGETS            += $(TARGET)

DEPLOY_SOURCE_BUNDLE = \
	vagrant ssh --command "\
	    rm -rf ~/'$(1)'; \
	    mkdir -p ~/'$(1)' || exit 1; \
	    cd ~/'$(1)' || exit 1; \
	    tar -xf $(GUEST_CACHE_DIR)/src-local.tar.xz || exit 1; \
	    git init || exit 1; \
	    git config user.name 'Gitian Builder' || exit 1; \
	    git config user.email 'vagrant@vagrant' || exit 1; \
	    find . -not \\( -path ./.git -prune \\) -type f -print0 | xargs -0 git add -f || exit 1; \
	    env GIT_COMMITTER_NAME='`cat '$(HOST_CACHE_DIR)'/src-local.committer-name`' \
	        GIT_COMMITTER_EMAIL='`cat '$(HOST_CACHE_DIR)'/src-local.committer-email`' \
	        GIT_COMMITTER_DATE='`cat '$(HOST_CACHE_DIR)'/src-local.committer-date`' \
	    git commit --author='`cat '$(HOST_CACHE_DIR)'/src-local.author`' \
	               --date='`cat '$(HOST_CACHE_DIR)'/src-local.author-date`' \
	               -m tip"

.PHONY: deploy-source
deploy-source: $(HOST_CACHE_DIR)/src-local.manifest
	$(call VAGRANT_UP)
	$(call DEPLOY_SOURCE_BUNDLE,$(PROJECT))
ifdef VAGRANT_CYCLE
	-$(call VAGRANT_HALT)
endif

define OUTPUT_RULE
$(HOST_CACHE_DIR)/$(1): $(HOST_OUTPUT_DIR)/$(1)
	cp '$(HOST_OUTPUT_DIR)/$(1)' '$(HOST_CACHE_DIR)/$(1)'
$(HOST_OUTPUT_DIR)/$(1): .stamp-$(2).manifest
endef

define TARGET_RULE
.PHONY: $(1)
$(1): $(foreach OUTPUT,$($(1)_OUTPUTS),$(HOST_OUTPUT_DIR)/$(OUTPUT)) .stamp-$(1).manifest
$(foreach OUTPUT,$($(1)_OUTPUTS),$(eval $(call OUTPUT_RULE,$(OUTPUT),$(1))))
.stamp-$(1).manifest: \
	.stamp-lxc-base.manifest \
	$(HOST_CACHE_DIR)/src-local.manifest \
	$(foreach TARGET,$($(1)_DEPS),$(TARGET)) \
	$(foreach PACKAGE,$($(1)_PKGS),$(HOST_CACHE_DIR)/$(PACKAGE))
	
	mkdir -p $(HOST_OUTPUT_DIR)
	
	$(call VAGRANT_UP)
	
	vagrant ssh --command "rm -rf ~/'$(PROJECT)'"
	vagrant ssh --command "rm -rf ~/gitian-builder/inputs/'$(PROJECT)'"
	$(call DEPLOY_SOURCE_BUNDLE,$(PROJECT))
	
	echo `$($(1)_MANIFEST)` > .$(1).manifest
	
	if ! cmp --silent '.$(1).manifest' '.stamp-$(1).manifest'; then \
	    vagrant ssh --command "\
	        rm -f ~/gitian-builder/cache; \
	        mkdir -p '$(GUEST_CACHE_DIR)/gitian/cache'; \
	        mkdir -p '$(GUEST_CACHE_DIR)/$(1)/`cat .$(1).manifest`/'; \
	        cd ~/gitian-builder; \
	        ln -s '$(GUEST_CACHE_DIR)/gitian/cache' cache || exit 1; \
	        env LXC_BRIDGE=lxcbr0 LXC_GUEST_IP=10.0.3.2 GITIAN_HOST_IP=10.0.3.1 USE_LXC=1 \
	            bin/gbuild ../$(PROJECT)/$($(1)_SCRIPT) --memory 2000 \
	                       --url $(PROJECT)=~/$(PROJECT) \
	                       --commit $(PROJECT)=master" || exit 1; \
	    $(foreach OUTPUT,$($(1)_OUTPUTS),vagrant ssh --command "cd ~/gitian-builder && \
	        mv 'build/out/$(OUTPUT)' \
	           '$(GUEST_CACHE_DIR)/$(1)/`cat .$(1).manifest`/'" || exit 1;) \
	    $(if $(or $($(1)_CLOSURE),$($(1)_DISTFILE)),vagrant ssh --command " \
	        cd ~/gitian-builder/build/out || exit 1; \
	        echo \`basename src/*.tar.gz | sed -e s:.tar.gz::\` >../basename || exit 1; \
	        tar -xf src/*.tar.gz \`cat ../basename\`/src/clientversion.h \
	                             \`cat ../basename\`/src/version.h \
	                             \`cat ../basename\`/src/version.cpp 2>&1 >/dev/null || exit 1; \
	        pushd \`cat ../basename\`/src; \
	        echo '#include <string>'             >main.cpp && \
	        echo '#include <iostream>'          >>main.cpp && \
	        echo '#include \"version.h\"'       >>main.cpp && \
	        echo 'int main(int, char**) {'      >>main.cpp && \
	        echo '  std::cout << CLIENT_BUILD;' >>main.cpp && \
	        echo '  return 0;'                  >>main.cpp && \
	        echo '}'                            >>main.cpp; \
	        c++ version.cpp main.cpp 2>&1 >/dev/null || exit 1; \
	        ./a.out | tee '$(GUEST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`/version \
	                      >../../../version || exit 1; \
	        popd; \
	        rm -rf \`cat ../basename\`" || exit 1;) \
	    $(if $($(1)_CLOSURE),vagrant ssh --command "cd ~/gitian-builder/build/out || exit 1; \
	        $($(1)_CLOSURE)" || exit 1;) \
	    $(if $($(1)_DISTFILE),vagrant ssh --command "cd ~/gitian-builder/build/out && \
	        mv \"$($(1)_DISTFILE)\" \
	           '$(GUEST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`/" || exit 1;) \
	    vagrant ssh --command "cd ~/gitian-builder && \
	        mv result/*-res.yml \
	           '$(GUEST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$($(1)_NAME)-gitian-res.yml' || exit 1; \
	        mv var/build.log \
	           '$(GUEST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$($(1)_NAME)-gitian-build.log' || exit 1; \
	        mv var/install.log \
	           '$(GUEST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$($(1)_NAME)-gitian-install.log'" || exit 1; \
	fi
	
	$(foreach OUTPUT,$($(1)_OUTPUTS),cp '$(HOST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$(OUTPUT)' '$(HOST_OUTPUT_DIR)/';)
	$(if $($(1)_DISTFILE),vagrant ssh --command "mkdir -p ~/gitian-builder/build/out && \
	    cd ~/gitian-builder/build/out && \
	    cp \"$(GUEST_CACHE_DIR)/$(1)/`cat .$(1).manifest`/version\" .. && \
	    cp \"$(GUEST_CACHE_DIR)/$(1)/`cat .$(1).manifest`/$($(1)_DISTFILE)\" \
	       '$(GUEST_OUTPUT_DIR)'")
	cp '$(HOST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$($(1)_NAME)-gitian-res.yml' '$(HOST_OUTPUT_DIR)/'
	cp '$(HOST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$($(1)_NAME)-gitian-build.log' '$(HOST_OUTPUT_DIR)/'
	cp '$(HOST_CACHE_DIR)/$(1)/'`cat .$(1).manifest`'/$($(1)_NAME)-gitian-install.log' '$(HOST_OUTPUT_DIR)/'
	
	mv .$(1).manifest .stamp-$(1).manifest
	touch .stamp-$(1).manifest
	
	vagrant ssh --command "rm -rf ~/'$(PROJECT)'"
	vagrant ssh --command "rm -rf ~/gitian-builder/inputs/'$(PROJECT)'"
ifdef VAGRANT_CYCLE
	-$(call VAGRANT_HALT)
endif

endef
$(foreach TARGET,$(TARGETS),$(eval $(call TARGET_RULE,$(TARGET))))

#
# End of File
#