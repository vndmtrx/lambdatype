# Diário de Implementação — LambdaType (Compilador Python)

## Estado Atual
- **Fase:** 0 (Preparação e Infraestrutura Base)
- **Status:** Iniciando a construção da infraestrutura principal do parser.

## Decisões Arquiteturais
- **Pipeline de três estágios:** O compilador tem três estágios separados, não dois. (1) **Parser** → AST bruta, só sintaxe, sem resolver nada. (2) **Passe semântico** → AST decorada, com numeração de footnotes, sanitização de IDs de âncora, desambiguação de anchors duplicados e validação de atributos `.([...])`. (3) **Emitter** → string de saída via hooks. O núcleo do parser não sabe qual backend está sendo gerado.
- **Posição em todo nó da AST:** Todo nó carrega `pos: (line, column)` do primeiro byte que o originou. Definido na Fase 0, não retrofitado depois. Motivo: LSP com posição correta de warning.
- **Hooks são funções, não flags:** `on_footnote_emit`, `on_media_emit`, `on_code_emit` são registrados como funções, e o núcleo só chama e concatena o retorno. Não é flag de formato passada ao emitter. Isso evita `if format == "html"` crescendo dentro do núcleo.

## Dúvidas e Lacunas da Spec
- [Lembrete para Fase 8]: A propriedade `pos` (line, column) dos nós inlines (ex: `**texto**`) deve apontar para o início do delimitador de abertura ou para o início do conteúdo textual em si? Decidido que apontará para o delimitador, por ser mais fiel ao início da estrutura e facilitar alertas úteis de sintaxe, mas valerá revisão quando o *inline scanner* for implementado.

## Próximo Passo imediato
- [ ] Definir classe base `ASTNode` com campo `pos: (line, column)`.
- [ ] Implementar `DocumentNode` (herdando de `ASTNode`) com `frontmatter` e lista de `children`.
- [ ] Estruturar `Frame` para controle de pilha (`kind`, `sigil_expected`).
- [ ] Implementar rotina de Pré-passe (normalização de BOM, conversão CRLF → LF, reconhecimento básico de `BlankLine`).
- [ ] Escrever Golden Tests básicos ("arquivo vazio", "apenas texto") para provar resiliência sem panic/exception.