graphviz-wasm — Corresponding Source mirror
==================================================

This repository publishes the Corresponding Source for the WebAssembly
build of graphviz (license: EPL-1.0) used in edgetools.io.

Contents
  build/      our build recipe: Dockerfile + helper scripts/config/patches.
              Rebuild with:  docker build build/
  upstream/   the exact upstream source archive(s) the build fetched,
              byte-identical and sha256-verified (see below).

Upstream sources:
  expat.tgz
    https://github.com/libexpat/libexpat/releases/download/R_2_8_1/expat-2.8.1.tar.gz
    sha256 a52eb72108be160e190b5cafa5bba8663f1313f2013e26060d1c18e26e31067b
  graphviz.tar.gz
    https://gitlab.com/api/v4/projects/4207231/packages/generic/graphviz-releases/15.0.0/graphviz-15.0.0.tar.gz
    sha256 06193df2f54cf0a505e6a2d154ed5df64d2e9ced50f8bf6e454049c1a59c8fd6
