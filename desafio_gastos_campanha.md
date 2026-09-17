# Desafio: Gastos de Campanha e Variáveis por Quartil

Duas tarefas pra estender a Fase 2 (clusterização) e preparar terreno pra uma Fase 3 (regras de associação) mais rica. Nenhuma das duas tem uma resposta "certa" fechada — façam as escolhas, documentem o porquê, e comparem com o que já foi feito nos notebooks.

---

## Parte 1 — Quanto cada candidato já gastou de campanha

Até aqui, a única informação financeira usada foi `Total_Bens` — o patrimônio **declarado antes da candidatura**, um retrato parado no tempo. O TSE também publica, **durante a própria campanha**, quanto cada candidato já arrecadou e gastou. É uma variável financeira diferente: não é "quanto a pessoa tem", é "quanto a campanha dela está investindo, agora".

### Onde buscar

Portal de Dados Abertos do TSE, conjunto **"Prestação de Contas Eleitorais - 2026"**:

- Página do conjunto: https://dadosabertos.tse.jus.br/dataset/prestacao-de-contas-eleitorais-2026
- Arquivo (zip, todas as UFs): https://cdn.tse.jus.br/estatistica/sead/odsele/prestacao_contas/prestacao_de_contas_eleitorais_candidatos_2026.zip

**Baixem pelo navegador, manualmente** — mesmo motivo do `consulta_cand` e do `bem_candidato`: o CDN do TSE (`cdn.tse.jus.br`) barra downloads automatizados (firewall Akamai, retorna 403 pra qualquer cliente que não seja um navegador de verdade). Salvem o zip em `dados/`, do mesmo jeito que os outros dois.

O sistema por trás desse dado se chama **IDC — Informações Durante a Campanha**: candidatos são obrigados a declarar periodicamente, ao longo da campanha (não só no fim), quanto já contrataram/gastaram. Ou seja, o número que vocês vão encontrar é literalmente "gasto até agora" — não uma prestação de contas final (essa só sai depois da eleição).

### O que fazer

1. **Descompactar e explorar.** O zip provavelmente contém mais de um CSV (não sabemos o nome exato ainda — façam parte do desafio descobrir). Pelo padrão histórico do TSE, esperem algo na linha de despesas **contratadas** (compromissos assumidos) separado de despesas **pagas** (o que já saiu de fato do caixa) — decidam qual das duas melhor representa "gasto até o momento", ou usem as duas e comparem.
2. **Achar a chave de junção.** Deve existir uma coluna equivalente a `SQ_CANDIDATO`, a mesma usada pra juntar `bem_candidato` ao `consulta_cand` na Fase 0. Confirmem antes de assumir.
3. **Agregar por candidato.** Cada candidato provavelmente aparece em várias linhas (uma por despesa) — somem pra chegar num valor total por `SQ_CANDIDATO`, mesma lógica do `Total_Bens` (`pivot_table`/`groupby` + soma).
4. **Criar `Total_Gasto_Campanha` e `Total_Gasto_Campanha_Log`.** Apliquem log (`np.log1p`) pela mesma razão do patrimônio: gasto de campanha tende a ser super assimétrico (a maioria gasta pouco, poucos gastam muito).
5. **Juntar ao `df_cluster`.** Atenção: candidatos sem nenhum registro de despesa até agora não vão aparecer no arquivo — ao fazer o join, isso vira `NaN`, mas o significado real é **zero gasto**, não "não sabemos". Tratem explicitamente (`fillna(0)` antes de logaritmizar, com uma nota explicando a decisão — mesmo espírito das notas de outlier que já existem no notebook).
6. **A pergunta de verdade:** incluir `Total_Gasto_Campanha_Log` como uma 4ª variável-driver (junto com `IDADE`, `ANOS_ESTUDO`, `Total_Bens_Log`) muda os clusters da Fase 2? Quem gasta mais é o mesmo grupo dos **Abastados**, ou gasto de campanha é uma dimensão diferente de patrimônio pessoal? Rodem o K-Means de novo com a variável nova e comparem (silhouette, composição dos clusters, radar).

---

## Parte 2 — Uma variável categórica por quartil, pra cada variável numérica do modelo

Peguem **cada variável numérica já usada na clusterização** (`IDADE`, `ANOS_ESTUDO`, `Total_Bens_Log` — e `Total_Gasto_Campanha_Log`, se decidirem incluir na Parte 1) e criem uma versão categórica em 4 faixas, usando quartis:

```python
df_cluster['IDADE_Faixa'] = pd.qcut(df_cluster['IDADE'], q=4, labels=['Q1', 'Q2', 'Q3', 'Q4'], duplicates='drop')
```

`Q1` = 25% mais baixo da variável, `Q4` = 25% mais alto — a mesma ideia já usada em outro contexto lá no início do projeto (`Total_Bens_Log_Cat`), só que agora aplicada a todas as variáveis-driver, não só ao patrimônio.

**Por quê:** o Apriori (Fase 3, `03_regras_associacao.ipynb`) só enxerga itens categóricos — é por isso que `IDADE`, `ANOS_ESTUDO` e `Total_Bens_Log` **não entram** na cesta hoje, só variáveis como partido, gênero e o nome do cluster. Com as faixas por quartil, vocês passam a poder incluir `IDADE_Faixa=Q1` (os mais jovens) como item de verdade, e achar regras do tipo *"IDADE_Faixa=Q1 + SG_PARTIDO=X → cluster=Jovens"* — sem jogar fora a informação numérica, só discretizando ela.

**Cuidado:** `pd.qcut` pode falhar ou gerar menos de 4 faixas quando a variável tem muitos valores repetidos — é o caso de `Total_Bens_Log`, que tem um monte de candidatos empatados em zero (lembrem do cluster "Sem bens e alta escolaridade": 100% com patrimônio zero). O parâmetro `duplicates='drop'` evita o erro, mas pode devolver menos de 4 categorias — verifiquem quantas faixas saíram de cada variável antes de seguir, e documentem se algum quartil ficou vazio ou desproporcional.

### Entrega

- As novas colunas (`Total_Gasto_Campanha`, `Total_Gasto_Campanha_Log`, e as `_Faixa` de cada variável-driver) incorporadas ao `df_cluster`.
- Uma resposta curta, com números, pra pergunta da Parte 1 (gasto muda os clusters?).
- Pelo menos uma regra de associação nova, na Fase 3, usando alguma das colunas `_Faixa` — mostrando que a discretização abriu uma pergunta que não dava pra fazer antes.
