# Relatório de ML · Refresca

**Global Solution 2026.2 · Inteligência Artificial & Machine Learning**
Notebook: [`refresca_ml.ipynb`](refresca_ml.ipynb) · Dados: [`dados/README.md`](dados/README.md)

## 1. Problema

Trabalhadores de rua da Vila Mariana, como entregadores, ambulantes e garis, passam o dia expostos ao sol e não sabem com antecedência quando o calor vai ficar perigoso. A Refresca quer avisá-los **na véspera**, para que levem água e planejem pausas na sombra.

**Pergunta de ML:** com os dados do clima de hoje, amanhã será um dia de calor perigoso em São Paulo?

É uma classificação binária: para cada dia, o modelo dá a probabilidade de o dia seguinte ser perigoso. Acima de um limiar, a Refresca emite o alerta.

## 2. Dados

- **Fonte:** INMET, dados históricos horários da estação automática A701 (São Paulo, Mirante de Santana), https://portal.inmet.gov.br/dadoshistoricos. São dados reais e públicos, sem simulação.
- **Período:** 2019 a 2025, com 61.368 registros horários.
- **Tratamento:** os valores `-9999` viraram nulos, os nomes das colunas foram padronizados entre os anos e o horário UTC foi convertido para o de Brasília. Foram descartados 16 dias com menos de 18 horas medidas, sobrando 2.542 dias.
- **Radiação:** a coluna tem 47% de horas vazias porque o INMET não registra radiação à noite. Na soma diária, essas horas contam como zero.
- **Base final:** 2.507 dias e 26 variáveis. Os dias perdidos em relação a 2.542 são os primeiros de cada trecho, que não têm os dias anteriores necessários para as variáveis de atraso.

## 3. Rótulo

Para cada hora, calculamos o **índice de calor** com a fórmula de Rothfusz (NOAA), que combina temperatura e umidade. Um dia é **perigoso** quando o índice máximo chega a **32 °C**: é o início da faixa de "cuidado extremo" da NOAA e o mesmo limite do nível "perigo" do totem e do app da Refresca.

Dias perigosos são raros, **4,9%** da base, e se concentram de setembro a março: 12,5% dos dias de janeiro e nenhum de maio a agosto. Eles também aumentaram: foram 4 em 2021 e 4 em 2022, contra 27 em 2023 e 36 em 2024.

## 4. Variáveis

Usamos só informações disponíveis até o fim do dia de hoje, para não vazar o futuro:

- **clima de hoje:** temperaturas, umidade, índice de calor máximo, pressão, radiação, chuva e vento;
- **dias anteriores:** valores de 1 e 2 dias antes e médias dos últimos 3 dias, para captar ondas de calor;
- **variações:** pressão e temperatura máxima em relação a ontem;
- **época do ano:** seno e cosseno do dia do ano.

O módulo [`preparar_variaveis.py`](preparar_variaveis.py) reproduz esse cálculo para o serviço de previsão. O notebook confere que os dois dão o mesmo resultado nos 722 dias do teste.

## 5. Modelo escolhido: XGBoost

- É o modelo mais forte, em geral, para dados tabulares com poucas dezenas de variáveis.
- Captura interações não lineares, como "a umidade pesa mais quando a temperatura já está alta", sem precisar normalizar os dados.
- Trata classes raras pelo `scale_pos_weight`.
- Mostra a importância de cada variável, o que ajuda a explicar o resultado.

## 6. Divisão treino e teste

A divisão é **pelo tempo**. Sortear dias vazaria informação, porque dias vizinhos são muito parecidos.

| Conjunto | Período | Dias | Dias perigosos |
| --- | --- | --- | --- |
| Treino | 03/01/2019 a 31/12/2023 | 1.785 | 67 (3,8%) |
| Teste | 01/01/2024 a 30/12/2025 | 722 | 56 (7,8%) |

O teste tem o dobro da proporção de dias perigosos do treino. Isso reflete os anos mais quentes de 2024 e 2025 e torna o teste mais difícil, porque o modelo enfrenta um clima mais extremo do que o que viu no treino.

## 7. Treino e hiperparâmetros

- **Busca em grade com 48 combinações:** `max_depth` (2, 3, 4, 6) × `learning_rate` (0,03; 0,1; 0,3) × `n_estimators` (100, 300) × `scale_pos_weight` (peso total de 25,6 ou sua raiz, 5,06).
- **Validação cruzada só no treino**, com `TimeSeriesSplit` em 3 partes, em que cada parte valida um período posterior ao que treinou. Com 5 partes, algumas validações ficavam sem nenhum dia perigoso. Com 3, as validações têm 8, 7 e 27 dias perigosos.
- **Métrica de escolha: PR AUC.** Com 4,9% de positivos, a acurácia engana: dizer sempre "não perigoso" acertaria 95% dos dias. A PR AUC mede se o modelo põe os dias perigosos no topo da lista de probabilidades.

| Posição | max_depth | learning_rate | n_estimators | scale_pos_weight | PR AUC treino (CV) | PR AUC validação (CV) |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 2 | 0,03 | 100 | 5,06 | 0,756 | **0,302** |
| 2 | 3 | 0,03 | 100 | 5,06 | 0,938 | 0,296 |
| 3 | 6 | 0,10 | 100 | 5,06 | 1,000 | 0,292 |
| 4 | 2 | 0,03 | 100 | 25,64 | 0,706 | 0,289 |

A busca preferiu árvores rasas, aprendizado lento e o peso moderado.

**Limiar do alerta:** também foi escolhido só com o treino, juntando as previsões das três validações e pegando o limiar com o maior F1. O resultado foi **0,46**.

## 8. Resultados

| Métrica | Treino | Teste | Regra "amanhã = hoje" (teste) |
| --- | --- | --- | --- |
| Precisão | 0,439 | 0,446 | 0,545 |
| Recall | 0,806 | **0,661** | 0,536 |
| F1 | 0,568 | 0,532 | 0,541 |
| F2 (recall pesa o dobro) | 0,691 | **0,603** | 0,538 |
| ROC AUC | 0,975 | **0,911** | 0,749 |
| PR AUC | 0,642 | **0,463** | 0,328 |
| Alertas emitidos | 123 | 83 | 55 |

O notebook mostra também a matriz de confusão, as curvas ROC de treino e de teste, a curva de escolha do limiar e a importância das variáveis. As variáveis que mais pesam são o índice de calor máximo de hoje, a temperatura máxima de hoje e a média da temperatura máxima nos últimos 3 dias, o que é coerente com ondas de calor que duram vários dias.

**Overfitting.** Na validação cruzada, a PR AUC de treino chega a 1,0 com árvores profundas, enquanto a de validação fica entre 0,25 e 0,30. O modelo decora com facilidade os poucos dias perigosos do treino, e por isso a busca escolheu árvores de profundidade 2. No modelo final, a PR AUC cai de 0,642 no treino para 0,463 no teste, e a ROC AUC de 0,975 para 0,911. É um overfitting moderado, esperado com só 67 exemplos positivos. A precisão e o F1 ficam estáveis entre treino e teste, o que indica que o limiar escolhido no treino funciona em dados novos.

**Comparação com a regra simples.** "Amanhã será perigoso se hoje foi perigoso" é uma referência forte, porque o calor persiste por vários dias. No F1, o modelo empata com ela (0,532 contra 0,541). Mas o modelo encontra mais dias perigosos (recall de 66% contra 54%) e separa muito melhor os dias de risco (ROC AUC de 0,911 contra 0,749).

## 9. Conclusão para o cidadão

Em 2024 e 2025 houve 56 dias de calor perigoso em São Paulo:

- **A Refresca avisaria na véspera em 37 deles (66%)**: cerca de 7 a cada 10 dias perigosos. A regra simples avisaria em 30 (54%), cerca de 5 a cada 10.
- O preço são mais alarmes falsos: dos 83 alertas da Refresca, 46 seriam em dias que não chegaram ao limite de perigo. Na regra simples, seriam 25 de 55.

Para o trabalhador de rua, esse equilíbrio faz sentido. Um alarme falso custa pouco: levar uma garrafa de água a mais num dia que ficou só quente. Já um dia perigoso sem aviso pode custar exaustão ou insolação. Além disso, os alarmes falsos não foram dias frescos: em 36 dos 46, o índice de calor chegou a 30 °C ou mais (mediana de 31,1 °C), perto do limite de perigo.

**Limitações e próximos passos**

- O modelo usa só a estação A701, na zona norte, a cerca de 10 km da Vila Mariana. Na Refresca, ele prevê o dia na cidade, e os totens medem o calor de cada esquina.
- Com mais anos de dados e outras estações (por exemplo, Interlagos), o modelo teria mais dias perigosos para aprender.
- Dados de previsão do tempo, em vez de só o clima de hoje, melhorariam muito o aviso da véspera. O modelo atual é uma base que a Refresca pode evoluir.
- Como 2023 a 2025 tiveram bem mais dias perigosos que os anos anteriores, o modelo deve ser retreinado todo ano.
