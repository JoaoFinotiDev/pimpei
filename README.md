# Pimpei

> Marketplace de cartas **Pimp** de One Piece TCG, com possibilidade de expansão futura para outros TCGs.

## Sobre o projeto

O Pimpei é um projeto de estudo e desenvolvimento de um produto real, criado para consolidar conceitos de orientação a objetos (OO) por meio de aplicações práticas.

## Objetivos de aprendizagem

- Aplicar conceitos de orientação a objetos em um domínio real.
- Desenvolver o marketplace em etapas pequenas, com entregas funcionais.
- Relacionar decisões de arquitetura aos conceitos de OO estudados em Java.
- Estudar aproximadamente **1 hora por dia**, priorizando os conhecimentos exigidos pela próxima fase.

## Escopo do produto

- Apenas cartas físicas.
- Exclusivamente cartas Pimp.
- Inicialmente, One Piece TCG; outros TCGs poderão ser considerados futuramente.

### O que é uma carta Pimp?

Neste projeto, uma Pimp é uma carta de **Arte Alternada (AA)**, independentemente da raridade ou do tipo da carta base. Isso inclui, por exemplo, cartas Crew, SP, AAs de Eventos e Leaders e promocionais exclusivas, como cartas serializadas.

A regra inicial adotada para identificar uma Pimp é: **se o código da carta tem sufixo, ela é considerada Pimp**.

Exemplos:

- `OP16-098`: carta normal.
- `OP16-098-AA`: arte alternada.
- `OP16-098-SP`: SP.
- `OP16-079` e `OP16-079-AA`.

Se surgir um sufixo que não represente uma Pimp, a regra deverá ser revista. O sufixo será armazenado como texto livre, sem enumeração fechada.

## Experiência de referência

A referência de interface é a página de carta da LigaOnePiece: a carta aparece no topo e, abaixo, é exibida a lista de ofertas.

Cada oferta deverá apresentar:

- Vendedor e selo de verificação.
- Condição da carta (por exemplo, `NM`).
- Idioma e acabamento (por exemplo, Foil).
- Indicador de foto da carta real.
- Preço atual e preço original riscado.
- Quantidade disponível.
- Botão de compra.

Filtros previstos:

- Vendedor.
- Cartas graduadas ou não graduadas.
- Quantidade mínima.

Também estão previstos resumos de preço médio no marketplace e de vendas dos últimos três meses. A interpretação de cada resumo como mínimo, médio e máximo ainda **não foi confirmada**.

Para cartas não graduadas, os atributos da carta — cor, custo e outros — serão usados principalmente para filtrar buscas. O centro do marketplace é a **oferta**.

### Fora do escopo inicial

- Lista de compras com melhor preço.
- Bazar de trocas.
- Buylist.
- Lista de desejos.
- Nota numérica de graduação.
- Cartas digitais.
- Cartas que não sejam Pimp.

## Tecnologias e arquitetura

| Componente | Decisão atual |
|---|---|
| Site e backend principal | Node.js |
| Frontend | React ou Vue — escolher apenas um |
| Banco de dados | PostgreSQL, ou SQLite inicialmente |
| Importação de cartas | Job que consome uma API pública e grava no banco próprio |
| Gráficos e estatísticas de preço | Serviço em Go, após o lançamento do marketplace |
| Deckbuilder | Serviço em Java, após o lançamento do marketplace |
| Comunicação entre serviços | HTTP/JSON com contratos simples |

O frontend **não deverá chamar diretamente a API pública de cartas**. Os dados serão importados para o banco do Pimpei por um job próprio.

O Deckbuilder em Java também precisará das cartas normais, importadas e marcadas como não vendáveis.

## Modelo de domínio

### `CodigoCarta`

Value Object responsável por interpretar o código da carta, incluindo set, número e sufixo. Deve expor o código-base e indicar se a carta é Pimp (`isPimp`).

### `CartaBase`

Representa a carta identificada pelo código-base. Atributos previstos:

- Nome.
- Tipo: Leader, Personagem, Evento ou Stage.
- Cor.
- Custo.
- Poder.
- Efeito.

Serve para filtros e, futuramente, para o Deckbuilder.

### `Variante`

Representa uma impressão específica da carta-base. Contém:

- Código completo.
- Sufixo.
- Set.
- Raridade.
- Imagem.

A página e as ofertas pertencem à variante. O set é uma propriedade da variante; versões normal, AA e SP podem pertencer a sets diferentes.

### `Anuncio`

Classe abstrata com vendedor, variante e preço.

#### `AnuncioEstoque`

Representa cartas não graduadas e intercambiáveis. Possui condição, idioma, acabamento e quantidade.

**Chave única:** vendedor + variante + condição + idioma + acabamento.

#### `AnuncioGraduado`

Representa uma carta graduada, de unidade única, associada a um `ExemplarGraduado`.

### `ExemplarGraduado`

Representa uma carta física individual graduada. Deve registrar:

- Dono.
- Nota.
- Laudo.
- Fotos.
- Localização física: com o vendedor, em trânsito, no armazém ou com o comprador.

### Outras entidades

- **`Venda`** — histórico utilizado nos resumos de preço dos últimos três meses.
- **`Pedido` e `ItemDoPedido`** — um pedido pode combinar itens de estoque e cartas graduadas.
- **`Envio`** — etiqueta, rastreio e responsável pelo frete.
- **`Pagamento`** — retenção e liberação de valores, em um fluxo semelhante a escrow.

## Regras e boas práticas já definidas

- Representar dinheiro em centavos (inteiro) ou com tipo decimal; **nunca usar `float`**. Criar uma classe `Dinheiro`.
- Representar `Condicao` como enum ordenado, por exemplo: `M`, `NM`, `SP`, `MP`, `HP`.
- Fazer a baixa de estoque de forma atômica no banco, com uma operação equivalente a `UPDATE ... WHERE quantidade >= n`.
- Reservar estoque por um período durante o pagamento.
- Considerar os padrões:
  - **Value Object** — por exemplo, `CodigoCarta` e `Dinheiro`.
  - **Strategy** — cálculo de frete e taxas.
  - **State** — estados de pedidos e graduação.
  - **Adapter** — fontes de dados e provedores de frete.
- Cartas serializadas numeradas podem precisar ser tratadas como unidades únicas; essa decisão permanece em aberto.

## Fluxos de negócio

### 1. Venda padrão

1. O vendedor anuncia a carta.
2. O comprador realiza o pagamento.
3. O sistema gera uma etiqueta com QR Code da Pimpei.
4. O vendedor posta o pacote.
5. O rastreio atualiza o pedido.
6. O pagamento é liberado após a entrega.

Estados previstos:

`Anunciado → Pago → Etiqueta gerada → Postado → Em trânsito → Entregue → Pagamento liberado`

### 2. Graduação e venda

1. O vendedor solicita a graduação.
2. Envia a carta à Pimpei.
3. A Pimpei recebe e avalia a carta, emitindo um laudo.
4. O vendedor escolhe entre receber a carta de volta ou deixá-la em custódia.

**Opção A — Devolução:** o vendedor paga o frete de ida e de volta e recebe a carta após a avaliação.

Estados:

`Solicitado → Aguardando envio → Recebida → Em avaliação → Graduada → Devolução solicitada → Enviada → Entregue`

**Opção B — Custódia:** a carta fica armazenada e é anunciada no site pelo preço pós-graduação. Quando vendida, a Pimpei envia a carta ao comprador.

Estados:

`Solicitado → Aguardando envio → Recebida → Em avaliação → Graduada → Em custódia → Anunciada → Vendida → Enviada ao comprador → Entregue → Pagamento liberado`

Na primeira versão, a graduação será um **laudo de condição com checklist e fotos**. A nota numérica fica para uma versão posterior, pois uma nota emitida pela Pimpei não terá reconhecimento de mercado no início.

## Importação de dados das cartas

### Fontes candidatas

- **OPTCG API (`optcgapi.com`)** — candidata principal, descrita no planejamento como gratuita e com dados de cartas, URLs de imagens e preços.
- **TCG Price Lookup API** — alternativa para preços ao vivo, com SDKs para JavaScript/TypeScript e Go.
- **opbindr** — evitar, pois o acesso aos dados é restrito.

> **Pendente de verificação:** limites de uso, termos e planos das APIs.

A documentação consultada no planejamento sugere que artes alternadas podem aparecer com um sufixo no ID da imagem (por exemplo, `_p1`) e com “Alternate Art” no nome. Isso ainda precisa ser confirmado inspecionando respostas JSON reais.

### Regras de importação

- Definir um formato canônico de código Pimpei: `SET-NÚMERO-SUFIXO`.
- Criar um adaptador por fonte, por meio de uma interface `FonteDeCartas`.
- Importar apenas códigos com sufixo para o catálogo vendável inicial.
- Usar **upsert** para evitar duplicações.
- Criar a `CartaBase` a partir do próprio registro da Pimp.
- Atualizar os dados por job agendado.
- Cadastrar manualmente cartas ausentes da API — possivelmente algumas serializadas — com fonte `MANUAL`.

As imagens pertencem à Bandai. O uso é considerado aceitável para um projeto didático, mas deverá ser reavaliado se o projeto se tornar um produto real.

## Decisões em aberto

- **Cartas serializadas:** confirmar se são numeradas e se cada unidade deverá ser um anúncio individual.
- **Taxas:** definir quem paga cada etapa da graduação e qual será a comissão em vendas de cartas mantidas em custódia.
- **Armazenagem:** definir prazo máximo de custódia e o que ocorre após o vencimento.
- **Retirada:** decidir se uma carta já anunciada no armazém pode ser retirada e qual será o custo.
- **Contestação:** definir o procedimento caso o dono discorde da avaliação ou o comprador receba uma carta diferente da anunciada.
- **APIs:** verificar limites, termos de uso e planos.
- **Frontend:** escolher entre React e Vue.
- **Banco inicial:** decidir entre PostgreSQL e SQLite.

Operar um marketplace real também envolve responsabilidades sobre extravio, seguro, impostos, transportadoras e proteção ao consumidor. Por enquanto, os processos físicos serão simulados; o foco é o software.

## Roadmap de desenvolvimento

O plano considera sessões de aproximadamente uma hora por dia.

| Fase | Duração estimada | Conteúdo e entrega |
|---|---:|---|
| 1. Fundamentos | 1 semana | Git/GitHub, terminal, JavaScript moderno (`async/await`, módulos) e HTTP/REST |
| 2. Catálogo | 2–3 semanas | API, classes em JS, regex, SQL, upsert, servidor Node (Express ou Fastify), banco, migrations, testes, job agendado e frontend básico. **Entrega:** `CodigoCarta` testado, importador idempotente, busca com filtros e página da variante |
| 3. Vendedores e anúncios | 2–3 semanas | Autenticação, autorização, validação, segurança web básica, chave única composta, `Dinheiro`, `Condicao`, filtros, paginação e upload. **Entrega:** página da variante com ofertas e filtros |
| 4. Carrinho, estoque e pedidos | 2–3 semanas | Transações SQL, concorrência, reserva com expiração, State, Strategy, idempotência e testes de integração. **Entrega:** teste com 20 compras simultâneas da última unidade e apenas um vencedor |
| 5. Testes e deploy | 2 semanas | Pirâmide de testes, CI com GitHub Actions, Docker, variáveis de ambiente, logs, backup e HTTPS. **Entrega:** lançamento do marketplace |
| 6. Frete real | 2 semanas | APIs de terceiros, webhooks com validação de assinatura, QR Code, PDF de etiqueta, filas, retries e Adapter |
| 7. Graduação e custódia | 3–4 semanas | Estados com histórico de auditoria, várias fotos, painel de avaliador, PDF do laudo, eventos de domínio e testes guiados por regras |
| 8. Serviço de gráficos em Go | 2 semanas | Go básico, `net/http`, acesso a banco, SQL de agregação, OpenAPI e gráficos no frontend |
| 9. Deckbuilder em Java | 4 semanas | OO em Java, Maven/Gradle, JUnit 5, API REST com Spring Boot ou Javalin e regras oficiais de deck |

### Pré-estudo

Antes das fases do projeto, está planejado um pré-estudo de cerca de três semanas, com uma hora por dia:

- **Semana 1:** Git, terminal, JavaScript moderno e HTTP.
- **Semana 2:** Node com Express/Fastify, SQL e PostgreSQL.
- **Semana 3:** classes em JavaScript, regex, testes unitários e React ou Vue básico.

Em paralelo às fases 1 a 4, serão feitos mini-exercícios de OO em Java, de 15 a 20 minutos por dia, usando o domínio do Pimpei — por exemplo, recriar `CodigoCarta` em Java.

## Marca e identidade visual

O nome "Pimpei", o logotipo e a identidade visual do projeto
não estão abrangidos pela licença MIT do código-fonte.

Todos os direitos sobre esses elementos são reservados aos
respectivos titulares. A licença do código não concede
permissão para utilizar o nome ou a identidade visual Pimpei
para identificar produtos ou serviços de terceiros.
