SUBMODULES =					\
	external/libsodium			\
	external/libwally-core			\
	external/gheap				\
	external/jsmn				\
	external/libbacktrace			\
	external/lnprototest

TOP := ../..
ifdef BUILD
CROSSCOMPILE_OPTS := --host="$(MAKE_HOST)" --build="$(BUILD)"
TARGET_DIR := external/$(MAKE_HOST)
else
TARGET_DIR := external/$(shell ${CC} -dumpmachine)
endif

LIBSODIUM_HEADERS := external/libsodium/src/libsodium/include/sodium.h
LIBWALLY_HEADERS := external/libwally-core/include/wally_bip32.h		\
			external/libwally-core/include/wally_core.h	\
			external/libwally-core/include/wally_psbt.h	\
			external/libwally-core/include/wally_crypto.h
LIBSECP_HEADERS := external/libwally-core/src/secp256k1/include/secp256k1_ecdh.h		\
		external/libwally-core/src/secp256k1/include/secp256k1_recovery.h		\
		external/libwally-core/src/secp256k1/include/secp256k1.h
JSMN_HEADERS := external/jsmn/jsmn.h
GHEAP_HEADERS := external/gheap/gheap.h
LIBBACKTRACE_HEADERS := external/libbacktrace/backtrace.h

EXTERNAL_HEADERS := $(LIBSODIUM_HEADERS) $(LIBWALLY_HEADERS) $(LIBSECP_HEADERS) $(JSMN_HEADERS) $(GHEAP_HEADERS) $(LIBBACKTRACE_HEADERS)
EXTERNAL_LIBS := ${TARGET_DIR}/libwallycore.a ${TARGET_DIR}/libsecp256k1.a ${TARGET_DIR}/libjsmn.a ${TARGET_DIR}/libbacktrace.a

EXTERNAL_INCLUDE_FLAGS :=					\
	-I external/libwally-core/include/			\
	-I external/libwally-core/src/secp256k1/include/	\
	-I external/jsmn/					\
	-I external/libbacktrace/				\
	-I external/gheap/					\
	-I ${TARGET_DIR}/libbacktrace-build

ifneq ($(HAVE_GOOD_LIBSODIUM),1)
EXTERNAL_INCLUDE_FLAGS += -I external/libsodium/src/libsodium/include \
			  -I external/libsodium/src/libsodium/include/sodium \
			  -I $(TARGET_DIR)/libsodium-build/src/libsodium/include
EXTERNAL_LIBS += ${TARGET_DIR}/libsodium.a
else
LDLIBS += -lsodium
endif

EXTERNAL_LDLIBS := -L${TARGET_DIR} $(patsubst lib%.a,-l%,$(notdir $(EXTERNAL_LIBS)))

submodcheck: $(FORCE)
ifneq ($(VERSION),)
	@tools/refresh-submodules.sh $(SUBMODULES)
	@cd external/libwally-core && ../../tools/refresh-submodules.sh src/secp256k1
endif

$(EXTERNAL_HEADERS): submodcheck

# We build libsodium, since Ubuntu xenial has one too old.
$(TARGET_DIR)/libsodium.a: $(TARGET_DIR)/libsodium-build/src/libsodium/libsodium.la
	$(MAKE) -C $(TARGET_DIR)/libsodium-build DESTDIR=$$(pwd)/$(TARGET_DIR) install-exec

$(TARGET_DIR)/libsodium-build/src/libsodium/libsodium.la: external/libsodium/src/libsodium/include/sodium.h
	cd external/libsodium && ./autogen.sh
	mkdir -p ${TARGET_DIR}/libsodium-build
	cd $(TARGET_DIR)/libsodium-build && $(TOP)/libsodium/configure CC="$(CC)" --enable-static=yes $(CROSSCOMPILE_OPTS) --enable-shared=no --prefix=/ --libdir=/ && $(MAKE)

# libsecp included in libwally.
# Wildcards here are magic.  See http://stackoverflow.com/questions/2973445/gnu-makefile-rule-generating-a-few-targets-from-a-single-source-file
$(TARGET_DIR)/libsecp256k1.% $(TARGET_DIR)/libwallycore.%: $(TARGET_DIR)/libwally-core-build/src/secp256k1/libsecp256k1.la $(TARGET_DIR)/libwally-core-build/src/libwallycore.la
	$(MAKE) -C $(TARGET_DIR)/libwally-core-build DESTDIR=$$(pwd)/$(TARGET_DIR) install-exec

# Build libwally-core.
$(TARGET_DIR)/libwally-core-build/src/libwallycore.% $(TARGET_DIR)/libwally-core-build/src/secp256k1/libsecp256k1.%: $(LIBWALLY_HEADERS) $(LIBSECP_HEADERS)
	cd external/libwally-core && ./tools/autogen.sh
	mkdir -p ${TARGET_DIR}/libwally-core-build
	cd ${TARGET_DIR}/libwally-core-build \
	&& PYTHON_VERSION=3 CFLAGS=-std=c99 ${TOP}/libwally-core/configure CC="$(CC)" \
		--enable-static=yes \
		$(CROSSCOMPILE_OPTS) \
		--enable-module-recovery \
		--enable-module-extrakeys \
		--enable-module-schnorrsig \
		--enable-elements \
		--enable-shared=no \
		--prefix=/ \
		--libdir=/ \
		--enable-debug \
	&& $(MAKE)

# If we tell Make that the above builds both, it runs it twice in
# parallel.  So we lie :(
external/jsmn/jsmn.c: external/jsmn/jsmn.h
	[ -f $@ ]

$(TARGET_DIR)/jsmn-build/jsmn.o: external/jsmn/jsmn.c Makefile
	@mkdir -p $(@D)
	$(COMPILE.c) -DJSMN_STRICT=1 -o $@ $<

$(TARGET_DIR)/libjsmn.a: $(TARGET_DIR)/jsmn-build/jsmn.o
	$(AR) rc $@ $<

# Need separate build dir: changes inside submodule make git think it's dirty.
$(TARGET_DIR)/libbacktrace.a: external/libbacktrace/backtrace.h
	mkdir -p $(TARGET_DIR)/libbacktrace-build
	cd $(TARGET_DIR)/libbacktrace-build && $(TOP)/libbacktrace/configure CC="$(CC)" --enable-static=yes $(CROSSCOMPILE_OPTS) --enable-shared=no --prefix=/ --libdir=/ && $(MAKE)
	$(MAKE) -C $(TARGET_DIR)/libbacktrace-build DESTDIR=$$(pwd)/$(TARGET_DIR) install-exec

distclean: external-distclean
clean: external-clean

external-clean:
	$(RM) $(EXTERNAL_LIBS) $(TARGET_DIR)/*.la $(TARGET_DIR)/*.o
	$(RM) $(TARGET_DIR)/jsmn-build/jsmn.o
	if [ -f ${TARGET_DIR}/libsodium-build/Makefile ]; then make -C ${TARGET_DIR}/libsodium-build clean; fi
	if [ -f ${TARGET_DIR}/libwally-core-build/Makefile ]; then make -C ${TARGET_DIR}/libwally-core-build clean; fi
	if [ -f ${TARGET_DIR}/libwally-core-build/src/Makefile ]; then make -C ${TARGET_DIR}/libwally-core-build/src clean; fi
	if [ -f ${TARGET_DIR}/libbacktrace-build/Makefile ]; then make -C ${TARGET_DIR}/libbacktrace-build clean; fi

external-distclean:
	make -C external/libsodium distclean || true
	$(RM) -rf ${TARGET_DIR}/libbacktrace-build ${TARGET_DIR}/libsodium-build ${TARGET_DIR}/libwally-core-build ${TARGET_DIR}/jsmn-build
	$(RM) -r `git status --ignored --porcelain external/libwally-core | grep '^!! ' | cut -c3-`
