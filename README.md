# Dashboard de Análise do Programa Bolsa Família

Este repositório contém o projeto de Business Intelligence desenvolvido no Power BI para análise dos repasses, beneficiários e abrangência municipal do Programa Bolsa Família. O projeto engloba desde o tratamento de dados em Linguagem M no Power Query até a implementação avançada de Segurança ao Nível da Linha (RLS) estática e dinâmica.

---

## 1. Visão Geral do Projeto

O objetivo deste painel é fornecer uma ferramenta de tomada de decisão para gestores públicos. O relatório foi dividido em duas perspectivas principais:
1. **Visão Geral Nacional:** Indicadores macro agregados para todo o território brasileiro.
2. **Detalhe por Região/Estado:** Visão analítica e granular, permitindo identificar disparidades regionais e os municípios com maior volume de recursos.

---

## 2. Arquitetura do Modelo Semântico

O modelo foi estruturado seguindo as melhores práticas de modelagem multidimensional , garantindo performance e suporte correto à propagação de filtros para a segurança de dados.

### Tabelas do Modelo
* **`fato_bolsa_familia`**: Contém os dados transacionais de repasses. Principais colunas utilizadas: `VALOR PARCELA`, `CPF`, `NOME MUNICÍPIO`, `ANO` e `MES`.
* **`dim_regioes`**: Tabela dimensional gerada via Power Query que mapeia cada uma das 27 Unidades Federativas (UF) à sua respectiva Região geográfica.
* **`dim_usuarios_rls`**: Tabela de mapeamento de segurança que associa o e-mail corporativo de cada gestor regional à região que ele possui permissão para visualizar.

### Relacionamentos e Cardinalidade
Para garantir o fluxo correto dos filtros, os relacionamentos foram configurados da seguinte forma:

1.  **`fato_bolsa_familia[uf]` ─── (Muitos para Um) ───> `dim_regioes[uf]`**
    * **Direção do filtro:** Única.
    * *Explicação:* Uma Unidade Federativa na tabela dimensão filtra os múltiplos registros de repasses ocorridos nela dentro da tabela fato.
2.  **`dim_regioes[Regiao]` <─── (Muitos para Um) ─── `dim_usuarios_rls[regiao_permitida]`**
    * **Direção do filtro:** Única.
    * *Explicação:* Como a tabela `dim_regioes` repete o nome da região para cada UF correspondente (ex: "Norte" aparece para AC, AM, etc.), ela assume o lado **Muitos (`*`)**, enquanto a tabela de usuários possui registros únicos por região, assumindo o lado **Um (`1`)**.

---

## 3. Medidas DAX Criadas

As métricas de negócio foram desenvolvidas utilizando a linguagem DAX, prezando pela performance e pela correta aplicação de regras de contagem e agregação.

* **Total de Repasses (R$):**
    Soma o valor total das parcelas liberadas pelo programa.
    ```dax
    Total Repasses = SUM(fato_bolsa_familia[VALOR PARCELA])
    ```
* **Total de Beneficiários:**
    Realiza a contagem distinta de CPFs únicos para mensurar o número real de cidadãos atendidos, evitando distorções causadas por múltiplas parcelas ao mesmo CPF.
    ```dax
    Total Beneficiarios = DISTINCTCOUNT(fato_bolsa_familia[CPF])
    ```
* **Ticket Médio (R$):**
    Calcula o valor médio pago por benefício de forma segura utilizando a função `DIVIDE` para evitar erros de divisão por zero.
    ```dax
    Ticket Medio = DIVIDE([Total Repasses], [Total Beneficiarios], 0)
    ```
* **Total de Municípios Atendidos:**
    Contagem distinta dos municípios impactados pelo programa.
    ```dax
    Total Municipios = DISTINCTCOUNT(fato_bolsa_familia[NOME MUNICÍPIO])
    ```

---

## 4. Dashboard


### Página 1 — Visão Geral Nacional
* **Cabeçalho de KPIs (Cartões):** Alinhamento horizontal no topo exibindo `Total Repasses`, `Total Beneficiários`, `Ticket Médio` e `Total Municípios`.
* **Análise Regional (Gráfico de Barras):** Gráfico exibindo a distribuição do `Total Repasses` por `Regiao`.
* **Filtros Temporais (Segmentação de Dados):** Segmentadores dinâmicos configurados como listas suspensas para seleção rápida de `ANO` e `MES`.

### Página 2 — Detalhe por Região/Estado
* **Matriz Analítica:** Tabela cruzada exibindo por `UF`: *Municípios Atendidos*, *Total Beneficiários*, *Total Repasses* e *Ticket Médio*, permitindo ordenação instantânea por qualquer coluna.
* **Gráfico de Barras Horizontais (Top 10 Municípios):** Exibe apenas os 10 municípios que mais receberam recursos, utilizando a regra de filtragem nativa por valor da medida `[Total Repasses]`.
* **Filtro Geográfico:** Segmentação de dados utilizando o campo `Regiao` para alternância rápida entre os mercados.

---

## 5. Implementação de Segurança (RLS)

Para atender aos requisitos de conformidade e privacidade de dados, foram implementados dois modelos de segurança (Row-Level Security):

### RLS Estático
Criado diretamente na guia de Modelagem para casos onde o acesso é fixo e imutável. Foram criados perfis como:
* `Admin`: Sem restrições de tabelas (visualiza o país inteiro).
* `Gestor_Norte`: Filtro aplicado em `dim_regioes`: `[Regiao] = "Norte"`
    * *(O mesmo padrão foi replicado para as funções `Gestor_Nordeste`, `Gestor_CentroOeste`, `Gestor_Sudeste` e `Gestor_Sul`).*

### RLS Dinâmico
Solução escalável que elimina a necessidade de manutenção manual de perfis. Criou-se uma única função denominada **`Gestor_Regional_Dinamico`**. 
O filtro aplicado na tabela `dim_usuarios_rls` captura de forma automática a identidade de quem realizou o login:
```dax
[email_usuario] = USERPRINCIPALNAME()