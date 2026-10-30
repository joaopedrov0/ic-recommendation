# Relatório Técnico e Científico: Recomendação por Popularidade e IBCF no IFVest

Este documento consolida:
1. **Diagnóstico Passivo do Repositório (`/root_lab/ifvest`)** e mapeamento de dados (atuais vs. novos);
2. **Proposta Matemática e Arquitetural** para os algoritmos **Baseado em Popularidade (*Popularity-Based*)** e **Filtragem Colaborativa Baseada em Itens (*IBCF*)**;
3. **Modelo de Arquitetura (`system-designer`)** salvo em [arquitetura_recomendacao_ifvest.json](file:///C:/root_lab/me/ic/arquitetura_recomendacao_ifvest.json);
4. **Pesquisa de Critérios Pedagógicos, Técnicos e Éticos/Legais (LGPD)** para Sistemas de Recomendação Educacionais (*Educational Recommender Systems — EdRS*).

---

## Parte I — Diagnóstico do Repositório `/root_lab/ifvest`

A inspeção passiva do monorepo revelou que a plataforma possui excelente cobertura de testes e separação modular, mas **ainda não dispõe de mecanismos de recomendação personalizada ou ranqueamento probabilístico**:

- **Seleção no IFQuiz** ([`select_next_quiz_question`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/quiz_question_queries.py#L77-L99)): aplica filtro estrito por disciplina e tópicos ([`filtered_question_ids`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/quiz_filters.py#L31-L58)) e realiza sorteio pseudoaleatório uniforme via [`session_question_chooser`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/quiz_question_queries.py#L44-L51).
- **Pergunta do Dia** ([`get_daily_question`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/quiz_service.py#L81-L154)): utiliza offset aritmético determinístico `(today.toordinal() * 37) % total_q`.
- **Simulados Aleatórios** ([`create_random_mocktest`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/mocktests_service.py#L87-L126)): seleciona questões via `ORDER BY RANDOM()` no PostgreSQL.
- **Estatísticas de Desempenho** ([`calculate_student_statistics`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/mocktest_attempts.py#L207-L224)): agrega taxa de acerto bruta por disciplina em simulados para exibir apenas a melhor disciplina e a disciplina a revisar ([`rank_disciplines`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/mocktest_attempts.py#L185-L205)).
- **Ausência de Dificuldade Real** ([0001-ocultar-seletor-de-dificuldade-do-quiz.md](file:///C:/root_lab/ifvest/docs/decisoes/0001-ocultar-seletor-de-dificuldade-do-quiz.md)): as 1.164 questões objetivas da Unicamp não trazem metadado de dificuldade na fonte e recebem `difficulty = 2` por padrão em [`unicamp_seed_service.py`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/unicamp_seed_service.py), o que motivou a ocultação temporária do seletor de dificuldade na tela de criação de quiz.

---

## Parte II — Mapeamento de Dados (Existentes vs. Novos Dados a Coletar)

### 2.1. Dados Já Disponíveis no Schema Atual ([database_model.md](file:///C:/root_lab/ifvest/ifvest-backend/docs/database_model.md))

| Tabela | Modelo SQLAlchemy | Campos Disponíveis para Recomendação |
| :--- | :--- | :--- |
| `question_history` | [`QuestionHistory`](file:///C:/root_lab/ifvest/ifvest-backend/src/models/quiz.py#L40-L65) | `user_id`, `question_id`, `quiz_id`, `chosen_alternative`, `is_correct`, `answered_at`. |
| `user_mocktest_answers` | [`UserMocktestAnswer`](file:///C:/root_lab/ifvest/ifvest-backend/src/models/mocktests.py#L83-L104) | `user_mocktest_id`, `question_id`, `selected_letter`, `is_correct`. |
| `user_mocktests` | [`UserMocktest`](file:///C:/root_lab/ifvest/ifvest-backend/src/models/mocktests.py#L56-L81) | `user_id`, `fixed_mocktest_id`, `score`, `started_at`, `completed_at`. |
| `multiple_choice_questions` | [`MultipleChoiceQuestion`](file:///C:/root_lab/ifvest/ifvest-backend/src/models/questions.py#L57-L81) | `question_id`, `exam_board`, `year`, `difficulty` e vínculos em [`QuestionTopic`](file:///C:/root_lab/ifvest/ifvest-backend/src/models/questions.py#L101-L115) (`Discipline -> Subject -> Topic`). |
| `question_reports` | [`QuestionReport`](file:///C:/root_lab/ifvest/ifvest-backend/src/models/admin.py#L13-L35) | `question_id`, `user_id`, `reason`, `status` (permite filtrar/penalizar questões com erro de cadastro). |

### 2.2. Novos Dados Recomendados para Coleta

1. **Tempo de Resposta por Questão (`response_time_ms: int | None`)**:
   - **Onde**: colunas em `question_history` e `user_mocktest_answers`.
   - **Finalidade**: distinguir acertos rápidos (domínio consolidado), acertos lentos (esforço cognitivo elevado) e erros instantâneos (*rapid guessing* / "chute" em menos de 5 segundos).
2. **Uso de Dica de Eliminação (`used_hint: bool`)**:
   - **Onde**: coluna booleana em `question_history`.
   - **Finalidade**: indicar se o estudante recorreu ao campo `dica_eliminar` fornecido por [`get_current_quiz_question`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/quiz_service.py#L499-L535) antes de acertar.
3. **Timestamp Individual em Simulados (`answered_at: datetime`)**:
   - **Onde**: coluna em `user_mocktest_answers` e persistência opcional na rota interativa `POST /mocktests/{mocktest_id}/answer` ([`evaluate_single_question_answer`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/mocktests_service.py#L129-L148)).
4. **Registro de Impressão/Exposição (`question_impressions` ou flag `was_skipped`)**:
   - **Finalidade**: mitigar o **viés de exposição** (saber quantas vezes uma questão foi sorteada/apresentada vs. quantas vezes foi efetivamente respondida ou abandonada).
5. **Favoritos / Marcadores de Revisão (`question_bookmarks`)**:
   - **Finalidade**: capturar feedback explícito positivo quando o aluno salva uma questão para revisar depois.

---

## Parte III — Especificação dos Algoritmos: Popularidade e IBCF

### 3.1. Algoritmo Baseado em Popularidade (*Popularity-Based*)

#### A. Estimador Bayesiano Empírico (Suavização de Laplace / IMDB Formula)
Para evitar que questões com poucas respostas ($N_i$ pequeno) dominem os extremos de dificuldade ou popularidade:

$$S_{\text{bayes}}(i) = \frac{N_i}{N_i + m} \cdot \hat{p}_i + \frac{m}{N_i + m} \cdot \mu$$

- $N_i$: total de resoluções da questão $i$ (`question_history` $\cup$ `user_mocktest_answers`).
- $\hat{p}_i = \frac{E_i}{N_i}$: taxa empírica de erro da questão $i$.
- $\mu$: taxa média global de erro em todo o banco de questões.
- $m$: suporte mínimo de confiança (ex.: $m = 10$).
- **Aplicação direta na ADR 0001**: mapeando $S_{\text{bayes}}(i)$ em 5 faixas (quantis ou limiares fixos), calibra-se automaticamente a coluna `difficulty` (`1` a `5`) das questões da Unicamp.

#### B. Popularidade com Decaimento Temporal Exponencial (*Time Decay*)
Para identificar questões, tópicos e simulados em alta nos últimos dias/semanas:

$$\text{Pop}_{\text{decay}}(i) = \phi(i) \cdot \sum_{k=1}^{N_i} w_k \cdot \exp\left(-\lambda \cdot \Delta t_k\right)$$

- $\Delta t_k$: tempo decorrido (em dias) desde a $k$-ésima interação; $\lambda = \frac{\ln(2)}{T_{1/2}}$ (ex.: meia-vida $T_{1/2} = 14\text{ dias}$).
- $\phi(i) \in \{0, 1\}$: anula ou reduz o score de questões com denúncias pendentes em [`QuestionReport`](file:///C:/root_lab/ifvest/ifvest-backend/src/models/admin.py#L13-L35).

---

### 3.2. Filtragem Colaborativa Baseada em Itens (IBCF)

#### A. Codificação da Matriz Usuário $\times$ Questão ($R \in \mathbb{R}^{|U| \times |I|}$)
Definimos $r_{u,i} \in [-1, +1]$ como o **índice de necessidade pedagógica** do aluno $u$ na questão $i$ (com decaimento temporal para priorizar o estado atual de conhecimento do estudante):
- $+1.0$: erro recente sem acerto posterior (lacuna ativa de conhecimento);
- $+0.5$: acerto com auxílio de dica (`used_hint = True`) ou tempo excessivo;
- $+0.3$: acerto recente após erro anterior (conceito em consolidação);
- $-0.5$: acerto direto e rápido na primeira tentativa (conceito dominado).

#### B. Similaridade de Cosseno Ajustada (*Adjusted Cosine Similarity*) com *Significance Weighting*
Desconta o viés de proficiência média $\bar{r}_u$ de cada estudante e amortece interseções pequenas ($|U_{i \cap j}| < \gamma$):

$$s(i, j) = \frac{\sum_{u \in U_{i \cap j}} (r_{u,i} - \bar{r}_u)(r_{u,j} - \bar{r}_u)}{\sqrt{\sum_{u \in U_{i \cap j}} (r_{u,i} - \bar{r}_u)^2} \cdot \sqrt{\sum_{u \in U_{i \cap j}} (r_{u,j} - \bar{r}_u)^2}} \cdot \min\left(1, \frac{|U_{i \cap j}|}{\gamma}\right)$$

#### C. Predição do Score de Vulnerabilidade ($\hat{r}_{u, i}$)
$$\hat{r}_{u, i} = \frac{\sum_{j \in \mathcal{N}_K(i) \cap I_u} s(i, j) \cdot r_{u, j}}{\sum_{j \in \mathcal{N}_K(i) \cap I_u} |s(i, j)|}$$

---

## Parte IV — Pesquisa: As Três Frentes de Critérios (Pedagógicos, Técnicos e Éticos)

Diferentemente do comércio eletrônico ou *streaming* (onde o objetivo é maximizar cliques ou tempo de tela), um **Sistema de Recomendação Educacional (*EdRS*)** interfere diretamente na trajetória cognitiva, emocional e acadêmica do estudante. Abaixo detalham-se as três frentes fundamentais investigadas na literatura científica e regulatória.

---

### 1. Frente Pedagógica (Ciência da Aprendizagem e Design Instrucional)

#### 1.0. "Deve-se estudar mais frequentemente aquilo que mais se erra?" — A Nuance Científica e o Roteamento Multi-Módulo
Como no IFVest **Questões (IFQuiz e Simulados), Flashcards e Materiais de Revisão** compartilham o mesmo identificador hierárquico (`discipline_id -> subject_id -> topic_id`), as estatísticas consolidadas podem recomendar tanto itens individuais quanto **disciplinas, assuntos ou tópicos**.

Contudo, a ciência cognitiva demonstra que a premissa *"recomendar mais frequentemente o que o aluno mais erra"* **não é uma verdade absoluta**:
1. **Quando a taxa de erro no tópico é muito alta (Acerto $< 50\%$)**: insistir em mais questões difíceis daquele tópico gera sobrecarga cognitiva, frustração e perda imediata das 3 vidas do IFQuiz. Nessa faixa, o sistema deve **mudar a modalidade recomendada dentro do mesmo `topic_id`**: recomendar primeiro o **Material de Revisão Teórica** e os **Flashcards de base** do tópico antes de apresentar novas baterias de questões.
2. **Quando a taxa de acerto está na Zona de Desenvolvimento Proximal ($55\%$ a $85\%$)**: é onde as **Questões Adaptativas (via IBCF)** produzem o maior ganho marginal de aprendizagem (Wilson et al., 2019).
3. **Quando o tópico já é dominado (Acerto $> 85\%$)**: **não se deve parar de recomendá-lo**, pois a Curva do Esquecimento de Ebbinghaus degradaria a retenção até a data do vestibular. O tópico deve entrar em regime de **Repetição Espaçada** ($7 \to 21 \to 60\text{ dias}$), priorizando flashcards rápidos de manutenção e questões espaçadas.

#### 1.1. Zona de Desenvolvimento Proximal (ZDP), *Desirable Difficulties* e a "Regra dos 85%"
- **Fundamentação Científica**: Vygotsky (1978) estabelece que a aprendizagem ótima ocorre na **Zona de Desenvolvimento Proximal (ZDP)** — tarefas que o estudante não resolve facilmente sozinho, mas consegue dominar com suporte (*scaffolding*). Robert Bjork (1994; 2011) cunhou o conceito de **Dificuldades Desejáveis (*Desirable Difficulties*)**, e **Wilson et al. (2019, *Nature Communications*)** demonstraram matematicamente a **"Regra dos 85%" (*The Eighty-Five Percent Rule for Optimal Learning*)**: a taxa de aprendizado humano e em algoritmos baseados em gradiente é maximizada quando a taxa de acerto fica em torno de **84,1% (taxa de erro ótima de ~15,87%)**.
- **Impacto Direto no IFVest (`QUIZ_MAX_LIVES = 3`)**:
  - Em [`quiz_service.py`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/quiz_service.py#L60), o estudante possui exatamente **3 vidas por quiz**.
  - Se o algoritmo IBCF ranquear e entregar **apenas** as questões com maior probabilidade de erro ($\hat{r}_{u,i}$ máximo), o aluno errará as 3 primeiras questões seguidas, perderá todas as vidas prematuramente e sofrerá **sobrecarga cognitiva e frustração**.
  - **Critério de Implementação (Mistura Pedagógica / *Scaffolding*)**: O `AdaptiveQuestionChooser` não deve selecionar o `argmax` bruto de vulnerabilidade em todas as posições do quiz. Para um quiz de $N$ questões, deve-se compor uma curva calibrada:
    - **60% a 70% de Questões de Consolidação/ZDP Moderada** ($\hat{r}_{u,i} \in [-0.1, +0.3]$, probabilidade de acerto estimada de ~80–85%);
    - **20% a 30% de Questões de Desafio/Reforço Direto** ($\hat{r}_{u,i} > +0.4$, focadas nas lacunas diagnosticadas pelo IBCF);
    - **Sequenciamento Progressivo (*Warm-up*)**: iniciar a posição 1 do quiz com uma questão de dificuldade moderada/acessível para gerar engajamento e preservar vidas iniciais.

#### 1.2. Prática de Recuperação Espaçada (*Spaced Retrieval Practice*) e Curva do Esquecimento
- **Fundamentação Científica**: Roediger & Karpicke (2006) e Ebbinghaus demonstram que a retenção de longo prazo para exames (como ENEM e Unicamp) exige **reexposição espaçada** a conceitos previamente estudados, e não apenas questões inéditas.
- **Critério de Implementação**:
  - Atualmente, [`select_next_quiz_question`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/quiz_question_queries.py#L93-L99) exclui apenas `answered_ids` da **sessão atual** do quiz.
  - Na recomendação personalizada, questões já respondidas com **acerto** devem sofrer supressão temporária e retornar como candidatas de revisão após um intervalo ótimo (ex.: $7, 14, 30\text{ dias}$), enquanto questões respondidas com **erro** devem ter suas vizinhas similares no IBCF ($j \in \mathcal{N}_K(i)$) priorizadas nos próximos $1\text{ a }3\text{ dias}$.

#### 1.3. Respeito a Pré-Requisitos e Cobertura Curricular (*Interleaving*)
- **Fundamentação Científica**: Na literatura de EdRS (Manouselis et al., 2012; Drachsler et al., 2015), a filtragem colaborativa pura sofre do **Problema de Pré-Requisito**: dois itens podem co-ocorrer, mas um exige base conceitual prévia. Além disso, a prática intercalada (*Interleaving* — Rohrer & Taylor, 2007) de diferentes subtópicos supera a prática em bloco (*Blocked Practice*) de questões quase idênticas.
- **Critério de Implementação**:
  - Ao gerar um Simulado Recomendado (`POST /api/v1/mocktests/recommended`), aplicar uma restrição de **Diversidade Intra-Lista**: limitar a no máximo $k$ questões do mesmo `topic_id` ([`QuestionTopic`](file:///C:/root_lab/ifvest/ifvest-backend/src/models/questions.py#L101-L115)) e ordenar por nível de dificuldade crescente ($S_{\text{bayes}}$ crescente).

#### 1.4. Modelos Abertos de Aprendiz (*Open Learner Models* — OLM) e Metacognição
- **Fundamentação Científica**: Bull & Kay (2010) e Zimmerman (2002) mostram que estudantes aprendem melhor quando compreendem seu próprio estado de domínio (autorregulação).
- **Critério de Implementação**: Toda recomendação IBCF/Popularidade deve retornar um atributo `explanation_reason` legível (ex.: *"Recomendada para reforçar Cinemática — estudantes que revisaram a questão Unicamp 2023 #4 também praticaram esta"*).

---

### 2. Frente Técnica (Engenharia de Recomendação, Viés Estatístico e Performance)

#### 2.1. Mitigação de Viés de Exposição e *Feedback Loops* (*Matthew Effect*)
- **Problema Técnico**: Como o próprio sistema escolhe quais questões entram nos quizzes e simulados aleatórios, questões recomendadas pelo algoritmo de Popularidade receberão ainda mais respostas, criando um ciclo vicioso (*Rich-Get-Richer* / *Exposure Bias*) que esconde questões boas recém-cadastradas.
- **Critério de Implementação**:
  1. **Normalização por Exposição (*Propensity-Aware Popularity*)**: dividir ou ponderar o engajamento pelo número de vezes que a questão foi sorteada/exibida, e não apenas pelo volume absoluto.
  2. **Exploração Estocástica (*Softmax / $\epsilon$-Greedy Sampling*)**: em vez de seleção determinística top-1 no `AdaptiveQuestionChooser`, usar amostragem proporcional a $\text{Softmax}(\text{score} / \tau)$ (com temperatura $\tau \approx 0.5$) ou reservar $\epsilon = 15\%$ das vagas de um quiz para questões pouco exploradas (*Cold Items*) do mesmo tópico.

#### 2.2. Dinâmica Temporal do Conhecimento (*Concept Drift* do Usuário)
- **Problema Técnico**: Em filmes ou músicas, uma avaliação dada há 6 meses continua representando o gosto do usuário. Na educação, **o estudante aprende**: um erro cometido há 3 meses em "Funções Quadráticas" não representa mais a proficiência atual do aluno se ele acertou 15 questões do tema na última semana.
- **Critério de Implementação**:
  - Ao calcular o vetor do estudante $r_{u,j}$ para a predição IBCF $\hat{r}_{u,i}$, aplicar uma **janela temporal ou peso exponencial** $e^{-\beta (t_{\text{agora}} - t_{u,j})}$ nas respostas do aluno, dando peso majoritário às últimas 20–30 resoluções naquela disciplina.

#### 2.3. Tratamento de Esparsidade, *Cold Start* e Escalabilidade no `ifvest-backend`
- **Problema Técnico**: Novas questões importadas via `.zip` ([`question_import_service.py`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/question_import_service.py)) entram com 0 resoluções; novos alunos entram sem histórico.
- **Critério de Implementação**:
  - **Hibridização Taxonômica no Item-Item**: quando $|U_{i \cap j}| < \gamma$, combinar a similaridade colaborativa com um prior baseado na árvore [`Discipline -> Subject -> Topic`](file:///C:/root_lab/ifvest/ifvest-backend/src/models/questions.py):
    $$s_{\text{híbrido}}(i, j) = \alpha_{ij} \cdot s_{\text{IBCF}}(i, j) + (1 - \alpha_{ij}) \cdot s_{\text{taxonomia}}(i, j)$$
    onde $\alpha_{ij} = \min\left(1, \frac{|U_{i \cap j}|}{\gamma}\right)$.
  - **Eficiência Computacional**: Com ~1.164 a 10.000 questões, a atualização da matriz $S(i, j)$ em Python puro sobre pares co-ocorrentes leva menos de $500\text{ ms}$ e não exige dependências pesadas (`torch`, `scipy`), mantendo a imagem Docker leve e rápida no CI.

#### 2.4. Métricas de Avaliação Offline e Online
Para validar a qualidade do recomendador sem cair em métricas enganosas:
- **Métricas Offline de Ranking**: *NDCG@K* e *Recall@K* em *Time-Based Split* (treinar com interações até o instante $t$ e prever as dificuldades/interações em $t+1$, nunca usando *random split* que causa *data leakage* do futuro para o passado).
- **Métricas de Qualidade Estrutural**: *Catalog Coverage* (% do banco de questões que é recomendado ao menos uma vez), *Intra-List Diversity (ILD)* (diversidade de tópicos dentro de um simulado recomendado) e *Calibration Error* (diferença entre a dificuldade prevista $S_{\text{bayes}}$ e a taxa real de acerto).
- **Métricas Pedagógicas Online**: Ganho de Aprendizagem Normalizado (*Normalized Learning Gain* — evolução da taxa de acerto do aluno no tópico após resolver questões recomendadas) e Taxa de Conclusão de Quizzes (redução de abandonos por perda precoce das 3 vidas).

---

### 3. Frente Ética, Legal e de Proteção de Dados (LGPD e IA Responsável)

#### 3.1. Conformidade com a LGPD (Lei nº 13.709/2018) e Proteção de Menores (Art. 14)
Plataformas preparatórias para vestibulares e ensino médio (como o IFVest) atendem amplamente **estudantes adolescentes (menores de 18 anos)**:
1. **Melhor Interesse do Adolescente (Art. 14 da LGPD)**: O tratamento de dados pessoais de crianças e adolescentes deve ser realizado em seu melhor interesse. A telemetria coletada (`is_correct`, `response_time_ms`, `used_hint`) deve ser usada **exclusivamente para apoio pedagógico ao próprio estudante** e melhoria agregada do banco de questões, nunca para ranqueamento punitivo ou exposição pública de vulnerabilidades.
2. **Princípio da Necessidade / Minimização de Dados (Art. 6º, III)**:
   - Coletar apenas variáveis estritamente ligadas à resolução da questão.
   - Não coletar telemetria comportamental invasiva (ex.: rastreamento contínuo de mouse, foco de janela/aba ou geolocalização).
   - Aplicar truncamento/teto no `response_time_ms` (ex.: `min(response_time_ms, 600_000)`) para não registrar períodos em que o estudante apenas deixou a aba aberta inativa.
3. **Direito à Transparência e Revisão de Decisões Automatizadas (Art. 6º, VI e Art. 20 da LGPD)**:
   - O estudante tem direito a informações claras e acessíveis sobre os critérios utilizados na recomendação automatizada de estudos.
   - O modo recomendado/adaptativo nunca deve eliminar a **autonomia do estudante**: o aluno deve sempre poder alternar livremente entre recomendação personalizada, filtros manuais e sorteio aleatório clássico.

#### 3.2. Privacidade na Filtragem Colaborativa ($k$-Anonimato na Matriz IBCF)
- **Por que o IBCF é mais seguro que o UBCF (*User-Based*)**: No UBCF, o sistema mantém vetores de similaridade direta entre usuários ($s(u, v)$), o que pode expor quais colegas têm perfil idêntico de erros. No **IBCF**, a matriz persistida [`question_similarities`](file:///C:/root_lab/me/ic/arquitetura_recomendacao_ifvest.json) armazena apenas relações **Questão $\times$ Questão** ($s(i, j)$), desacoplando a identidade individual dos estudantes na camada de modelo.
- **Prevenção de Inferência por Co-ocorrência Mínima**: Se apenas 1 estudante respondeu duas questões raras $i$ e $j$, a entrada $s(i, j)$ refletiria exclusivamente o histórico daquele indivíduo. Portanto, exigir um limiar mínimo de co-avaliadores (ex.: `common_users >= 5`, atuando como **$k$-anonimato** com $k=5$) antes de persistir ou exibir $s(i, j)$ protege a privacidade individual e aumenta a robustez estatística.

#### 3.3. Prevenção de Bolhas de Filtro Curriculares (*Curriculum Filter Bubbles*) e Equidade
- **Risco Ético-Pedagógico**: Se um estudante tem dificuldade inicial em Física e facilidade em Biologia, um algoritmo míope que otimize apenas engajamento pode recomendar apenas Biologia (para fazê-lo acertar e ganhar pontos fáceis no ranking), ou um algoritmo míope de erro pode recomendar 100% de Física (negligenciando Química, História e Matemática, que têm peso igual na 1ª fase da Unicamp/ENEM).
- **Critério de Mitigação**:
  - Separar claramente **Recomendações de Reforço Focal** (dentro de uma disciplina escolhida pelo aluno) de **Recomendações Globais de Simulado** (que devem respeitar a distribuição proporcional de disciplinas da prova-alvo, usando o IBCF apenas para escolher *quais* questões dentro da cota de cada disciplina serão incluídas).
  - Não alterar a pontuação por acerto ([`POINTS_PER_CORRECT_ANSWER = 20`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/quiz_scoring.py)) de forma opaca que prejudique estudantes que aceitam desafios adaptativos mais difíceis no ranking semanal/geral ([`quiz_ranking.py`](file:///C:/root_lab/ifvest/ifvest-backend/src/services/quiz_ranking.py)).

---

## Parte V — Resumo da Arquitetura no `system-designer`

O arquivo de cenário interativo foi gerado e validado sem erros em [arquitetura_recomendacao_ifvest.json](file:///C:/root_lab/me/ic/arquitetura_recomendacao_ifvest.json), contemplando:
- **Camada de Apresentação (`IFVest Frontend`)**: telas `/quiz`, `/quiz/criar`, `/quiz/sessao` e `/simulados` com navegação interna, coleta de `response_time_ms`/`used_hint` e exibição de justificativas XAI.
- **Camada de Aplicação e Recomendação (`IFVest Backend API`, `Motor Popularidade & IBCF` e `Cache de Similaridade e Ranking`)**: rotas FastAPI, `AdaptiveQuestionChooser` injetado via `Protocol`, worker de cálculo de Popularidade Bayesiana + Time Decay e matriz IBCF com cosseno ajustado.
- **Camada de Dados (`IFVest Database` - PostgreSQL 16)**: entidades `users`, `multiple_choice_questions`, `question_history`, `user_mocktest_answers`, e as novas tabelas materializadas `question_similarities` e `question_popularity_stats` com seus relacionamentos mapeados.

---

## Referências Bibliográficas Fundamentais

1. **Wilson, R. C., Shenhav, A., Straccia, M. A., & Cohen, J. D. (2019).** *The Eighty Five Percent Rule for optimal learning*. Nature Communications, 10(1), 4646.
2. **Bjork, E. L., & Bjork, R. A. (2011).** *Making things hard on yourself, but in a good way: Creating desirable difficulties to enhance learning*. Psychology and the Real World, 2, 56-64.
3. **Roediger, H. L., & Karpicke, J. D. (2006).** *Test-enhanced learning: Taking memory tests improves long-term retention*. Psychological Science, 17(3), 249-255.
4. **Sarwar, B., Karypis, G., Konstan, J., & Riedl, J. (2001).** *Item-based collaborative filtering recommendation algorithms*. Proceedings of the 10th International Conference on World Wide Web (WWW '01), 285-295.
5. **Manouselis, N., Drachsler, H., Verbert, K., & Duval, E. (2012).** *Recommender Systems for Learning*. SpringerBriefs in Electrical and Computer Engineering.
6. **Bull, S., & Kay, J. (2010).** *Open Learner Models*. In Advances in Intelligent Tutoring Systems (pp. 301-322). Springer.
7. **Brasil (2018).** *Lei Geral de Proteção de Dados Pessoais (LGPD)* — Lei nº 13.709, de 14 de agosto de 2018 (especialmente Art. 6º sobre princípios de minimização/transparência, Art. 14 sobre tratamento de dados de crianças e adolescentes e Art. 20 sobre revisão de decisões automatizadas).
