# Creating UDF for Impala (C++, Docker, Crypto lib)

В связи с тем, что в Impala отсутствуют многие функции доступные на обычных РСУБД, например, Upper/Lower для кириллицы, или хэш-функции md5, sha256 для сверки контрольных сумм и пр. вещей, то появляется необходимость создавать свои User Defint Function. Легче и быстрее всего UDF создать на проде, но, к сожалению, данный вариант не всегда доступен в связи с тем, что при компиляции той или иной UDF могут возникнуть проблемы с версионностью и взаимозависимостью библиотек etc. Одно из решений данной проблемы это воспользоваться docker образом, что и сделаем далее ниже. 

### И так, для этого нам потребуется установка:
0. Если у вас ОС Windows, то понадобиться установить программу виртуализации, например, VirtualBox, чтобы развернуть ОС на базе Linux, на Win на мой взгляд, Docker работает неполноценно.
1. Устанавливаем Docker
2. Скачиваем Docker образ под ОС, которая у вас на проде
> docker pull *oraclelinux7.5*
3. Переходим в папку с Dockerfile-OS ("-OS" уберите перед запуском), и собираем образ с ОС и инструментами для разработки UDF
> cd /path

> docker build -t *impala-udf-base* .

4. Создаем новый образ на основе предыдущего образа, где компилируем hash UDF 

4.1 Перходим в папку с UDF (udfimp)
> cd /path

4.2 Клонируем в него репозиторий криптографии
> git clone https://github.com/weidai11/cryptopp

4.3 Клонируем в него репозиторий с базовым сэмплом от Cloudera
> git clone https://github.com/cloudera/impala-udf-samples

4.4 В папке с Dockerfile-UDF запускаем новую сборку ("-UDF" уберите перед запуском). В результате у вас скомпилируется нужная динамическая библиотека.
> docker build -t *image name* .

5. Создаем и запускаем Docker-контейнер из Docker-образа
> docker run -it -d --name *имя контейнера* impala-udf-base
6. Копируем необходимые файлы или папку из контейнера на хост-машину, в нашем случае папка /udfmp/build - там будет сгенерированная динамическая библиотека <b>libudfcrypto.so</b>

> docker cp *id контейнера*:*/path_docker_container* *"/path_local"*

> cd *"/path_local"*

### Далее способ 2
7. Копирование с хост-машины в контейнер, в нашем случае надо туда перенести папку impala-udf-master
> docker cp *"/path_local"* *id контейнера*:*/path_docker_container*
8. Входим в docker-контейнер
> docker exec -t -i dd7518a91b66 /bin/bash
9. Далее заходим в нужную папку (impala-udf-master) и компилим *cmake . && make* в результате получим несколько .so библиотек для создания пользовательских функций Upper, Lower, String, Date etc.
10. Переносим полученные .so на прод сервер в папку /user/impala/udfs

#### Создаем пользовательские функции по аналогии:
> create function if not exists sha256(string) returns string location '/user/impala/udfs/libudfcrypto.so' SYMBOL='SHA256';
> 
> create function if not exists md5(string) returns string location '/user/impala/udfs/libudfcrypto.so' SYMBOL='MD5';
> 
> create function if not exists upper_ruutf8(string) returns string location '/user/impala/udfs/libudf-strings.so' SYMBOL='upper_ruutf8';
> 
> create function if not exists lower_ruutf8(string) returns string location '/user/impala/udfs/libudf-strings.so' SYMBOL='lower_ruutf8';
> 
> create function if not exists length_ruutf8_str(string) returns string location '/user/impala/udfs/libudf-strings.so' SYMBOL='length_ruutf8_str';
> 
> create function if not exists substr_ruutf8_str(string,string,string) returns string location '/user/impala/udfs/libudf-strings.so' SYMBOL='substr_ruutf8_str';
> 
> create function if not exists substr2_ruutf8_str(string,string) returns string location '/user/impala/udfs/libudf-strings.so' SYMBOL='substr2_ruutf8_str';
> 
> create function if not exists translate_ruutf8(string,string,string) returns string location '/user/impala/udfs/libudf-strings.so' SYMBOL='translate_ruutf8';

#### Примеры select'а:
select    
default.length_ruutf8_str("привет мИр"),
default.substr_ruutf8_str("привет мИр",'1','10'),
default.substr2_ruutf8_str("привет мИр",'5'),
default.translate_ruutf8('KFC PRIMORSKAYA  CAFE','qwerенгшщASDFРОЛД','mnbячсмPIIYTЙЦУК')

### Полезные ссылки:
[Docker](https://community.vscale.io/hc/ru/community/posts/211783625-%D0%9E%D1%81%D0%BD%D0%BE%D0%B2%D1%8B-%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D1%8B-%D1%81-Docker) 

[Hash UDF](https://github.com/ScalefreeCOM/impala-crypto-udf)

[Create UDF_1](https://impala.apache.org/docs/build/html/topics/impala_create_function.html)

[Create UDF_1](https://impala.apache.org/docs/build/html/topics/impala_udf.html#udfs)
