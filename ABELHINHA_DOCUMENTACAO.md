# Abelhinha (Sugestão de Venda)

## 1. Objetivo da Funcionalidade

Oferecer descontos progressivos por volume ou quantidade, incentivando compras maiores através de sugestões automáticas conforme lote múltiplo ou faixas de quantidade configuradas por produto e centro.

## 2. Quando o Usuário Utiliza

- Ao configurar descontos por volume via CSV.
- Quando produto exibe ícone de abelha no pedido.
- Para consultar descontos disponíveis por quantidade.
- Na análise de campanhas de desconto por volume.

## 2.1. Estrutura do CSV

**Lote Múltiplo:**
```
centro;produto;multiplo;desconto;vigencia_inicial;vigencia_final;trava_lote
1001;MAT001;10;0.05;2025-01-01;2025-12-31;true
1001;MAT001;20;0.10;2025-01-01;2025-12-31;true
```

**Faixa de Quantidade:**
```
centro;produto;faixa_inicial;faixa_final;desconto;vigencia_inicial;vigencia_final
1001;MAT001;1;10;0.03;2025-01-01;2025-12-31
1001;MAT001;11;50;0.07;2025-01-01;2025-12-31
1001;MAT001;51;null;0.12;2025-01-01;2025-12-31
```

**Campos obrigatórios:**
- `centro`: código numérico do centro de distribuição.
- `produto`: código do material (alfanumérico).
- `desconto`: formato decimal (0.05 = 5%).
- `vigencia_inicial`: formato YYYY-MM-DD.
- `vigencia_final`: formato YYYY-MM-DD ou null para indefinido.

**Encoding:** UTF-8 com BOM.
**Delimitador:** ponto e vírgula (;).
**Limite:** 2000 linhas por arquivo.

## 3. Fluxo Simplificado

1. Admin acessa `/configuracoes/abelhinha`.
2. Faz upload de CSV com centro, produto, faixas/múltiplos, vigência, percentual.
3. Sistema valida arquivo.
4. Processamento assíncrono via RabbitMQ.
5. Integração com Elasticsearch.
6. Vendedor busca produto no pedido.
7. Ícone de abelha aparece quando há desconto configurado.
8. Ao informar quantidade, desconto é calculado automaticamente.
9. Tooltip mostra faixas de desconto disponíveis.

## 4. Regras de Negócio Importantes

**Lote múltiplo:**
- Desconto por múltiplos (10, 20, 30 unidades).
- Pode ter trava de lote (força quantidade em múltiplos).
- Distribui quantidade pelos múltiplos disponíveis, resto sem desconto.

**Faixa de quantidade:**
- Desconto por faixas (1-10, 11-20, acima de 20).
- Faixa final pode ser null (acima de X).
- Encontra faixa que contém quantidade informada.

**Geral:**
- Vigência inicial <= vigência final.
- Vigência final >= data atual - 1 dia.
- Desconto só aplica se não houver preço manual.
- Limite 2000 registros por lote no processamento.

## 4.1. Algoritmo de Cálculo

**Lote Múltiplo com Trava:**
```
Quantidade informada: 35
Múltiplos cadastrados: 10 (5%), 20 (10%), 30 (15%)
Trava ativa: força arredondamento para múltiplo válido
Resultado: quantidade ajustada para 30, desconto 15%
```

**Lote Múltiplo sem Trava:**
```
Quantidade informada: 35
Múltiplos cadastrados: 10 (5%), 20 (10%), 30 (15%)
Distribuição: 30 unidades com 15%, 5 unidades sem desconto
Desconto médio ponderado: (30*0.15 + 5*0) / 35 = 12.86%
```

**Faixa de Quantidade:**
```
Quantidade informada: 25
Faixas: 1-10 (3%), 11-50 (7%), 51+ (12%)
Quantidade 25 está na faixa 11-50
Resultado: desconto 7% aplicado sobre total
```

**Ordem de precedência:**
1. Verifica vigência ativa.
2. Valida ausência de preço manual.
3. Busca configuração específica centro + produto.
4. Aplica cálculo conforme tipo (lote ou faixa).
5. Retorna desconto final ou 0% se nenhuma regra aplicável.

## 5. Integrações Envolvidas

**RabbitMQ:**
- Fila: `abelhinha.import.queue`
- Exchange: `abelhinha.exchange` (tipo direct)
- Routing key: `abelhinha.import`
- Dead letter queue: `abelhinha.import.dlq`
- TTL mensagem: 24 horas
- Retry automático: 3 tentativas com backoff exponencial

**Elasticsearch:**
- Índice: `descontos_abelhinha`
- Refresh interval: 5s
- Shards: 1 primário, 1 réplica
- Campos indexados: `centro`, `produto`, `tipo_desconto`, `vigencia_inicial`, `vigencia_final`
- Mapeamento:
  - `centro`: keyword
  - `produto`: keyword
  - `tipo_desconto`: keyword (LOTE_MULTIPLO | FAIXA_QUANTIDADE)
  - `multiplo`: integer
  - `faixa_inicial`: integer
  - `faixa_final`: integer (nullable)
  - `desconto`: float
  - `vigencia_inicial`: date
  - `vigencia_final`: date (nullable)
  - `trava_lote`: boolean

**Email:**
- Template: `abelhinha-erro-importacao`
- Destinatários: admin, responsável pela importação
- Anexa log de erros em formato JSON.

## 5.1. Endpoints da API

**POST `/api/abelhinha/importar`**
- Body: `multipart/form-data`
- Campos: `arquivo` (file), `tipo` (LOTE_MULTIPLO | FAIXA_QUANTIDADE)
- Response: `{ "id": "uuid", "status": "PROCESSANDO", "total_linhas": 1500 }`
- Timeout: 30s (processamento continua em background)

**GET `/api/abelhinha/status/:id`**
- Response: `{ "id": "uuid", "status": "CONCLUIDO|PROCESSANDO|ERRO", "processadas": 1500, "erro": null }`

**GET `/api/abelhinha/descontos`**
- Query params: `centro`, `produto`, `quantidade`
- Response:
```json
{
  "tem_desconto": true,
  "tipo": "LOTE_MULTIPLO",
  "desconto_aplicado": 0.15,
  "detalhes": {
    "multiplos": [
      { "valor": 10, "desconto": 0.05 },
      { "valor": 20, "desconto": 0.10 },
      { "valor": 30, "desconto": 0.15 }
    ]
  }
}
```

**DELETE `/api/abelhinha/descontos/:id`**
- Desativa desconto específico (soft delete).
- Response: `{ "success": true }`

**GET `/api/abelhinha/exportar`**
- Query params: `centro` (opcional), `produto` (opcional), `vigente` (boolean)
- Response: CSV com descontos ativos.
- Stream response para grandes volumes.

## 5.2. Modelo de Dados

**Tabela: `descontos_abelhinha`**
```sql
id: BIGINT PRIMARY KEY AUTO_INCREMENT
centro_id: VARCHAR(10) NOT NULL
produto_id: VARCHAR(50) NOT NULL
tipo_desconto: ENUM('LOTE_MULTIPLO', 'FAIXA_QUANTIDADE') NOT NULL
multiplo: INT NULL
faixa_inicial: INT NULL
faixa_final: INT NULL
percentual_desconto: DECIMAL(5,4) NOT NULL
vigencia_inicial: DATE NOT NULL
vigencia_final: DATE NULL
trava_lote: BOOLEAN DEFAULT FALSE
ativo: BOOLEAN DEFAULT TRUE
criado_em: TIMESTAMP DEFAULT CURRENT_TIMESTAMP
criado_por: VARCHAR(100)
atualizado_em: TIMESTAMP ON UPDATE CURRENT_TIMESTAMP

INDEX idx_centro_produto (centro_id, produto_id)
INDEX idx_vigencia (vigencia_inicial, vigencia_final)
INDEX idx_ativo (ativo)
```

**Constraints:**
- `CHECK (percentual_desconto >= 0 AND percentual_desconto <= 1)`
- `CHECK (vigencia_inicial <= vigencia_final OR vigencia_final IS NULL)`
- `CHECK ((tipo_desconto = 'LOTE_MULTIPLO' AND multiplo IS NOT NULL) OR (tipo_desconto = 'FAIXA_QUANTIDADE' AND faixa_inicial IS NOT NULL))`

## 5.3. Configurações Técnicas

**Variáveis de Ambiente:**
- `ABELHINHA_BATCH_SIZE`: tamanho do lote de processamento (padrão: 500)
- `ABELHINHA_MAX_FILE_SIZE`: tamanho máximo do CSV em MB (padrão: 10)
- `ABELHINHA_IMPORT_TIMEOUT`: timeout do processamento em segundos (padrão: 300)
- `ELASTICSEARCH_ABELHINHA_INDEX`: nome do índice (padrão: descontos_abelhinha)
- `RABBITMQ_ABELHINHA_QUEUE`: nome da fila (padrão: abelhinha.import.queue)

**Cache:**
- Redis key pattern: `abelhinha:desconto:{centro}:{produto}`
- TTL: 1 hora
- Invalidação: ao finalizar importação de desconto para centro/produto específico

**Performance:**
- Busca no Elasticsearch: média < 50ms
- Processamento CSV 2000 linhas: média 2-5 minutos
- Query SQL com índices: < 100ms

## 6. Erros Comuns e Como Resolver

### Erro
"Tipo de importação incorreta"

### Possíveis causas
- Layout CSV não corresponde ao tipo selecionado.

### Como o N1 deve validar
1. Verificar tipo selecionado: lote múltiplo ou faixa.
2. Conferir colunas do CSV.
3. Baixar modelo correto.

---

### Erro
"Faixa inicial maior que final"

### Possíveis causas
- Faixas invertidas no CSV.

### Como o N1 deve validar
1. Revisar faixas do CSV.
2. Faixa inicial deve ser <= faixa final.

---

### Erro
"Desconto maior que 100%"

### Possíveis causas
- Percentual inválido no CSV.

### Como o N1 deve validar
1. Verificar coluna de desconto.
2. Valores devem estar entre 0 e 1 (0% a 100%).
3. Exemplo: 0.05 = 5%, 0.15 = 15%.

---

### Erro
"Existem materiais/centros não cadastrados"

### Possíveis causas
- Códigos de produto ou centro inválidos.

### Como o N1 deve validar
1. Verificar produtos e centros no CSV.
2. Confirmar se existem no sistema.
3. Corrigir códigos inválidos.

## 7. Validações Técnicas

**Validação de Arquivo:**
- Extensão: `.csv` apenas
- Tamanho máximo: 10MB
- Encoding: UTF-8 (detecta automaticamente)
- Delimitador: `;` (validação estrita)

**Validação de Dados:**
- Centro existe: query em `centros` com cache de 1h
- Produto existe: query em `produtos` com cache de 1h
- Desconto: regex `^0(\.[0-9]{1,4})?$|^1(\.0{1,4})?$`
- Data: formato ISO 8601 (YYYY-MM-DD)
- Múltiplo: > 0 e <= 10000
- Faixas: inicial >= 0, final > inicial ou null

**Validação de Regras:**
- Não permite sobreposição de faixas para mesmo centro/produto/vigência
- Múltiplos duplicados para mesmo centro/produto geram erro
- Vigência não pode conflitar com desconto já ativo

## 8. Logs e Monitoramento

**Eventos logados:**
- `ABELHINHA_IMPORT_STARTED`: início do processamento
- `ABELHINHA_IMPORT_VALIDATING`: validação de cada linha
- `ABELHINHA_IMPORT_ERROR`: erro em linha específica
- `ABELHINHA_IMPORT_COMPLETED`: finalização com sucesso
- `ABELHINHA_IMPORT_FAILED`: falha total do processamento
- `ABELHINHA_DISCOUNT_APPLIED`: desconto aplicado em pedido
- `ABELHINHA_DISCOUNT_NOT_FOUND`: busca sem resultado

**Estrutura de log:**
```json
{
  "timestamp": "2025-02-21T10:30:00Z",
  "level": "INFO",
  "event": "ABELHINHA_IMPORT_COMPLETED",
  "usuario": "admin@leomadeiras.com",
  "import_id": "uuid",
  "linhas_processadas": 1500,
  "linhas_erro": 3,
  "duracao_ms": 125000,
  "metadata": {
    "tipo": "LOTE_MULTIPLO",
    "centros_afetados": [1001, 1002]
  }
}
```

**Métricas monitoradas:**
- Taxa de sucesso de importações (%)
- Tempo médio de processamento (ms)
- Tamanho médio de arquivo (linhas)
- Descontos aplicados por dia
- Taxa de erro por tipo de validação

## 9. Segurança

**Permissões necessárias:**
- Importação CSV: perfil `ADMIN` ou `GERENTE_COMERCIAL`
- Consulta descontos: perfil `VENDEDOR` ou superior
- Exclusão descontos: perfil `ADMIN` apenas

**Validações de segurança:**
- Upload de arquivo: validação de MIME type (text/csv)
- Sanitização de inputs: remove caracteres especiais SQL
- Rate limiting: 5 importações por hora por usuário
- Auditoria: registra todas as operações com usuário e timestamp

**Proteção de dados:**
- CSV temporário armazenado em storage criptografado
- Remoção automática após processamento (máx 24h)
- Logs não expõem dados sensíveis de clientes

## 10. Requisitos de Sistema

**Backend:**
- Node.js >= 18.x
- PostgreSQL >= 13.x
- Elasticsearch >= 7.10
- RabbitMQ >= 3.9
- Redis >= 6.x

**Dependências principais:**
- `@elastic/elasticsearch`: ^8.x
- `amqplib`: ^0.10.x
- `ioredis`: ^5.x
- `csv-parse`: ^5.x
- `joi`: ^17.x (validação de schemas)

**Recursos de infraestrutura:**
- CPU: 2 cores mínimo para worker de processamento
- RAM: 2GB mínimo, 4GB recomendado
- Disco: 10GB para logs e arquivos temporários
- Network: latência < 100ms entre serviços

## 11. Troubleshooting Técnico

**Importação travada no status PROCESSANDO:**
1. Verificar worker RabbitMQ: `rabbitmqctl list_consumers`
2. Verificar logs do worker: `tail -f logs/abelhinha-worker.log`
3. Verificar mensagens na DLQ: buscar em `abelhinha.import.dlq`
4. Reprocessar manualmente: `POST /api/abelhinha/reprocessar/:id`

**Desconto não aparece no pedido:**
1. Verificar índice Elasticsearch: `GET /descontos_abelhinha/_search?q=produto:XXX`
2. Verificar cache Redis: `redis-cli GET abelhinha:desconto:{centro}:{produto}`
3. Validar vigência: `SELECT * FROM descontos_abelhinha WHERE produto_id='XXX' AND ativo=true`
4. Verificar logs: buscar por `ABELHINHA_DISCOUNT_NOT_FOUND`

**Performance degradada:**
1. Verificar índices Elasticsearch: `GET /_cat/indices/descontos_abelhinha`
2. Analisar slow queries: `SELECT * FROM pg_stat_statements WHERE query LIKE '%descontos_abelhinha%'`
3. Verificar tamanho da fila RabbitMQ: `rabbitmqctl list_queues`
4. Limpar cache Redis se necessário: `redis-cli DEL abelhinha:*`

**Comandos úteis:**
```bash
# Reindexar Elasticsearch
curl -X POST "localhost:9200/descontos_abelhinha/_refresh"

# Limpar fila RabbitMQ
rabbitmqctl purge_queue abelhinha.import.queue

# Verificar conexões ativas
netstat -an | grep :5672  # RabbitMQ
netstat -an | grep :9200  # Elasticsearch

# Backup de descontos ativos
psql -c "COPY (SELECT * FROM descontos_abelhinha WHERE ativo=true) TO '/tmp/backup_abelhinha.csv' WITH CSV HEADER"
```

## 12. Validações Técnicas

**Validação de Arquivo:**
- Extensão: `.csv` apenas
- Tamanho máximo: 10MB
- Encoding: UTF-8 (detecta automaticamente)
- Delimitador: `;` (validação estrita)

**Validação de Dados:**
- Centro existe: query em `centros` com cache de 1h
- Produto existe: query em `produtos` com cache de 1h
- Desconto: regex `^0(\.[0-9]{1,4})?$|^1(\.0{1,4})?$`
- Data: formato ISO 8601 (YYYY-MM-DD)
- Múltiplo: > 0 e <= 10000
- Faixas: inicial >= 0, final > inicial ou null

**Validação de Regras:**
- Não permite sobreposição de faixas para mesmo centro/produto/vigência
- Múltiplos duplicados para mesmo centro/produto geram erro
- Vigência não pode conflitar com desconto já ativo

## 13. Logs e Monitoramento

**Eventos logados:**
- `ABELHINHA_IMPORT_STARTED`: início do processamento
- `ABELHINHA_IMPORT_VALIDATING`: validação de cada linha
- `ABELHINHA_IMPORT_ERROR`: erro em linha específica
- `ABELHINHA_IMPORT_COMPLETED`: finalização com sucesso
- `ABELHINHA_IMPORT_FAILED`: falha total do processamento
- `ABELHINHA_DISCOUNT_APPLIED`: desconto aplicado em pedido
- `ABELHINHA_DISCOUNT_NOT_FOUND`: busca sem resultado

**Estrutura de log:**
```json
{
  "timestamp": "2025-02-21T10:30:00Z",
  "level": "INFO",
  "event": "ABELHINHA_IMPORT_COMPLETED",
  "usuario": "admin@leomadeiras.com",
  "import_id": "uuid",
  "linhas_processadas": 1500,
  "linhas_erro": 3,
  "duracao_ms": 125000,
  "metadata": {
    "tipo": "LOTE_MULTIPLO",
    "centros_afetados": [1001, 1002]
  }
}
```

**Métricas monitoradas:**
- Taxa de sucesso de importações (%)
- Tempo médio de processamento (ms)
- Tamanho médio de arquivo (linhas)
- Descontos aplicados por dia
- Taxa de erro por tipo de validação

## 14. Segurança

**Permissões necessárias:**
- Importação CSV: perfil `ADMIN` ou `GERENTE_COMERCIAL`
- Consulta descontos: perfil `VENDEDOR` ou superior
- Exclusão descontos: perfil `ADMIN` apenas

**Validações de segurança:**
- Upload de arquivo: validação de MIME type (text/csv)
- Sanitização de inputs: remove caracteres especiais SQL
- Rate limiting: 5 importações por hora por usuário
- Auditoria: registra todas as operações com usuário e timestamp

**Proteção de dados:**
- CSV temporário armazenado em storage criptografado
- Remoção automática após processamento (máx 24h)
- Logs não expõem dados sensíveis de clientes

## 15. Testes

**Testes Unitários:**
- `DescontoCalculator.calcularLoteMultiplo()`: valida distribuição de quantidade
- `DescontoCalculator.calcularFaixaQuantidade()`: valida seleção de faixa
- `CsvValidator.validarEstrutura()`: valida formato e colunas
- `VigenciaValidator.estaVigente()`: valida datas

**Testes de Integração:**
- Fluxo completo de importação com RabbitMQ
- Sincronização PostgreSQL -> Elasticsearch
- Invalidação de cache Redis após importação
- Notificação de email em caso de erro

**Testes de Carga:**
- Importação simultânea de 5 arquivos com 2000 linhas cada
- 100 consultas/segundo no endpoint de descontos
- Elasticsearch com 1M de registros ativos

**Casos de Teste Críticos:**
```
Caso 1: Lote múltiplo com trava
- Input: 35 unidades, múltiplos [10, 20, 30]
- Expected: ajusta para 30, desconto do múltiplo 30

Caso 2: Lote múltiplo sem trava
- Input: 35 unidades, múltiplos [10, 20, 30]
- Expected: 30 com desconto, 5 sem desconto

Caso 3: Faixa com limite null
- Input: 1000 unidades, faixas [1-10, 11-50, 51-null]
- Expected: aplica desconto da faixa 51+

Caso 4: Sobreposição de vigência
- Input: centro 1001, produto MAT001, vigência 2025-01-01 a 2025-06-30
- Existing: mesmo centro/produto, vigência 2025-05-01 a 2025-12-31
- Expected: erro de validação
```

## 16. Exemplo de Integração Frontend

**Consulta de desconto ao digitar quantidade:**
```javascript
async function buscarDesconto(centro, produto, quantidade) {
  const response = await fetch(
    `/api/abelhinha/descontos?centro=${centro}&produto=${produto}&quantidade=${quantidade}`
  );
  const data = await response.json();
  
  if (data.tem_desconto) {
    aplicarDesconto(data.desconto_aplicado);
    exibirTooltip(data.detalhes);
  }
}

const buscarDescontoDebounced = debounce(buscarDesconto, 500);
```

**Exibição do ícone de abelha:**
```javascript
async function verificarDescontoDisponivel(centro, produto) {
  const response = await fetch(
    `/api/abelhinha/descontos?centro=${centro}&produto=${produto}&quantidade=1`
  );
  const data = await response.json();
  return data.tem_desconto;
}

if (await verificarDescontoDisponivel(centro, produto)) {
  mostrarIconeAbelha();
}
```

## 6. Erros Comuns e Como Resolver
