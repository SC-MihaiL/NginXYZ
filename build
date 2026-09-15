#!/bin/bash

# 0. Разбор аргументов
LUAJIT_OPT=""
FORCE_CLEAN_OPENSSL=0

for arg in "$@"; do
    case $arg in
        --with-luajit-opt=*)
            LUAJIT_OPT="${arg#*=}"
            ;;
        --force-clean-openssl)
            FORCE_CLEAN_OPENSSL=1
            ;;
        *)
            echo "Unknown argument: $arg"
            ;;
    esac
done

# 1. Глобальная чистка
echo "--- Cleaning up Nginx and OpenSSL ---"
cd ~/build/nginx
rm -rf objs Makefile
# Чистим OpenSSL, так как quictls больше не используем — НО только когда
# есть основания думать, что инкрементальная сборка небезопасна: сменилась
# версия gcc (тогда старые .obj с несовместимым LTO-байткодом от прошлого
# компилятора останутся на диске и молча всё сломают), либо чистка запрошена
# явно флагом --force-clean-openssl. Если компилятор тот же — просто отдаём
# OpenSSL на инкрементальный make, он пересоберёт только реально изменившееся
# (обычно ничего) и это займёт секунды вместо ~5 минут.
cd ../openssl

GCC_VERSION_MARKER=".build_gcc_version"
CURRENT_GCC_VERSION="$(gcc --version | head -n1)"

NEED_CLEAN=0
if [ "$FORCE_CLEAN_OPENSSL" = "1" ]; then
    NEED_CLEAN=1
    echo "OpenSSL: чистка запрошена явно (--force-clean-openssl)"
elif [ ! -f "$GCC_VERSION_MARKER" ]; then
    NEED_CLEAN=1
    echo "OpenSSL: маркер версии gcc не найден, чистим на всякий случай"
elif [ "$(cat "$GCC_VERSION_MARKER")" != "$CURRENT_GCC_VERSION" ]; then
    NEED_CLEAN=1
    echo "OpenSSL: версия gcc изменилась, чистим"
else
    echo "OpenSSL: версия gcc не менялась ($CURRENT_GCC_VERSION), чистку пропускаем"
fi

if [ "$NEED_CLEAN" = "1" ]; then
    # Чистим по-настоящему (git clean), а не только Makefile -
    # иначе make clean не запускается (его вызов завязан на "if [ -f Makefile ]")
    if [ -d .git ]; then
        git clean -xdf
    else
        echo "WARNING: ../openssl is not a git repo, doing manual cleanup instead"
        rm -rf Makefile configdata.pm makefile .openssl
        find . -name '*.obj' -delete
        find . -name '*.o' -delete
        find . -name '*.a' -delete
        find . -name '*.d.tmp' -delete
    fi
fi

echo "$CURRENT_GCC_VERSION" > "$GCC_VERSION_MARKER"

cd ../nginx

# 2. Фикс для zlib-ng
echo "--- Patching zlib-ng ---"
cd ../zlib-ng

# Создаем структуру, которую ждет Nginx
mkdir -p win32

# Создаем заглушку для ВСЕХ возможных вызовов
echo -e "all:\n\ninstall:\n\nlibz.a:\n\ndistclean:\n\t@echo 'skip distclean'\nclean:\n\t@echo 'skip clean'" > Makefile
# Копируем этот же файл в win32, чтобы флаг -f win32/Makefile.gcc его нашел
cp Makefile win32/Makefile.gcc

cd ../nginx

# 2.5 Проверка и сборка LuaJIT
LUAJIT_LIB_FILE="../luajit2/objs/libluajit.a"

if [ ! -f "$LUAJIT_LIB_FILE" ]; then
    echo "--- LuaJIT library not found, building it ---"
    (cd ../luajit2 && ./build.sh gcc "$LUAJIT_OPT")

    if [ ! -f "$LUAJIT_LIB_FILE" ]; then
        echo "Error: LuaJIT build failed, library still missing at $LUAJIT_LIB_FILE"
        exit 1
    fi
else
    echo "--- LuaJIT library found ($LUAJIT_LIB_FILE), skipping build ---"
fi

# 3. Переменные окружения
export PERL=../../../../usr/bin/perl
export LUAJIT_LIB=../luajit2/objs
export LUAJIT_INC=../luajit2/objs
export LIBS=../zlib-ng/libz.a
export NJS_QUICKJS=NO
export NJS_HAVE_LITTLE_ENDIAN=1
export LIBDRIZZLE_INC=../libdrizzle-redux/build/include
export LIBDRIZZLE_LIB=../libdrizzle-redux/build/src/.libs

# 4. Директории и иконка
echo "--- Making dirs and compiling icon's ---"
mkdir -p objs
cd objs
mkdir -p temp logs pid conf
cd ..

rm -rf ../nginx/src/os/win32/nginxyz_icon.o > /dev/null 2>&1
windres ../nginx/src/os/win32/nginxyz.rc -O \
coff -o ../nginx/src/os/win32/nginxyz_icon.o

rm -rf ../nginx/src/os/win32/nginx_icon.o > /dev/null 2>&1
windres ../nginx/src/os/win32/nginx.rc -O \
coff -o ../nginx/src/os/win32/nginx_icon.o

# 5. Конфигурация
echo "--- Starting configure ---"
./auto/configure \
    --with-cc=gcc \
    --prefix= \
    --sbin-path=nginx.exe \
    --conf-path=conf/nginx.conf \
    --pid-path=pid/nginx.pid \
    --error-log-path=logs/error.log \
    --http-log-path=logs/access.log \
    --http-client-body-temp-path=temp/client_body_temp \
    --http-proxy-temp-path=temp/proxy_temp \
    --http-fastcgi-temp-path=temp/fastcgi_temp \
    --http-uwsgi-temp-path=temp/uwsgi_temp \
    --http-scgi-temp-path=temp/scgi_temp \
    --with-cc-opt="-march=native -O3 -flto=$(nproc) -fstack-protector-strong \
    -D_WIN32_WINNT=0x0A00 -DLIBXML_STATIC -DLIBXSLT_STATIC -DLIBEXSLT_STATIC \
    -DNONDLL -DBGD_WIN32 -DBGDWIN32 -DGD_STATIC -I$LUAJIT_INC -I../zlib-ng -I/mingw64/include \
    -I/mingw64/include/libxml2" \
    --with-ld-opt="-static -flto=$(nproc) -L$LUAJIT_LIB \
    ../nginx_upstream_module/third_party/yajl/libyajl.a \
    ../nginx_upstream_module/third_party/msgpuck/libmsgpuck.a \
    -Wl,--whole-archive ../lua-cjson/liblua_cjson.a -Wl,--no-whole-archive \
    -lmaxminddb -lluajit -lexslt \
    -lgd -limagequant -lwebp -lsharpyuv -lpng -ljpeg \
    -lfreetype -lharfbuzz -lgraphite2 -lbz2 \
    -lbrotlienc -lbrotlidec -lbrotlicommon \
    -lxslt -lxml2 -lexpat -lintl -liconv -llzma -lz \
    -L../openssl/.openssl/lib -lssl -lcrypto \
    -lsecur32 -lshfolder -lbcrypt -lwldap32 -lshlwapi -lws2_32 -luser32 -lruntimeobject -lcrypt32 -lgdi32 -ladvapi32 -lole32 -lwinmm \
    -lntdll -lnetapi32 -lrpcrt4 -ldwrite -lfontconfig -luserenv -luuid -lstdc++ -lmsvcrt \
    -Wl,--export-all-symbols \
    ../zlib-ng/libz.a \
	../nginx/src/os/win32/nginxyz_icon.o" \
    --with-file-aio \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_realip_module \
    --with-http_addition_module \
    --with-http_xslt_module \
    --with-http_image_filter_module \
    --with-http_geoip_module \
    --with-http_sub_module \
    --with-http_dav_module \
    --with-http_flv_module \
    --with-http_mp4_module \
    --with-http_gunzip_module \
    --with-http_gzip_static_module \
    --with-http_auth_request_module \
    --with-http_random_index_module \
    --with-http_secure_link_module \
    --with-http_slice_module \
    --with-http_stub_status_module \
    --with-http_json_module \
    --with-mail \
    --with-stream \
    --with-stream_ssl_module \
    --with-stream_realip_module \
    --with-stream_geoip_module \
    --with-stream_ssl_preread_module \
    --with-pcre=../pcre2 \
    --with-pcre-opt="-O3 -march=native -flto=$(nproc)" \
    --with-pcre-conf-opt="--enable-jit --enable-pcre2-16 --enable-pcre2-32 --enable-newline-is-anycrlf --enable-pcre2grep-libz --enable-pcre2grep-libbz2 --enable-year2038" \
    --with-pcre-jit \
    --with-zlib=../zlib-ng \
    --with-openssl=../openssl \
    --with-openssl-opt="PERL=$PERL no-shared mingw64 -march=native -flto no-tests enable-pie enable-weak-ssl-ciphers enable-md2 enable-rc5 enable-tls-deprecated-ec enable-tfo -DOPENSSL_TLS_SECURITY_LEVEL=0" \
    --add-module=../ngx_devel_kit \
    --add-module=../lua-nginx-module \
    --add-module=../stream-lua-nginx-module \
    --add-module=../lua-upstream-nginx-module \
    --add-module=../set-misc-nginx-module \
    --add-module=../array-var-nginx-module \
    --add-module=../form-input-nginx-module \
    --add-module=../encrypted-session-nginx-module \
    --add-module=../iconv-nginx-module \
    --add-module=../echo-nginx-module \
    --add-module=../xss-nginx-module \
    --add-module=../srcache-nginx-module \
    --add-module=../memc-nginx-module \
    --add-module=../redis-nginx-module \
    --add-module=../redis2-nginx-module \
    --add-module=../rds-json-nginx-module \
    --add-module=../rds-csv-nginx-module \
    --add-module=../njs/nginx \
    --add-module=../nginx_upstream_module \
    --add-module=../nginx-module-vts \
    --add-module=../ngx_brotli \
    --add-module=../zstd-nginx-module \
    --add-module=../ngx-fancyindex \
    --add-module=../headers-more-nginx-module \
    --add-module=../testcookie-nginx-module \
    --add-module=../nginx-http-flv-module \
    --add-module=../ngx_http_proxy_connect_module \
    --add-module=../nginx-dav-ext-module \
    --add-module=../ngx_cache_purge \
    --add-module=../ngx_http_geoip2_module && \

# 5. Сборка (На полную катушку!)
echo "--- Starting make with all cores ---"
make -j$(nproc) PERL="$PERL"