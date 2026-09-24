# Modelagem e Catálogo de Dados (Etapa 4.3)

## Escopo e referência dos metadados

Projeto: **MVP — Gramados e resultados do Brasileirão 2023–2024**. Catálogo transcrito a partir de `05_catalogo_dados.csv`, exportado do Databricks e recebido em 24/09/2026. O arquivo contém **7 tabelas e 67 colunas**, todas com descrição preenchida. Os nomes, tipos e descrições abaixo reproduzem esse arquivo; os domínios e a linhagem foram acrescentados a partir das regras do projeto.

Todas as tabelas pertencem a `workspace.mvp_gramados` e foram persistidas em Delta. As quantidades de registros são as verificadas nas evidências da execução; o CSV de metadados, por si só, não contém contagens de registros.

## Modelo e granularidade

Foi adotado um modelo de tabelas planas organizado em Bronze, Silver e Gold. A Bronze conserva os dados recebidos; a Silver converte tipos e associa partidas à pesquisa histórica; a Gold reúne indicadores por tipo de gramado, ano e clube mandante. As chaves abaixo são **chaves lógicas esperadas**, não uma declaração de restrições PRIMARY KEY ou FOREIGN KEY aplicadas no banco.

| Tabela | Colunas | Registros na execução | Chave lógica |
|---|---:|---:|---|
| `bronze_partidas` | 16 | 8.785 | `ID` |
| `bronze_gramados` | 6 | 40 | `arena, na versão atual` |
| `silver_gramados` | 6 | 40 | `arena, na versão atual` |
| `silver_partidas_gramados` | 22 | 760 | `ID` |
| `gold_resumo_gramado` | 5 | 3 | `tipo_gramado` |
| `gold_resumo_ano_gramado` | 6 | 6 | `ano + tipo_gramado` |
| `gold_resumo_clube_gramado` | 6 | Não aferida no CSV de metadados | `mandante + tipo_gramado` |

A pesquisa contém 40 **nomes de estádio**, que não equivalem necessariamente a 40 locais físicos. Nomes alternativos podem representar o mesmo estádio. Na versão atual, cada nome possui uma linha. Se forem adicionados múltiplos períodos por nome, a chave da pesquisa e a junção precisarão considerar os intervalos e impedir sobreposição ou duplicação de partidas.

## Convenções dos domínios

Os domínios indicam valores esperados e regras de interpretação; não significam que foram criadas restrições automáticas no banco. Na Bronze, até os campos de conteúdo numérico são STRING. Campos sem enumeração fechada, como nomes e observações, têm domínio textual aberto. Os valores mínimo e máximo observados não devem ser confundidos com limites permanentes de negócio.

A exportação não contém a propriedade de nulabilidade. Não se afirma aqui que existam restrições NOT NULL. A qualidade foi avaliada separadamente no notebook `04_qualidade_dados`.

## `bronze_partidas`

**Granularidade:** Uma partida da base histórica original.

**Origem e processamento:** CSV campeonato-brasileiro-full.csv; leitura com cabeçalho, campos multilinha e escape de aspas, mantendo as 16 colunas como texto.

**Notebook responsável:** `01_preparacao`. Nome completo: `workspace.mvp_gramados.bronze_partidas`.

| Coluna | Tipo persistido | Descrição do catálogo | Domínio esperado | Origem / transformação |
|---|---|---|---|---|
| `ID` | `string` | Identificador da partida na fonte. Esperado: inteiro positivo e unico, armazenado como texto na Bronze. | Inteiro ≥ 1; único por partida; sem máximo de negócio definido. | campeonato-brasileiro-full.csv → ID; preservado como texto. |
| `rodata` | `string` | Numero da rodada, armazenado como texto. Nome original da coluna preservado da fonte. Esperado: inteiro positivo. | Inteiro ≥ 1; máximo depende da edição do campeonato. Não impor 38 à base histórica inteira. | campeonato-brasileiro-full.csv → rodata; preservado como texto. |
| `data` | `string` | Data da partida na fonte, em texto no formato dd/MM/yyyy. | Data válida em dd/MM/yyyy na Bronze; na Silver, de 01/01/2023 a 31/12/2024 pelo filtro. | campeonato-brasileiro-full.csv → data; preservado como texto. |
| `hora` | `string` | Horario da partida informado pela fonte, armazenado como texto. | Texto HH:mm ou HH:mm:ss; horas 00–23, minutos e segundos 00–59; fuso não validado. | campeonato-brasileiro-full.csv → hora; preservado como texto. |
| `mandante` | `string` | Nome do clube mandante conforme registrado na fonte. | Nome de clube presente na fonte; domínio textual aberto. Mandante e visitante devem ser distintos na partida. | campeonato-brasileiro-full.csv → mandante; preservado como texto. |
| `visitante` | `string` | Nome do clube visitante conforme registrado na fonte. | Nome de clube presente na fonte; domínio textual aberto. Mandante e visitante devem ser distintos na partida. | campeonato-brasileiro-full.csv → visitante; preservado como texto. |
| `formacao_mandante` | `string` | Formacao tatica do mandante informada pela fonte. Pode estar ausente. | Texto de formação tática, por exemplo 4-2-3-1, ou ausência; sem lista fechada validada. | campeonato-brasileiro-full.csv → formacao_mandante; preservado como texto. |
| `formacao_visitante` | `string` | Formacao tatica do visitante informada pela fonte. Pode estar ausente. | Texto de formação tática, por exemplo 4-2-3-1, ou ausência; sem lista fechada validada. | campeonato-brasileiro-full.csv → formacao_visitante; preservado como texto. |
| `tecnico_mandante` | `string` | Nome do tecnico do mandante informado pela fonte. Pode estar ausente. | Nome de técnico ou ausência; domínio textual aberto. | campeonato-brasileiro-full.csv → tecnico_mandante; preservado como texto. |
| `tecnico_visitante` | `string` | Nome do tecnico do visitante informado pela fonte. Pode estar ausente. | Nome de técnico ou ausência; domínio textual aberto. | campeonato-brasileiro-full.csv → tecnico_visitante; preservado como texto. |
| `vencedor` | `string` | Vencedor informado pela fonte; hifen representa empate. Campo original preservado, sujeito a divergencias com o placar. | Nome do mandante, nome do visitante ou hífen (-) para empate; esperado coerente com o placar. | campeonato-brasileiro-full.csv → vencedor; preservado como texto. |
| `arena` | `string` | Nome do estadio conforme registrado na fonte. Um estadio pode possuir diferentes nomes. | Nome de estádio; domínio textual aberto, com possíveis nomes alternativos do mesmo local. | campeonato-brasileiro-full.csv → arena; preservado como texto. |
| `mandante_Placar` | `string` | Gols do mandante, armazenados como texto. Esperado: inteiro maior ou igual a zero. | Inteiro ≥ 0; sem máximo de negócio fixado. Valores altos exigem análise, não exclusão automática. | campeonato-brasileiro-full.csv → mandante_Placar; preservado como texto. |
| `visitante_Placar` | `string` | Gols do visitante, armazenados como texto. Esperado: inteiro maior ou igual a zero. | Inteiro ≥ 0; sem máximo de negócio fixado. Valores altos exigem análise, não exclusão automática. | campeonato-brasileiro-full.csv → visitante_Placar; preservado como texto. |
| `mandante_Estado` | `string` | Sigla da unidade federativa do clube mandante, conforme a fonte. | AC, AL, AP, AM, BA, CE, DF, ES, GO, MA, MT, MS, MG, PA, PB, PR, PE, PI, RJ, RN, RS, RO, RR, SC, SP, SE, TO. | campeonato-brasileiro-full.csv → mandante_Estado; preservado como texto. |
| `visitante_Estado` | `string` | Sigla da unidade federativa do clube visitante, conforme a fonte. | AC, AL, AP, AM, BA, CE, DF, ES, GO, MA, MT, MS, MG, PA, PB, PR, PE, PI, RJ, RN, RS, RO, RR, SC, SP, SE, TO. | campeonato-brasileiro-full.csv → visitante_Estado; preservado como texto. |

## `bronze_gramados`

**Granularidade:** Um nome de estádio e seu intervalo de classificação pesquisado.

**Origem e processamento:** CSV gramados.csv, elaborado a partir da pesquisa histórica; seis colunas carregadas como texto.

**Notebook responsável:** `01_preparacao`. Nome completo: `workspace.mvp_gramados.bronze_gramados`.

| Coluna | Tipo persistido | Descrição do catálogo | Domínio esperado | Origem / transformação |
|---|---|---|---|---|
| `arena` | `string` | Nome do estadio usado para associar a pesquisa as partidas por igualdade de texto. Origem: gramados.csv; valor carregado como texto. | Nome de estádio; domínio textual aberto, com possíveis nomes alternativos do mesmo local. | gramados.csv → arena; preservado como texto. |
| `tipo_gramado` | `string` | Classificacao historica pesquisada: natural, sintetico, hibrido ou nao_confirmado. Origem: gramados.csv; valor carregado como texto. | natural, sintetico, hibrido, nao_confirmado. Na junção, nulo também pode ocorrer por falta de correspondência. | gramados.csv → tipo_gramado; preservado como texto. |
| `inicio_validade` | `string` | Inicio inclusivo do intervalo adotado na pesquisa para classificar os jogos. Nao representa necessariamente a data de instalacao do gramado. Origem: gramados.csv; valor carregado como texto. | Data válida (yyyy-MM-dd no CSV de pesquisa); início ≤ fim. Ausência admitida em pesquisa pendente; ambas obrigatórias para elegibilidade. | gramados.csv → inicio_validade; preservado como texto. |
| `fim_validade` | `string` | Fim inclusivo do intervalo adotado na pesquisa para classificar os jogos. Nao representa necessariamente a data de retirada do gramado. Origem: gramados.csv; valor carregado como texto. | Data válida (yyyy-MM-dd no CSV de pesquisa); início ≤ fim. Ausência admitida em pesquisa pendente; ambas obrigatórias para elegibilidade. | gramados.csv → fim_validade; preservado como texto. |
| `fonte_url` | `string` | URL ou URLs das referencias consultadas. O preenchimento nao garante que a classificacao esteja confirmada. Origem: gramados.csv; valor carregado como texto. | Um ou mais endereços de referências, possivelmente separados por quebra de linha. URL preenchida não comprova classificação nem disponibilidade do site. | gramados.csv → fonte_url; preservado como texto. |
| `observacoes` | `string` | Notas da pesquisa, incluindo nomes alternativos, evidencias, inferencias e pendencias. Origem: gramados.csv; valor carregado como texto. | Texto livre de evidências, hipóteses de continuidade, nomes alternativos e pendências; sem enumeração fechada. | gramados.csv → observacoes; preservado como texto. |

## `silver_gramados`

**Granularidade:** Um nome de estádio e seu intervalo de classificação pesquisado, com datas convertidas.

**Origem e processamento:** bronze_gramados; conversão das duas datas com TRY_CAST para DATE.

**Notebook responsável:** `02_tratamento`. Nome completo: `workspace.mvp_gramados.silver_gramados`.

| Coluna | Tipo persistido | Descrição do catálogo | Domínio esperado | Origem / transformação |
|---|---|---|---|---|
| `arena` | `string` | Nome do estadio usado para associar a pesquisa as partidas por igualdade de texto. Origem: bronze_gramados. | Nome de estádio; domínio textual aberto, com possíveis nomes alternativos do mesmo local. | bronze_gramados.arena; preservado. |
| `tipo_gramado` | `string` | Classificacao historica pesquisada: natural, sintetico, hibrido ou nao_confirmado. Origem: bronze_gramados. | natural, sintetico, hibrido, nao_confirmado. Na junção, nulo também pode ocorrer por falta de correspondência. | bronze_gramados.tipo_gramado; preservado. |
| `inicio_validade` | `date` | Inicio inclusivo do intervalo adotado na pesquisa para classificar os jogos. Nao representa necessariamente a data de instalacao do gramado. Origem: bronze_gramados; convertido para DATE, com nulo quando ausente ou nao conversivel. | Data válida (yyyy-MM-dd no CSV de pesquisa); início ≤ fim. Ausência admitida em pesquisa pendente; ambas obrigatórias para elegibilidade. | bronze_gramados.inicio_validade → TRY_CAST AS DATE. |
| `fim_validade` | `date` | Fim inclusivo do intervalo adotado na pesquisa para classificar os jogos. Nao representa necessariamente a data de retirada do gramado. Origem: bronze_gramados; convertido para DATE, com nulo quando ausente ou nao conversivel. | Data válida (yyyy-MM-dd no CSV de pesquisa); início ≤ fim. Ausência admitida em pesquisa pendente; ambas obrigatórias para elegibilidade. | bronze_gramados.fim_validade → TRY_CAST AS DATE. |
| `fonte_url` | `string` | URL ou URLs das referencias consultadas. O preenchimento nao garante que a classificacao esteja confirmada. Origem: bronze_gramados. | Um ou mais endereços de referências, possivelmente separados por quebra de linha. URL preenchida não comprova classificação nem disponibilidade do site. | bronze_gramados.fonte_url; preservado. |
| `observacoes` | `string` | Notas da pesquisa, incluindo nomes alternativos, evidencias, inferencias e pendencias. Origem: bronze_gramados. | Texto livre de evidências, hipóteses de continuidade, nomes alternativos e pendências; sem enumeração fechada. | bronze_gramados.observacoes; preservado. |

## `silver_partidas_gramados`

**Granularidade:** Uma partida de 2023–2024 com a pesquisa de gramado e o status de elegibilidade.

**Origem e processamento:** bronze_partidas tratada e filtrada para 2023–2024, associada por LEFT JOIN à pesquisa de gramados tratada usando igualdade exata de arena; cálculo de status pela classificação e pela data.

**Notebook responsável:** `02_tratamento`. Nome completo: `workspace.mvp_gramados.silver_partidas_gramados`.

| Coluna | Tipo persistido | Descrição do catálogo | Domínio esperado | Origem / transformação |
|---|---|---|---|---|
| `arena` | `string` | Nome do estádio conforme a base de partidas. Utilizado para associar a partida à pesquisa de gramados. Nomes diferentes podem representar o mesmo estádio. | Nome de estádio; domínio textual aberto, com possíveis nomes alternativos do mesmo local. | bronze_partidas.arena; preservado no recorte. |
| `ID` | `int` | Identificador da partida na fonte original. Sem valores ausentes ou repetidos no recorte analisado. | Inteiro ≥ 1; único por partida; sem máximo de negócio definido. | bronze_partidas.ID → TRY_CAST AS INT. |
| `rodada` | `int` | Número da rodada do campeonato. Coluna originalmente chamada rodata. | Inteiro ≥ 1; máximo depende da edição do campeonato. Não impor 38 à base histórica inteira. | bronze_partidas.rodata → renomeada rodada e TRY_CAST AS INT. |
| `data` | `date` | Data da partida, utilizada no recorte de 2023–2024 e na verificação da validade do gramado. | Data válida em dd/MM/yyyy na Bronze; na Silver, de 01/01/2023 a 31/12/2024 pelo filtro. | bronze_partidas.data → TRY_TO_TIMESTAMP com dd/MM/yyyy → DATE; filtro 2023–2024. |
| `hora` | `string` | Horário informado na fonte original, mantido como texto. Fuso horário não validado. | Texto HH:mm ou HH:mm:ss; horas 00–23, minutos e segundos 00–59; fuso não validado. | bronze_partidas.hora; preservado no recorte. |
| `mandante` | `string` | Clube que possui o mando da partida, independentemente do estádio utilizado. | Nome de clube presente na fonte; domínio textual aberto. Mandante e visitante devem ser distintos na partida. | bronze_partidas.mandante; preservado no recorte. |
| `visitante` | `string` | Clube visitante da partida. | Nome de clube presente na fonte; domínio textual aberto. Mandante e visitante devem ser distintos na partida. | bronze_partidas.visitante; preservado no recorte. |
| `formacao_mandante` | `string` | Formação tática do mandante informada na fonte. Não utilizada nos indicadores deste MVP. | Texto de formação tática, por exemplo 4-2-3-1, ou ausência; sem lista fechada validada. | bronze_partidas.formacao_mandante; preservado no recorte. |
| `formacao_visitante` | `string` | Formação tática do visitante informada na fonte. Não utilizada nos indicadores deste MVP. | Texto de formação tática, por exemplo 4-2-3-1, ou ausência; sem lista fechada validada. | bronze_partidas.formacao_visitante; preservado no recorte. |
| `tecnico_mandante` | `string` | Nome do técnico do mandante informado na fonte. | Nome de técnico ou ausência; domínio textual aberto. | bronze_partidas.tecnico_mandante; preservado no recorte. |
| `tecnico_visitante` | `string` | Nome do técnico do visitante informado na fonte. | Nome de técnico ou ausência; domínio textual aberto. | bronze_partidas.tecnico_visitante; preservado no recorte. |
| `vencedor` | `string` | Vencedor informado na fonte; o hífen indica empate. Os indicadores do MVP são calculados pelos placares. | Nome do mandante, nome do visitante ou hífen (-) para empate; esperado coerente com o placar. | bronze_partidas.vencedor; preservado no recorte. |
| `mandante_Placar` | `int` | Quantidade de gols marcados pelo mandante na partida. | Inteiro ≥ 0; sem máximo de negócio fixado. Valores altos exigem análise, não exclusão automática. | bronze_partidas.mandante_Placar → TRY_CAST AS INT. |
| `visitante_Placar` | `int` | Quantidade de gols marcados pelo visitante na partida. | Inteiro ≥ 0; sem máximo de negócio fixado. Valores altos exigem análise, não exclusão automática. | bronze_partidas.visitante_Placar → TRY_CAST AS INT. |
| `mandante_Estado` | `string` | Sigla do estado do clube mandante conforme a fonte. | AC, AL, AP, AM, BA, CE, DF, ES, GO, MA, MT, MS, MG, PA, PB, PR, PE, PI, RJ, RN, RS, RO, RR, SC, SP, SE, TO. | bronze_partidas.mandante_Estado; preservado no recorte. |
| `visitante_Estado` | `string` | Sigla do estado do clube visitante conforme a fonte. | AC, AL, AP, AM, BA, CE, DF, ES, GO, MA, MT, MS, MG, PA, PB, PR, PE, PI, RJ, RN, RS, RO, RR, SC, SP, SE, TO. | bronze_partidas.visitante_Estado; preservado no recorte. |
| `tipo_gramado` | `string` | Classificação da pesquisa: natural, sintetico, hibrido ou nao_confirmado. A aplicação à partida depende do período de validade. | natural, sintetico, hibrido, nao_confirmado. Na junção, nulo também pode ocorrer por falta de correspondência. | Pesquisa de gramados tratada.tipo_gramado; trazido pela junção em arena. |
| `inicio_validade` | `date` | Início inclusivo do intervalo considerado para a classificação nesta pesquisa. Não representa necessariamente a data de instalação. | Data válida (yyyy-MM-dd no CSV de pesquisa); início ≤ fim. Ausência admitida em pesquisa pendente; ambas obrigatórias para elegibilidade. | Pesquisa de gramados tratada.inicio_validade; trazido pela junção em arena. |
| `fim_validade` | `date` | Fim inclusivo do intervalo considerado para a classificação nesta pesquisa. Não representa necessariamente a data de retirada ou troca. | Data válida (yyyy-MM-dd no CSV de pesquisa); início ≤ fim. Ausência admitida em pesquisa pendente; ambas obrigatórias para elegibilidade. | Pesquisa de gramados tratada.fim_validade; trazido pela junção em arena. |
| `fonte_url` | `string` | Endereço ou endereços das fontes consultadas na pesquisa sobre o gramado. | Um ou mais endereços de referências, possivelmente separados por quebra de linha. URL preenchida não comprova classificação nem disponibilidade do site. | Pesquisa de gramados tratada.fonte_url; trazido pela junção em arena. |
| `observacoes` | `string` | Notas sobre evidências, nomes alternativos do estádio, limitações e hipóteses de continuidade da classificação. | Texto livre de evidências, hipóteses de continuidade, nomes alternativos e pendências; sem enumeração fechada. | Pesquisa de gramados tratada.observacoes; trazido pela junção em arena. |
| `status_gramado` | `string` | Resultado da verificação: sem_classificacao, pesquisa_pendente, periodo_nao_informado, dentro_do_periodo ou fora_do_periodo. Apenas dentro_do_periodo entra nas análises Gold. | sem_classificacao, pesquisa_pendente, periodo_nao_informado, dentro_do_periodo, fora_do_periodo. | Derivado da classificação, das duas datas de validade e da data da partida; precedência descrita na seção de regras. |

## `gold_resumo_gramado`

**Granularidade:** Um tipo de gramado nas partidas elegíveis de 2023–2024.

**Origem e processamento:** silver_partidas_gramados filtrada por dentro_do_periodo; agrupamento por tipo_gramado.

**Notebook responsável:** `03_analise`. Nome completo: `workspace.mvp_gramados.gold_resumo_gramado`.

| Coluna | Tipo persistido | Descrição do catálogo | Domínio esperado | Origem / transformação |
|---|---|---|---|---|
| `tipo_gramado` | `string` | Tipo de gramado das partidas elegiveis: natural, sintetico ou hibrido. | natural, sintetico, hibrido. | tipo_gramado da Silver elegível; chave do agrupamento. |
| `quantidade_partidas` | `bigint` | Quantidade de partidas do grupo com status dentro_do_periodo. Inteiro positivo. | Inteiro de 1 a N, sendo N o total elegível (737 nesta execução). | COUNT(*) das partidas elegíveis do grupo. |
| `media_gols` | `double` | Media da soma dos gols do mandante e do visitante nas partidas do grupo. Valor maior ou igual a zero. | Real ≥ 0; entre o menor e o maior total de gols das partidas do grupo. Sem teto universal fixado. | AVG(mandante_Placar + visitante_Placar). |
| `vitorias_mandante` | `bigint` | Quantidade de partidas do grupo em que o placar do mandante supera o do visitante. Entre zero e quantidade_partidas. | Inteiro de 0 a quantidade_partidas do grupo. | SUM(CASE WHEN mandante_Placar > visitante_Placar THEN 1 ELSE 0 END). |
| `taxa_vitoria_mandante` | `double` | Proporcao de vitorias do mandante no grupo: vitorias_mandante divididas por quantidade_partidas. Valor entre 0 e 1. | Real entre 0 e 1, inclusive; multiplicar por 100 apenas para apresentação percentual. | AVG(CASE WHEN mandante_Placar > visitante_Placar THEN 1 ELSE 0 END); equivalente a vitórias / quantidade. |

## `gold_resumo_ano_gramado`

**Granularidade:** Uma combinação de ano e tipo de gramado nas partidas elegíveis.

**Origem e processamento:** silver_partidas_gramados filtrada por dentro_do_periodo; agrupamento por ano da data e tipo_gramado.

**Notebook responsável:** `03_analise`. Nome completo: `workspace.mvp_gramados.gold_resumo_ano_gramado`.

| Coluna | Tipo persistido | Descrição do catálogo | Domínio esperado | Origem / transformação |
|---|---|---|---|---|
| `ano` | `int` | Ano extraido da data da partida. Valores do recorte: 2023 e 2024. | 2023 ou 2024. | YEAR(data); chave do agrupamento. |
| `tipo_gramado` | `string` | Tipo de gramado das partidas elegiveis: natural, sintetico ou hibrido. | natural, sintetico, hibrido. | tipo_gramado da Silver elegível; chave do agrupamento. |
| `quantidade_partidas` | `bigint` | Quantidade de partidas do grupo com status dentro_do_periodo. Inteiro positivo. | Inteiro de 1 a N, sendo N o total elegível (737 nesta execução). | COUNT(*) das partidas elegíveis do grupo. |
| `media_gols` | `double` | Media da soma dos gols do mandante e do visitante nas partidas do grupo. Valor maior ou igual a zero. | Real ≥ 0; entre o menor e o maior total de gols das partidas do grupo. Sem teto universal fixado. | AVG(mandante_Placar + visitante_Placar). |
| `vitorias_mandante` | `bigint` | Quantidade de partidas do grupo em que o placar do mandante supera o do visitante. Entre zero e quantidade_partidas. | Inteiro de 0 a quantidade_partidas do grupo. | SUM(CASE WHEN mandante_Placar > visitante_Placar THEN 1 ELSE 0 END). |
| `taxa_vitoria_mandante` | `double` | Proporcao de vitorias do mandante no grupo: vitorias_mandante divididas por quantidade_partidas. Valor entre 0 e 1. | Real entre 0 e 1, inclusive; multiplicar por 100 apenas para apresentação percentual. | AVG(CASE WHEN mandante_Placar > visitante_Placar THEN 1 ELSE 0 END); equivalente a vitórias / quantidade. |

## `gold_resumo_clube_gramado`

**Granularidade:** Uma combinação de clube mandante e tipo de gramado nas partidas elegíveis.

**Origem e processamento:** silver_partidas_gramados filtrada por dentro_do_periodo; agrupamento por mandante e tipo_gramado.

**Notebook responsável:** `03_analise`. Nome completo: `workspace.mvp_gramados.gold_resumo_clube_gramado`.

| Coluna | Tipo persistido | Descrição do catálogo | Domínio esperado | Origem / transformação |
|---|---|---|---|---|
| `mandante` | `string` | Clube mandante usado no agrupamento junto ao tipo de gramado. | Nome de clube presente na fonte; domínio textual aberto. Mandante e visitante devem ser distintos na partida. | mandante da Silver elegível; chave do agrupamento. |
| `tipo_gramado` | `string` | Tipo de gramado das partidas elegiveis: natural, sintetico ou hibrido. | natural, sintetico, hibrido. | tipo_gramado da Silver elegível; chave do agrupamento. |
| `quantidade_partidas` | `bigint` | Quantidade de partidas do grupo com status dentro_do_periodo. Inteiro positivo. | Inteiro de 1 a N, sendo N o total elegível (737 nesta execução). | COUNT(*) das partidas elegíveis do grupo. |
| `media_gols` | `double` | Media da soma dos gols do mandante e do visitante nas partidas do grupo. Valor maior ou igual a zero. | Real ≥ 0; entre o menor e o maior total de gols das partidas do grupo. Sem teto universal fixado. | AVG(mandante_Placar + visitante_Placar). |
| `vitorias_mandante` | `bigint` | Quantidade de partidas do grupo em que o placar do mandante supera o do visitante. Entre zero e quantidade_partidas. | Inteiro de 0 a quantidade_partidas do grupo. | SUM(CASE WHEN mandante_Placar > visitante_Placar THEN 1 ELSE 0 END). |
| `taxa_vitoria_mandante` | `double` | Proporcao de vitorias do mandante no grupo: vitorias_mandante divididas por quantidade_partidas. Valor entre 0 e 1. | Real entre 0 e 1, inclusive; multiplicar por 100 apenas para apresentação percentual. | AVG(CASE WHEN mandante_Placar > visitante_Placar THEN 1 ELSE 0 END); equivalente a vitórias / quantidade. |

## Regras da associação histórica e elegibilidade

A partida é associada à pesquisa por igualdade exata de `arena`, mediante LEFT JOIN. O tipo de gramado atual de um estádio não é aplicado retroativamente: a data do jogo deve pertencer ao intervalo pesquisado. Os limites são inclusivos e representam o intervalo adotado na pesquisa, não necessariamente as datas de instalação ou retirada do gramado. Hipóteses de continuidade precisam permanecer registradas nas observações.

O status é calculado nesta ordem:

1. Tipo de gramado nulo: `sem_classificacao`.
2. Tipo `nao_confirmado`: `pesquisa_pendente`.
3. Início ou fim de validade nulo: `periodo_nao_informado`.
4. Data da partida entre início e fim, inclusive: `dentro_do_periodo`.
5. Demais casos: `fora_do_periodo`.

A execução apresentou 760 linhas e 760 IDs distintos na Silver: 737 com `dentro_do_periodo` e 23 com `pesquisa_pendente`. As pendências correspondem a Antônio Accioly (18), Raulino de Oliveira (3), Luso-Brasileiro (1) e Orlando Scarpelli (1). Todos os 760 jogos permanecem na Silver; somente os 737 elegíveis entram nas tabelas Gold.

## Campos derivados e interpretação dos indicadores

`ano`, `total_gols` e `vitoria_mandante` são derivados durante a análise. `total_gols = mandante_Placar + visitante_Placar`; `vitoria_mandante` vale 1 quando o mandante marca mais gols e 0 nos demais casos. Eles não são colunas adicionais persistidas na Silver exportada. `ano` é persistido na Gold por ano; os demais alimentam as agregações.

A taxa persistida é uma proporção de 0 a 1. O campo de apresentação `vitorias_mandante_pct`, exibido nos resultados, é a taxa multiplicada por 100 e não integra as 67 colunas persistidas deste catálogo. As médias e taxas são arredondadas para exibição, sem confundir o valor exibido com o valor armazenado.

Os indicadores descrevem associações observadas. Não demonstram que o gramado causou diferenças nos resultados; composição dos clubes, adversários, temporadas e exclusões também podem influenciar as comparações.

## Qualidade e limites conhecidos

- Na Bronze histórica, as formações táticas possuem 4.975 ausências por coluna e os técnicos 4.610 por coluna. Esses campos são mantidos e não entram nos indicadores.
- Quatro registros de pesquisa têm início e fim ausentes e estão classificados como `nao_confirmado`.
- Foi detectada uma divergência interna entre vencedor e placar no ID 1114, de 2005, fora do recorte analítico. A Bronze foi preservada; os indicadores usam os placares. Essa comparação não estabelece, isoladamente, qual campo está factualmente correto.
- As verificações apresentadas não encontraram IDs repetidos, duplicatas integrais, datas ou horários inválidos na Bronze. Verificações de formato e consistência não equivalem a auditoria externa de cada fato.
- No recorte de 760 partidas, os totais de gols observados vão de 0 a 10. Valores extremos foram mantidos; não são automaticamente erros.
- Endereços de referência e observações permitem rastrear a pesquisa, mas não garantem que cada fonte comprove todo o intervalo. As limitações históricas e as inferências de continuidade devem ser consideradas.

## Evidências e integração com o README

O [README](../README.md) apresenta a modelagem e incorpora as evidências disponíveis da execução em nuvem. O catálogo transcrito neste documento contém as 67 colunas da exportação de metadados, sem depender de uma captura individual de cada coluna.

- [Sete tabelas no Catalog Explorer](evidencias/08_catalogo_completo.png).
- [Descrições das tabelas consultadas no catálogo](evidencias/09_descricoes_tabela.png).
- [Cobertura e unicidade das partidas após a junção](evidencias/18_cobertura_e_pendencias.png).

Os comentários das tabelas e das 22 colunas de `silver_partidas_gramados` são aplicados no notebook `03_analise`. As outras 45 colunas são documentadas no notebook `05_catalogo_dados`, que também verifica os metadados. Caso as tabelas sejam recriadas e percam comentários, é necessário reaplicar as respectivas células de ambos os notebooks.
