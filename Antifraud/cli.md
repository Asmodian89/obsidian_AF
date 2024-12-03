В [[Увр от ГРЧЦ]] есть встроенный механизм обогащения данных [[clickhouse]].
Запускается из папки где развернут Увр.
Команда имеет вид:
/etc/uvr/bin/cli 178.238.113.14 7565 update-records CALL_ID=XXXXXX,CALL_TYPE=XXXX,ID_SRC=YYYY,ID_DST=YYYY,DURATION=YYYY

Поля CALL_ID и CALL_TYPE обязательные - по ним однозначно идентифицируется запись в базе. Т.е. при обогащении данных всё равно нужно делать SELECT.
CALL_TYPE может быть IN или OUT.