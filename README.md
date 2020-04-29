# Creating UDF for Impala (C++, Docker, Crypto lib)

В связи с тем что в Impala отсутствуют многие функции доступные на обычных РСУБД, например, Upper/Lower для кириллицы, или хэш-функции md5, sha256 для сверки контрольных сумм, то приходится создавать свои User Defint Function.
