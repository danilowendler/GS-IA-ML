# Dados

Dados **reais e públicos**, sem nenhuma simulação.

## Fonte

| Item | Valor |
| --- | --- |
| Instituição | INMET · Instituto Nacional de Meteorologia |
| Base | Dados históricos anuais das estações automáticas (BDMEP) |
| Link | https://portal.inmet.gov.br/dadoshistoricos |
| Estação | A701 · São Paulo, Mirante de Santana (zona norte) |
| Granularidade | Horária (horário UTC, convertido para o horário de Brasília no notebook) |
| Período | 1º de janeiro de 2019 a 31 de dezembro de 2025 (61.368 registros horários) |
| Data do download | 2026-11-08 |

## Arquivos

Os CSVs originais da estação A701, um por ano, estão em `brutos/`, sem nenhuma alteração. Eles ficam no repositório (cerca de 1 MB cada) para que o notebook rode sem depender do site do INMET.

Para baixar de novo, o próprio notebook (seção 1) busca os ZIPs anuais no link acima e extrai só o CSV da A701. Se o site estiver fora do ar, baixe os ZIPs manualmente e coloque os CSVs da A701 em `brutos/`.

## Variáveis usadas

| Coluna original do INMET | Nome no notebook |
| --- | --- |
| TEMPERATURA DO AR - BULBO SECO, HORARIA (°C) | `temp` |
| TEMPERATURA MÁXIMA NA HORA ANT. (AUT) (°C) | `temp_max_hora` |
| UMIDADE RELATIVA DO AR, HORARIA (%) | `umidade` |
| PRESSAO ATMOSFERICA AO NIVEL DA ESTACAO, HORARIA (mB) | `pressao` |
| RADIACAO GLOBAL (Kj/m²) | `radiacao` |
| PRECIPITAÇÃO TOTAL, HORÁRIO (mm) | `precip` |
| VENTO, VELOCIDADE HORARIA (m/s) | `vento` |

## Tratamento

- Valores `-9999` (medição ausente) viram nulos.
- Valores fisicamente impossíveis (temperatura fora de -5 °C a 45 °C, umidade fora de 1% a 100%) viram nulos.
- Nomes de colunas padronizados, já que mudam entre os anos.
- Dias com menos de 18 horas medidas são descartados.

## Limitação

A estação A701 fica no Mirante de Santana, a cerca de 10 km da Vila Mariana. O modelo prevê o dia na cidade; o calor de cada esquina da Vila Mariana é medido pelos totens da Refresca.
