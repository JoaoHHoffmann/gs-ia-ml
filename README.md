# Dá Pra Ir? | Inteligência Artificial & Machine Learning

App que ajuda pessoas com deficiência, pessoas autistas e famílias atípicas a saber, antes de sair de casa, se um lugar vai funcionar para elas. Este repositório tem a parte de IA e ML do projeto.

## Integrantes

Turma: 3ESPX

- Carlos Henrique - RM558003
- Mauricio Alves - RM556214
- Ian Monteiro - RM558652
- Bruno Silva - RM550416
- João Hoffmann - RM550763

## Ficha da CitizenTech

| Item | Descrição |
|---|---|
| Nome | Dá Pra Ir? |
| Cidade e bairro | São Paulo, Vila Clementino (subprefeitura Vila Mariana) |
| Cidadão beneficiado | Pessoas com deficiência, pessoas autistas e famílias atípicas, principalmente quem vai à AACD, ao Hospital do Servidor Público Estadual e ao Hospital São Paulo |
| Problema | Pessoas com deficiência, pessoas autistas e famílias atípicas só descobrem se um lugar vai funcionar para elas quando já chegaram lá: barulho demais, elevador parado, calçada quebrada ou sem rampa no caminho |
| Como funciona | Cada lugar tem uma ficha feita por quem já foi (como é, como foi e como está), e a pessoa diz o que importa para ela, recebe um plano para a visita e depois responde "deu certo?". A parte de ML mostra quais problemas de calçada e rampa a Prefeitura demora a resolver |
| O que a parte de IA e ML faz | Um modelo XGBoost prevê se um chamado de calçada, obstáculo ou rampa aberto no SP156 vai ser resolvido em até 30 dias, usando dados públicos de 2024 e 2025 |
| Repositório | https://github.com/JoaoHHoffmann/gs-ia-ml |
| Indicador de impacto | Porcentagem de chamados de acessibilidade resolvidos em até 30 dias na Vila Mariana: 35,6% em 2025. O número vem direto dos dados abertos do SP156, e o app ajuda a acompanhar e cobrar essa taxa ano a ano |
| ODS | 3 (Saúde e bem-estar), 10 (Redução das desigualdades) e 11 (Cidades e comunidades sustentáveis) |

## Estrutura

```
README.md
ml/
  notebook_ml.ipynb      notebook com download, preparação, modelo e conclusão
  RELATORIO_ML.md        relatório do problema à conclusão
  requirements.txt       bibliotecas e versões
  dados/
    README.md            fontes, colunas e limitações dos dados
    *.csv                dados já filtrados, gerados pelo notebook
    brutos/              arquivos originais baixados (não vão para o GitHub)
```

## Como rodar

1. Crie o ambiente virtual com Python 3.12 e instale as bibliotecas:
   ```
   py -3.12 -m venv .venv
   .venv\Scripts\python -m pip install -r ml/requirements.txt
   ```
2. Baixe `2024.zip` e `2025.zip` em https://portal.inmet.gov.br/dadoshistoricos e salve em `ml/dados/brutos/`.
3. Abra `ml/notebook_ml.ipynb` com o `.venv` como kernel e rode todas as células. O notebook baixa os dados do SP156 sozinho.

Os detalhes estão em [ml/dados/README.md](ml/dados/README.md).

## Links

- [Notebook](ml/notebook_ml.ipynb)
- [Relatório](ml/RELATORIO_ML.md)
- [README dos dados](ml/dados/README.md)

## Vídeo

[link do YouTube]
