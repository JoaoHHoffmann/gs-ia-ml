# Relatório de Machine Learning | Dá Pra Ir?

Global Solution FIAP 2026, tema CitizenTech. Disciplina de Inteligência Artificial & Machine Learning.

## 1. Problema e pergunta de ML

O Dá Pra Ir? é um app para pessoas com deficiência, pessoas autistas e famílias atípicas. Hoje elas só descobrem se um lugar vai funcionar para elas quando já chegaram lá: barulho demais, elevador parado, calçada quebrada ou sem rampa no caminho. O modelo de ML foca na parte da calçada, que é a que tem dado público.

O recorte do projeto é a Vila Clementino, na subprefeitura Vila Mariana, em São Paulo. Ali ficam a AACD, o Hospital do Servidor Público Estadual e o Hospital São Paulo (Unifesp), lugares que recebem todos os dias muitas pessoas em cadeira de rodas ou com mobilidade reduzida.

A pergunta do modelo é "Tá funcionando?": quando alguém abre um chamado no SP156 sobre calçada, obstáculo ou rampa, esse chamado vai ser resolvido em até 30 dias? É um problema de classificação binária. O modelo treina com os chamados da cidade inteira, com a subprefeitura como variável, e a leitura final é feita para a Vila Mariana.

## 2. Dados reais

Usamos duas fontes públicas, com dados de 2024 e 2025:

| Fonte | O que usamos | Link |
|---|---|---|
| SP156, Prefeitura de São Paulo | 8 CSVs trimestrais de chamados (4.480.951 chamados no total) | https://dados.prefeitura.sp.gov.br/dataset/dados-do-sp156 |
| INMET | Dados horários da estação A701 (Mirante de Santana), agregados em chuva por dia | https://portal.inmet.gov.br/dadoshistoricos |

Dos chamados do SP156 ficamos com cinco serviços ligados a andar na calçada: manutenção de calçada pública, obstáculo na calçada, calçada particular danificada, rebaixamento de guia para travessia e fiscalização de rebaixamento irregular. São 26.390 chamados, sendo 1.535 da Vila Mariana. A fonte, os links de cada arquivo, as colunas e as limitações de cada dataset estão em `ml/dados/README.md`.

## 3. Escolha do modelo

Escolhemos o XGBoost porque os dados são uma tabela com variáveis categóricas e numéricas, e modelos de árvore com boosting costumam ir bem nesse tipo de dado. Ele também aceita um peso para a classe menor (`scale_pos_weight`), o que ajuda aqui, já que só cerca de 31% dos chamados são resolvidos no prazo. E ele mostra quais variáveis pesaram na previsão, o que facilita explicar o resultado.

Para saber se o XGBoost realmente vale a pena, comparamos com duas coisas. A primeira é uma referência que sempre responde a classe mais comum ("não resolvido"), que mostra o mínimo que qualquer modelo precisa superar. A segunda é um Random Forest com 200 árvores e `class_weight="balanced"`, outro modelo de árvore, mais simples de ajustar.

Não usamos LSTM porque cada linha é um chamado independente, e não uma sequência no tempo. Também não usamos MLP: em dados de tabela, ela costuma perder para modelos de árvore e não mostra quais variáveis pesaram. KMeans com PCA serve para agrupar dados sem rótulo, e aqui temos um alvo claro para prever.

## 4. Preparação e divisão treino/teste

O alvo `resolvido_30d` vale 1 quando o chamado tem status Finalizada e foi fechado em até 30 dias, e 0 quando foi fechado depois disso ou ainda está aberto ou em andamento. O status foi registrado em agosto de 2026 e o último chamado é de dezembro de 2025, então todos tiveram pelo menos 7 meses para ser fechados.

O modelo usa seis variáveis, todas conhecidas no dia em que o chamado é aberto:

- serviço, subprefeitura e canal (viram colunas de 0 e 1 com one-hot, 44 colunas no total);
- dia da semana da abertura;
- fila de pedidos: quantos chamados destes serviços foram abertos na mesma subprefeitura nos 7 dias anteriores;
- chuva total nos 7 dias anteriores, medida pelo INMET.

A divisão entre treino e teste é pelo tempo. O treino tem os chamados abertos em 2024 (10.348 chamados, 31,4% resolvidos) e o teste tem os abertos em 2025 (15.256 chamados, 31,5% resolvidos). Se a divisão fosse aleatória, o modelo usaria chamados de depois para prever chamados de antes, o que não acontece no uso real.

As decisões mais importantes da preparação foram estas:

| Decisão | Motivo |
|---|---|
| 2º trimestre de 2024 fora | O arquivo não tem a coluna Serviço, e sem ela não dá para separar os serviços de acessibilidade |
| Mês fora das variáveis | O treino não tem abril, maio e junho, e o modelo seria testado em meses que nunca viu |
| Órgão fora das variáveis | É o órgão no dia em que o arquivo foi gerado, não na abertura; usar seria vazamento de dados |
| Pesquisa de satisfação fora | Só existe depois do atendimento; também seria vazamento |
| 193 chamados cancelados fora | Um chamado cancelado não foi resolvido nem deixou de ser resolvido |
| 128 chamados sem subprefeitura válida fora | Vazia, "não encontrada" ou Usinas de Asfalto, que é um órgão e não um lugar |
| 465 chamados de 01 a 07/01/2024 e de 01 a 07/07/2024 fora | Os 7 dias anteriores caem em datas sem dados, e a fila e a chuva sairiam erradas |

Tema, assunto e distrito também ficaram de fora, porque repetem o que já está em serviço e subprefeitura.

## 5. Treino com diferentes parâmetros

No XGBoost testamos 12 combinações de três hiperparâmetros com `GridSearchCV`: número de árvores (200 e 400), profundidade máxima (3, 5 e 7) e taxa de aprendizado (0,05 e 0,1). A validação cruzada usa `TimeSeriesSplit` com 5 partes, só dentro do treino de 2024 e com os chamados em ordem de abertura. Assim, cada validação é feita em chamados abertos depois dos usados para treinar. A métrica da busca é o ROC-AUC, que mede se o modelo dá probabilidade maior aos chamados que de fato foram resolvidos. O teste de 2025 só é usado no fim.

As 3 melhores e as 3 piores combinações na validação estão abaixo. A tabela completa, com as 12, está na seção 5 do notebook.

| | Árvores | Profundidade | Taxa | ROC-AUC treino | ROC-AUC validação |
|---|---|---|---|---|---|
| Melhor | **200** | **5** | **0,05** | **0,936** | **0,867** |
| 2ª | 200 | 3 | 0,05 | 0,906 | 0,866 |
| 3ª | 400 | 3 | 0,05 | 0,918 | 0,866 |
| 10ª | 400 | 7 | 0,05 | 0,974 | 0,860 |
| 11ª | 400 | 5 | 0,1 | 0,971 | 0,859 |
| 12ª | 400 | 7 | 0,1 | 0,988 | 0,854 |

O ROC-AUC de treino da busca (0,936 na melhor) é a média das 5 partes da validação cruzada, em que cada treino usa só uma parte de 2024. O modelo final, treinado com 2024 inteiro, tem 0,920 no treino, que é o número da seção 6.

A melhor combinação foi 200 árvores, profundidade 5 e taxa 0,05. Da 2ª à 5ª posição, quatro combinações empatam em 0,866. A validação muda pouco entre as 12 (de 0,854 a 0,867), mas o treino sobe de 0,906 para 0,988 conforme o modelo fica mais complexo. Ou seja, as combinações mais complexas decoram o treino sem prever melhor. O gráfico "ROC-AUC no treino e na validação das 12 combinações" está na seção 6 do notebook.

## 6. Resultados em treino e teste

| Modelo | Acurácia (treino / teste) | Precisão | Recall | F1 | ROC-AUC |
|---|---|---|---|---|---|
| Referência (classe mais comum) | 0,686 / 0,685 | 0,000 / 0,000 | 0,000 / 0,000 | 0,000 / 0,000 | 0,500 / 0,500 |
| Random Forest | 0,964 / 0,765 | 0,902 / 0,605 | 0,994 / 0,730 | 0,946 / 0,661 | 0,997 / 0,833 |
| **XGBoost** | 0,825 / 0,764 | 0,664 / 0,588 | 0,893 / 0,840 | 0,762 / 0,692 | 0,920 / **0,849** |

Precisão, recall e F1 são da classe "resolvido". O XGBoost foi o melhor no teste, com ROC-AUC de 0,849 e F1 de 0,692, e encontra 84% dos chamados que foram resolvidos.

A acurácia sozinha engana aqui. A referência acerta 68,5% no teste sem acertar nenhum chamado resolvido, só porque a maioria não é resolvida. E o Random Forest e o XGBoost têm quase a mesma acurácia (0,765 e 0,764), mesmo com o XGBoost separando melhor as duas classes.

Os dois modelos de árvore têm overfitting. O Random Forest é quase perfeito no treino (ROC-AUC de 0,997) e cai para 0,833 no teste. O XGBoost vai de 0,920 para 0,849. Ele já era o modelo principal, e o teste confirma a escolha: tem o melhor ROC-AUC e uma queda menor que a do Random Forest.

No teste, a matriz de confusão do XGBoost mostra que, dos 4.807 chamados resolvidos, ele acertou 4.040 e errou 767. Dos 10.449 não resolvidos, acertou 7.619 e marcou 2.830 como resolvidos. O gráfico está na seção 6 do notebook.

## 7. Conclusão e leitura para o cidadão

Em 2025, só 31,5% dos chamados de calçada, obstáculo e rampa da cidade foram resolvidos em até 30 dias. Na Vila Mariana foi um pouco melhor, 35,6%. A diferença maior está no tipo de problema:

| Serviço (Vila Mariana, 2025) | Chamados | Resolvidos em até 30 dias | Previsto pelo modelo |
|---|---|---|---|
| Calçada pública - manutenção | 230 | 73,9% | 67,3% |
| Guias para travessia - rebaixamento | 39 | 71,8% | 79,2% |
| Obstáculo na calçada | 218 | 19,3% | 12,3% |
| Calçada particular danificada | 291 | 14,8% | 12,0% |
| Fiscalização de rebaixamento irregular | 20 | 5,0% | 5,8% |

"Previsto pelo modelo" é a probabilidade média prevista, corrigida para tirar o efeito do peso dado aos resolvidos no treino (explicado na seção 6 do notebook).

O modelo aprende principalmente essa diferença entre tipos de serviço e regiões. Chuva, fila de pedidos e dia da semana quase não influenciam. O maior valor dele está em mostrar quais pedidos ficam parados. No app, isso vira um aviso simples: "esse tipo de problema costuma levar mais de 30 dias; se o obstáculo na calçada estava lá semana passada, provavelmente ainda está". Esses números também servem de prova para cobrar a Prefeitura.

O erro mais perigoso é o modelo dizer que um chamado foi resolvido quando não foi. Isso aconteceu com 2.830 chamados em 2025, 27% dos não resolvidos. Quando o modelo diz que um chamado vai ser resolvido, ele acerta só 59% das vezes (precisão de 0,588). Para quem usa cadeira de rodas, é confiar que a calçada foi arrumada e encontrar o problema lá. Por isso o aviso do app deve ser cauteloso, e a confirmação vem dos relatos de quem passou pelo lugar.

Os dados públicos respondem só parte da pergunta do app. Como é o lugar por dentro e se deu certo para outras famílias, só a comunidade sabe. Quando o app tiver esses relatos, o mesmo processo de treino pode ser usado para prever "vai dar certo pra mim?".

Os gráficos de importância das variáveis e de real contra previsto na Vila Mariana estão na seção 6 do notebook.

## 8. Limitações

- "Finalizada" quer dizer que a Prefeitura encerrou o chamado, não que a calçada foi consertada.
- O 2º trimestre de 2024 ficou fora do treino.
- Chamados não são o mesmo que problemas: um mesmo buraco pode ter vários chamados ou nenhum.
- O status de todos os chamados foi registrado em agosto de 2026, uma única vez.
- A chuva é medida em um ponto só, no Mirante de Santana, e o modelo usa chamados da cidade inteira.
- Alguns serviços têm poucos chamados na Vila Mariana (a fiscalização de rebaixamento teve só 20 em 2025), então essas porcentagens são pouco confiáveis.

## 9. ODS

- **ODS 3, Saúde e bem-estar:** ajuda pessoas que vão a consultas e terapias na região da AACD e dos hospitais a chegar com segurança.
- **ODS 10, Redução das desigualdades:** dá às pessoas com deficiência e às famílias atípicas uma informação que hoje elas não têm antes de sair de casa.
- **ODS 11, Cidades e comunidades sustentáveis:** mostra quais problemas de calçada ficam parados e dá números para cobrar a Prefeitura por uma cidade mais acessível.
