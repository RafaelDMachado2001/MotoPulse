# Modelo de dados

Todos os IDs são UUID. Timestamps são UTC e preservam `original_offset`; unidades canônicas: m, s, m/s, °C, rpm e litros. Ausência é `NULL` acompanhada de motivo em `missing_fields`/`issue`; nunca usar zero como ausente. Dinheiro é inteiro em centavos mais moeda ISO-4217.

## Tabelas

- `motorcycle(id, make, model, year, vin NULL, nickname NULL, created_at, updated_at)`; único `(make,model,year,vin)` quando VIN existe.
- `odometer_reading(id, motorcycle_id FK, read_at, value_m, source, note NULL)`; valor não negativo e não regressivo sem `correction`.
- `maintenance_record(id, motorcycle_id FK, occurred_at, type, description, odometer_reading_id NULL, cost_cents NULL, currency NULL, notes NULL, created_at)`; custo e moeda juntos.
- `maintenance_item(id, record_id FK, name, quantity NULL, part_number NULL, cost_cents NULL, currency NULL)`; `maintenance_plan(id,motorcycle_id FK,name,interval_m NULL,interval_days NULL,active)`; ao menos um intervalo.
- `trip(id,motorcycle_id FK,started_at NULL,ended_at NULL,status,source,distance_m NULL,duration_s NULL,summary_json NULL,created_at,updated_at)`; estados válidos conforme arquitetura.
- `trip_segment(id,trip_id FK,sequence,started_at,ended_at NULL,status)` único `(trip_id,sequence)`.
- `location_point(id,segment_id FK,sequence,recorded_at,original_offset,monotonic_ms NULL,latitude,longitude,accuracy_m NULL,altitude_m NULL,speed_mps NULL,source)`; lat/lon válidos, índice `(segment_id,sequence)`.
- `telemetry_import(id,motorcycle_id FK,source_file_name,source_hash,raw_uri,imported_at,encoding NULL,delimiter NULL,parser_version,status,error_summary NULL)`; hash único; bytes preservados.
- `telemetry_session(id,motorcycle_id FK,trip_id NULL FK,import_id NULL FK,started_at,ended_at NULL,source)`; trip e import podem faltar.
- `telemetry_sample(id,session_id FK,recorded_at,original_offset,rpm NULL,speed_mps NULL,gear NULL,engine_temp_c NULL,fuel_consumption_l_per_100km NULL,values_json NULL,missing_fields_json NULL)`; índice `(session_id,recorded_at)`.
- `sample_issue(id,sample_id FK,field,code,detail)`.
- `metric_result(id,trip_id NULL FK,session_id NULL FK,metric_type,value NULL,unit,method_version,coverage_json,evidence_json,calculated_at)`; ao menos um escopo.
- `baseline_profile(id,motorcycle_id FK,metric_type,cohort_definition_json,sample_count,p50,p90,mean,stddev,version,calculated_at)`.
- `finding(id,trip_id NULL FK,kind,metric_type,observed_value NULL,expected_value NULL,deviation NULL,confidence NULL,evidence_json,classification,created_at)`; kind em measurement/deviation/hypothesis/recommendation.
- `report(id,scope,trip_id NULL,period_start NULL,period_end NULL,structured_input_json,narrative,uncertainty_json,generated_at)`.

Foreign keys são obrigatórias; exclusão de moto é proibida enquanto houver histórico. Índices adicionais: `(motorcycle_id, occurred_at)`, `(motorcycle_id,started_at)`, `source_hash`, `(trip_id,metric_type)`. Derivados jamais sobrescrevem bruto; recalculação registra versão.
