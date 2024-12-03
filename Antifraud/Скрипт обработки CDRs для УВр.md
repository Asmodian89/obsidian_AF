```python


import os
import zipfile
import json
import csv
import time
import logging
from datetime import datetime
from clickhouse_driver import Client

# Настройка логирования
logging.basicConfig(filename='script.log', level=logging.INFO,
                    format='%(asctime)s - %(levelname)s - %(message)s')

# Настройки
WORK_DIR = '/etc/uvr'
OPR_DIR = '/etc/uvr/digests/operators'
CHECK_INTERVAL_FILES = 60  # интервал проверки .zip файлов в секундах
CHECK_INTERVAL_OPR = 300  # интервал проверки наличия OPR файлов в секундах
CLICKHOUSE_HOST = 'localhost'
CLICKHOUSE_DB = 'uvr_908'

# Функция для поиска актуального OPR файла в рабочей директории
def get_latest_opr_csv(work_dir):
    csv_files = [f for f in os.listdir(work_dir) if f.startswith('OPR_') and f.endswith('.csv')]
    if not csv_files:
        return None
    csv_files.sort(key=lambda x: datetime.strptime('_'.join(x.split('_')[1:]), '%Y_%m_%d_%H_%M_%S.csv'))
    return os.path.join(work_dir, csv_files[-1])

# Функция для проверки соответствия даты файла текущей дате
def is_file_date_current(file_name):
    try:
        date_str = '_'.join(file_name.split('_')[1:4])
        file_date = datetime.strptime(date_str, '%Y_%m_%d')
        return file_date.date() == datetime.now().date()
    except ValueError:
        logging.error(f"Ошибка извлечения даты из имени файла: {file_name}")
        return False

# Функция для обработки OPR файла (CSV)
def process_opr_csv(csv_file):
    opr_data = {}
    row_count = 0

    try:
        with open(csv_file, 'r', encoding='utf-8') as f:
            reader = csv.DictReader(f, delimiter=';')
            for row in reader:
                opr_data[row['inn']] = row['id_src']
                row_count += 1

        logging.info(f"Прочитан файл OPR: {csv_file}, количество строк: {row_count}")
        return opr_data, row_count
    except Exception as e:
        logging.error(f"Ошибка при чтении CSV файла {csv_file}: {str(e)}")
        return None, 0

# Функция для разархивирования OPR файла в рабочую директорию
def unzip_file(zip_file, extract_to):
    try:
        with zipfile.ZipFile(zip_file, 'r') as zip_ref:
            zip_ref.extractall(extract_to)
        logging.info(f"Разархивирован файл {zip_file} в {extract_to}")
    except Exception as e:
        logging.error(f"Ошибка при разархивировании файла {zip_file}: {str(e)}")

# Функция для поиска самого нового OPR файла в ZIP формате
def get_newest_opr_file(directory, pattern, extension='.zip'):
    opr_files = [f for f in os.listdir(directory) if f.startswith(pattern) and f.endswith(extension)]
    if not opr_files:
        return None
    opr_files.sort(key=lambda x: datetime.strptime('_'.join(x.split('_')[1:]), '%Y_%m_%d_%H_%M_%S' + extension))
    return os.path.join(directory, opr_files[-1])

# Обработка OPR файлов с учетом актуальности даты
def process_opr_file():
    csv_file = get_latest_opr_csv(WORK_DIR)
    if csv_file and is_file_date_current(csv_file):
        logging.info(f"Найден актуальный CSV файл: {csv_file}")
        opr_data, row_count = process_opr_csv(csv_file)
        return opr_data, row_count

    opr_file = get_newest_opr_file(OPR_DIR, 'OPR_', '.zip')
    if opr_file:
        unzip_file(opr_file, WORK_DIR)
        new_csv_file = os.path.join(WORK_DIR, os.path.splitext(os.path.basename(opr_file))[0] + '.csv')

        if os.path.exists(new_csv_file):
            opr_data, row_count = process_opr_csv(new_csv_file)
            if csv_file and csv_file != new_csv_file:
                os.remove(csv_file)
                logging.info(f"Удален старый CSV файл: {csv_file}")

            return opr_data, row_count
        else:
            logging.error(f"CSV файл не найден после разархивирования: {new_csv_file}")

    logging.error(f"Не найден OPR файл с текущей датой в директории: {OPR_DIR}")
    return None, 0

# Функция для разархивирования и обработки JSON-файла из ZIP
def process_zip_file(zip_file):
    extract_dir = '/tmp/unzipped_files'
    os.makedirs(extract_dir, exist_ok=True)
    
    json_file_path = None
    try:
        with zipfile.ZipFile(zip_file, 'r') as zip_ref:
            zip_ref.extractall(extract_dir)
        logging.info(f"Файл {zip_file} успешно разархивирован в {extract_dir}")

        json_files = [f for f in os.listdir(extract_dir) if f.endswith('.json')]
        if not json_files:
            logging.error(f"JSON файл не найден в архиве {zip_file}")
            return []

        json_file_path = os.path.join(extract_dir, json_files[0])
        data = process_json_file(json_file_path)

        return data
    except Exception as e:
        logging.error(f"Ошибка при разархивировании файла {zip_file}: {str(e)}")
        return []
    finally:
        if json_file_path and os.path.exists(json_file_path):
            os.remove(json_file_path)
            logging.info(f"Удален разархивированный JSON файл: {json_file_path}")

# Функция для чтения JSON-файла
def process_json_file(json_file):
    data = []
    encodings = ['utf-8', 'ISO-8859-1', 'cp1251']

    for encoding in encodings:
        try:
            with open(json_file, 'r', encoding=encoding) as f:
                for line in f:
                    try:
                        obj = json.loads(line.strip())
                        data.append(obj)
                    except json.JSONDecodeError as e:
                        logging.error(f"Ошибка при декодировании строки JSON: {str(e)}")
            logging.info(f"Прочитан JSON файл: {json_file} с кодировкой {encoding}")
            break
        except UnicodeDecodeError as e:
            logging.warning(f"Не удалось прочитать файл {json_file} с кодировкой {encoding}: {str(e)}")
            continue

    if not data:
        logging.error(f"Не удалось прочитать файл {json_file} с доступными кодировками.")
    
    return data

# Функция для выполнения SELECT и последующего ALTER TABLE
def select_and_update_data(client, num_a, num_b, new_id_src, new_id_dst, duration, num_c, date):
    where_clause = f"NUM_A = '{num_a}' AND NUM_B = '{num_b}'"
    if num_c:
        where_clause += f" AND NUM_C = '{num_c}'"
    where_clause += f" AND ABS(toUnixTimestamp(DATE) - toUnixTimestamp('{date}')) <= 300"

    # Выполняем SELECT для проверки наличия строк
    select_query = f"SELECT COUNT(*) FROM calls WHERE {where_clause}"
    try:
        result = client.execute(select_query)
        count = result[0][0]
        if count > 0:
            # Формируем UPDATE только если SELECT нашел строки
            update_clauses = []
            if new_id_src is not None and new_id_src != '11069':
                update_clauses.append(f"ID_SRC = {new_id_src}")
            if new_id_dst is not None and new_id_dst != '11069':
                update_clauses.append(f"ID_DST = {new_id_dst}")
            if duration is not None:
                update_clauses.append(f"DURATION = {duration}")
            if num_c is not None:
                update_clauses.append(f"NUM_C = {num_c}")

            if update_clauses:
                update_query = f"""
                ALTER TABLE calls
                UPDATE {', '.join(update_clauses)}
                WHERE {where_clause}
                """
                client.execute(update_query)
                return 1  # Считаем успешным, если запросы прошли без ошибок
    except Exception as e:
        logging.error(f"Ошибка при выполнении SELECT или ALTER TABLE в ClickHouse: {str(e)}")
    
    return 0

# Функция для обработки данных из JSON и обновления ClickHouse
def compare_with_clickhouse_and_update(data, opr_data):
    client = Client(CLICKHOUSE_HOST, database=CLICKHOUSE_DB)
    
    total_json_rows = len(data)
    alter_count = 0

    for obj in data:
        try:
            date = datetime.strptime(obj['DATE'], '%Y-%m-%dT%H:%M:%S%z').replace(tzinfo=None).strftime('%Y-%m-%d %H:%M:%S')
        except ValueError:
            logging.error(f"Ошибка при преобразовании даты для NUM_A: {obj['NUM_A']} NUM_B: {obj['NUM_B']}")
            continue

        id_src = opr_data.get(obj['INN_SRC'], None) if obj['INN_SRC'] != '11069' else None
        id_dst = opr_data.get(obj['INN_DST'], None) if obj['INN_DST'] != '11069' else None
        duration = obj.get('DURATION', None)
        num_c = obj.get('NUM_C', None)

        # Выполняем SELECT и, если найдено, сразу выполняем UPDATE
        if select_and_update_data(client, obj['NUM_A'], obj['NUM_B'], id_src, id_dst, duration, num_c, date):
            alter_count += 1

    logging.info(f"Обработано строк в JSON: {total_json_rows}, количество успешных ALTER TABLE запросов: {alter_count}")
    client.disconnect()

# Основной цикл проверки файлов
def main():
    last_opr_check = time.time() - CHECK_INTERVAL_OPR
    opr_data = {}

    while True:
        if time.time() - last_opr_check >= CHECK_INTERVAL_OPR:
            opr_data, row_count = process_opr_file()
            last_opr_check = time.time()

        json_zip_file = get_newest_opr_file(WORK_DIR, '', '.zip')
        if json_zip_file:
            json_data = process_zip_file(json_zip_file)
            if json_data and opr_data:
                compare_with_clickhouse_and_update(json_data, opr_data)
        else:
            logging.info("Более новый OPR .zip файл не найден, ожидаем следующий.")

        time.sleep(CHECK_INTERVAL_FILES)

if __name__ == "__main__":
    main()
```
