# Dados

Usamos duas fontes públicas, as duas com dados reais de 2024 e 2025.

Os arquivos originais ficam em `ml/dados/brutos/`, que não vai para o GitHub: eles somam cerca de 1,1 GB e já são públicos nas fontes oficiais. Nesta pasta ficam só os CSVs já filtrados e pequenos, gerados pelo notebook.

## Como refazer os dados do zero

1. Na raiz do projeto, crie o ambiente virtual com Python 3.12 e instale as bibliotecas:
   ```
   py -3.12 -m venv .venv
   .venv\Scripts\python -m pip install -r ml/requirements.txt
   ```
2. Baixe os arquivos `2024.zip` e `2025.zip` do INMET pelo navegador, em https://portal.inmet.gov.br/dadoshistoricos, e salve os dois em `ml/dados/brutos/` com esses mesmos nomes. Crie a pasta se ela ainda não existir. Esse passo é manual porque o download pelo notebook não funcionou.
3. Abra `ml/notebook_ml.ipynb`, escolha o `.venv` como kernel e rode todas as células. O notebook baixa os 8 CSVs do SP156 (cerca de 930 MB) e gera os dois CSVs desta pasta. Arquivos que já estão em `ml/dados/brutos/` não são baixados de novo.

Para rodar pelo terminal, a partir da raiz do projeto:
```
.venv\Scripts\python -m nbconvert --to notebook --execute --inplace ml\notebook_ml.ipynb
```

## Arquivos desta pasta

| Arquivo | O que é | Gerado a partir de |
|---|---|---|
| `sp156_acessibilidade.csv` | Chamados do SP156 de cinco serviços de calçada, obstáculo e rampa | SP156 |
| `inmet_a701_diario.csv` | Chuva por dia na estação A701 | INMET |

## 1. SP156: chamados da Prefeitura de São Paulo

**Fonte:** Prefeitura de São Paulo, portal de dados abertos.
- Página: https://dados.prefeitura.sp.gov.br/dataset/dados-do-sp156
- API: https://dados.prefeitura.sp.gov.br/api/3/action/package_show?id=dados-do-sp156
- Arquivos: 8 CSVs trimestrais (1º a 4º trimestre de 2024 e de 2025), de 85 MB a 155 MB cada. Os links estão no notebook.

**Período usado:** chamados abertos de 01/01/2024 a 31/12/2025, menos o 2º trimestre de 2024 (veja as limitações).

**Como o CSV foi montado:** cada arquivo é lido em partes de 200 mil linhas. Ficam só os chamados destes cinco serviços (26.390 chamados, sendo 1.535 da subprefeitura Vila Mariana):

| Serviço | Chamados na cidade | Na Vila Mariana |
|---|---|---|
| Calçada pública - solicitar manutenção | 9.986 | 447 |
| Acessibilidade - solicitar avaliação de obstáculo na calçada | 7.389 | 406 |
| Calçada particular - denunciar calçada danificada ou inexistente | 7.099 | 563 |
| Guias para travessia de pedestres - Solicitar rebaixamento | 1.153 | 85 |
| Guia rebaixada - solicitar fiscalização de rebaixamento irregular | 763 | 34 |

Os cinco foram escolhidos a partir de uma busca mais ampla pelas palavras calçada, rampa, rebaixamento, obstáculo e acessibilidade. Ficaram de fora serviços que não tratam de circular pela calçada, como poda de árvore, manutenção de sarjeta, pintura de guia e relógios de rua. O filtro usa o nome do serviço, porque o mesmo serviço às vezes aparece em mais de um assunto (por exemplo, em "Calçadas, guias e postes" e em "Fiação e Postes").

**Colunas de `sp156_acessibilidade.csv`:**
- `data_abertura`: dia em que o chamado foi aberto.
- `data_finalizacao`: dia em que o chamado foi fechado. Fica vazia nos chamados ainda abertos ou em andamento e em 4 chamados finalizados (veja as limitações).
- `canal`: por onde o chamado foi aberto (portal, aplicativo, telefone etc.).
- `tema`, `assunto`, `servico`: classificação do chamado, do mais geral ao mais específico.
- `status`: situação do chamado na data em que o arquivo foi gerado (Finalizada, Aberto, Em Andamento ou Cancelada).
- `orgao`: órgão da Prefeitura responsável pelo chamado no momento em que o arquivo foi gerado.
- `subprefeitura`, `distrito`: onde o problema fica.

As colunas `tema`, `assunto`, `distrito` e `orgao` ficam no CSV para descrever os dados, mas não entram no modelo. O motivo está no notebook.

**Colunas que não guardamos e por quê:**
- Logradouro, Bairro, Número, CEP, Latitude e Longitude: mostram o endereço exato de quem reclamou, e o distrito já basta para o nosso recorte.
- Pesquisa de satisfação: só existe depois que o chamado é atendido, então usar essas colunas para prever se ele será atendido seria vazamento de dados. Elas não aparecem nos CSVs, só na aba "Satisfação" dos arquivos XLSX, que não usamos.

**Alvo do modelo (`resolvido_30d`, criado no notebook, não salvo no CSV):**
- 1 (resolvido): status Finalizada e no máximo 30 dias entre `data_abertura` e `data_finalizacao`.
- 0 (não resolvido): status Finalizada com mais de 30 dias, ou status Aberto ou Em Andamento.
- Fora do modelo: status Cancelada (193 chamados), porque um chamado cancelado não foi resolvido nem deixou de ser resolvido.

O status foi registrado em agosto de 2026 e o último chamado foi aberto em dezembro de 2025, então todos tiveram pelo menos 7 meses para ser fechados. Quem ainda está aberto certamente não foi resolvido em 30 dias. Por isso não excluímos os últimos 30 dias do período.

| Ano | Não resolvido (0) | Resolvido (1) | Total | % resolvido |
|---|---|---|---|---|
| 2024 (treino) | 7.457 | 3.394 | 10.851 | 31,3% |
| 2025 (teste) | 10.503 | 4.843 | 15.346 | 31,6% |

Na preparação do modelo saem mais 128 chamados sem subprefeitura válida e 465 abertos de 01 a 07/01/2024 e de 01 a 07/07/2024, que não têm os 7 dias anteriores com dados. Por isso o modelo usa 10.348 chamados no treino e 15.256 no teste.

**Limitações conhecidas:**
- **O 2º trimestre de 2024 (abril a junho) ficou de fora.** O CSV desse trimestre não tem a coluna Serviço. O XLSX do mesmo trimestre também foi conferido e não tem a coluna Serviço na aba "Solicitações". Só com o assunto não dá para separar os serviços de acessibilidade dos outros (por exemplo, "Calçadas, guias e postes" mistura obstáculo na calçada com pintura de guias). A coluna Serviço aparece só na aba "Satisfação" do XLSX, que tem apenas os chamados avaliados depois de atendidos, então usar essa aba enviesaria os dados. Com isso, o treino do modelo fica sem abril, maio e junho.
- **A variável "mês" fica fora do modelo**, porque o treino não tem abril, maio e junho. O modelo seria testado em meses que nunca viu.
- "Finalizada" quer dizer que a Prefeitura encerrou o chamado, não necessariamente que a calçada foi consertada.
- A parte dos chamados com data de finalização é menor em 2025 (de 54% a 66% por mês) do que em 2024 (de 66% a 75% por mês). Mesmo assim, a parte resolvida em até 30 dias é quase igual nos dois anos (31,3% e 31,6%). A diferença está nos chamados fechados depois de 30 dias ou ainda abertos.
- O CSV do 1º trimestre de 2025 tem 1.048.575 linhas, o limite do Excel. Dessas, 742.904 são chamados e as outras 305.671 são linhas totalmente vazias no fim do arquivo, que o notebook descarta.
- Os nomes das colunas dos CSVs são diferentes dos que aparecem na API (DataStore) do portal. A API não serve para este trabalho: só alguns trimestres estão nela, e mesmo esses estão incompletos (por exemplo, o 1º trimestre de 2024 vai só até 17/01/2024).
- Os arquivos estão na codificação do Windows (cp1252) e usam ponto e vírgula como separador.
- As datas não têm hora, só o dia.
- O status e o órgão valem para a data em que o arquivo foi gerado. Todos os arquivos aparecem como atualizados no portal em 10/08/2026, e a última data de finalização encontrada é 07/08/2026.
- Um mesmo problema pode ter vários chamados, e um problema pode ser resolvido sem chamado. O número de chamados não é o número de problemas na cidade.
- 28 chamados não têm subprefeitura, e outros 100 têm um valor que não é uma subprefeitura ("não encontrada" ou Usinas de Asfalto). Esses 128 ficam fora do modelo. Cerca de 0,1% dos chamados também não têm distrito, mas o distrito não entra no modelo.
- 4 dos 17.239 chamados com status Finalizada não têm data de finalização. Como não dá para contar os dias até o fechamento, eles entram como "não resolvido" no modelo. O efeito nos resultados é desprezível.

## 2. INMET: dados meteorológicos horários

**Fonte:** Instituto Nacional de Meteorologia (INMET), dados históricos.
- Página: https://portal.inmet.gov.br/dadoshistoricos
- Arquivos: `2024.zip` e `2025.zip` (https://portal.inmet.gov.br/uploads/dadoshistoricos/2024.zip e .../2025.zip), salvos em `ml/dados/brutos/` com esses mesmos nomes.
- Estação usada: A701, São Paulo, Mirante de Santana.

**Período usado:** 01/01/2024 a 31/12/2025.

**Como o CSV foi montado:** as horas vêm em UTC e foram convertidas para o horário de Brasília (UTC menos 3 horas), para que a chuva caia no mesmo dia da abertura do chamado. Depois, os dados foram agregados por dia. O modelo usa a chuva total dos 7 dias antes da abertura de cada chamado.

**Colunas de `inmet_a701_diario.csv`:**
- `data`: dia, no horário de Brasília.
- `chuva_mm`: chuva total do dia, em milímetros.
- `horas_com_dado`: quantas das 24 horas do dia tinham medição.

**Limitações conhecidas:**
- O portal do INMET recusou o download feito pelo notebook. Os dois ZIPs foram baixados manualmente pelo navegador. O notebook tenta baixar e, se não conseguir, explica como fazer o download manual.
- Por causa da conversão de fuso, o dia 31/12/2025 fica sem as 3 últimas horas, que estariam no arquivo de 2026.
- Horas sem medição (valor -9999 no arquivo original) ficam vazias. Menos de 0,5% das horas estão vazias. Nesses dias, a chuva total pode estar um pouco abaixo do real. A coluna `horas_com_dado` mostra quais dias foram afetados.
- A chuva é medida em um ponto só, a estação do Mirante de Santana, na zona norte. Como o modelo usa chamados da cidade inteira, a chuva de cada região pode ser diferente da medida ali.
