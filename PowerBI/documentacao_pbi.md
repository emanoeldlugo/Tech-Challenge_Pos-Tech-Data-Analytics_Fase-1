# Documentação do Projeto Power BI: Análise de Entregas e NPS

Esta documentação fornece uma visão abrangente do projeto Power BI (formato `.pbip`), detalhando todas as camadas de desenvolvimento técnico (ETL, Modelagem, DAX) e traduzindo essas lógicas em um contexto claro de negócios.

---

## 1. 📥 POWER QUERY (ETL)

A camada de preparação de dados utiliza várias fontes, majoritariamente arquivos `.csv` (provavelmente do dataset público da Olist), aplicando regras de limpeza e transformação.

### Tabelas Dimensão (Apoio e Cadastros)
- **`d_customers` (Clientes):**
  - **Fonte:** `olist_customers_dataset.csv`
  - **Transformações:** Importação, promoção de cabeçalhos e tipagem de dados.
  - **Negócio:** Contém informações demográficas dos clientes (ID, cidade, estado, CEP).
- **`d_sellers` (Vendedores):**
  - **Fonte:** `olist_sellers_dataset.csv`
  - **Transformações:** Importação, tipagem de dados e criação de uma coluna de índice (`seller_id_aux`) para criar um identificador numérico único.
  - **Negócio:** Cadastro dos vendedores parceiros do marketplace.
- **`d_regioes` (Regiões):**
  - **Fonte:** Inserção manual de dados via JSON/Base64.
  - **Transformações:** Mapeamento explícito entre `Região` (ex: Sudeste, Sul) e `UF`.
  - **Negócio:** Agrupamento geográfico para análises macro.
- **`product_category_name_translation`:**
  - **Fonte:** `product_category_name_translation.csv`
  - **Negócio:** Tabela de de/para que traduz os nomes das categorias de produtos para o inglês.

### Tabelas Fato (Eventos e Transações)
- **`f_orders` (Pedidos):**
  - **Fonte:** `olist_orders_dataset.csv`
  - **Transformações e Regras de Negócio:**
    - **Filtro de Período:** Apenas pedidos aprovados entre 01/01/2017 e 30/08/2018.
    - **Filtro de Status:** Apenas pedidos com status `"delivered"` (entregues) e com a data de entrega ao cliente preenchida.
    - **Colunas Derivadas (MUITO IMPORTANTE):**
      - `flg_atraso`: Compara a data estimada com a data real de entrega. Se a data real for maior, marca como `"Entregue com atraso"`, senão `"Entregue no prazo"`.
      - `dif_entregue_estimado`: Calcula a diferença exata em dias entre a entrega real e a estimativa.
      - `fx_atraso` e `fx_atraso_amostra_diagnostica`: Agrupa o atraso em faixas (ex: "Entregue no prazo", "1 a 15 dias", "15+ dias", "2+ dias"). Isso facilita a análise de gravidade dos atrasos.
- **`f_order_items` (Itens do Pedido):**
  - **Fonte:** `olist_order_items_dataset.csv`
  - **Transformações:** Substituição de pontos por vírgulas nas colunas monetárias (`price` e `freight_value`) para conversão correta de decimal.
  - **Negócio:** Detalha os produtos vendidos dentro de cada pedido e seus respectivos valores.
- **`f_order_reviews` (Avaliações):**
  - **Fonte:** `olist_order_reviews_dataset.csv`
  - **Transformações:** Cria a coluna `class_nps` categorizando a nota (`review_score` de 1 a 5):
    - Nota 5 = **PROMOTOR**
    - Nota 4 = **NEUTRO**
    - Notas 1, 2 e 3 = **DETRATOR**
  - **Negócio:** Base de avaliações dos clientes, essencial para o cálculo do NPS.

---

## 2. 📊 DAX (Medidas e Colunas Calculadas)

As medidas foram organizadas em duas tabelas virtuais principais.

### Tabela: `_medidas` (Indicadores Gerais)
Focada em monitorar a saúde do negócio no estado atual (Descritivo).

- **Métricas de Volume e Receita:**
  - `pedidos_total`: Contagem total de pedidos.
  - `qtd_clientes` / `qtd_reviews`: Contagens distintas de clientes e avaliações.
  - `receita`: Soma do preço (`price`) dos itens vendidos.
- **Métricas de Logística (Atrasos):**
  - `pedidos_em_atraso`: Contagem de pedidos filtrados por `flg_atraso = "Entregue com atraso"`.
  - `pedidos_atraso_taxa`: Percentual de pedidos atrasados sobre o total (`pedidos_em_atraso` / `pedidos_total`).
- **Métricas de Qualidade (NPS - Net Promoter Score):**
  - `nps_promotores` / `nps_detratores`: Conta avaliações com classificação PROMOTOR ou DETRATOR.
  - `nps`: Cálculo oficial do NPS `(% Promotores - % Detratores)`.
  - `nps_medio_total`: Calcula a média geral do NPS ignorando filtros de localidade ou vendedor (usado como baseline de comparação).
- **Métricas de Risco:**
  - `receita_em_risco`: Soma a receita de pedidos onde o cliente foi classificado como DETRATOR.
  - `receita_em_risco_%`: Percentual da receita comprometida pela insatisfação do cliente.

### Tabela: `_medidas_prescritiva` (Simulações e Cenários)
Focada em análises "What-If" (O que aconteceria se resolvermos um problema X?).

- **Identificação do Ponto Crítico:**
  - No DAX, foram criadas lógicas (em colunas calculadas na `f_order_reviews` e `f_order_items`) para isolar um grupo crítico: **Vendedores de São Paulo (SP) com mais de 2 dias de atraso**.
- **Métricas Simuladas:**
  - `class_nps_ajustado_regiao`: Simula que, se resolvermos o problema de atraso dos vendedores de SP, esses clientes detratores passariam a ser promotores.
  - `nps_critico` e `receita_em_risco_critico`: Recalculam o NPS e a Receita em Risco assumindo que esse problema foi solucionado, permitindo ver o impacto de melhoria.
  - `nps_projecao_conservador`, `nps_projecao_moderado`, `nps_projecao_otimista`: Simulam diferentes cenários de conversão de detratores em promotores variando em volumes fixos (+115, +230, etc).

---

## 3. 🔗 MODELAGEM DE DADOS (RELACIONAMENTOS)

O modelo segue uma estrutura mista de **Star Schema / Snowflake Schema** focada no `order_id` (ID do pedido).

- **Tabelas Fato:**
  - `f_orders` atua como a Fato Principal (Header do pedido).
  - `f_order_items` (Detalhes do pedido) liga-se a `f_orders` via `order_id` (N:1) com filtro bidirecional habilitado, permitindo que os filtros fluam livremente entre itens e pedidos.
  - `f_order_reviews` (Avaliações) também se liga a `f_orders` via `order_id` (N:1), bidirecional.
- **Tabelas Dimensão:**
  - `d_customers` filtra `f_orders` (1:N, bidirecional).
  - `d_sellers` filtra `f_order_items` via `seller_id` (1:N, bidirecional).
  - O relacionamento geográfico `f_geolocation` liga-se a `d_regioes` via Estado/UF.
- **Calendário:**
  - Há uma tabela de datas explícita (`d_calendar`) baseada em `CALENDARAUTO()`, relacionada com a data de compra (`order_purchase_timestamp`).
  
> **Visão de Negócio:** O modelo é robusto e centrado no Pedido. A ativação de *cross-filtering* (filtros bidirecionais) permite que ao selecionar um Vendedor (`d_sellers`), o modelo filtre automaticamente os Itens, que por sua vez filtram o Pedido, que por fim filtram o Cliente e as Avaliações. Isso permite rastrear quem comprou, quem vendeu e como foi avaliado de ponta a ponta.

---

## 4. 📈 REPORTS (PÁGINAS E VISUAIS)

O relatório possui três páginas principais construídas com um storytelling de maturidade analítica:

### 1. Análise Descritiva (`849d3f09bb3e6015acd7`)
- **Objetivo (Negócio):** Mostrar "O que está acontecendo?".
- **Visuais Esperados:** Cartões (KPIs) de Receita Total, Volume de Pedidos, NPS Geral, Taxa de Atrasos e Receita em Risco. Gráficos de linha mostrando a evolução do NPS e da Receita ao longo do tempo.

### 2. Análise Diagnóstica (`1eb927b2b803499a3a23`)
- **Objetivo (Negócio):** Mostrar "Por que está acontecendo?".
- **Visuais Esperados:** Gráficos de barras relacionando Taxas de Atraso por Estado/Região. Cruzamento do NPS x Faixa de Atraso (mostrando que quanto maior o atraso, menor o NPS). Tabelas destacando os piores vendedores em atrasos e os maiores detratores. Aqui é revelado o vilão da história: Vendedores de SP.

### 3. Análise Prescritiva (`b22b80c907b4c3bbd036`)
- **Objetivo (Negócio):** Mostrar "O que devemos fazer e qual será o impacto?".
- **Visuais Esperados:** Simuladores e cenários *What-If*. Gráficos de cascata (Waterfall) ou cartões comparativos mostrando "NPS Atual vs NPS Simulado" e "Receita Salva" caso a ineficiência de frete dos vendedores de SP seja resolvida. Uso de parâmetros para visualizar os cenários Conservador, Moderado e Otimista.

---

## 5. 🧠 VISÃO DE NEGÓCIO (INTERPRETAÇÃO E INSIGHTS)

### Qual é o objetivo geral do dashboard?
Monitorar a correlação direta entre a **eficiência logística** (entregas no prazo) e a **satisfação do cliente** (NPS), traduzindo essa relação em **impacto financeiro** (Receita em Risco).

### Quais decisões ele apoia?
- **Gestão de Parceiros:** Permite descredenciar ou aplicar treinamentos aos vendedores (`d_sellers`) que mais geram detratores.
- **Otimização Logística:** Direciona os esforços de melhoria de malha logística para as rotas ou estados mais críticos (comprovadamente, São Paulo).
- **Projeção de Metas:** Permite à diretoria estabelecer metas factíveis baseadas em simulações do que aconteceria se a operação fosse otimizada.

### Principais Insights e Narrativa
1. **O Atraso destrói o NPS:** Há uma lógica forte no ETL agrupando os atrasos. O modelo prova matematicamente que clientes que sofrem atraso tendem a dar notas de 1 a 3 (Detratores).
2. **Receita em Risco:** O problema logístico não afeta apenas a "marca", ele afeta o bolso. A métrica `receita_em_risco` quantifica o valor financeiro dos clientes que provavelmente não voltarão a comprar.
3. **O Gargalo de São Paulo:** Através da análise prescritiva, descobriu-se que se a empresa criar um plano de ação para zerar atrasos maiores que 2 dias específicos dos **Vendedores de São Paulo**, o NPS geral da empresa dará um salto significativo e milhões em receita em risco serão preservados.

### Limitações ou Pontos de Atenção
- **Uso de Filtros Bidirecionais:** O modelo faz uso extensivo de relacionamentos bidirecionais (ex: entre avaliações, itens e pedidos). Embora facilite a análise transversal para o usuário, pode gerar ambiguidade de filtragem em modelos maiores ou problemas de performance no futuro.
- **Classificação Prescritiva Estática:** A métrica simulada foca *hardcoded* no estado de "SP" e atraso de "2+ dias" dentro do DAX. Se o gargalo da empresa mudar de estado no ano seguinte, o código DAX precisará de manutenção. Idealmente, essa parametrização poderia vir de uma tabela auxiliar dinâmica.
