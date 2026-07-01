FROM emscripten/emsdk:5.0.7 AS builder

ENV DEBIAN_FRONTEND=noninteractive

ARG VERSION=15.0.0
ENV VERSION=${VERSION}
ARG SOURCE_SHA256=06193df2f54cf0a505e6a2d154ed5df64d2e9ced50f8bf6e454049c1a59c8fd6

WORKDIR /src

ARG EXPAT_VERSION=2.8.1
ARG EXPAT_SHA256=a52eb72108be160e190b5cafa5bba8663f1313f2013e26060d1c18e26e31067b
ADD --checksum=sha256:${EXPAT_SHA256} \
    https://github.com/libexpat/libexpat/releases/download/R_2_8_1/expat-${EXPAT_VERSION}.tar.gz /tmp/expat.tgz
RUN set -eux; \
    cd /tmp; tar xf expat.tgz; cd "expat-${EXPAT_VERSION}"; \
    emcmake cmake -B build -DCMAKE_INSTALL_PREFIX=/opt -DCMAKE_BUILD_TYPE=Release \
      -DBUILD_SHARED_LIBS=OFF -DEXPAT_BUILD_TOOLS=OFF -DEXPAT_BUILD_EXAMPLES=OFF \
      -DEXPAT_BUILD_TESTS=OFF -DEXPAT_BUILD_DOCS=OFF -DEXPAT_SHARED_LIBS=OFF; \
    cmake --build build --target install -j"$(nproc)"; \
    [ -f /opt/lib/libexpat.a ] && [ -f /opt/include/expat.h ]; \
    rm -rf /tmp/expat*

ADD --checksum=sha256:${SOURCE_SHA256} \
    https://gitlab.com/api/v4/projects/4207231/packages/generic/graphviz-releases/${VERSION}/graphviz-${VERSION}.tar.gz \
    graphviz.tar.gz
RUN tar -xzf graphviz.tar.gz && rm graphviz.tar.gz

WORKDIR /src/graphviz-${VERSION}

COPY js ./js

RUN set -eux; \
    PKG_CONFIG_PATH=/opt/lib/pkgconfig \
    emconfigure ./configure \
      --disable-shared --enable-static --disable-ltdl \
      --with-expat --without-zlib \
      --without-pangocairo --without-libgd --without-webp \
      --without-poppler --without-rsvg --without-ghostscript --without-lasi \
      --without-gts --without-ann --without-glut --without-freetype2 \
      --without-x --without-gtk --without-gdk --without-gdk-pixbuf \
      --without-gtkgl --without-gtkglext --without-glade --without-qt \
      --without-quartz --without-devil --with-ortho=no --without-smyrna \
      --disable-swig --disable-sharp --disable-go --disable-guile \
      --disable-java --disable-javascript --disable-lua --disable-perl \
      --disable-php --disable-python --disable-python3 --disable-r \
      --disable-ruby --disable-tcl --disable-man-pdfs \
      CPPFLAGS="-I/opt/include" \
      LDFLAGS="-L/opt/lib" \
      CFLAGS="-O2 -fexceptions -I/opt/include -Wno-error=implicit-function-declaration -Wno-error=incompatible-pointer-types -Wno-error=int-conversion -Wno-error=implicit-int" \
      CXXFLAGS="-O2 -fexceptions -I/opt/include"; \
    grep -q "define HAVE_EXPAT 1" config.h \
      || { echo "FATAL: expat not detected by configure (HAVE_EXPAT unset) — HTML-like labels would silently degrade" >&2; exit 1; }

RUN set -eux; \
    make -j"$(nproc)" V=1 || echo "top-level make returned nonzero (tolerated; relinking objects manually)"; \
    DOT_O="$(find cmd/dot -name 'dot_static-dot.o' -o -name '*dot_static*dot.o' | head -1)"; \
    [ -n "$DOT_O" ] || DOT_O="$(find cmd/dot -name '*-dot.o' | head -1)"; \
    BUILTINS_O="$(find cmd/dot -name '*dot_builtins.o' | head -1)"; \
    [ -n "$DOT_O" ]      || { echo "FATAL: dot.o not built — real compile failure" >&2; exit 1; }; \
    [ -n "$BUILTINS_O" ] || { echo "FATAL: dot_builtins.o not built — real compile failure" >&2; exit 1; }; \
    echo "dot object:      $DOT_O"; \
    echo "builtins object: $BUILTINS_O"

RUN set -eux; \
    DOT_O="$(find cmd/dot -name '*-dot.o' ! -name '*builtins*' | head -1)"; \
    BUILTINS_O="$(find cmd/dot -name '*dot_builtins.o' | head -1)"; \
    [ -n "$DOT_O" ] && [ -n "$BUILTINS_O" ]; \
    PLUGIN_LIBS=""; \
    for p in dot_layout neato_layout core vt kitty; do \
      a="$(find plugin/$p -name "libgvplugin_${p}_C.a" | head -1)"; \
      [ -n "$a" ] || a="$(find plugin/$p -name "libgvplugin_${p}.a" | head -1)"; \
      [ -n "$a" ] || { echo "FATAL: plugin archive for $p not found" >&2; exit 1; }; \
      PLUGIN_LIBS="$PLUGIN_LIBS $a"; \
    done; \
    CORE_LIBS=""; \
    for l in gvc pathplan xdot cgraph cdt util common label sparse pack \
             dotgen neatogen twopigen circogen fdpgen sfdpgen patchwork osage \
             rbtree ortho vpsc; do \
      a="$(find lib/$l -name "lib${l}_C.a" 2>/dev/null | head -1)"; \
      [ -n "$a" ] || a="$(find lib/$l -name "lib${l}.a" 2>/dev/null | head -1)"; \
      [ -n "$a" ] && CORE_LIBS="$CORE_LIBS $a"; \
    done; \
    echo "PLUGIN_LIBS:$PLUGIN_LIBS"; echo "CORE_LIBS:$CORE_LIBS"; \
    mkdir -p /dist; \
    emcc -O0 -fexceptions \
      "$DOT_O" "$BUILTINS_O" \
      $PLUGIN_LIBS $CORE_LIBS /opt/lib/libexpat.a \
      -o /dist/graphviz.mjs \
      -sMODULARIZE=1 -sEXPORT_ES6=1 -sEXIT_RUNTIME=0 -sINVOKE_RUN=0 \
      -sFORCE_FILESYSTEM=1 \
      -sALLOW_MEMORY_GROWTH=1 -sINITIAL_MEMORY=33554432 -sMAXIMUM_MEMORY=1073741824 \
      -sSTACK_SIZE=8388608 \
      -sERROR_ON_UNDEFINED_SYMBOLS=0 \
      -sALLOW_TABLE_GROWTH=1 \
      -sEXPORTED_RUNTIME_METHODS=callMain,FS,ENV \
      -sEXPORTED_FUNCTIONS=_main,_malloc,_free \
      --post-js /src/graphviz-${VERSION}/js/post.js; \
    ls -lh /dist

RUN set -eux; \
    [ "$(head -c4 /dist/graphviz.wasm | od -An -tx1 | tr -d ' \n')" = "0061736d" ] \
      || { echo "FATAL: graphviz.wasm has no wasm magic" >&2; exit 1; }; \
    [ "$(stat -c%s /dist/graphviz.wasm)" -gt 1048576 ] \
      || { echo "FATAL: graphviz.wasm implausibly small" >&2; exit 1; }; \
    grep -q "export default" /dist/graphviz.mjs \
      || { echo "FATAL: graphviz.mjs is not an ES6 module" >&2; exit 1; }

COPY test ./test
RUN set -eux; \
    cp /dist/graphviz.mjs /dist/graphviz.wasm test/; \
    cd test; node smoke.mjs; rm -f graphviz.mjs graphviz.wasm

RUN set -eux; \
    printf '%s\n' "${VERSION}" > /dist/GRAPHVIZ_VERSION; \
    echo "=== dist ==="; ls -lh /dist

FROM scratch AS exportor
COPY --from=builder /dist /dist
