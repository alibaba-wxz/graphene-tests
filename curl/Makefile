# Use one of these commands to build the manifest for curl:
#
# - make
# - make DEBUG=1
# - make SGX=1
# - make SGX=1 DEBUG=1
#
# Use `make clean` to remove Graphene-generated files.

THIS_DIR := $(dir $(lastword $(MAKEFILE_LIST)))
CURL_DIR ?= /usr/bin/

# Relative path to Graphene root and key for enclave signing
GRAPHENEDIR ?= ../../../../..
GRAPHENEKEY ?= $(GRAPHENEDIR)/Pal/src/host/Linux-SGX/signer/enclave-key.pem

ifeq ($(DEBUG),1)
GRAPHENEDEBUG = inline
else
GRAPHENEDEBUG = none
endif

.PHONY: all
all: curl.manifest pal_loader
ifeq ($(SGX),1)
all: curl.token
endif

# curl dependencies (generated from ldd). For SGX, the manifest needs to list all the libraries
# loaded during execution, so that the signer can include the file hashes.

# We need to replace Glibc dependencies with Graphene-specific Glibc. The Glibc binaries are
# already listed in the manifest template, so we can skip them from the ldd results.
GLIBC_DEPS = linux-vdso.so.1 /lib64/ld-linux-x86-64.so.2 libc.so.6 libm.so.6 librt.so.1 \
			 libdl.so.2 libpthread.so.0 libutil.so.1 libresolv.so.2 libnss_dns.so.2

# List all the curl dependencies, besides Glibc libraries
.INTERMEDIATE: curl-deps
curl-deps:
	@ldd $(CURL_DIR)curl | \
		awk '{if ($$2 =="=>") {print $$1}}' | \
		sort | uniq | grep -v -x $(patsubst %,-e %,$(GLIBC_DEPS)) > $@

# Generate manifest rules for curl dependencies
.INTERMEDIATE: curl-trusted-libs
curl-trusted-libs: curl-deps
	@for F in `cat curl-deps`; do \
		P=`ldd $(CURL_DIR)curl | grep $$F | awk '{print $$3; exit}'`; \
		N=`echo $$F | tr --delete '.' | tr --delete '-'`; \
		echo -n "sgx.trusted_files.$$N = file:$$P\\\\n"; \
	done > $@

curl.manifest: curl.manifest.template curl-trusted-libs
	@sed -e 's|$$(GRAPHENEDIR)|'"$(GRAPHENEDIR)"'|g' \
		-e 's|$$(GRAPHENEDEBUG)|'"$(GRAPHENEDEBUG)"'|g' \
		-e 's|$$(CURL_DIR)|'"$(CURL_DIR)"'|g' \
		-e 's|$$(CURL_TRUSTED_LIBS)|'"`cat curl-trusted-libs`"'|g' \
		$< > $@

# Generate SGX-specific manifest, enclave signature, and token for enclave initialization
curl.manifest.sgx: curl.manifest
	$(GRAPHENEDIR)/Pal/src/host/Linux-SGX/signer/pal-sgx-sign \
		-libpal $(GRAPHENEDIR)/Runtime/libpal-Linux-SGX.so \
		-key $(GRAPHENEKEY) \
		-manifest $< -output $@

curl.sig: curl.manifest.sgx

curl.token: curl.sig
	$(GRAPHENEDIR)/Pal/src/host/Linux-SGX/signer/pal-sgx-get-token \
		-output $@ -sig $^

# Extra executables
pal_loader:
	ln -s $(GRAPHENEDIR)/Runtime/pal_loader $@

.PHONY: check
check: all
	./pal_loader curl.manifest example.com > OUTPUT
	@grep -q "Example Domain" OUTPUT && echo "[ Success 1/1 ]"
	@rm OUTPUT

.PHONY: clean
clean:
	$(RM) *.manifest *.manifest.sgx *.token *.sig pal_loader OUTPUT

.PHONY: distclean
distclean: clean
