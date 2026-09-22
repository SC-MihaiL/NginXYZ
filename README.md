<img src="./NginXYZ.png" align="left" width="150" hspace="15">

# NginXYZ

**A custom, high-performance native nginx build for Windows 10/11** — statically linked, aggressively optimized, and bundled with a large stack of popular third-party modules (LuaJIT/OpenResty ecosystem, njs, Brotli, Zstd, GeoIP2, and more) that normally require piecing together a build yourself.

<br clear="left"/>

> Forked from [nginx/nginx](https://github.com/nginx/nginx). Built with MSYS2/MinGW-w64.

[![Project Status: Active](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)
[![License](https://img.shields.io/badge/License-BSD%202--Clause-blue.svg)](./LICENSE)

*(Русская версия ридми: [README.ru.md](./README.ru.md))*

---

## Why NginXYZ?

Official nginx for Windows is explicitly a proof-of-concept build with a minimal module set. Getting a "real" nginx — with Lua scripting, njs, modern compression, GeoIP, and the rest of the OpenResty-adjacent ecosystem — working natively on Windows normally means compiling everything yourself against MinGW, which is fiddly and full of platform-specific gotchas (icons, `zlib-ng` quirks, OpenSSL cross-compilation, etc).

NginXYZ does that work for you: one `build.sh` script that configures and statically links nginx with all of the below into a single `nginx.exe`.

## Features

- **Native Windows binary** — statically linked, no external DLL dependencies for the bundled libraries.
- **LuaJIT (FFI) scripting** — full OpenResty-style Lua module stack (`lua-nginx-module`, `stream-lua-nginx-module`, `lua-upstream-nginx-module`, and friends).
- **Modular NJS** — nginx's native JavaScript scripting module.
- **Modern compression** — Brotli (`ngx_brotli`) and Zstandard (`zstd-nginx-module`) in addition to gzip.
- **GeoIP2** support via `ngx_http_geoip2_module` + libmaxminddb.
- **Image processing** — `http_image_filter_module` with GD, WebP, and imagequant.
- **Observability** — live status via `nginx-module-vts`.
- **Caching & proxying extras** — `srcache-nginx-module`, `memc`/`redis`/`redis2` modules, `ngx_cache_purge`, `ngx_http_proxy_connect_module`.
- **Streaming** — `nginx-http-flv-module`, `http_mp4_module`, `http_flv_module`.
- **Optimized for the target CPU** — compiled with `-O3 -march=native -flto`, PIE, stack protector, and TFO enabled.

## Bundled third-party modules

<details>
<summary>Full list (click to expand)</summary>

**Lua / OpenResty stack**
`ngx_devel_kit`, `lua-nginx-module`, `stream-lua-nginx-module`, `lua-upstream-nginx-module`, `set-misc-nginx-module`, `array-var-nginx-module`, `form-input-nginx-module`, `encrypted-session-nginx-module`, `iconv-nginx-module`, `echo-nginx-module`, `xss-nginx-module`, `srcache-nginx-module`

**Caching / upstream backends**
`memc-nginx-module`, `redis-nginx-module`, `redis2-nginx-module`, `rds-json-nginx-module`, `rds-csv-nginx-module`, `ngx_cache_purge`, `nginx_upstream_module`

**Scripting & compression**
`njs` (nginx JavaScript), `ngx_brotli`, `zstd-nginx-module`

**Misc / ops**
`nginx-module-vts`, `ngx-fancyindex`, `headers-more-nginx-module`, `testcookie-nginx-module`, `nginx-http-flv-module`, `ngx_http_proxy_connect_module`, `nginx-dav-ext-module`, `ngx_http_geoip2_module`

**Core nginx flags enabled**
`http_ssl`, `http_v2`, `http_realip`, `http_addition`, `http_xslt`, `http_image_filter`, `http_geoip`, `http_sub`, `http_dav`, `http_flv`, `http_mp4`, `http_gunzip`, `http_gzip_static`, `http_auth_request`, `http_random_index`, `http_secure_link`, `http_slice`, `http_stub_status`, `http_json`, `mail`, `stream`, `stream_ssl`, `stream_realip`, `stream_geoip`, `stream_ssl_preread`, `file-aio`

**Statically linked libraries**
LuaJIT, OpenSSL, zlib-ng, PCRE2 (JIT-enabled), libxml2/libxslt/libexslt, GD (+ libwebp, libpng, libjpeg, imagequant, freetype, harfbuzz), libmaxminddb, yajl, msgpuck, lua-cjson, bzip2, lzma

</details>

## Requirements

- Windows 10/11, x86-64 CPU with **SSE4.2 or newer**.
- [MSYS2](https://www.msys2.org/) with the MinGW-w64 toolchain for building from source.
- The dependency repositories referenced by `build.sh` cloned as sibling directories (`nginx`, `openssl`, `zlib-ng`, `pcre2`, `luajit2`, and one directory per `--add-module=../...` entry). See `build.sh` for the exact expected paths.

> **Note on portability:** the default build uses `-march=native`, so a binary built on one machine is **not guaranteed to run on another CPU** — if the target CPU lacks an instruction set the build host has, you'll get an illegal instruction crash. If you're redistributing binaries rather than building locally, replace `-march=native` with a more conservative target (e.g. `-march=x86-64-v2`, which covers SSE4.2/POPCNT-class CPUs).

## Building

```bash
# from inside MSYS2 MinGW64 shell, with all sibling dependency repos in place
./build.sh

# optional flags:
./build.sh --with-luajit-opt="..."   # pass extra options through to LuaJIT's build.sh
./build.sh --force-clean-openssl     # force a full OpenSSL rebuild (normally only
                                      # done automatically when the gcc version changes)
```

The script:
1. Cleans previous nginx build artifacts, and conditionally cleans OpenSSL (only on compiler-version change, or when forced — incremental OpenSSL rebuilds otherwise, which is much faster).
2. Patches `zlib-ng` with the Makefile layout nginx's configure script expects on Windows.
3. Builds LuaJIT if it hasn't been built yet.
4. Compiles the Windows resource icon.
5. Runs `auto/configure` with the full module list and static linking flags, then `make -j$(nproc)`.

The resulting binary is placed at `objs/nginx.exe`.

## ⚠️ Security note: TLS compatibility mode

This build currently configures OpenSSL with **`OPENSSL_TLS_SECURITY_LEVEL=0`** and enables legacy/weak options (`enable-weak-ssl-ciphers`, `enable-tls-deprecated-ec`, `enable-md2`, `enable-rc5`) for compatibility with old clients. **This intentionally weakens the TLS stack compared to a standard nginx/OpenSSL build.**

If you don't specifically need legacy-client compatibility, do not use this build for anything security-sensitive without first removing/tightening these OpenSSL configure options. A future goal is to make this an opt-in build flag rather than the default.

## Bleeding edge

This isn't a "build once, forget it" project. The tree is rebuilt every couple of commits against upstream [nginx/nginx master](https://github.com/nginx/nginx), so fixes and new features from mainline nginx tend to show up here quickly rather than waiting for a stable tag. Expect this repo to track upstream closely — and expect occasional breakage as a result, since third-party modules don't always keep pace with nginx internals changing.

## Status

Actively developed, pre-release / early stage. Expect rough edges — see [Issues](../../issues) for known gaps.

## Credits

Built on top of [nginx](https://nginx.org/), [njs](https://github.com/nginx/njs), the [OpenResty](https://openresty.org/) Lua module ecosystem, and the many third-party nginx module authors listed above.

## License

[2-clause BSD-like license](./LICENSE) (same as upstream nginx).
