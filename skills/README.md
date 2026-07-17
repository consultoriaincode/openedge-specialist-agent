# Skills de Agente AdvPL/TLPP

Uma coleção de **10 skills** de agente IA para o ecossistema **TOTVS Datasul ERP**, abrangendo as linguagens de programação **Progress Openedge**. Essas skills orientam assistentes de IA em fluxos de trabalho estruturados para geração de código, revisão de qualidade, testes e documentação dentro da plataforma Datasul.

---

## Sumário

- [Skills por Categoria](#skills-por-categoria)
  - [Geração de código](#geração-de-código)
  - [Qualidade de Código e Revisão](#qualidade-de-código-e-revisão)
  - [Testes](#testes)
  - [Documentação e Planejamento](#documentação-e-planejamento)
- [Referência Rápida: Qual Skill Usar?](#refer?ncia-rápida-qual-skill-usar)

---

## Skills por Categoria

### Geração de Código

Skills que geram estruturas de código prontas para produção seguindo padrões do framework TOTVS.

| Skill | Descrição |
|-------|-----------|
| [code-generator](code-generator/SKILL.md) | Gera códigos em Openedge utilizando boas práticas |
| [ddk-generator](ddk-generator/SKILL.md) | Gera códigos utilizando DDK |
| [rest-endpoint-generator](rest-endpoint-generator/SKILL.md) | Gera endpoints REST com e sem utapi e utapi-utils. Segue os padrões de API com ou sem integração ao PO-UI. |
| [rest-client-generator](rest-client-generator/SKILL.md) | Gera código para consumo de APIs REST externas utilizando as ferramentas e bibliotecas do .NET. Cobre os verbos GET, POST, PUT e DELETE, configuração de headers, parâmetros de query/path, serialização e desserialização de JSON, autenticação (No Auth, HTTP Basic, Bearer/JWT e OAuth 2.0), timeout, SSL/TLS, tratamento de códigos HTTP, exceções, políticas de retry e boas práticas de integração. |
| [entry-point-designer](entry-point-designer/SKILL.md) | Realiza o desenvolvimento de costumização através dos pontos de entrada do produto em telas clássicas. |

### Qualidade de Código e Revisão

Skills que aplicam padrões de qualidade, detectam problemas e melhoram código existente.

| Skill | Descrição |
|-------|-----------|
| [code-review](code-review/SKILL.md) | Realiza revisão completa de código OpenEdge ABL, avaliando segurança, performance, consultas ao banco de dados, gerenciamento de transações, uso de buffers, temp-tables e queries, tratamento de erros, gerenciamento de recursos, código legado, padrões de desenvolvimento, documentação, Clean Code e possíveis problemas de compilação. Os resultados são classificados por severidade e incluem recomendações práticas de melhoria baseadas em boas práticas de desenvolvimento e qualidade de código. |
| [refactor](refactor/SKILL.md) | Refatoração cirúrgica de código para melhorar a manutenibilidade sem alterar o comportamento. Aborda 15 code smells (métodos longos, código duplicado, condicionais aninhados, números mágicos, etc.) com padrões seguros de extração e orientações específicas para Openedge.|

### Testes

Skills que geram scripts de testes automatizados tanto para lógica de negócio quanto para validação de interface.

| Skill | Descrição |
|-------|-----------|
| [test-generator](test-generator/SKILL.md) | Gera testes unitários para rotinas AdvPL/TLPP, cobrindo validação de regras de negócio, funções, métodos, tratamentos de erro, cenários positivos e negativos, casos de borda, uso de mocks e stubs quando aplicável, além de boas práticas para organização, nomenclatura e manutenção da suíte de testes. |

### Documentação e Planejamento

Skills para documentar código e planejar trabalhos de implementação.

| Skill | Descrição |
|-------|-----------|
| [documentation-writer](documentation-writer/SKILL.md) | Gera blocos de comentários para os códigos, incluindo cabeçalho e nomenclaturas de métodos e comportamentos. |
| [data-dictionary-lookup](data-dictionary-lookup/SKILL.md) | Consulta o dicionário de dados. Também utilizado durante refatorações e melhorias de código para validação de impacto no dicionário. |

---

## Referência rápida: Qual Skill Usar?

| Eu quero... | Use esta skill |
|-------------|----------------|
| Construir uma API REST | `rest-endpoint-generator` |
| Consumir uma API REST externa | `rest-client-generator` |
| Revisar qualidade de código | `code-review` |
| Melhorar estrutura de código | `refactor` |
| Criar testes de unidade | `test-generator` |
| Adicionar documentação| `documentation-writer` |
| Consultar dicionário de dados do Datasul | `data-dictionary-lookup` |
| Criar códigos |`code-generator` |
| Criar códigos utilizando DDK |`ddk-generator` |
| Criar códigos utilizando os pontos de costumização de telas padrão do produto |`entry-point-designer` |
