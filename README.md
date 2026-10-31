# Farmaviva · Vendas de uma rede de farmácias com Databricks e Power BI

Projeto de portfólio de ponta a ponta: dados fictícios de vendas de uma rede de **48 farmácias**, tratados no **Databricks** (arquitetura medalhão) e apresentados no **Power BI**, com um **mapa 3D do Brasil** (uma coluna por loja) e um **painel 3D** de evolução por mês, região e categoria.

> **Dados 100% fictícios.** Nenhuma pessoa, loja ou venda é real.

![Demonstração do relatório Farmaviva](imagens/farmaviva_demo.gif)

## Resultados em números

| Indicador | Valor |
|---|---:|
| Vendas (linhas válidas) | 283.762 |
| Receita líquida | R$ 23.224.538,09 |
| Custo total | R$ 15.822.545,26 |
| Margem bruta | R$ 7.401.992,09 (31,9%) |
| Unidades vendidas | 442.830 |
| Lojas | 48 |
| Linhas em quarentena de qualidade | 6.024 |

Todos os totais do Power BI foram conferidos contra o Databricks.

## O que este projeto mostra

- Ingestão de arquivos CSV brutos e organização em camadas **bronze, prata e ouro**.
- **Qualidade de dados** com tabela de quarentena e motivo de rejeição.
- **Modelo estrela** com tabela de datas, medidas DAX e conferência de totais entre Databricks e Power BI.
- **Governança e LGPD** aplicadas desde o desenho do modelo.
- Visualização geográfica 3D com **Azure Maps**, painel 3D e dica de ferramenta personalizada por loja.

## Visão geral da arquitetura

```
CSV (33 arquivos de vendas + lojas, clientes, produtos)
        │
        ▼
  BRONZE  →  dados crus, tudo como texto + arquivo de origem e data de ingestão
        │
        ▼
  PRATA   →  tipos corretos, limpeza, padronização, quarentena
        │
        ▼
  OURO    →  modelo estrela (dimensões + fato + agregado mensal)
        │
        ▼
  CSV exportado  →  Power BI (modelo, medidas DAX, relatório)
```

Tudo foi feito no **Databricks Free Edition**, no catálogo `workspace`, com os esquemas `fv_bronze`, `fv_silver` e `fv_gold`.

## Camadas

### Bronze
- Leitura dos CSV (UTF-8, separador `;`) com `read_files`, mantendo **todas as colunas como texto**, para nunca perder dado na entrada.
- Colunas de controle: `_arquivo_origem` e `_data_ingestao`.
- Volume: **289.786** linhas de vendas (33 arquivos, de jan/2024 em diante), 98 produtos, 48 lojas, 9.000 clientes.

### Prata
- Conversão de tipos com `try_cast`, padronização de textos e remoção de duplicidades.
- **Quarentena** (`fv_silver.vendas_quarentena`) para linhas inválidas, com a coluna `motivo_rejeicao`.
- Resultado de vendas: **283.762** linhas válidas + **6.024** em quarentena = **289.786** (confere com a bronze).

| Motivo da quarentena | Linhas |
|---|---:|
| Duplicada | 3.446 |
| Quantidade inválida (zero) | 1.119 |
| Preço inválido (sem preço) | 879 |
| Produto inexistente no cadastro | 580 |

- Os 580 itens de produto inexistente vêm de 50 códigos (99 a 148) que não estão no cadastro de produtos.
- Padronização de **categorias** de produto: 21 variações na bronze, 14 após ajustar caixa e 6 após padronizar acentos e símbolos.
- Padronização de **canais** de venda, que ficaram em 4: Loja física, Aplicativo, Site e Televendas (E-commerce e Site foram unificados).

### Ouro (modelo estrela)

| Tabela | Conteúdo |
|---|---|
| `dim_calendario` | 1.096 dias (2024 a 2026), com ano, trimestre, mês e dia da semana |
| `dim_loja` | 48 lojas, região, formato, área, faixa de área e coordenadas para o mapa |
| `dim_produto` | 98 produtos, categoria, grupo (medicamentos ou não), preço, custo e margem de lista |
| `dim_cliente` | 9.001 clientes (inclui o cliente `-1`, "Cliente não identificado"), **sem nome** |
| `fato_vendas` | 283.762 linhas, com valor bruto, desconto, receita líquida, custo e margem |
| `agg_vendas_mensal` | Resumo mensal por região, categoria e canal |

### Mapa: lojas da mesma cidade
Lojas de uma mesma cidade têm a mesma coordenada original, o que empilharia as colunas. Na ouro, criei `latitude_mapa` e `longitude_mapa`, espalhando essas lojas em círculo (raio de 0,12 grau), e mantive as coordenadas originais. O mapa passa a ter **48 posições distintas**.

## Governança e LGPD

- **Minimização:** a camada ouro **não tem o nome do cliente**. O relatório usa ID, faixa etária, UF e fidelidade.
- **Menores de 18 anos:** cadastros anteriores aos 18 anos foram sinalizados na prata, e a faixa etária da ouro não identifica pessoas.
- **Acesso:** em um ambiente real, bronze e prata teriam acesso restrito, e só a ouro seria aberta aos analistas.
- **Rastreabilidade:** cada linha da bronze guarda o arquivo de origem e a data de ingestão.
- Os dados da Farmaviva são fictícios, mas o desenho segue as boas práticas que seriam exigidas em dados reais.

## Power BI

![Página principal do relatório](imagens/pagina_principal.png)

- **Modelo:** 4 relacionamentos 1:\* com filtro em uma direção (`dim_calendario`, `dim_loja`, `dim_produto` e `dim_cliente` para `fato_vendas`), e `dim_calendario` marcada como tabela de datas.
- **Medidas DAX** (tabela `_Medidas`): Receita Líquida, Custo Total, Margem Bruta, Margem %, Itens Vendidos, Cupons, Ticket Médio, Itens por Cupom e Desconto %.
- **Mapa 3D (Azure Maps):** camada de coluna 3D, uma coluna por loja (48), altura pela receita e cor por região.
- **Dica de ferramenta personalizada:** ao passar o mouse numa coluna, uma página de dica mostra só loja, vendas, receita e margem, sem latitude nem longitude.
- **Painel 3D:** colunas por mês, região e categoria, com filtros de período, região e canal, mix por categoria e participação dos canais. Usa a tabela `agg_vendas_mensal`.

![Dica de ferramenta por loja](imagens/dica_loja.png)

### Nota sobre "itens"
Na tabela `agg_vendas_mensal`, a coluna `itens` conta **linhas de venda** (283.762), enquanto `quantidade` soma as **unidades vendidas** (442.830). O painel usa `quantidade`, para ficar consistente com o mapa e com o Databricks.

## Decisões de projeto

- **Quarentena, não exclusão:** linhas inválidas ficam guardadas com o motivo. Itens com quantidade zero (1.119) e sem preço (879) permanecem na quarentena, sem recuperar o preço pelo cadastro. Recuperação de preço fica como **melhoria futura**.
- **Sem particionamento nem clustering:** a fato tem cerca de 284 mil linhas, então o ganho seria mínimo. Em volumes maiores, usaria *liquid clustering* por data e loja.
- **DDL como guia:** o arquivo DDL recebido do grupo serviu de referência de desenho. As tabelas foram criadas com `CREATE TABLE AS SELECT`.
- **Exportação para CSV:** as tabelas da ouro foram exportadas em CSV para o Power BI. Uma **versão 2** pode testar a conexão direta Databricks → Power BI.

## Desafios e aprendizados

### Tipos de dados na importação para o Power BI
**Problema:** na primeira importação dos CSV, as colunas de valor da `fato_vendas` foram lidas como **número inteiro**, e os valores ficaram sem centavos e com ordem de grandeza errada. O mesmo ocorreu com latitude e longitude da `dim_loja`.

**Causa:** os tipos foram detectados na primeira importação com a configuração regional em português do Brasil, que lê o ponto decimal como separador de milhar. O Power BI guardou esses tipos na etapa "Tipo Alterado", e eles permaneceram mesmo depois de corrigir a configuração regional.

**Solução:** no Power Query, refiz os tipos manualmente em cada tabela (`type number` para valores e coordenadas, `Int64.Type` para inteiros, `type date` e `type text`).

**Resultado:** os totais do Power BI passaram a coincidir com os do Databricks.

### Azure Maps no Power BI
**Problema:** o visual Azure Maps não abria e mostrava a mensagem de que não era possível acessar o serviço.

**Causa:** o visual exige entrada com uma conta organizacional e três configurações de locatário habilitadas no portal de administração do Power BI: uso do visual do Azure Maps, processamento fora da região geográfica e processamento por subprocessadores dos serviços online da Microsoft.

**Solução:** configurei o ambiente de teste com conta organizacional própria e habilitei as configurações. Depois disso o mapa 3D passou a carregar normalmente.

## Créditos

- O visual personalizado de colunas 3D do painel é do **Favela dos Dados**, e os scripts de referência fazem parte do pacote recebido do grupo. O crédito é deles.
- Pipeline no Databricks, tratamento de qualidade, modelo, medidas, mapa 3D, dica de ferramenta, layout do relatório, conferências e documentação: **Luana Ferreira de Sousa**.

## Como reproduzir

1. Subir os CSV para um volume do Databricks (`farmaviva_raw`).
2. Executar as etapas bronze, prata e ouro na ordem, criando as tabelas descritas acima.
3. Exportar as 6 tabelas da ouro em CSV.
4. No Power BI Desktop, importar os CSV (separador `;`, UTF-8), conferir os tipos, montar o modelo e as medidas DAX.
5. Para o mapa 3D, usar o visual **Azure Maps** (requer conta organizacional e as configurações de locatário citadas acima).

## Melhorias futuras

- Conectar o Power BI direto ao Databricks (SQL Warehouse).
- Recuperar o preço dos itens sem preço pelo cadastro de produtos, com regra documentada.
- Ordenar nomes de mês e dia da semana no calendário e acrescentar medidas de inteligência de tempo (ano anterior, acumulado no ano, média móvel).
- Versão em inglês e em espanhol deste README.

---
*Autora: Luana Ferreira de Sousa · [GitHub](https://github.com/lua008Ferreira) · Dados fictícios · Projeto de portfólio*
