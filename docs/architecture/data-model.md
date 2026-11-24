# Modelo de dados

Este documento é o contrato do esquema local. Os blocos `sql` executam na ordem em que aparecem, sem adaptação, em SQLite 3.37 ou superior com JSON (necessários para `STRICT` e `json_*`). A versão embutida pela biblioteca do aplicativo é [A VERIFICAR] (spike S3 de `architecture.md`). É documentação: o código de migração pertence à F1.

## Regras gerais

1. **Identidade.** `id` é UUID em texto, gerado pelo aplicativo. O banco garante presença e unicidade; o formato UUID é verificado pelo aplicativo (T1.16).
2. **Instantes.** UTC no formato único `AAAA-MM-DDTHH:MM:SS.sssZ`, com milissegundos sempre presentes, para que a ordem do texto seja a ordem do tempo. Datas inexistentes, como 30 de fevereiro, são recusadas.
3. **Offset.** Todo instante de evento (leitura, manutenção, viagem, segmento, ponto, sessão, período de relatório) tem o offset original, `+HH:MM` ou `-HH:MM`. Instantes de auditoria (`created_at`, `updated_at`, `imported_at`, `calculated_at` e semelhantes) são só UTC. Hora local sem fuso conhecido nunca é convertida: fica em `telemetry_session.started_local`, e `started_at` fica ausente com motivo.
4. **Unidades canônicas.** Metros, segundos, m/s, graus Celsius, rpm e km/L como lido. Dinheiro é centavo inteiro mais código de moeda de três letras. L/100 km é apenas derivado, em `metric_result`.
5. **Ausência.** Medição ou valor monetário ausente é NULL com motivo, nunca zero. O motivo é um código `snake_case` em coluna `*_missing_reason` ou, nas tabelas largas, em `missing_fields_json` (objeto campo -> código). O motivo existe se e somente se o valor falta.
6. **Opcionais estruturais, sem motivo.** `vin`, `nickname`, `note`, `notes`, `part_number`, FKs de vínculo (`trip_id`, `session_id`, `sample_id`, `plan_item_id`, `corrects_reading_id`, `baseline_profile_id`, `metric_result_id`), `interval_m` e `interval_days` (ao menos um), períodos de relatório fora do escopo `period` e os campos de conclusão de `telemetry_parse_run`, `trip_segment` e `raw_deletion`, controlados pelo estado. `row_number` e `raw_values_json` só existem em amostras vindas de arquivo.
7. **Bruto e derivado.** Bruto: arquivo original, `telemetry_import`, `telemetry_sample` e `location_point`. Bruto não é alterado nem apagado, exceto pela exclusão definitiva autorizada (decisão D2 do ADR 0002). Derivados têm versão, não são alterados e podem ser recalculados.
8. **Conexão.** Toda conexão liga `foreign_keys` e `recursive_triggers`, que o SQLite não persiste. Sem `recursive_triggers`, `INSERT OR REPLACE` apagaria bruto sem disparar o gatilho de exclusão. `journal_mode=WAL` é ligado pelo aplicativo. T1.17 verifica os três.

## Conexão e versão

```sql
PRAGMA foreign_keys = ON;
PRAGMA recursive_triggers = ON;
PRAGMA user_version = 1;
```

## Garagem e odômetro

`nickname` é opcional e digitado pela pessoa no aplicativo. Nunca é preenchido nem sugerido a partir do CSV: o apelido do veículo no CSV só existe no arquivo bruto (D3). `is_demo` separa dados de demonstração dos reais.

Uma leitura é efetiva quando nenhuma outra a corrige. Leituras efetivas não regridem no tempo; a correção substitui a leitura apontada e exige nota. Leituras não são alteradas nem apagadas. Troca de painel com reinício da contagem não está modelada: [A VERIFICAR].

```sql
CREATE TABLE motorcycle (
  id TEXT NOT NULL PRIMARY KEY,
  make TEXT NOT NULL,
  model TEXT NOT NULL,
  year INTEGER NOT NULL CHECK (year BETWEEN 1900 AND 2200),
  vin TEXT UNIQUE,
  nickname TEXT,
  is_demo INTEGER NOT NULL DEFAULT 0 CHECK (is_demo IN (0, 1)),
  created_at TEXT NOT NULL CHECK (
    created_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(created_at, 1, 19)) IS substr(created_at, 1, 19)
    AND substr(created_at, 12, 2) <= '23'
  ),
  updated_at TEXT NOT NULL CHECK (
    updated_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(updated_at, 1, 19)) IS substr(updated_at, 1, 19)
    AND substr(updated_at, 12, 2) <= '23'
  )
) STRICT;

CREATE TABLE odometer_reading (
  id TEXT NOT NULL PRIMARY KEY,
  motorcycle_id TEXT NOT NULL REFERENCES motorcycle (id) ON DELETE RESTRICT,
  read_at TEXT NOT NULL CHECK (
    read_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(read_at, 1, 19)) IS substr(read_at, 1, 19)
    AND substr(read_at, 12, 2) <= '23'
  ),
  original_offset TEXT NOT NULL CHECK (
    original_offset GLOB '[+-][01][0-9]:[0-5][0-9]' AND substr(original_offset, 2) <= '14:00'
  ),
  value_m INTEGER NOT NULL CHECK (value_m >= 0),
  source TEXT NOT NULL CHECK (source IN ('user', 'maintenance', 'trip', 'import')),
  note TEXT,
  corrects_reading_id TEXT UNIQUE,
  created_at TEXT NOT NULL CHECK (
    created_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(created_at, 1, 19)) IS substr(created_at, 1, 19)
    AND substr(created_at, 12, 2) <= '23'
  ),
  UNIQUE (id, motorcycle_id),
  FOREIGN KEY (corrects_reading_id, motorcycle_id)
    REFERENCES odometer_reading (id, motorcycle_id) ON DELETE RESTRICT,
  CHECK (corrects_reading_id <> id),
  CHECK (corrects_reading_id IS NULL OR length(trim(coalesce(note, ''))) > 0)
) STRICT;

CREATE INDEX ix_odometer_moto_time ON odometer_reading (motorcycle_id, read_at);

CREATE TRIGGER odometer_no_regression
BEFORE INSERT ON odometer_reading
WHEN EXISTS (
  SELECT 1
  FROM odometer_reading AS r
  WHERE r.motorcycle_id = NEW.motorcycle_id
    AND r.id IS NOT NEW.corrects_reading_id
    AND NOT EXISTS (
      SELECT 1 FROM odometer_reading AS c WHERE c.corrects_reading_id = r.id
    )
    AND (
      (r.read_at <= NEW.read_at AND r.value_m > NEW.value_m)
      OR (r.read_at > NEW.read_at AND r.value_m < NEW.value_m)
    )
)
BEGIN
  SELECT RAISE(ABORT, 'odometer regression without correction');
END;

CREATE TRIGGER odometer_no_update
BEFORE UPDATE ON odometer_reading
BEGIN
  SELECT RAISE(ABORT, 'odometer reading is append-only');
END;

CREATE TRIGGER odometer_no_delete
BEFORE DELETE ON odometer_reading
BEGIN
  SELECT RAISE(ABORT, 'odometer reading is append-only');
END;

CREATE TRIGGER motorcycle_demo_fixed
BEFORE UPDATE OF is_demo ON motorcycle
BEGIN
  SELECT RAISE(ABORT, 'is_demo is fixed');
END;
```

## Manutenção

`maintenance_plan` define intervalos e `plan_item` lista o que o plano prevê. `maintenance_record` é o serviço executado e `maintenance_item` o que foi feito nele, apontando opcionalmente para o item previsto. `record_plan` diz quais planos o serviço cumpriu. Plano e registro são sempre da mesma moto.

```sql
CREATE TABLE maintenance_record (
  id TEXT NOT NULL PRIMARY KEY,
  motorcycle_id TEXT NOT NULL REFERENCES motorcycle (id) ON DELETE RESTRICT,
  occurred_at TEXT NOT NULL CHECK (
    occurred_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(occurred_at, 1, 19)) IS substr(occurred_at, 1, 19)
    AND substr(occurred_at, 12, 2) <= '23'
  ),
  original_offset TEXT NOT NULL CHECK (
    original_offset GLOB '[+-][01][0-9]:[0-5][0-9]' AND substr(original_offset, 2) <= '14:00'
  ),
  type TEXT NOT NULL CHECK (type IN ('service', 'repair', 'inspection', 'other')),
  description TEXT NOT NULL,
  odometer_reading_id TEXT,
  odometer_missing_reason TEXT CHECK (
    odometer_missing_reason GLOB '[a-z]*' AND odometer_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  cost_cents INTEGER CHECK (cost_cents >= 0),
  currency TEXT CHECK (currency GLOB '[A-Z][A-Z][A-Z]'),
  cost_missing_reason TEXT CHECK (
    cost_missing_reason GLOB '[a-z]*' AND cost_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  notes TEXT,
  created_at TEXT NOT NULL CHECK (
    created_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(created_at, 1, 19)) IS substr(created_at, 1, 19)
    AND substr(created_at, 12, 2) <= '23'
  ),
  UNIQUE (id, motorcycle_id),
  FOREIGN KEY (odometer_reading_id, motorcycle_id)
    REFERENCES odometer_reading (id, motorcycle_id) ON DELETE RESTRICT,
  CHECK ((odometer_reading_id IS NULL) = (odometer_missing_reason IS NOT NULL)),
  CHECK ((cost_cents IS NULL) = (cost_missing_reason IS NOT NULL)),
  CHECK ((cost_cents IS NULL) = (currency IS NULL))
) STRICT;

CREATE TABLE maintenance_plan (
  id TEXT NOT NULL PRIMARY KEY,
  motorcycle_id TEXT NOT NULL REFERENCES motorcycle (id) ON DELETE RESTRICT,
  name TEXT NOT NULL,
  interval_m INTEGER CHECK (interval_m > 0),
  interval_days INTEGER CHECK (interval_days > 0),
  active INTEGER NOT NULL DEFAULT 1 CHECK (active IN (0, 1)),
  created_at TEXT NOT NULL CHECK (
    created_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(created_at, 1, 19)) IS substr(created_at, 1, 19)
    AND substr(created_at, 12, 2) <= '23'
  ),
  UNIQUE (id, motorcycle_id),
  CHECK (interval_m IS NOT NULL OR interval_days IS NOT NULL)
) STRICT;

CREATE TABLE plan_item (
  id TEXT NOT NULL PRIMARY KEY,
  plan_id TEXT NOT NULL REFERENCES maintenance_plan (id) ON DELETE RESTRICT,
  name TEXT NOT NULL,
  part_number TEXT,
  UNIQUE (plan_id, name)
) STRICT;

CREATE TABLE maintenance_item (
  id TEXT NOT NULL PRIMARY KEY,
  record_id TEXT NOT NULL REFERENCES maintenance_record (id) ON DELETE RESTRICT,
  plan_item_id TEXT REFERENCES plan_item (id) ON DELETE RESTRICT,
  name TEXT NOT NULL,
  quantity REAL CHECK (quantity > 0),
  quantity_missing_reason TEXT CHECK (
    quantity_missing_reason GLOB '[a-z]*' AND quantity_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  part_number TEXT,
  cost_cents INTEGER CHECK (cost_cents >= 0),
  currency TEXT CHECK (currency GLOB '[A-Z][A-Z][A-Z]'),
  cost_missing_reason TEXT CHECK (
    cost_missing_reason GLOB '[a-z]*' AND cost_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  CHECK ((quantity IS NULL) = (quantity_missing_reason IS NOT NULL)),
  CHECK ((cost_cents IS NULL) = (cost_missing_reason IS NOT NULL)),
  CHECK ((cost_cents IS NULL) = (currency IS NULL))
) STRICT;

CREATE TABLE record_plan (
  record_id TEXT NOT NULL,
  plan_id TEXT NOT NULL,
  motorcycle_id TEXT NOT NULL,
  PRIMARY KEY (record_id, plan_id),
  FOREIGN KEY (record_id, motorcycle_id)
    REFERENCES maintenance_record (id, motorcycle_id) ON DELETE RESTRICT,
  FOREIGN KEY (plan_id, motorcycle_id)
    REFERENCES maintenance_plan (id, motorcycle_id) ON DELETE RESTRICT
) STRICT;

CREATE INDEX ix_record_moto_time ON maintenance_record (motorcycle_id, occurred_at);
CREATE INDEX ix_item_record ON maintenance_item (record_id);

CREATE TRIGGER item_plan_same_moto_insert
BEFORE INSERT ON maintenance_item
WHEN NEW.plan_item_id IS NOT NULL AND NOT EXISTS (
  SELECT 1
  FROM plan_item AS i
  JOIN maintenance_plan AS p ON p.id = i.plan_id
  JOIN maintenance_record AS r ON r.motorcycle_id = p.motorcycle_id
  WHERE i.id = NEW.plan_item_id AND r.id = NEW.record_id
)
BEGIN
  SELECT RAISE(ABORT, 'plan item motorcycle mismatch');
END;

CREATE TRIGGER item_plan_fixed
BEFORE UPDATE OF record_id, plan_item_id ON maintenance_item
BEGIN
  SELECT RAISE(ABORT, 'maintenance item links are fixed');
END;

CREATE TRIGGER record_moto_fixed
BEFORE UPDATE OF motorcycle_id ON maintenance_record
BEGIN
  SELECT RAISE(ABORT, 'motorcycle is fixed');
END;

CREATE TRIGGER plan_moto_fixed
BEFORE UPDATE OF motorcycle_id ON maintenance_plan
BEGIN
  SELECT RAISE(ABORT, 'motorcycle is fixed');
END;

CREATE TRIGGER plan_item_plan_fixed
BEFORE UPDATE OF plan_id ON plan_item
BEGIN
  SELECT RAISE(ABORT, 'plan is fixed');
END;
```

## Viagem e GPS do aplicativo

Viagem criada de importação pode não ter instante UTC conhecido; nesse caso, e só nele, aceita-se motivo em vez da data. O banco não exige fim posterior ao início, porque o relógio pode ser ajustado durante a viagem; a duração usa `monotonic_ms`. O segmento fecha em toda pausa ou interrupção: no máximo um aberto por viagem. O banco recusa transição de estado fora da tabela de `architecture.md`; as guardas (permissão, confirmação) são do aplicativo (T2.8). `finished` e `cancelled` são finais. Viagem não é apagada nem substituída. Distâncias e duração não ficam em `trip`: são derivados em `metric_result`.

`location_point` guarda apenas GPS gravado pelo aplicativo. **O GPS do CSV mora somente em `telemetry_sample`**, e nunca é copiado para cá.

```sql
CREATE TABLE trip (
  id TEXT NOT NULL PRIMARY KEY,
  motorcycle_id TEXT NOT NULL REFERENCES motorcycle (id) ON DELETE RESTRICT,
  status TEXT NOT NULL CHECK (
    status IN ('draft', 'recording', 'paused', 'interrupted', 'finished', 'cancelled')
  ),
  source TEXT NOT NULL CHECK (source IN ('gps', 'import', 'demo')),
  started_at TEXT CHECK (
    started_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(started_at, 1, 19)) IS substr(started_at, 1, 19)
    AND substr(started_at, 12, 2) <= '23'
  ),
  started_original_offset TEXT CHECK (
    started_original_offset GLOB '[+-][01][0-9]:[0-5][0-9]' AND substr(started_original_offset, 2) <= '14:00'
  ),
  started_missing_reason TEXT CHECK (
    started_missing_reason GLOB '[a-z]*' AND started_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  ended_at TEXT CHECK (
    ended_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(ended_at, 1, 19)) IS substr(ended_at, 1, 19)
    AND substr(ended_at, 12, 2) <= '23'
  ),
  ended_original_offset TEXT CHECK (
    ended_original_offset GLOB '[+-][01][0-9]:[0-5][0-9]' AND substr(ended_original_offset, 2) <= '14:00'
  ),
  ended_missing_reason TEXT CHECK (
    ended_missing_reason GLOB '[a-z]*' AND ended_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  created_at TEXT NOT NULL CHECK (
    created_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(created_at, 1, 19)) IS substr(created_at, 1, 19)
    AND substr(created_at, 12, 2) <= '23'
  ),
  updated_at TEXT NOT NULL CHECK (
    updated_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(updated_at, 1, 19)) IS substr(updated_at, 1, 19)
    AND substr(updated_at, 12, 2) <= '23'
  ),
  UNIQUE (id, motorcycle_id),
  CHECK ((started_at IS NULL) = (started_original_offset IS NULL)),
  CHECK ((ended_at IS NULL) = (ended_original_offset IS NULL)),
  CHECK (started_missing_reason IS NULL OR (started_at IS NULL AND source = 'import')),
  CHECK (ended_missing_reason IS NULL OR (ended_at IS NULL AND source = 'import')),
  CHECK (
    status IN ('draft', 'cancelled')
    OR started_at IS NOT NULL
    OR started_missing_reason IS NOT NULL
  ),
  CHECK (
    status <> 'finished'
    OR ended_at IS NOT NULL
    OR ended_missing_reason IS NOT NULL
  )
) STRICT;

CREATE UNIQUE INDEX ux_trip_active
  ON trip (motorcycle_id) WHERE status IN ('recording', 'paused', 'interrupted');
CREATE INDEX ix_trip_moto_time ON trip (motorcycle_id, started_at);

CREATE TABLE trip_segment (
  id TEXT NOT NULL PRIMARY KEY,
  trip_id TEXT NOT NULL REFERENCES trip (id) ON DELETE RESTRICT,
  sequence INTEGER NOT NULL CHECK (sequence >= 0),
  status TEXT NOT NULL CHECK (status IN ('open', 'closed', 'interrupted')),
  started_at TEXT NOT NULL CHECK (
    started_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(started_at, 1, 19)) IS substr(started_at, 1, 19)
    AND substr(started_at, 12, 2) <= '23'
  ),
  started_original_offset TEXT NOT NULL CHECK (
    started_original_offset GLOB '[+-][01][0-9]:[0-5][0-9]' AND substr(started_original_offset, 2) <= '14:00'
  ),
  ended_at TEXT CHECK (
    ended_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(ended_at, 1, 19)) IS substr(ended_at, 1, 19)
    AND substr(ended_at, 12, 2) <= '23'
  ),
  ended_original_offset TEXT CHECK (
    ended_original_offset GLOB '[+-][01][0-9]:[0-5][0-9]' AND substr(ended_original_offset, 2) <= '14:00'
  ),
  UNIQUE (trip_id, sequence),
  CHECK ((status = 'open') = (ended_at IS NULL)),
  CHECK ((ended_at IS NULL) = (ended_original_offset IS NULL))
) STRICT;

CREATE UNIQUE INDEX ux_segment_open ON trip_segment (trip_id) WHERE status = 'open';

CREATE TABLE location_point (
  id TEXT NOT NULL PRIMARY KEY,
  segment_id TEXT NOT NULL REFERENCES trip_segment (id) ON DELETE RESTRICT,
  sequence INTEGER NOT NULL CHECK (sequence >= 0),
  recorded_at TEXT NOT NULL CHECK (
    recorded_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(recorded_at, 1, 19)) IS substr(recorded_at, 1, 19)
    AND substr(recorded_at, 12, 2) <= '23'
  ),
  original_offset TEXT NOT NULL CHECK (
    original_offset GLOB '[+-][01][0-9]:[0-5][0-9]' AND substr(original_offset, 2) <= '14:00'
  ),
  monotonic_ms INTEGER NOT NULL CHECK (monotonic_ms >= 0),
  latitude REAL NOT NULL CHECK (latitude BETWEEN -90 AND 90),
  longitude REAL NOT NULL CHECK (longitude BETWEEN -180 AND 180),
  accuracy_m REAL CHECK (accuracy_m >= 0),
  accuracy_missing_reason TEXT CHECK (
    accuracy_missing_reason GLOB '[a-z]*' AND accuracy_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  altitude_m REAL,
  altitude_missing_reason TEXT CHECK (
    altitude_missing_reason GLOB '[a-z]*' AND altitude_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  speed_mps REAL CHECK (speed_mps >= 0),
  speed_missing_reason TEXT CHECK (
    speed_missing_reason GLOB '[a-z]*' AND speed_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  UNIQUE (segment_id, sequence),
  CHECK ((accuracy_m IS NULL) = (accuracy_missing_reason IS NOT NULL)),
  CHECK ((altitude_m IS NULL) = (altitude_missing_reason IS NOT NULL)),
  CHECK ((speed_mps IS NULL) = (speed_missing_reason IS NOT NULL))
) STRICT;

CREATE TRIGGER trip_demo_match
BEFORE INSERT ON trip
WHEN (NEW.source = 'demo') <> (SELECT is_demo FROM motorcycle WHERE id = NEW.motorcycle_id)
BEGIN
  SELECT RAISE(ABORT, 'demo data must not mix with real data');
END;

CREATE TRIGGER trip_transition
BEFORE UPDATE OF status ON trip
WHEN NEW.status <> OLD.status
  AND OLD.status || '>' || NEW.status NOT IN (
    'draft>recording',
    'draft>cancelled',
    'recording>paused',
    'recording>interrupted',
    'recording>finished',
    'recording>cancelled',
    'paused>recording',
    'paused>finished',
    'paused>cancelled',
    'interrupted>recording',
    'interrupted>paused',
    'interrupted>finished',
    'interrupted>cancelled'
  )
BEGIN
  SELECT RAISE(ABORT, 'invalid trip transition');
END;

CREATE TRIGGER trip_no_delete
BEFORE DELETE ON trip
BEGIN
  SELECT RAISE(ABORT, 'trip is permanent');
END;

CREATE TRIGGER trip_origin_fixed
BEFORE UPDATE OF motorcycle_id, source ON trip
BEGIN
  SELECT RAISE(ABORT, 'trip origin is fixed');
END;

CREATE TRIGGER segment_trip_fixed
BEFORE UPDATE OF trip_id ON trip_segment
BEGIN
  SELECT RAISE(ABORT, 'segment trip is fixed');
END;
```

## Importação e telemetria

`telemetry_import` identifica os bytes: um hash SHA-256, único no banco. Reimportar os mesmos bytes, mesmo renomeados, reutiliza o registro. A unicidade vale para todo o banco: os mesmos bytes não entram em outra moto. Moto de demonstração não recebe importação: seus dados usam sessão `demo`. `telemetry_parse_run` é uma leitura desses bytes por uma versão do parser, e só ela guarda versão, estado, encoding, delimitador, regra decimal e metadados. Leitura `failed` pode voltar a `pending`; `parsed` e `rejected` são finais. Cada leitura de CSV gera no máximo uma sessão.

`decimal_rules_json` diz o separador por coluna. No único arquivo observado, latitude e longitude usam vírgula e as demais colunas usam ponto; não se supõe isso para outros arquivos.

**Como lido e normalizado.** `raw_values_json` guarda o texto exato de cada célula, por nome de coluna, inclusive `"N"`, `"0.0"`, velocidade da roda em km/h e as colunas sem campo próprio. As colunas tipadas são a normalização: `speed_mps` é a velocidade da roda dividida por 3,6, `engine_temp_c` é a temperatura da água e `fuel_km_per_l` mantém 0.0 e 99.9, de significado [A VERIFICAR]. Célula ilegível ou fora de faixa fica NULL na coluna tipada com motivo e intacta em `raw_values_json`. `row_number` é a linha física do arquivo, a partir de 1.

`elapsed_ms` pode repetir ou regredir em arquivos futuros; a linha é mantida e gera `sample_issue`. Linha sem `elapsed` legível não gera amostra, só `sample_issue`.

**Identidade (D3).** O nome de usuário e o apelido do veículo permanecem só no arquivo bruto. `metadata_json` aceita apenas cinco chaves escalares, e o gatilho recusa qualquer outra. `raw_uri` deriva do hash. Se o valor do nome ou do apelido aparecer em título, comentário ou nome do arquivo, o importador omite esse campo; em `detail` e `error_summary` nunca se copia conteúdo de célula. A comparação com os valores do arquivo é do importador (T3.16).

```sql
CREATE TABLE telemetry_import (
  id TEXT NOT NULL PRIMARY KEY,
  motorcycle_id TEXT NOT NULL REFERENCES motorcycle (id) ON DELETE RESTRICT,
  source_hash TEXT NOT NULL UNIQUE
    CHECK (length(source_hash) = 64 AND source_hash NOT GLOB '*[^0-9a-f]*'),
  raw_uri TEXT NOT NULL CHECK (raw_uri = 'raw/' || source_hash || '.csv'),
  byte_count INTEGER NOT NULL CHECK (byte_count >= 0),
  source_file_name TEXT,
  source_file_name_missing_reason TEXT CHECK (
    source_file_name_missing_reason GLOB '[a-z]*' AND source_file_name_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  imported_at TEXT NOT NULL CHECK (
    imported_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(imported_at, 1, 19)) IS substr(imported_at, 1, 19)
    AND substr(imported_at, 12, 2) <= '23'
  ),
  UNIQUE (id, motorcycle_id),
  CHECK ((source_file_name IS NULL) = (source_file_name_missing_reason IS NOT NULL))
) STRICT;

CREATE TABLE telemetry_parse_run (
  id TEXT NOT NULL PRIMARY KEY,
  import_id TEXT NOT NULL,
  motorcycle_id TEXT NOT NULL,
  parser_version TEXT NOT NULL,
  status TEXT NOT NULL CHECK (status IN ('pending', 'parsed', 'rejected', 'failed')),
  encoding TEXT,
  delimiter TEXT,
  decimal_rules_json TEXT CHECK (
    json_valid(decimal_rules_json) AND json_type(decimal_rules_json) = 'object'
  ),
  metadata_json TEXT CHECK (
    json_valid(metadata_json) AND json_type(metadata_json) = 'object'
  ),
  started_at TEXT NOT NULL CHECK (
    started_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(started_at, 1, 19)) IS substr(started_at, 1, 19)
    AND substr(started_at, 12, 2) <= '23'
  ),
  finished_at TEXT CHECK (
    finished_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(finished_at, 1, 19)) IS substr(finished_at, 1, 19)
    AND substr(finished_at, 12, 2) <= '23'
  ),
  error_summary TEXT,
  UNIQUE (import_id, parser_version),
  UNIQUE (id, motorcycle_id),
  UNIQUE (id, import_id),
  FOREIGN KEY (import_id, motorcycle_id)
    REFERENCES telemetry_import (id, motorcycle_id) ON DELETE RESTRICT,
  CHECK ((status = 'pending') = (finished_at IS NULL)),
  CHECK (
    status <> 'parsed'
    OR (
      encoding IS NOT NULL
      AND delimiter IS NOT NULL
      AND decimal_rules_json IS NOT NULL
      AND metadata_json IS NOT NULL
    )
  ),
  CHECK ((status IN ('rejected', 'failed')) = (error_summary IS NOT NULL))
) STRICT;

CREATE TABLE telemetry_session (
  id TEXT NOT NULL PRIMARY KEY,
  motorcycle_id TEXT NOT NULL REFERENCES motorcycle (id) ON DELETE RESTRICT,
  source TEXT NOT NULL CHECK (source IN ('rideology_csv', 'ble', 'demo')),
  trip_id TEXT,
  parse_run_id TEXT UNIQUE,
  time_source TEXT NOT NULL
    CHECK (time_source IN ('csv_title', 'file_name', 'device', 'user', 'unknown')),
  started_at TEXT CHECK (
    started_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(started_at, 1, 19)) IS substr(started_at, 1, 19)
    AND substr(started_at, 12, 2) <= '23'
  ),
  started_missing_reason TEXT CHECK (
    started_missing_reason GLOB '[a-z]*' AND started_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  started_local TEXT
    CHECK (
      strftime('%Y-%m-%dT%H:%M:%S', started_local) IS started_local
      AND ifnull(substr(started_local, 12, 2) <= '23', 1)
    ),
  started_precision_s INTEGER CHECK (started_precision_s > 0),
  ended_at TEXT CHECK (
    ended_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(ended_at, 1, 19)) IS substr(ended_at, 1, 19)
    AND substr(ended_at, 12, 2) <= '23'
  ),
  ended_missing_reason TEXT CHECK (
    ended_missing_reason GLOB '[a-z]*' AND ended_missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  original_offset TEXT CHECK (
    original_offset GLOB '[+-][01][0-9]:[0-5][0-9]' AND substr(original_offset, 2) <= '14:00'
  ),
  UNIQUE (id, motorcycle_id),
  UNIQUE (id, parse_run_id),
  FOREIGN KEY (trip_id, motorcycle_id)
    REFERENCES trip (id, motorcycle_id) ON DELETE RESTRICT,
  FOREIGN KEY (parse_run_id, motorcycle_id)
    REFERENCES telemetry_parse_run (id, motorcycle_id) ON DELETE RESTRICT,
  CHECK ((source = 'rideology_csv') = (parse_run_id IS NOT NULL)),
  CHECK ((started_at IS NULL) = (started_missing_reason IS NOT NULL)),
  CHECK ((ended_at IS NULL) = (ended_missing_reason IS NOT NULL)),
  CHECK (ended_at IS NULL OR started_at IS NOT NULL),
  CHECK ((started_at IS NULL) = (original_offset IS NULL)),
  CHECK ((time_source = 'unknown') = (started_at IS NULL AND started_local IS NULL)),
  CHECK ((time_source = 'unknown') = (started_precision_s IS NULL))
) STRICT;

CREATE TABLE telemetry_sample (
  id TEXT NOT NULL PRIMARY KEY,
  session_id TEXT NOT NULL REFERENCES telemetry_session (id) ON DELETE RESTRICT,
  sequence INTEGER NOT NULL CHECK (sequence >= 0),
  row_number INTEGER CHECK (row_number >= 1),
  raw_values_json TEXT CHECK (
    json_valid(raw_values_json) AND json_type(raw_values_json) = 'object'
  ),
  elapsed_ms INTEGER NOT NULL CHECK (elapsed_ms >= 0),
  recorded_at TEXT CHECK (
    recorded_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(recorded_at, 1, 19)) IS substr(recorded_at, 1, 19)
    AND substr(recorded_at, 12, 2) <= '23'
  ),
  original_offset TEXT CHECK (
    original_offset GLOB '[+-][01][0-9]:[0-5][0-9]' AND substr(original_offset, 2) <= '14:00'
  ),
  latitude REAL CHECK (latitude BETWEEN -90 AND 90),
  longitude REAL CHECK (longitude BETWEEN -180 AND 180),
  rpm REAL CHECK (rpm >= 0),
  speed_mps REAL CHECK (speed_mps >= 0),
  gear TEXT CHECK (gear IN ('N', '1', '2', '3', '4', '5', '6')),
  engine_temp_c REAL,
  fuel_km_per_l REAL CHECK (fuel_km_per_l >= 0),
  missing_fields_json TEXT NOT NULL DEFAULT '{}' CHECK (
    json_valid(missing_fields_json) AND json_type(missing_fields_json) = 'object'
  ),
  UNIQUE (session_id, sequence),
  UNIQUE (session_id, row_number),
  UNIQUE (id, session_id),
  CHECK ((row_number IS NULL) = (raw_values_json IS NULL)),
  CHECK ((recorded_at IS NULL) = (original_offset IS NULL)),
  CHECK ((latitude IS NULL) = (longitude IS NULL)),
  CHECK ((recorded_at IS NULL) = ifnull(json_extract(missing_fields_json, '$.recorded_at') GLOB '[a-z]*', 0)),
  CHECK ((latitude IS NULL) = ifnull(json_extract(missing_fields_json, '$.latitude') GLOB '[a-z]*', 0)),
  CHECK ((longitude IS NULL) = ifnull(json_extract(missing_fields_json, '$.longitude') GLOB '[a-z]*', 0)),
  CHECK ((rpm IS NULL) = ifnull(json_extract(missing_fields_json, '$.rpm') GLOB '[a-z]*', 0)),
  CHECK ((speed_mps IS NULL) = ifnull(json_extract(missing_fields_json, '$.speed_mps') GLOB '[a-z]*', 0)),
  CHECK ((gear IS NULL) = ifnull(json_extract(missing_fields_json, '$.gear') GLOB '[a-z]*', 0)),
  CHECK ((engine_temp_c IS NULL) = ifnull(json_extract(missing_fields_json, '$.engine_temp_c') GLOB '[a-z]*', 0)),
  CHECK ((fuel_km_per_l IS NULL) = ifnull(json_extract(missing_fields_json, '$.fuel_km_per_l') GLOB '[a-z]*', 0))
) STRICT;

CREATE TABLE sample_issue (
  id TEXT NOT NULL PRIMARY KEY,
  import_id TEXT NOT NULL,
  parse_run_id TEXT NOT NULL,
  session_id TEXT,
  sample_id TEXT,
  scope TEXT NOT NULL CHECK (scope IN ('file', 'row')),
  row_number INTEGER CHECK (row_number >= 1),
  field TEXT,
  code TEXT NOT NULL CHECK (
    code GLOB '[a-z]*' AND code NOT GLOB '*[^a-z0-9_]*'
  ),
  detail TEXT NOT NULL,
  FOREIGN KEY (parse_run_id, import_id)
    REFERENCES telemetry_parse_run (id, import_id) ON DELETE RESTRICT,
  FOREIGN KEY (session_id, parse_run_id)
    REFERENCES telemetry_session (id, parse_run_id) ON DELETE RESTRICT,
  FOREIGN KEY (sample_id, session_id)
    REFERENCES telemetry_sample (id, session_id) ON DELETE RESTRICT,
  CHECK ((scope = 'row') = (row_number IS NOT NULL)),
  CHECK (sample_id IS NULL OR (session_id IS NOT NULL AND scope = 'row'))
) STRICT;

CREATE INDEX ix_session_moto_time ON telemetry_session (motorcycle_id, started_at);
CREATE INDEX ix_session_trip ON telemetry_session (trip_id);
CREATE INDEX ix_sample_session_elapsed ON telemetry_sample (session_id, elapsed_ms);
CREATE INDEX ix_issue_run_row ON sample_issue (parse_run_id, row_number);

CREATE TRIGGER import_not_demo
BEFORE INSERT ON telemetry_import
WHEN (SELECT is_demo FROM motorcycle WHERE id = NEW.motorcycle_id) = 1
BEGIN
  SELECT RAISE(ABORT, 'demo data must not mix with real data');
END;

CREATE TRIGGER parse_run_final
BEFORE UPDATE ON telemetry_parse_run
WHEN OLD.status IN ('parsed', 'rejected')
BEGIN
  SELECT RAISE(ABORT, 'parse run is final');
END;

CREATE TRIGGER parse_run_origin_fixed
BEFORE UPDATE OF import_id, motorcycle_id, parser_version ON telemetry_parse_run
BEGIN
  SELECT RAISE(ABORT, 'parse run origin is fixed');
END;

CREATE TRIGGER parse_run_metadata_insert
BEFORE INSERT ON telemetry_parse_run
WHEN NEW.metadata_json IS NOT NULL AND EXISTS (
  SELECT 1 FROM json_each(NEW.metadata_json)
  WHERE key NOT IN ('title', 'comment', 'distance_km', 'time_min', 'rideology_score')
    OR type IN ('object', 'array')
)
BEGIN
  SELECT RAISE(ABORT, 'metadata key not allowed');
END;

CREATE TRIGGER parse_run_metadata_update
BEFORE UPDATE OF metadata_json ON telemetry_parse_run
WHEN NEW.metadata_json IS NOT NULL AND EXISTS (
  SELECT 1 FROM json_each(NEW.metadata_json)
  WHERE key NOT IN ('title', 'comment', 'distance_km', 'time_min', 'rideology_score')
    OR type IN ('object', 'array')
)
BEGIN
  SELECT RAISE(ABORT, 'metadata key not allowed');
END;

CREATE TRIGGER session_demo_match
BEFORE INSERT ON telemetry_session
WHEN (NEW.source = 'demo') <> (SELECT is_demo FROM motorcycle WHERE id = NEW.motorcycle_id)
BEGIN
  SELECT RAISE(ABORT, 'demo data must not mix with real data');
END;

CREATE TRIGGER session_origin_fixed
BEFORE UPDATE OF motorcycle_id, source, parse_run_id ON telemetry_session
BEGIN
  SELECT RAISE(ABORT, 'session origin is fixed');
END;

CREATE TRIGGER sample_row_matches_source
BEFORE INSERT ON telemetry_sample
WHEN (NEW.row_number IS NOT NULL) <> (
  SELECT source = 'rideology_csv' FROM telemetry_session WHERE id = NEW.session_id
)
BEGIN
  SELECT RAISE(ABORT, 'row_number is required for csv and only for csv');
END;

CREATE TRIGGER sample_no_identity
BEFORE INSERT ON telemetry_sample
WHEN NEW.raw_values_json IS NOT NULL AND EXISTS (
  SELECT 1 FROM json_each(NEW.raw_values_json)
  WHERE lower(key) LIKE '%nickname%' OR lower(key) LIKE '%account%'
    OR type IN ('object', 'array')
)
BEGIN
  SELECT RAISE(ABORT, 'identity column not allowed');
END;
```

## Proteção do bruto e exclusão definitiva

`UPDATE` e `DELETE` em bruto são barrados. A exceção é a exclusão definitiva (D2), por este procedimento:

1. A interface mostra o alvo e o que será apagado, e pede que a pessoa digite a frase de confirmação.
2. O aplicativo insere `raw_deletion` com estado `authorized`. O alvo precisa existir; para importação, o hash precisa coincidir. `expected_rows` registra quantas linhas brutas existem e é conferido pelo banco.
3. Em uma transação, apaga os derivados que apontam para o alvo e depois o bruto. O gatilho só libera o bruto desse alvo. Enquanto a autorização está aberta, o alvo não recebe bruto, leitura, sessão, segmento nem derivado novo.
4. Na exclusão de importação, remove o arquivo bruto e seu material auxiliar.
5. Marca `completed`, aceito só se o bruto do alvo não existe mais e `deleted_rows` é igual a `expected_rows`. Cancelar só é aceito se nada foi apagado. O registro nunca é alterado depois nem apagado.

Se o aplicativo morrer no meio, a autorização aberta é retomada na próxima abertura, com nova confirmação na tela, ou cancelada se nada foi apagado. O alvo `trip` apaga os pontos e mantém a viagem, que depois de concluída a exclusão não recebe mais pontos nem segmentos. Não há exclusão por prazo: retenção segue aberta.

```sql
CREATE TABLE raw_deletion (
  id TEXT NOT NULL PRIMARY KEY,
  target_kind TEXT NOT NULL CHECK (target_kind IN ('telemetry_import', 'trip')),
  target_id TEXT NOT NULL,
  source_hash TEXT,
  reason TEXT NOT NULL CHECK (length(trim(reason)) > 0),
  confirmation TEXT NOT NULL CHECK (confirmation = 'EXCLUIR DEFINITIVAMENTE'),
  confirmed_at TEXT NOT NULL CHECK (
    confirmed_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(confirmed_at, 1, 19)) IS substr(confirmed_at, 1, 19)
    AND substr(confirmed_at, 12, 2) <= '23'
  ),
  expected_rows INTEGER NOT NULL CHECK (expected_rows >= 0),
  status TEXT NOT NULL CHECK (status IN ('authorized', 'completed', 'cancelled')),
  deleted_rows INTEGER CHECK (deleted_rows >= 0),
  raw_file_deleted_at TEXT CHECK (
    raw_file_deleted_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(raw_file_deleted_at, 1, 19)) IS substr(raw_file_deleted_at, 1, 19)
    AND substr(raw_file_deleted_at, 12, 2) <= '23'
  ),
  finished_at TEXT CHECK (
    finished_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(finished_at, 1, 19)) IS substr(finished_at, 1, 19)
    AND substr(finished_at, 12, 2) <= '23'
  ),
  CHECK ((target_kind = 'telemetry_import') = (source_hash IS NOT NULL)),
  CHECK ((status = 'authorized') = (finished_at IS NULL)),
  CHECK (status <> 'completed' OR ifnull(deleted_rows = expected_rows, 0)),
  CHECK (status <> 'cancelled' OR (deleted_rows IS NULL AND raw_file_deleted_at IS NULL)),
  CHECK (
    status <> 'completed'
    OR target_kind <> 'telemetry_import'
    OR raw_file_deleted_at IS NOT NULL
  )
) STRICT;

CREATE UNIQUE INDEX ux_raw_deletion_open
  ON raw_deletion (target_kind, target_id) WHERE status = 'authorized';

CREATE TRIGGER raw_deletion_insert
BEFORE INSERT ON raw_deletion
WHEN NEW.status <> 'authorized'
  OR (
    NEW.target_kind = 'telemetry_import' AND NOT EXISTS (
      SELECT 1 FROM telemetry_import
      WHERE id = NEW.target_id AND source_hash = NEW.source_hash
    )
  )
  OR (
    NEW.target_kind = 'trip' AND NOT EXISTS (
      SELECT 1 FROM trip
      WHERE id = NEW.target_id AND status IN ('finished', 'cancelled')
    )
  )
BEGIN
  SELECT RAISE(ABORT, 'invalid deletion target');
END;

CREATE TRIGGER raw_deletion_count
BEFORE INSERT ON raw_deletion
WHEN NEW.expected_rows <> CASE NEW.target_kind
    WHEN 'telemetry_import' THEN (
      SELECT count(*)
      FROM telemetry_sample AS a
      JOIN telemetry_session AS s ON s.id = a.session_id
      JOIN telemetry_parse_run AS r ON r.id = s.parse_run_id
      WHERE r.import_id = NEW.target_id
    )
    ELSE (
      SELECT count(*)
      FROM location_point AS p
      JOIN trip_segment AS s ON s.id = p.segment_id
      WHERE s.trip_id = NEW.target_id
    )
  END
BEGIN
  SELECT RAISE(ABORT, 'expected_rows differs from stored rows');
END;

CREATE TRIGGER raw_deletion_update
BEFORE UPDATE ON raw_deletion
WHEN OLD.status <> 'authorized'
  OR NEW.status = 'authorized'
  OR NEW.id IS NOT OLD.id
  OR NEW.target_kind IS NOT OLD.target_kind
  OR NEW.target_id IS NOT OLD.target_id
  OR NEW.source_hash IS NOT OLD.source_hash
  OR NEW.reason IS NOT OLD.reason
  OR NEW.confirmation IS NOT OLD.confirmation
  OR NEW.confirmed_at IS NOT OLD.confirmed_at
  OR NEW.expected_rows IS NOT OLD.expected_rows
BEGIN
  SELECT RAISE(ABORT, 'deletion record is fixed');
END;

CREATE TRIGGER raw_deletion_complete
BEFORE UPDATE ON raw_deletion
WHEN NEW.status = 'completed' AND (
  (
    NEW.target_kind = 'telemetry_import'
    AND EXISTS (SELECT 1 FROM telemetry_import WHERE id = NEW.target_id)
  )
  OR (
    NEW.target_kind = 'trip' AND EXISTS (
      SELECT 1
      FROM location_point AS p
      JOIN trip_segment AS s ON s.id = p.segment_id
      WHERE s.trip_id = NEW.target_id
    )
  )
)
BEGIN
  SELECT RAISE(ABORT, 'raw data still exists');
END;

CREATE TRIGGER raw_deletion_no_derived_left
BEFORE UPDATE ON raw_deletion
WHEN NEW.status = 'completed' AND NEW.target_kind = 'trip' AND (
  EXISTS (SELECT 1 FROM metric_result WHERE trip_id = NEW.target_id)
  OR EXISTS (SELECT 1 FROM finding WHERE trip_id = NEW.target_id)
  OR EXISTS (SELECT 1 FROM report WHERE trip_id = NEW.target_id)
)
BEGIN
  SELECT RAISE(ABORT, 'derived data still exists');
END;

CREATE TRIGGER raw_deletion_cancel
BEFORE UPDATE ON raw_deletion
WHEN NEW.status = 'cancelled' AND (
  NEW.expected_rows <> CASE NEW.target_kind
    WHEN 'telemetry_import' THEN (
      SELECT count(*)
      FROM telemetry_sample AS a
      JOIN telemetry_session AS s ON s.id = a.session_id
      JOIN telemetry_parse_run AS r ON r.id = s.parse_run_id
      WHERE r.import_id = NEW.target_id
    )
    ELSE (
      SELECT count(*)
      FROM location_point AS p
      JOIN trip_segment AS s ON s.id = p.segment_id
      WHERE s.trip_id = NEW.target_id
    )
  END
  OR (
    NEW.target_kind = 'telemetry_import'
    AND NOT EXISTS (SELECT 1 FROM telemetry_import WHERE id = NEW.target_id)
  )
)
BEGIN
  SELECT RAISE(ABORT, 'raw data already deleted');
END;

CREATE TRIGGER raw_deletion_no_delete
BEFORE DELETE ON raw_deletion
BEGIN
  SELECT RAISE(ABORT, 'deletion record is permanent');
END;

CREATE VIEW open_deletion_session AS
SELECT s.id AS session_id, s.trip_id AS trip_id
FROM raw_deletion AS d
JOIN telemetry_parse_run AS r ON r.import_id = d.target_id
JOIN telemetry_session AS s ON s.parse_run_id = r.id
WHERE d.target_kind = 'telemetry_import' AND d.status = 'authorized';

CREATE VIEW open_deletion_trip AS
SELECT target_id AS trip_id
FROM raw_deletion
WHERE target_kind = 'trip' AND status = 'authorized'
UNION
SELECT trip_id
FROM open_deletion_session
WHERE trip_id IS NOT NULL;

CREATE VIEW open_deletion_motorcycle AS
SELECT t.motorcycle_id AS motorcycle_id
FROM raw_deletion AS d
JOIN trip AS t ON t.id = d.target_id
WHERE d.target_kind = 'trip' AND d.status = 'authorized'
UNION
SELECT i.motorcycle_id
FROM raw_deletion AS d
JOIN telemetry_import AS i ON i.id = d.target_id
WHERE d.target_kind = 'telemetry_import' AND d.status = 'authorized';

CREATE TRIGGER location_insert_open_deletion
BEFORE INSERT ON location_point
WHEN EXISTS (
  SELECT 1
  FROM raw_deletion AS d
  JOIN trip_segment AS s ON s.trip_id = d.target_id
  WHERE s.id = NEW.segment_id AND d.target_kind = 'trip' AND d.status <> 'cancelled'
)
BEGIN
  SELECT RAISE(ABORT, 'target has a deletion');
END;

CREATE TRIGGER segment_insert_open_deletion
BEFORE INSERT ON trip_segment
WHEN EXISTS (
  SELECT 1 FROM raw_deletion AS d
  WHERE d.target_id = NEW.trip_id AND d.target_kind = 'trip' AND d.status <> 'cancelled'
)
BEGIN
  SELECT RAISE(ABORT, 'target has a deletion');
END;

CREATE TRIGGER parse_run_insert_open_deletion
BEFORE INSERT ON telemetry_parse_run
WHEN EXISTS (
  SELECT 1 FROM raw_deletion AS d
  WHERE d.target_id = NEW.import_id
    AND d.target_kind = 'telemetry_import'
    AND d.status = 'authorized'
)
BEGIN
  SELECT RAISE(ABORT, 'target has a deletion');
END;

CREATE TRIGGER session_insert_open_deletion
BEFORE INSERT ON telemetry_session
WHEN EXISTS (
  SELECT 1
  FROM raw_deletion AS d
  JOIN telemetry_parse_run AS r ON r.import_id = d.target_id
  WHERE r.id = NEW.parse_run_id
    AND d.target_kind = 'telemetry_import'
    AND d.status = 'authorized'
)
BEGIN
  SELECT RAISE(ABORT, 'target has a deletion');
END;

CREATE TRIGGER sample_insert_open_deletion
BEFORE INSERT ON telemetry_sample
WHEN EXISTS (
  SELECT 1 FROM open_deletion_session WHERE session_id = NEW.session_id
)
BEGIN
  SELECT RAISE(ABORT, 'target has a deletion');
END;

CREATE TRIGGER parse_run_no_delete
BEFORE DELETE ON telemetry_parse_run
WHEN NOT EXISTS (
  SELECT 1 FROM raw_deletion AS d
  WHERE d.target_id = OLD.import_id
    AND d.target_kind = 'telemetry_import'
    AND d.status = 'authorized'
)
BEGIN
  SELECT RAISE(ABORT, 'parse run deletion not authorized');
END;

CREATE TRIGGER location_no_update
BEFORE UPDATE ON location_point
BEGIN
  SELECT RAISE(ABORT, 'raw location is immutable');
END;

CREATE TRIGGER location_no_delete
BEFORE DELETE ON location_point
WHEN NOT EXISTS (
  SELECT 1
  FROM raw_deletion AS d
  JOIN trip_segment AS s ON s.trip_id = d.target_id
  WHERE s.id = OLD.segment_id AND d.target_kind = 'trip' AND d.status = 'authorized'
)
BEGIN
  SELECT RAISE(ABORT, 'raw location deletion not authorized');
END;

CREATE TRIGGER sample_no_update
BEFORE UPDATE ON telemetry_sample
BEGIN
  SELECT RAISE(ABORT, 'raw sample is immutable');
END;

CREATE TRIGGER sample_no_delete
BEFORE DELETE ON telemetry_sample
WHEN NOT EXISTS (
  SELECT 1
  FROM raw_deletion AS d
  JOIN telemetry_parse_run AS r ON r.import_id = d.target_id
  JOIN telemetry_session AS s ON s.parse_run_id = r.id
  WHERE s.id = OLD.session_id
    AND d.target_kind = 'telemetry_import'
    AND d.status = 'authorized'
)
BEGIN
  SELECT RAISE(ABORT, 'raw sample deletion not authorized');
END;

CREATE TRIGGER import_no_update
BEFORE UPDATE ON telemetry_import
BEGIN
  SELECT RAISE(ABORT, 'raw import is immutable');
END;

CREATE TRIGGER import_no_delete
BEFORE DELETE ON telemetry_import
WHEN NOT EXISTS (
  SELECT 1 FROM raw_deletion AS d
  WHERE d.target_id = OLD.id
    AND d.target_kind = 'telemetry_import'
    AND d.status = 'authorized'
)
BEGIN
  SELECT RAISE(ABORT, 'raw import deletion not authorized');
END;
```

Amostra de sessão BLE ou de demonstração não pertence a uma importação e por isso não tem exclusão autorizada nesta versão; o BLE é da F6 e na demonstração os pontos de viagem podem ser excluídos, mas as amostras não. Um novo tipo de alvo será necessário se isso for preciso.

## Validade do GPS e distância oficial

A validade é derivada e versionada, separada do ponto bruto, para que mudar um limiar não altere o bruto. Só `valid` entra na distância oficial.

| Situação | Avaliação | Motivo |
|---|---|---|
| GPS do aplicativo sem precisão | `invalid` | `no_accuracy` |
| GPS do aplicativo com precisão pior que 25 m [PROPOSTA] | `invalid` | `low_accuracy` |
| Sem coordenada, ou indicação de sem sinal [A VERIFICAR] no CSV | `invalid` | `no_signal` |
| Três ou mais coordenadas idênticas com roda acima de 5 km/h [PROPOSTA] | `suspect` | `frozen` |
| Velocidade implícita acima de 90 m/s [PROPOSTA] | `suspect` | `jump` |
| Tempo repetido ou regressivo | `invalid` | `time_regression` |
| Nenhuma das anteriores | `valid` | — |

O GPS do CSV não traz precisão; por isso ela não é exigida dele. Congelamento do GPS do aplicativo usa a velocidade do próprio ponto, e o critério é [A VERIFICAR] no spike S1.

Regra de distância (D1): soma-se a distância entre pontos consecutivos, ambos `valid`, no mesmo segmento ou sessão, com intervalo de até 10 s no aplicativo e 3 s no CSV [PROPOSTA]. Nada é interpolado: trecho congelado, com salto ou sem sinal fica fora. Se o aplicativo tem ao menos um par utilizável na viagem, a fonte oficial é só `gps_app`; senão, `gps_csv`. As fontes não são combinadas na mesma viagem. Sem par utilizável, a distância é ausente com motivo.

`coverage_json` da distância oficial traz `source`, `covered_ms` e `excluded_ms`, e a interface mostra a cobertura junto do valor. A escolha da fonte é do aplicativo (T3.12).

```sql
CREATE TABLE gps_evaluation (
  id TEXT NOT NULL PRIMARY KEY,
  location_point_id TEXT REFERENCES location_point (id) ON DELETE RESTRICT,
  sample_id TEXT REFERENCES telemetry_sample (id) ON DELETE RESTRICT,
  rule_version TEXT NOT NULL,
  validity TEXT NOT NULL CHECK (validity IN ('valid', 'suspect', 'invalid')),
  reason TEXT CHECK (
    reason GLOB '[a-z]*' AND reason NOT GLOB '*[^a-z0-9_]*'
  ),
  evaluated_at TEXT NOT NULL CHECK (
    evaluated_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(evaluated_at, 1, 19)) IS substr(evaluated_at, 1, 19)
    AND substr(evaluated_at, 12, 2) <= '23'
  ),
  UNIQUE (location_point_id, rule_version),
  UNIQUE (sample_id, rule_version),
  CHECK ((location_point_id IS NULL) <> (sample_id IS NULL)),
  CHECK ((validity = 'valid') = (reason IS NULL))
) STRICT;

CREATE TRIGGER gps_valid_needs_data
BEFORE INSERT ON gps_evaluation
WHEN NEW.validity = 'valid' AND (
  EXISTS (
    SELECT 1 FROM location_point
    WHERE id = NEW.location_point_id AND accuracy_m IS NULL
  )
  OR EXISTS (
    SELECT 1 FROM telemetry_sample
    WHERE id = NEW.sample_id AND latitude IS NULL
  )
)
BEGIN
  SELECT RAISE(ABORT, 'valid gps needs accuracy (app) or coordinates (csv)');
END;

CREATE TRIGGER gps_evaluation_insert_open_deletion
BEFORE INSERT ON gps_evaluation
WHEN EXISTS (
  SELECT 1
  FROM location_point AS p
  JOIN trip_segment AS s ON s.id = p.segment_id
  JOIN raw_deletion AS d ON d.target_id = s.trip_id
  WHERE p.id = NEW.location_point_id AND d.target_kind = 'trip' AND d.status = 'authorized'
)
OR EXISTS (
  SELECT 1
  FROM telemetry_sample AS a
  JOIN open_deletion_session AS o ON o.session_id = a.session_id
  WHERE a.id = NEW.sample_id
)
BEGIN
  SELECT RAISE(ABORT, 'target has a deletion');
END;

CREATE TRIGGER gps_evaluation_no_delete
BEFORE DELETE ON gps_evaluation
WHEN NOT EXISTS (
  SELECT 1
  FROM location_point AS p
  JOIN trip_segment AS s ON s.id = p.segment_id
  JOIN raw_deletion AS d ON d.target_id = s.trip_id
  WHERE p.id = OLD.location_point_id AND d.target_kind = 'trip' AND d.status = 'authorized'
)
AND NOT EXISTS (
  SELECT 1
  FROM telemetry_sample AS a
  JOIN open_deletion_session AS o ON o.session_id = a.session_id
  WHERE a.id = OLD.sample_id
)
BEGIN
  SELECT RAISE(ABORT, 'derived deletion not authorized');
END;

CREATE TRIGGER gps_evaluation_no_update
BEFORE UPDATE ON gps_evaluation
BEGIN
  SELECT RAISE(ABORT, 'derived data is versioned');
END;
```

## Derivados

Derivado não é alterado, apagado nem substituído por `INSERT OR REPLACE` (com `recursive_triggers` ligado, regra geral 8). A única exclusão possível ocorre com autorização aberta em `raw_deletion`:

| Derivado | Pode ser apagado quando |
|---|---|
| `gps_evaluation` | seu ponto é da viagem alvo, ou sua amostra é da importação alvo |
| `metric_result` | é da viagem alvo, de sessão da importação alvo ou de viagem vinculada a essa sessão |
| `finding` | como a métrica, por vínculo direto ou pela métrica citada |
| `report` | é da viagem coberta; relatórios de período e manutenção, se a moto tem autorização aberta |
| `baseline_profile` | nunca: é agregado, sem referência a linhas brutas; nova versão substitui a anterior no uso |

Enquanto a autorização está aberta não entra derivado novo do alvo, o que também impede a substituição nessa janela. Escolher quais relatórios de período contêm o alvo é do aplicativo (T3.17, T7.3).

`calculation_key` é o SHA-256 da entrada normalizada (identificadores e hashes das fontes, versão da regra de GPS e parâmetros). Mesma entrada e mesma `method_version` produzem a mesma chave: recalcular não duplica. Mudou a entrada ou o método, nasce outra linha e a anterior permanece.

Tipos reservados de `metric_type`: `distance_official`, `distance_wheel`, `distance_rideology` (todas em `m`, lado a lado, só a primeira é oficial), `duration` (`s`), `fuel_l_per_100km` e `stop_duration` (`s`, com os intervalos em `evidence_json`). A distância e o tempo do cabeçalho do CSV são comparativos.

```sql
CREATE TABLE metric_result (
  id TEXT NOT NULL PRIMARY KEY,
  motorcycle_id TEXT NOT NULL REFERENCES motorcycle (id) ON DELETE RESTRICT,
  trip_id TEXT,
  session_id TEXT,
  metric_type TEXT NOT NULL,
  value REAL,
  missing_reason TEXT CHECK (
    missing_reason GLOB '[a-z]*' AND missing_reason NOT GLOB '*[^a-z0-9_]*'
  ),
  unit TEXT NOT NULL,
  method_version TEXT NOT NULL,
  calculation_key TEXT NOT NULL,
  coverage_json TEXT NOT NULL CHECK (
    json_valid(coverage_json) AND json_type(coverage_json) = 'object'
  ),
  evidence_json TEXT NOT NULL CHECK (
    json_valid(evidence_json) AND json_type(evidence_json) = 'object'
  ),
  calculated_at TEXT NOT NULL CHECK (
    calculated_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(calculated_at, 1, 19)) IS substr(calculated_at, 1, 19)
    AND substr(calculated_at, 12, 2) <= '23'
  ),
  UNIQUE (id, motorcycle_id),
  FOREIGN KEY (trip_id, motorcycle_id)
    REFERENCES trip (id, motorcycle_id) ON DELETE RESTRICT,
  FOREIGN KEY (session_id, motorcycle_id)
    REFERENCES telemetry_session (id, motorcycle_id) ON DELETE RESTRICT,
  CHECK ((trip_id IS NULL) <> (session_id IS NULL)),
  CHECK ((value IS NULL) = (missing_reason IS NOT NULL)),
  CHECK (metric_type NOT LIKE 'distance_%' OR unit = 'm'),
  CHECK (
    metric_type <> 'distance_official'
    OR value IS NULL
    OR (
      value >= 0
      AND ifnull(json_extract(coverage_json, '$.source') IN ('gps_app', 'gps_csv'), 0)
      AND ifnull(json_type(coverage_json, '$.covered_ms') = 'integer', 0)
      AND ifnull(json_type(coverage_json, '$.excluded_ms') = 'integer', 0)
    )
  )
) STRICT;

CREATE UNIQUE INDEX ux_metric_trip
  ON metric_result (trip_id, metric_type, method_version, calculation_key)
  WHERE trip_id IS NOT NULL;
CREATE UNIQUE INDEX ux_metric_session
  ON metric_result (session_id, metric_type, method_version, calculation_key)
  WHERE session_id IS NOT NULL;

CREATE TABLE baseline_profile (
  id TEXT NOT NULL PRIMARY KEY,
  motorcycle_id TEXT NOT NULL REFERENCES motorcycle (id) ON DELETE RESTRICT,
  metric_type TEXT NOT NULL,
  cohort_key TEXT NOT NULL,
  cohort_definition_json TEXT NOT NULL CHECK (
    json_valid(cohort_definition_json) AND json_type(cohort_definition_json) = 'object'
  ),
  sample_count INTEGER NOT NULL CHECK (sample_count >= 0),
  p50 REAL,
  p90 REAL,
  mean REAL,
  stddev REAL CHECK (stddev >= 0),
  missing_fields_json TEXT NOT NULL DEFAULT '{}' CHECK (
    json_valid(missing_fields_json) AND json_type(missing_fields_json) = 'object'
  ),
  version TEXT NOT NULL,
  calculated_at TEXT NOT NULL CHECK (
    calculated_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(calculated_at, 1, 19)) IS substr(calculated_at, 1, 19)
    AND substr(calculated_at, 12, 2) <= '23'
  ),
  UNIQUE (id, motorcycle_id),
  UNIQUE (motorcycle_id, metric_type, cohort_key, version),
  CHECK ((p50 IS NULL) = ifnull(json_extract(missing_fields_json, '$.p50') GLOB '[a-z]*', 0)),
  CHECK ((p90 IS NULL) = ifnull(json_extract(missing_fields_json, '$.p90') GLOB '[a-z]*', 0)),
  CHECK ((mean IS NULL) = ifnull(json_extract(missing_fields_json, '$.mean') GLOB '[a-z]*', 0)),
  CHECK ((stddev IS NULL) = ifnull(json_extract(missing_fields_json, '$.stddev') GLOB '[a-z]*', 0))
) STRICT;

CREATE TABLE finding (
  id TEXT NOT NULL PRIMARY KEY,
  motorcycle_id TEXT NOT NULL REFERENCES motorcycle (id) ON DELETE RESTRICT,
  trip_id TEXT,
  session_id TEXT,
  baseline_profile_id TEXT,
  metric_result_id TEXT,
  metric_type TEXT NOT NULL,
  kind TEXT NOT NULL
    CHECK (kind IN ('measurement', 'deviation', 'hypothesis', 'recommendation')),
  observed_value REAL,
  expected_value REAL,
  deviation REAL,
  confidence REAL CHECK (confidence BETWEEN 0 AND 1),
  missing_fields_json TEXT NOT NULL DEFAULT '{}' CHECK (
    json_valid(missing_fields_json) AND json_type(missing_fields_json) = 'object'
  ),
  evidence_json TEXT NOT NULL CHECK (
    json_valid(evidence_json) AND json_type(evidence_json) = 'object'
  ),
  algorithm_version TEXT NOT NULL,
  created_at TEXT NOT NULL CHECK (
    created_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(created_at, 1, 19)) IS substr(created_at, 1, 19)
    AND substr(created_at, 12, 2) <= '23'
  ),
  FOREIGN KEY (trip_id, motorcycle_id)
    REFERENCES trip (id, motorcycle_id) ON DELETE RESTRICT,
  FOREIGN KEY (session_id, motorcycle_id)
    REFERENCES telemetry_session (id, motorcycle_id) ON DELETE RESTRICT,
  FOREIGN KEY (baseline_profile_id, motorcycle_id)
    REFERENCES baseline_profile (id, motorcycle_id) ON DELETE RESTRICT,
  FOREIGN KEY (metric_result_id, motorcycle_id)
    REFERENCES metric_result (id, motorcycle_id) ON DELETE RESTRICT,
  CHECK (
    kind <> 'deviation'
    OR (
      baseline_profile_id IS NOT NULL
      AND observed_value IS NOT NULL
      AND expected_value IS NOT NULL
      AND deviation IS NOT NULL
    )
  ),
  CHECK ((observed_value IS NULL) = ifnull(json_extract(missing_fields_json, '$.observed_value') GLOB '[a-z]*', 0)),
  CHECK ((expected_value IS NULL) = ifnull(json_extract(missing_fields_json, '$.expected_value') GLOB '[a-z]*', 0)),
  CHECK ((deviation IS NULL) = ifnull(json_extract(missing_fields_json, '$.deviation') GLOB '[a-z]*', 0)),
  CHECK ((confidence IS NULL) = ifnull(json_extract(missing_fields_json, '$.confidence') GLOB '[a-z]*', 0))
) STRICT;

CREATE TABLE report (
  id TEXT NOT NULL PRIMARY KEY,
  motorcycle_id TEXT NOT NULL REFERENCES motorcycle (id) ON DELETE RESTRICT,
  scope TEXT NOT NULL CHECK (scope IN ('trip', 'period', 'maintenance')),
  trip_id TEXT,
  period_start TEXT CHECK (
    period_start GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(period_start, 1, 19)) IS substr(period_start, 1, 19)
    AND substr(period_start, 12, 2) <= '23'
  ),
  period_end TEXT CHECK (
    period_end GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(period_end, 1, 19)) IS substr(period_end, 1, 19)
    AND substr(period_end, 12, 2) <= '23'
  ),
  period_original_offset TEXT CHECK (
    period_original_offset GLOB '[+-][01][0-9]:[0-5][0-9]' AND substr(period_original_offset, 2) <= '14:00'
  ),
  structured_input_json TEXT NOT NULL CHECK (
    json_valid(structured_input_json) AND json_type(structured_input_json) = 'object'
  ),
  input_hash TEXT NOT NULL,
  generation_version TEXT NOT NULL,
  narrative TEXT NOT NULL,
  uncertainty_json TEXT NOT NULL CHECK (
    json_valid(uncertainty_json) AND json_type(uncertainty_json) = 'object'
  ),
  generated_at TEXT NOT NULL CHECK (
    generated_at GLOB '???????????????????.[0-9][0-9][0-9]Z'
    AND strftime('%Y-%m-%dT%H:%M:%S', substr(generated_at, 1, 19)) IS substr(generated_at, 1, 19)
    AND substr(generated_at, 12, 2) <= '23'
  ),
  FOREIGN KEY (trip_id, motorcycle_id)
    REFERENCES trip (id, motorcycle_id) ON DELETE RESTRICT,
  CHECK ((scope = 'trip') = (trip_id IS NOT NULL)),
  CHECK ((scope = 'period') = (period_start IS NOT NULL)),
  CHECK ((period_start IS NULL) = (period_end IS NULL)),
  CHECK ((period_start IS NULL) = (period_original_offset IS NULL)),
  CHECK (period_end > period_start)
) STRICT;

CREATE TRIGGER metric_no_update
BEFORE UPDATE ON metric_result
BEGIN
  SELECT RAISE(ABORT, 'derived data is versioned');
END;

CREATE TRIGGER baseline_no_update
BEFORE UPDATE ON baseline_profile
BEGIN
  SELECT RAISE(ABORT, 'derived data is versioned');
END;

CREATE TRIGGER finding_no_update
BEFORE UPDATE ON finding
BEGIN
  SELECT RAISE(ABORT, 'derived data is versioned');
END;

CREATE TRIGGER report_no_update
BEFORE UPDATE ON report
BEGIN
  SELECT RAISE(ABORT, 'derived data is versioned');
END;

CREATE TRIGGER metric_insert_open_deletion
BEFORE INSERT ON metric_result
WHEN EXISTS (SELECT 1 FROM open_deletion_trip WHERE trip_id = NEW.trip_id)
OR EXISTS (SELECT 1 FROM open_deletion_session WHERE session_id = NEW.session_id)
BEGIN
  SELECT RAISE(ABORT, 'target has a deletion');
END;

CREATE TRIGGER metric_no_delete
BEFORE DELETE ON metric_result
WHEN NOT EXISTS (SELECT 1 FROM open_deletion_trip WHERE trip_id = OLD.trip_id)
AND NOT EXISTS (SELECT 1 FROM open_deletion_session WHERE session_id = OLD.session_id)
BEGIN
  SELECT RAISE(ABORT, 'derived deletion not authorized');
END;

CREATE TRIGGER finding_insert_open_deletion
BEFORE INSERT ON finding
WHEN EXISTS (SELECT 1 FROM open_deletion_trip WHERE trip_id = NEW.trip_id)
OR EXISTS (SELECT 1 FROM open_deletion_session WHERE session_id = NEW.session_id)
OR EXISTS (
  SELECT 1
  FROM metric_result AS m
  WHERE m.id = NEW.metric_result_id
    AND (
      m.trip_id IN (SELECT trip_id FROM open_deletion_trip)
      OR m.session_id IN (SELECT session_id FROM open_deletion_session)
    )
)
BEGIN
  SELECT RAISE(ABORT, 'target has a deletion');
END;

CREATE TRIGGER finding_no_delete
BEFORE DELETE ON finding
WHEN NOT EXISTS (SELECT 1 FROM open_deletion_trip WHERE trip_id = OLD.trip_id)
AND NOT EXISTS (SELECT 1 FROM open_deletion_session WHERE session_id = OLD.session_id)
AND NOT EXISTS (
  SELECT 1
  FROM metric_result AS m
  WHERE m.id = OLD.metric_result_id
    AND (
      m.trip_id IN (SELECT trip_id FROM open_deletion_trip)
      OR m.session_id IN (SELECT session_id FROM open_deletion_session)
    )
)
BEGIN
  SELECT RAISE(ABORT, 'derived deletion not authorized');
END;

CREATE TRIGGER report_insert_open_deletion
BEFORE INSERT ON report
WHEN EXISTS (SELECT 1 FROM open_deletion_motorcycle WHERE motorcycle_id = NEW.motorcycle_id)
BEGIN
  SELECT RAISE(ABORT, 'target has a deletion');
END;

CREATE TRIGGER report_no_delete
BEFORE DELETE ON report
WHEN NOT EXISTS (SELECT 1 FROM open_deletion_trip WHERE trip_id = OLD.trip_id)
AND (
  OLD.scope = 'trip'
  OR NOT EXISTS (SELECT 1 FROM open_deletion_motorcycle WHERE motorcycle_id = OLD.motorcycle_id)
)
BEGIN
  SELECT RAISE(ABORT, 'derived deletion not authorized');
END;

CREATE TRIGGER baseline_no_delete
BEFORE DELETE ON baseline_profile
BEGIN
  SELECT RAISE(ABORT, 'derived data is versioned');
END;
```

## Regras que o banco não garante

| Regra | Quem garante | Teste |
|---|---|---|
| Formato UUID | aplicativo | T1.16 |
| Pragmas em toda conexão | adaptador SQLite | T1.17 |
| Guardas das transições (permissão, confirmação) | aplicação | T2.8 |
| Apelido do cadastro nunca vir do CSV | importador | T3.16 |
| Quais relatórios de período contêm o alvo da exclusão | aplicação | T3.17, T7.3 |
| Limiares de validade do GPS | domínio | T2.10, T3.11 |
| Prioridade do GPS do aplicativo | aplicação | T3.12 |
| Identidade ausente em textos livres, logs e exportação | importador e exportador | T3.16, T7.4, T7.5 |
| Igualdade entre hash registrado e bytes do arquivo | importador | T3.1 |
| Evidência não vazia e verdadeira nos achados | domínio | T4.6 |
| Remoção do arquivo físico na exclusão definitiva | adaptador de arquivos | T3.17 |

## Justificativa de cada objeto

Objeto sem requisito ou decisão não entra no esquema. Índices só aceleram uma consulta ou garantem unicidade.

| Tabela | Justificativa |
|---|---|
| `motorcycle` | RF-01, D3 |
| `odometer_reading` | RF-05, A4 |
| `maintenance_record` | RF-02, A3, A4 |
| `maintenance_plan` | RF-04, A4 |
| `plan_item` | RF-04, A4 |
| `maintenance_item` | RF-03, A3, A4 |
| `record_plan` | RF-04, A4 |
| `trip` | RF-07, A5 |
| `trip_segment` | RF-08, A5 |
| `location_point` | RF-09, A8 |
| `telemetry_import` | RF-10, A8 |
| `telemetry_parse_run` | RNF-03, A8 |
| `telemetry_session` | RF-11, RF-27, A6 |
| `telemetry_sample` | RF-12, RF-16, A7 |
| `sample_issue` | RF-10 |
| `raw_deletion` | RF-25, D2 |
| `gps_evaluation` | RF-09, RF-23, D1, P3 |
| `metric_result` | RF-12, RF-19, RNF-03, A9 |
| `baseline_profile` | RF-13 |
| `finding` | RF-13 |
| `report` | RF-14, RF-21 |

| Gatilho | Justificativa |
|---|---|
| `odometer_no_regression` | RF-05, A4 |
| `odometer_no_update` | RF-05, A4 |
| `odometer_no_delete` | RF-05, A4 |
| `motorcycle_demo_fixed` | RF-06 |
| `item_plan_same_moto_insert` | RF-04, A4 |
| `item_plan_fixed` | RF-04, A4 |
| `record_moto_fixed` | RF-04, A4 |
| `plan_moto_fixed` | RF-04, A4 |
| `plan_item_plan_fixed` | RF-04, A4 |
| `trip_demo_match` | RF-06 |
| `trip_transition` | RF-07, A5 |
| `trip_no_delete` | RF-07, RF-25, D2 |
| `trip_origin_fixed` | RNF-08, D2 |
| `segment_trip_fixed` | RNF-08, D2 |
| `import_not_demo` | RF-06 |
| `parse_run_final` | RNF-03, A8 |
| `parse_run_origin_fixed` | RNF-08, D2 |
| `parse_run_metadata_insert` | RNF-09, D3 |
| `parse_run_metadata_update` | RNF-09, D3 |
| `session_demo_match` | RF-06 |
| `session_origin_fixed` | RNF-08, D2 |
| `sample_row_matches_source` | RF-10, A7 |
| `sample_no_identity` | RNF-09, D3 |
| `raw_deletion_insert` | RF-25, D2 |
| `raw_deletion_count` | RF-25, D2 |
| `raw_deletion_update` | RF-25, D2 |
| `raw_deletion_complete` | RF-25, D2 |
| `raw_deletion_no_derived_left` | RF-25, D2 |
| `raw_deletion_cancel` | RF-25, D2 |
| `raw_deletion_no_delete` | RF-25, D2 |
| `location_insert_open_deletion` | RF-25, D2 |
| `segment_insert_open_deletion` | RF-25, D2 |
| `parse_run_insert_open_deletion` | RF-25, D2 |
| `session_insert_open_deletion` | RF-25, D2 |
| `sample_insert_open_deletion` | RF-25, D2 |
| `parse_run_no_delete` | RNF-03, A8 |
| `location_no_update` | RNF-03, RNF-08, D2 |
| `location_no_delete` | RNF-03, RNF-08, D2 |
| `sample_no_update` | RNF-03, RNF-08, D2 |
| `sample_no_delete` | RNF-03, RNF-08, D2 |
| `import_no_update` | RNF-03, RNF-08, D2 |
| `import_no_delete` | RNF-03, RNF-08, D2 |
| `gps_valid_needs_data` | RF-09, RF-23, D1 |
| `gps_evaluation_insert_open_deletion` | RF-25, D2 |
| `gps_evaluation_no_delete` | RNF-03, A8 |
| `gps_evaluation_no_update` | RNF-03, A8 |
| `metric_no_update` | RNF-03, A8 |
| `baseline_no_update` | RNF-03, A8 |
| `finding_no_update` | RNF-03, A8 |
| `report_no_update` | RNF-03, A8 |
| `metric_insert_open_deletion` | RF-25, D2 |
| `metric_no_delete` | RNF-03, A8 |
| `finding_insert_open_deletion` | RF-25, D2 |
| `finding_no_delete` | RNF-03, A8 |
| `report_insert_open_deletion` | RF-25, D2 |
| `report_no_delete` | RNF-03, A8 |
| `baseline_no_delete` | RNF-03, A8 |

| Visão | Justificativa |
|---|---|
| `open_deletion_session` | RF-25, D2 |
| `open_deletion_trip` | RF-25, D2 |
| `open_deletion_motorcycle` | RF-25, D2 |

| Índice | Justificativa |
|---|---|
| `ix_odometer_moto_time` | RF-05 |
| `ix_record_moto_time` | RF-02 |
| `ix_item_record` | RF-03 |
| `ux_trip_active` | RF-07 |
| `ix_trip_moto_time` | RF-21 |
| `ux_segment_open` | RF-07, A5 |
| `ix_session_moto_time` | RF-21 |
| `ix_session_trip` | RF-11 |
| `ix_sample_session_elapsed` | RF-20 |
| `ix_issue_run_row` | RF-10 |
| `ux_raw_deletion_open` | RF-25, D2 |
| `ux_metric_trip` | RF-12, RNF-03 |
| `ux_metric_session` | RF-12, RNF-03 |
