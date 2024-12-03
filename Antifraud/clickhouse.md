Ещё одна субд в нашем зоопарке.
немного инфы по таблицам (пока никакие параметры не трогали всё из коробки от #УврОтГРЧЦ)

 CREATE TABLE system.mutations
(
    `database` String,
    `table` String,
    `mutation_id` String,
    `command` String,
    `create_time` DateTime,
    `block_numbers.partition_id` Array(String),
    `block_numbers.number` Array(Int64),
    `parts_to_do_names` Array(String),
    `parts_to_do` Int64,
    `is_done` UInt8,
    `latest_failed_part` String,
    `latest_fail_time` DateTime,
    `latest_fail_reason` String
)
ENGINE = SystemMutations
COMMENT 'SYSTEM TABLE is built on the fly.' │


 CREATE TABLE uvr_980.calls
(
    `NUM_A` String,
    `NUM_B` String,
    `NUM_D` String,
    `NUM_C` String,
    `DATE` DateTime64(9),
    `DATE_ACT` DateTime64(0),
    `ID_SRC` Nullable(UInt32),
    `ID_DST` Nullable(UInt32),
    `SESSION_ID` UInt64,
    `ID_REL` UInt8,
    `VRF_RSP` Int16,
    `RELEASE_CODE` Nullable(UInt8),
    `DURATION` Nullable(UInt16),
    `ID_UVR_T` Nullable(UInt16),
    `CALL_ID` String,
    `CALL_TYPE` UInt8,
    INDEX NUM_A_IX CALL_ID TYPE set(0) GRANULARITY 8192
)
ENGINE = ReplacingMergeTree
PARTITION BY toUInt64(formatDateTime(DATE, '%Y%m%d%H'))
ORDER BY (DATE, NUM_A, NUM_B, CALL_TYPE)
TTL (toDateTime(DATE) + toIntervalYear(1)) + toIntervalHour(1)
SETTINGS merge_with_ttl_timeout = 3600, index_granularity = 8192 │


пока это всё возможно будем пробовать другие параметры например TTL
https://clickhouse.com/docs/ru/engines/table-engines/mergetree-family/mergetree
всё будет зависеть от того насколько эффективно будет работать [[Алгоритм обогащения данных clickhouse]]
