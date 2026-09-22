# NginXYZ

**Кастомная высокопроизводительная нативная сборка nginx для Windows 10/11** — со статической линковкой, агрессивной оптимизацией и большим набором популярных сторонних модулей (LuaJIT/OpenResty, njs, Brotli, Zstd, GeoIP2 и др.), которые обычно приходится собирать вручную по отдельности.

> Форк [nginx/nginx](https://github.com/nginx/nginx). Собирается через MSYS2/MinGW-w64.

[![Project Status: Active](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)
[![License](https://img.shields.io/badge/License-BSD%202--Clause-blue.svg)](./LICENSE)

---

## Зачем это нужно

Официальная сборка nginx под Windows прямо заявлена как proof-of-concept с минимальным набором модулей. Чтобы получить «полноценный» nginx — со скриптингом на Lua, njs, современным сжатием, GeoIP и остальной экосистемой уровня OpenResty — на Windows, обычно приходится собирать всё это самостоятельно под MinGW, а это возня с платформенными нюансами (иконки, особенности `zlib-ng`, кросс-компиляция OpenSSL и т.д.).

NginXYZ берёт эту работу на себя: один скрипт `build.sh`, который конфигурирует и статически линкует nginx со всем перечисленным ниже в один `nginx.exe`.

## Возможности

- **Нативный Windows-бинарник** — статическая линковка, без внешних DLL для встроенных библиотек.
- **Скриптинг LuaJIT (FFI)** — полный стек модулей в духе OpenResty (`lua-nginx-module`, `stream-lua-nginx-module`, `lua-upstream-nginx-module` и др.).
- **Модульный NJS** — нативный JavaScript-модуль nginx.
- **Современное сжатие** — Brotli (`ngx_brotli`) и Zstandard (`zstd-nginx-module`) в дополнение к gzip.
- **GeoIP2** через `ngx_http_geoip2_module` + libmaxminddb.
- **Обработка изображений** — `http_image_filter_module` с GD, WebP и imagequant.
- **Наблюдаемость** — живая статистика через `nginx-module-vts`.
- **Кэширование и проксирование** — `srcache-nginx-module`, модули `memc`/`redis`/`redis2`, `ngx_cache_purge`, `ngx_http_proxy_connect_module`.
- **Стриминг** — `nginx-http-flv-module`, `http_mp4_module`, `http_flv_module`.
- **Оптимизация под конкретный CPU** — сборка с флагами `-O3 -march=native -flto`, PIE, stack protector и включённым TFO.

## Список сторонних модулей

<details>
<summary>Полный список (раскрыть)</summary>

**Lua / OpenResty-стек**
`ngx_devel_kit`, `lua-nginx-module`, `stream-lua-nginx-module`, `lua-upstream-nginx-module`, `set-misc-nginx-module`, `array-var-nginx-module`, `form-input-nginx-module`, `encrypted-session-nginx-module`, `iconv-nginx-module`, `echo-nginx-module`, `xss-nginx-module`, `srcache-nginx-module`

**Кэширование / бэкенды**
`memc-nginx-module`, `redis-nginx-module`, `redis2-nginx-module`, `rds-json-nginx-module`, `rds-csv-nginx-module`, `ngx_cache_purge`, `nginx_upstream_module`

**Скриптинг и сжатие**
`njs` (nginx JavaScript), `ngx_brotli`, `zstd-nginx-module`

**Разное / эксплуатация**
`nginx-module-vts`, `ngx-fancyindex`, `headers-more-nginx-module`, `testcookie-nginx-module`, `nginx-http-flv-module`, `ngx_http_proxy_connect_module`, `nginx-dav-ext-module`, `ngx_http_geoip2_module`

**Встроенные флаги ядра nginx**
`http_ssl`, `http_v2`, `http_realip`, `http_addition`, `http_xslt`, `http_image_filter`, `http_geoip`, `http_sub`, `http_dav`, `http_flv`, `http_mp4`, `http_gunzip`, `http_gzip_static`, `http_auth_request`, `http_random_index`, `http_secure_link`, `http_slice`, `http_stub_status`, `http_json`, `mail`, `stream`, `stream_ssl`, `stream_realip`, `stream_geoip`, `stream_ssl_preread`, `file-aio`

**Статически слинкованные библиотеки**
LuaJIT, OpenSSL, zlib-ng, PCRE2 (с JIT), libxml2/libxslt/libexslt, GD (+ libwebp, libpng, libjpeg, imagequant, freetype, harfbuzz), libmaxminddb, yajl, msgpuck, lua-cjson, bzip2, lzma

</details>

## Требования

- Windows 10/11, процессор x86-64 с поддержкой **SSE4.2 или новее**.
- [MSYS2](https://www.msys2.org/) с тулчейном MinGW-w64 для сборки из исходников.
- Репозитории зависимостей, на которые ссылается `build.sh`, склонированы как соседние директории (`nginx`, `openssl`, `zlib-ng`, `pcre2`, `luajit2` и по одной директории на каждый `--add-module=../...`). Точные пути смотри в `build.sh`.

> **О переносимости бинарника:** по умолчанию используется `-march=native`, поэтому бинарник, собранный на одной машине, **не гарантированно запустится на другом CPU** — если у целевого процессора нет набора инструкций, доступного на машине сборки, получите падение с illegal instruction. Если планируешь раздавать готовые бинарники, а не собирать локально под свой CPU, замени `-march=native` на что-то более консервативное (например, `-march=x86-64-v2`, что покрывает CPU уровня SSE4.2/POPCNT).

## Сборка

```bash
# из шелла MSYS2 MinGW64, при наличии всех соседних репозиториев-зависимостей
./build.sh

# опциональные флаги:
./build.sh --with-luajit-opt="..."   # передать доп. опции в build.sh LuaJIT
./build.sh --force-clean-openssl     # принудительно пересобрать OpenSSL целиком
                                      # (обычно происходит автоматически только
                                      # при смене версии gcc)
```

Что делает скрипт:
1. Чистит предыдущие артефакты сборки nginx, а OpenSSL чистит условно (только при смене версии компилятора либо при явном флаге — иначе идёт быстрая инкрементальная пересборка).
2. Патчит `zlib-ng`, создавая структуру Makefile, которую ожидает configure-скрипт nginx на Windows.
3. Собирает LuaJIT, если он ещё не собран.
4. Компилирует иконку в виде Windows-ресурса.
5. Запускает `auto/configure` с полным списком модулей и флагами статической линковки, затем `make -j$(nproc)`.

Готовый бинарник оказывается в `objs/nginx.exe`.

## ⚠️ О безопасности: режим совместимости TLS

Сейчас сборка конфигурирует OpenSSL с **`OPENSSL_TLS_SECURITY_LEVEL=0`** и включёнными legacy/слабыми опциями (`enable-weak-ssl-ciphers`, `enable-tls-deprecated-ec`, `enable-md2`, `enable-rc5`) — ради совместимости со старыми клиентами. **Это осознанно ослабляет TLS-стек по сравнению со стандартной сборкой nginx/OpenSSL.**

Если совместимость со старыми клиентами тебе не нужна конкретно — не используй эту сборку там, где важна безопасность, без предварительного ужесточения этих опций конфигурации OpenSSL. В планах — сделать это опциональным флагом сборки, а не поведением по умолчанию.

## Bleeding edge

Это не проект в стиле "собрал один раз и забыл". Дерево пересобирается буквально через каждые пару коммитов апстрима [nginx/nginx master](https://github.com/nginx/nginx), так что фиксы и новые фичи из mainline nginx попадают сюда быстро, а не ждут очередного stable-тега. Repo плотно следует за апстримом — а значит, возможны и периодические поломки, потому что сторонние модули не всегда успевают за изменениями во внутренностях nginx.

## Статус проекта

В активной разработке, ранняя стадия / пререлиз. Возможны шероховатости — актуальные пробелы см. в [Issues](../../issues).

## Благодарности

Построено поверх [nginx](https://nginx.org/), [njs](https://github.com/nginx/njs), экосистемы Lua-модулей [OpenResty](https://openresty.org/) и множества авторов сторонних модулей nginx, перечисленных выше.

## Лицензия

[2-clause BSD-like лицензия](./LICENSE) (как и у оригинального nginx).
