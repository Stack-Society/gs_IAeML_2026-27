# Diagnóstico de aprendizagem e lacunas por questionário — componente de IA

Entrega da disciplina **Inteligência Artificial & Machine Learning** da Global Solution 2026.2 (CitizenTech).

Este repositório é a parte de IA do **AprendIA**: o modelo de Machine Learning, treinado em Python e exportado em ONNX, e o contrato de integração. Quem usa o modelo em produção é a **API REST em Java** do projeto, que executa o arquivo ONNX **dentro da própria API, offline e sem Python no servidor**.

## Integrantes
| Nome | RM |
|---|---|
| Leonardo Fernandes Mesquita | RM:559623 |
| Vitor de Lima Domingues | RM:561008 |
| Giovanni Romano Provazi | RM:560434 |
| Caio Berardo de Araújo | RM:560357 |
| Felipe Carlos Abreu | RM:559476 |

## Ficha da CitizenTech

| Campo | Preenchimento |
|---|---|
| Nome da CitizenTech | *AprendIA* |
| Cidade e bairro | São Paulo |
| Cidadão beneficiado | Alunos do 1º ao 5º ano e seus professores |
| Problema, em uma frase | Professores não têm um jeito rápido e padronizado de saber em quais conteúdos cada aluno está abaixo do esperado para a sua série, nem o que fazer a respeito. |
| Como funciona | O aluno responde a um questionário da sua série (24 questões de Língua Portuguesa e Matemática). O sistema corrige, um modelo de ML estima a chance de lacuna em cada um dos 8 conteúdos, e o sistema gera os insights: nível por disciplina, conteúdos a reforçar em ordem de urgência e uma estratégia de estudo para cada um. Casos inválidos ou duvidosos vão para o professor. |
| O que cada parte faz | **IA (este repositório):** modelo treinado e exportado em ONNX, com o contrato de integração e os golden tests. **API Java:** cadastro de escolas e alunos, correção, execução do modelo, insights, confirmação do professor e histórico, com persistência no Oracle. |
| Link do repositório | *https://github.com/Stack-Society/gs_IAeML_2026-27* (IA) · *https://github.com/Stack-Society/gs_SOAeWebServices_2026-27* (API Java) |
| Indicador de impacto | % de alunos com diagnóstico no início do ano; tempo até a intervenção; % de lacunas superadas no questionário seguinte |
| ODS conectados | ODS 4 (Educação de qualidade) e ODS 10 (Redução das desigualdades) |
| Referências | BNCC; SAEB; Compromisso Nacional Criança Alfabetizada (MEC) |

## A solução AprendIA

O AprendIA é um único projeto dividido em partes. Esta entrega cobre a IA; a API Java usa o que ela gera.

```
                 ┌──────────── este repositório (Python) ────────────┐
 base simulada ─►│ notebook: treino, avaliação, limiar e guardrails  │
                 │   └─► models/modelo.onnx + models/modelo_info.json │
                 │   └─► integracao/golden_tests.csv                  │
                 └───────────────────────┬────────────────────────────┘
                                         │ arquivos gerados
                                         ▼
 professor / app ──JSON──► API REST Java (Spring Boot) ──► ONNX Runtime (modelo.onnx)
                                         │
                                         └──► Oracle: escolas, alunos, questionário, diagnósticos e histórico
```

| Parte | Responsabilidade | Tecnologia |
|---|---|---|
| **IA** (este repositório) | gerar e avaliar o modelo, definir o limiar, a margem de dúvida, as regras de nível e o catálogo de estratégias; exportar o modelo e o contrato | Python, scikit-learn, ONNX |
| **API REST** | receber as respostas, validar, corrigir, executar o modelo, montar os insights, gravar o diagnóstico e registrar a confirmação do professor | Java 25, Spring Boot, ONNX Runtime, Oracle |

**O que a API Java usa desta entrega:**

| Arquivo | Uso na API |
|---|---|
| `models/modelo.onnx` | carregado uma vez, quando a API sobe (`ModeloIaService`), e executado a cada diagnóstico |
| `models/modelo_info.json` | origem dos gabaritos, conteúdos, ordem das features, limiar (0,40), margem (0,05), regra de nível e estratégias, que a API grava no banco nas migrações |
| `integracao/golden_tests.csv` | conferência da API: as probabilidades em Java diferem no máximo 0,0000005 das do notebook, e situações e níveis batem em 100% dos casos válidos |

O diagnóstico de um aluno passa a ser um recurso da API:

```http
POST /alunos/{id}/diagnosticos
{ "respostas": { "q1": "A", ..., "q24": "B" } }
```

A resposta traz o nível de cada disciplina, a situação de cada conteúdo (com a probabilidade do modelo) e os conteúdos a reforçar com a estratégia de cada um. Os casos **a confirmar** são decididos pelo professor em `PATCH /alunos/{id}/diagnosticos/{diagnosticoId}/conteudos/{conteudo}`; a decisão do modelo continua gravada, para auditoria.

## Como funciona, passo a passo

```
respostas ──► validar ──► corrigir ──► 13 features ──► modelo ──► 8 probabilidades de lacuna
                │                                                         │
           inválido: para                                    limiar + margem de dúvida
           (sem diagnóstico)                                              │
                                                                          ▼
                     insights ◄── nível por disciplina ◄── lacuna / sem_lacuna / a_confirmar
```

1. **Questionário.** Cada série (1º ao 5º ano) tem o seu questionário, com a mesma estrutura: 24 questões de múltipla escolha (A, B, C), 3 para cada um dos 8 conteúdos.

   | Disciplina | Conteúdos |
   |---|---|
   | Língua Portuguesa | decodificação, localizar informação, inferência, ortografia |
   | Matemática | números, operações, geometria, grandezas e medidas |

2. **Validação (guardrail de entrada).** Série inexistente, questão em branco ou resposta fora de A/B/C: o questionário é recusado e não recebe diagnóstico.
3. **Correção.** Conta os acertos (0 a 3) em cada conteúdo, com o gabarito da série. É uma regra exata, sem IA.
4. **Modelo.** Uma regressão logística por conteúdo recebe as 13 features (8 contagens de acertos e a série) e devolve a **probabilidade de lacuna** em cada conteúdo. Cada decisão olha também os outros conteúdos: quem vai mal em inferência tem mais chance de ter lacuna em localizar informação.
5. **Decisão com guardrail de saída.** Cada conteúdo vira **lacuna** (probabilidade ≥ 0,45), **sem lacuna** (< 0,35) ou **a confirmar pelo professor** (entre 0,35 e 0,45, onde o modelo está em dúvida).
6. **Insights.**
   - **Nível por disciplina:** 0 lacunas = adequado; 1–2 = em desenvolvimento; 3–4 = abaixo do esperado.
   - **Conteúdos a reforçar:** em ordem de urgência.
   - **Estratégias de estudo:** uma por conteúdo, tiradas de um catálogo fixo e revisável.

## Resultados

Conjunto de teste: 1.640 alunos de 13 escolas que o modelo nunca viu. Métricas por conteúdo, com média entre os 8 conteúdos.

| | Recall de lacuna | Precisão de lacuna | F1-macro |
|---|---|---|---|
| Dummy (sempre "sem lacuna") | 0% | 0% | 0,000 |
| Regra do corte (lacuna se errou alguma questão do conteúdo) | 81,6% | 57,9% | 0,677 |
| **Regressão logística (modelo entregue)** | **81,8%** | **61,3%** | **0,700** |

**Como ler:**
- **Recall:** das lacunas que existiam, quantas foram encontradas. É a métrica prioritária, porque deixar uma lacuna sem aviso é o erro mais grave.
- **Precisão:** dos conteúdos apontados como lacuna, quantos eram de fato lacuna. Quanto maior, menos reforço desperdiçado.
- **F1:** equilíbrio entre os dois.

O modelo encontra a mesma proporção de lacunas que a melhor regra simples, com menos alarmes falsos. Com o guardrail, 91,9% dos conteúdos recebem decisão automática, e as lacunas reais que ficam sem nenhum aviso caem de 18,3% (regra) para 15,3%.

O ganho é **real, mas modesto**: com 3 questões por conteúdo, há pouca informação para qualquer método.

> **Atenção:** a base é **simulada** (ver abaixo). Com alunos reais, os números devem ser menores.

## Dados

`data/questionarios.csv` é uma base **simulada** de 8.000 alunos de 60 escolas, do 1º ao 5º ano, gerada pelo próprio notebook com semente fixa. Não existe base pública com respostas individuais de alunos a este questionário, e a GS proíbe dados pessoais reais.

Como a simulação funciona (detalhes no passo 5 do notebook):
- **Domínio do aluno.** Cada aluno tem, em cada conteúdo, um domínio em relação ao esperado para a sua série. Ele soma:
  - a média da série;
  - um efeito geral do aluno;
  - um efeito da disciplina;
  - um efeito da escola;
  - uma variação própria do conteúdo.
- **Lacuna verdadeira:** domínio abaixo de zero.
- **Respostas:** quem domina acerta; quem não domina chuta (acerta 1 em 3).
- **Problemas reais, incluídos de propósito:**
  - 2% de questionários com questões em branco;
  - 3% dos rótulos trocados, imitando erros de avaliação do professor.

| Coluna do CSV | Significado |
|---|---|
| `aluno_id`, `escola_id` | identificadores anônimos |
| `serie` | série do aluno (1 a 5) |
| `q1` … `q24` | alternativa marcada (`A`, `B`, `C`) ou vazio |
| `lacuna_<conteúdo>` | rótulo do professor: 1 = lacuna, 0 = sem lacuna |

Os parâmetros, os gabaritos e as estratégias de estudo são **ilustrativos**, não números ou orientações oficiais. Para usar dados reais, basta substituir o CSV.

## Estrutura

```
├── diagnostico_lacunas_aprendizagem.ipynb  notebook com todo o trabalho, comentado linha a linha (entrega principal)
├── data/questionarios.csv                  base simulada (gerada pelo notebook)
├── models/modelo.onnx                      modelo treinado, executado pela API Java (ONNX Runtime)
├── models/modelo_info.json                 contrato: gabaritos, features, limiar, margem, regras, estratégias, métricas
├── integracao/golden_tests.csv             casos de teste com as saídas esperadas, usados para conferir a API Java
└── requirements.txt                        dependências com versões exatas
```

Etapas do notebook:

| Passos | Conteúdo |
|---|---|
| 1 a 4 | bibliotecas, constantes, questionário e correção |
| 5 a 7 | base simulada, features e análise exploratória |
| 8 a 11 | separação treino/teste por escola, baselines, comparação de modelos e ajuste |
| 12 a 14 | limiar e margem de dúvida, avaliação no teste e explicação do modelo |
| 15 a 18 | diagnóstico com insights, exportação ONNX, contrato e golden tests, limitações |

## Como executar

Requer **Python 3.11 ou superior**.

```bash
python -m venv .venv
.venv\Scripts\activate            # Windows (Linux/macOS: source .venv/bin/activate)
pip install -r requirements.txt
jupyter notebook diagnostico_lacunas_aprendizagem.ipynb   # depois: Kernel → Restart & Run All
```

As células precisam rodar em ordem. A semente fixa garante os mesmos resultados e os mesmos arquivos em qualquer execução.

## Integração com a API Java

A API REST em Java **não roda o notebook nem precisa de Python**: ela usa só os arquivos gerados por ele.

- **Dependência na API:** ONNX Runtime para Java (`com.microsoft.onnxruntime:onnxruntime`). O scikit-learn e as demais bibliotecas de treino ficam só neste repositório.
- **Onde cada parte fica na API:**

  | Etapa | Classe da API Java |
  |---|---|
  | carregar e executar o `modelo.onnx` | `ia/ModeloIaService` |
  | validar e corrigir as respostas | `diagnostico/CorrecaoService` |
  | situação de cada conteúdo e nível da disciplina | `diagnostico/Situacao`, `diagnostico/NivelDisciplina` |
  | catálogo do questionário, gabaritos e estratégias | tabelas do Oracle, populadas a partir do contrato nas migrações |

- **Versão do modelo:** cada diagnóstico gravado pela API guarda a `versao_modelo` do contrato (atualmente `2.0`), para saber qual modelo gerou cada resultado.

- **Fluxo, nesta ordem** (todos os valores estão em `models/modelo_info.json`; a referência de cada etapa é a função de mesmo nome no notebook, e a API reproduz as mesmas regras):
  1. **Validar.** A série precisa ser de 1 a 5, e as 24 questões precisam ter resposta `A`, `B` ou `C`; caso contrário, o status é `invalido`, o modelo não é chamado e a API responde **400 Questionário inválido**, sem gravar nada.
  2. **Calcular as features** (`float32`, 13 valores, na ordem de `ordem_features`): os acertos (0 a 3) em cada conteúdo, usando o **gabarito da série**, e depois `serie_1` … `serie_5`, com 1 na série do aluno e 0 nas demais.
  3. **Rodar o modelo:** entrada `acertos` `[1, 13]`; usar a saída `probabilities` `[1, 8]`, na ordem de `ordem_saidas`. **Ignorar a saída `label`**, que usa o limiar padrão 0,5.
  4. **Decidir cada conteúdo:** `a_confirmar` se a distância entre a probabilidade e `limiar_lacuna` (0,40) for menor que `margem_duvida` (0,05); senão, `lacuna` se a probabilidade for maior ou igual a 0,40; senão, `sem_lacuna`.
  5. **Montar os insights:**
     - **nível por disciplina:** 0 lacunas → adequado; 1–2 → em desenvolvimento; 3–4 → abaixo do esperado. Se os conteúdos `a_confirmar` puderem mudar o nível, o nível fica "a confirmar";
     - **conteúdos a reforçar:** lacunas primeiro, depois os `a_confirmar`; em cada grupo, a maior probabilidade primeiro;
     - **textos:** `estrategias`, `orientacao_por_nivel` e `nome_exibicao`.
  6. **Confirmação do professor (na API).** Os conteúdos `a_confirmar` são decididos pelo professor; a decisão do modelo continua gravada e o nível da disciplina é recalculado com a decisão final.
- **Validação:** os casos de `integracao/golden_tests.csv` conferem a API contra o notebook:
  - o status;
  - a situação de cada conteúdo;
  - o nível de cada disciplina;
  - as probabilidades, que devem diferir menos de 0,0001.

  **Resultado na API Java:** nos 40 casos válidos, as probabilidades diferem no máximo 0,0000005 das do notebook, e a situação dos 8 conteúdos e o nível das 2 disciplinas batem em 100% dos casos.

## Decisões principais

| Decisão | Por quê | Alternativa descartada |
|---|---|---|
| **Multirrótulo por conteúdo** | um aluno pode ter nenhuma, uma ou várias lacunas, e a tarefa pede os conteúdos a reforçar | um nível único por disciplina: diria *quanto* o aluno sabe, mas não *o que* falta |
| **Um classificador por conteúdo, com todas as features** (`OneVsRestClassifier`) | usa o desempenho nos outros conteúdos da mesma disciplina para decidir melhor do que olhando só as 3 questões | `MultiOutputClassifier`: mesmo resultado, mas o ONNX sai como lista de 8 matrizes, mais difícil de usar na API Java |
| **Separação treino/teste por escola** | o modelo será usado em escolas que nunca viu; misturar alunos da mesma escola geraria vazamento de dados | separação aleatória por aluno: daria um resultado otimista |
| **Regressão logística** | empatou com o Random Forest na validação cruzada e é a mais simples e explicável | Random Forest e Gradient Boosting |
| **Limiar escolhido pelo recall** (0,40) | o modelo precisa encontrar pelo menos tantas lacunas quanto a melhor regra simples | limiar padrão 0,5 (perderia lacunas) ou F2 (peso arbitrário) |
| **Falhar fechado** | questionários inválidos não recebem diagnóstico, e conteúdos duvidosos ficam "a confirmar pelo professor" | decidir sempre: erros silenciosos perto do limiar |
| **Estratégias de um catálogo fixo** | podem ser revisadas por pedagogos, são testáveis e funcionam offline, sem enviar dados de crianças a terceiros | texto gerado por LLM: pode inventar orientações e exige internet |
| **ONNX** | o modelo treinado em Python roda dentro da API Java com o ONNX Runtime, sem Python no servidor e sem amarração com a versão do scikit-learn; formato aberto e mais seguro | pickle/joblib: pode executar código ao carregar e só abre em Python, com a mesma versão do scikit-learn; `skops`: seguro, mas exige Python e o scikit-learn no servidor da API |

## Conceitos principais

| Termo | Significado neste projeto |
|---|---|
| Feature | informação de entrada do modelo: acertos em cada conteúdo e a série |
| Rótulo | o que se quer prever: lacuna (1) ou não (0) em cada conteúdo |
| Multirrótulo | várias respostas sim/não para o mesmo aluno (8, uma por conteúdo) |
| One-hot | a série vira 5 colunas de 0/1, com 1 apenas na série do aluno |
| Validação cruzada | treinar e avaliar 5 vezes, em partes diferentes do treino, para comparar modelos sem usar o teste |
| Vazamento de dados | informação do teste "vazando" para o treino e inflando o resultado |
| Baseline | solução simples usada como régua mínima |
| Limiar | probabilidade a partir da qual a resposta vira "lacuna" |
| Guardrail | regra fixa que impede o sistema de agir sobre entradas inválidas ou previsões duvidosas |
| Golden tests | casos com respostas esperadas, para conferir que a API Java reproduz o notebook |
| Contrato de integração | arquivo `modelo_info.json`, que define entrada, saída e regras para que o notebook (Python) e a API (Java) sejam desenvolvidos separadamente e deem o mesmo resultado |

## Limitações

- A base é simulada.
- Os conteúdos, os gabaritos, a regra de nível e as estratégias são ilustrativos.
- O questionário não foi validado por pedagogos nem testado com alunos.
- Com só 3 questões por conteúdo, o desempenho tem um limite.
- O modelo apoia a triagem e não substitui a avaliação do professor.

Detalhes e próximos passos estão no passo 18 do notebook.
