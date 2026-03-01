# PRD: Geração de Arquivo de Períodos - Simples Nacional e MEI (Spring Boot Batch)

## Introdução

Este documento descreve os requisitos para a implementação de um processo batch em Spring Boot responsável por gerar arquivos TXT de Períodos do Simples Nacional (SN) e MEI, com base no indicador de sincronização. O processo seleciona períodos pendentes de sincronização, gera os arquivos com layout fixo, grava-os em diretório de rede e atualiza o indicador de sincronização no banco de dados.

---

## Goals

- Automatizar a geração dos arquivos de períodos SN e MEI sem intervenção manual
- Garantir rastreabilidade por meio do indicador de sincronização (`INDI_SINCRONIZACAO`)
- Suportar múltiplos cenários de execução: agendado, manual e reprocessamento de competências
- Gerar arquivos com nomenclatura padronizada e layout fixo conforme especificação
- Garantir atomicidade: só atualizar o indicador após gravação bem-sucedida do arquivo

---

## User Stories

### US-001: Configuração do Job Spring Batch para Períodos SN e MEI

**Descrição:** Como desenvolvedor, preciso configurar o Job Spring Batch com dois Steps independentes (um para SN e outro para MEI) para que o processo possa ser executado de forma isolada ou conjunta.

**Critérios de Aceite:**

- [ ] Job `gerarArquivosPeriodosJob` criado com dois Steps: `stepPeriodoSimples` e `stepPeriodoMei`
- [ ] Cada Step é independente (falha em um não impede execução do outro)
- [ ] Job configurado para aceitar parâmetro de data/hora de execução (usado na nomenclatura do arquivo)
- [ ] Projeto compila sem erros (`mvn clean package` ou equivalente)
- [ ] Typecheck/lint passa

---

### US-002: Leitura dos Períodos do Simples Nacional pendentes de sincronização

**Descrição:** Como sistema, preciso consultar no banco de dados os períodos do tipo "Simples" com `INDI_SINCRONIZACAO = 'N'` para que somente registros não sincronizados sejam incluídos no arquivo.

**Critérios de Aceite:**

- [ ] `ItemReader` implementado com `JdbcCursorItemReader` (ou `JdbcPagingItemReader`)
- [ ] Query seleciona apenas registros com `TIPO_PERIODO = 'Simples'` e `INDI_SINCRONIZACAO = 'N'`
- [ ] Campos retornados: `NUMR_CNPJ_BASE`, `DATA_INCLUSAO_SIMPLES_NACIONAL`, `DATA_EXCLUSAO_SIMPLES_NACIONAL`, `INDI_PERIODO_CANCELADO`, `NUMR_OPCAO_SIMPLES_NACIONAL`
- [ ] Leitura paginada/cursor para suportar grandes volumes sem estouro de memória
- [ ] Typecheck/lint passa

---

### US-003: Geração do arquivo TXT do Simples Nacional com layout fixo

**Descrição:** Como sistema, preciso gerar um arquivo TXT com layout de largura fixa para os períodos SN para que o arquivo possa ser processado pelo sistema receptor.

**Critérios de Aceite:**

- [ ] `ItemWriter` grava arquivo em diretório temporário/local antes de mover para destino de rede
- [ ] Nomenclatura do arquivo: `01-0715-PER-YYYYMMDD-HHMMSS-SSS.TXT` (data/hora da execução do Job)
- [ ] Layout de cada linha respeita exatamente as colunas abaixo (sem separador, largura fixa):

| Campo                          | Tamanho | Formato  | Alinhamento       |
| ------------------------------ | ------- | -------- | ----------------- |
| NUMR_CNPJ_BASE                 | 8       | Alfa     | Esquerda, pad ' ' |
| DATA_INCLUSAO_SIMPLES_NACIONAL | 8       | Numérico | Direita, pad '0'  |
| DATA_EXCLUSAO_SIMPLES_NACIONAL | 8       | Numérico | Direita, pad '0'  |
| INDI_PERIODO_CANCELADO         | 1       | Alfa     | Esquerda, pad ' ' |
| NUMR_OPCAO_SIMPLES_NACIONAL    | 9       | Numérico | Direita, pad '0'  |
| **Total por linha**            | **34**  |          |                   |

- [ ] Cada registro ocupa exatamente 34 caracteres por linha (+ quebra de linha `\r\n` ou `\n`)
- [ ] Datas em formato `YYYYMMDD` (8 dígitos, sem separadores)
- [ ] Campos nulos/vazios preenchidos com espaços (alfa) ou zeros (numérico)
- [ ] Typecheck/lint passa
- [ ] Se não houver registros do tipo correspondente, o Step deve finalizar com sucesso sem gerar arquivo.

---

### US-004: Leitura dos Períodos do MEI pendentes de sincronização

**Descrição:** Como sistema, preciso consultar no banco de dados os períodos do tipo "MEI" com `INDI_SINCRONIZACAO = 'N'` para que somente registros não sincronizados do MEI sejam incluídos no arquivo específico do MEI.

**Critérios de Aceite:**

- [ ] `ItemReader` implementado com `JdbcCursorItemReader` (ou `JdbcPagingItemReader`)
- [ ] Query seleciona apenas registros com `TIPO_PERIODO = 'MEI'` e `INDI_SINCRONIZACAO = 'N'`
- [ ] Campos retornados: `NUMR_CNPJ_BASE`, `DATA_INCLUSAO_SIMPLES_NACIONAL`, `DATA_EXCLUSAO_SIMPLES_NACIONAL`, `INDI_PERIODO_CANCELADO`, `NUMR_OPCAO_SIMPLES_NACIONAL`, `NUMR_OPCAO_SIMEI`
- [ ] Leitura paginada/cursor para suportar grandes volumes sem estouro de memória
- [ ] Typecheck/lint passa

---

### US-005: Geração do arquivo TXT do MEI com layout fixo

**Descrição:** Como sistema, preciso gerar um arquivo TXT com layout de largura fixa para os períodos MEI para que o arquivo possa ser processado pelo sistema receptor com os dados adicionais do SIMEI.

**Critérios de Aceite:**

- [ ] `ItemWriter` grava arquivo com nomenclatura: `01-0715-PERMEI-YYYYMMDD-HHMMSS.TXT`
- [ ] Layout de cada linha respeita exatamente as colunas abaixo (sem separador, largura fixa):

| Campo                          | Tamanho | Formato  | Alinhamento       |
| ------------------------------ | ------- | -------- | ----------------- |
| NUMR_CNPJ_BASE                 | 8       | Alfa     | Esquerda, pad ' ' |
| DATA_INCLUSAO_SIMPLES_NACIONAL | 8       | Numérico | Direita, pad '0'  |
| DATA_EXCLUSAO_SIMPLES_NACIONAL | 8       | Numérico | Direita, pad '0'  |
| INDI_PERIODO_CANCELADO         | 1       | Alfa     | Esquerda, pad ' ' |
| NUMR_OPCAO_SIMPLES_NACIONAL    | 9       | Numérico | Direita, pad '0'  |
| NUMR_OPCAO_SIMEI               | 9       | Numérico | Direita, pad '0'  |
| **Total por linha**            | **43**  |          |                   |

- [ ] Cada registro ocupa exatamente 43 caracteres por linha (+ quebra de linha)
- [ ] Datas em formato `YYYYMMDD` (8 dígitos, sem separadores)
- [ ] Campos nulos/vazios preenchidos com espaços (alfa) ou zeros (numérico)
- [ ] Typecheck/lint passa

---

### US-006: Gravação dos arquivos no diretório de rede

**Descrição:** Como sistema, preciso gravar os arquivos gerados no diretório de rede `\\arquivosmf.economia.go.gov.br\SSN\Eventos` para que o sistema receptor possa consumi-los.

**Critérios de Aceite:**

- [ ] Caminho de destino configurável via `application.properties` / `application.yml` (não hardcoded)
- [ ] Arquivo gravado atomicamente: gerado em local temporário e movido para destino final apenas após escrita completa
- [ ] Em caso de falha na gravação (ex: rede indisponível), o Step falha e o indicador `INDI_SINCRONIZACAO` **não** é atualizado
- [ ] Log registra: nome do arquivo gerado, quantidade de registros e caminho de destino
- [ ] Typecheck/lint passa

---

### US-007: Atualização do indicador de sincronização após gravação bem-sucedida

**Descrição:** Como sistema, preciso atualizar o indicador `INDI_SINCRONIZACAO` de 'N' para 'S' nos registros incluídos no arquivo apenas após a gravação bem-sucedida para garantir consistência entre o arquivo gerado e o banco de dados.

**Critérios de Aceite:**

- [ ] Atualização executada em batch (`UPDATE ... WHERE NUMR_CNPJ_BASE IN (...)` ou equivalente eficiente)
- [ ] Atualização ocorre **somente após** confirmação de gravação do arquivo no destino
- [ ] Se a atualização falhar, o Job falha e o erro é logado com detalhes dos registros afetados
- [ ] Transação garante que todos os registros do arquivo são atualizados ou nenhum é
- [ ] Typecheck/lint passa

---

### US-008: Agendamento e modos de execução do Job

**Descrição:** Como operador, preciso que o Job suporte execução agendada (scheduled), manual via linha de comando e reprocessamento de competências em atraso para que a operação seja flexível.

**Critérios de Aceite:**

- [ ] Job agendado via `@Scheduled` com expressão cron configurável em `application.properties`
- [ ] Job pode ser disparado manualmente via `CommandLineRunner` ou endpoint REST seguro (opcional)
- [ ] Parâmetro `dataHoraExecucao` injetado automaticamente no Job quando agendado
- [ ] Log de início e fim do Job com duração e status (COMPLETED / FAILED)
- [ ] Typecheck/lint passa

---

### US-009: Tratamento de erros e resiliência

**Descrição:** Como operador, preciso que o Job trate falhas de forma resiliente (com retry e skip configuráveis) para que erros pontuais não interrompam o processamento completo do lote.

**Critérios de Aceite:**

- [ ] Configuração de `retry` para erros transitórios de I/O (ex: falha de rede) com no mínimo 3 tentativas
- [ ] Configuração de `skip` para registros com dados inválidos (ex: CNPJ base nulo), com log do registro pulado
- [ ] Limite de skip configurável (`skip-limit`) em `application.properties`
- [ ] Falhas fatais (ex: banco indisponível) encerram o Job imediatamente com status FAILED
- [ ] Typecheck/lint passa

---

### US-010: Testes unitários e de integração

**Descrição:** Como desenvolvedor, preciso de testes automatizados cobrindo os componentes principais do batch para garantir a qualidade e evitar regressões.

**Critérios de Aceite:**

- [ ] Teste unitário do `ItemProcessor`/`ItemWriter` para formatação do layout fixo (SN e MEI)
- [ ] Teste de integração do Job completo usando banco em memória (H2) ou Testcontainers
- [ ] Teste valida que o indicador `INDI_SINCRONIZACAO` é atualizado para 'S' após execução bem-sucedida
- [ ] Teste valida que arquivos gerados possuem o tamanho correto por linha (34 bytes para SN, 43 bytes para MEI)
- [ ] Todos os testes passam (`mvn test` ou equivalente)

---

## Functional Requirements

- **FR-1:** O sistema deve selecionar apenas os registros com `INDI_SINCRONIZACAO = 'N'` agrupados por `TIPO_PERIODO` ('Simples' ou 'MEI').
- **FR-2:** O arquivo SN deve ser nomeado `01-0715-PER-YYYYMMDD-HHMMSS.TXT` onde data/hora refere-se ao momento de início do Job.
- **FR-3:** O arquivo MEI deve ser nomeado `01-0715-PERMEI-YYYYMMDD-HHMMSS.TXT` com a mesma data/hora do Job.
- **FR-4:** O layout do arquivo SN deve ter exatamente 34 caracteres por linha (campos sem separador).
- **FR-5:** O layout do arquivo MEI deve ter exatamente 43 caracteres por linha (campos sem separador).
- **FR-6:** Campos alfanuméricos devem ser alinhados à esquerda e padded com espaços. Campos numéricos devem ser alinhados à direita e padded com zeros.
- **FR-7:** Datas devem ser gravadas no formato `YYYYMMDD` (8 dígitos, sem separadores).
- **FR-8:** Os arquivos devem ser gravados em `\\arquivosmf.economia.go.gov.br\SSN\Eventos`.
- **FR-9:** O indicador `INDI_SINCRONIZACAO` deve ser atualizado para 'S' apenas após gravação bem-sucedida do arquivo.
- **FR-10:** O caminho de destino, cron de agendamento e limites de retry/skip devem ser configuráveis via `application.properties`.
- **FR-11:** Caso não existam registros com `INDI_SINCRONIZACAO = 'N'`, o Job deve completar com sucesso sem gerar arquivo.
- **FR-12:** O Job deve registrar em log: data/hora de início, quantidade de registros processados por tipo, nome dos arquivos gerados e status final.

---

## Non-Goals (Fora do Escopo)

- Não contempla validação de regras de negócio do Simples Nacional ou MEI (ex: elegibilidade de CNPJ)
- Não envia arquivos via SFTP, FTP ou e-mail — apenas gravação em diretório de rede UNC
- Não inclui interface gráfica ou dashboard de monitoramento
- Não contempla geração de arquivos para outros regimes tributários além de SN e MEI
- Não implementa lógica de compactação ou criptografia dos arquivos gerados
- Não inclui integração com sistemas externos além do banco de dados de origem e o diretório de destino

---

## Technical Considerations

- **Framework:** Spring Boot com Spring Batch (projeto existente já configurado)
- **Banco de dados:** Relacional existente (adaptadores JDBC — sem acoplamento a vendor específico)
- **Conexão de rede:** Diretório UNC (`\\servidor\SSN\Eventos`) — deve estar montado/acessível no sistema operacional onde o batch executa; considerar uso de `java.nio.file.Path` ou `java.io.File` para compatibilidade
- **Charset:** Verificar com a equipe receptora se o arquivo deve ser gravado em `ISO-8859-1` (Latin-1) ou `UTF-8`
- **Linha de quebra:** Verificar se o receptor espera `\r\n` (Windows) ou `\n` (Unix)
- **Tamanho de lote:** Configurar `chunk-size` adequado (sugestão inicial: 1000 registros por chunk) para equilibrar memória e performance
- **Concorrência:** O Job não deve ter instâncias paralelas simultâneas — usar `JobParameters` com timestamp para evitar colisão no `JobRepository`
- **JobRepository:** Usar banco de dados existente (não banco em memória) para persistência dos metadados do batch em produção

---

## Success Metrics

- Arquivos gerados com 100% dos registros `INDI_SINCRONIZACAO = 'N'` do tipo correspondente
- Zero registros atualizados para 'S' em caso de falha na gravação do arquivo
- Job executa em menos de 5 minutos para lotes de até 100.000 registros
- Zero falhas silenciosas — toda exceção deve ser logada e o Job deve terminar com status FAILED

---

## Open Questions

1. Qual é o charset esperado pelo sistema receptor: `ISO-8859-1` ou `UTF-8`?
2. A quebra de linha deve ser `\r\n` (CRLF) ou `\n` (LF)?
3. O diretório de rede UNC é mapeado como drive de rede no servidor ou será acessado via caminho UNC direto?
4. Deve ser gerado um arquivo de log/controle (ex: `.CTL`) junto com o TXT para o sistema receptor?
5. O Job deve gerar os dois arquivos (SN e MEI) sempre na mesma execução, ou podem ser executados independentemente?
6. Existe alguma janela de horário restrita para a execução do Job (ex: fora do horário comercial)?
7. Em caso de reprocessamento, o Job deve reprocessar apenas registros que voltaram a ter `INDI_SINCRONIZACAO = 'N'` ou existe outro mecanismo?
