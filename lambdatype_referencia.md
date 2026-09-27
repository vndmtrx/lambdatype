# [λT] LambdaType (`.lt`)
> *"Less ambiguity. Pure typography."*

## Referência Rápida

### Estrutura
| Sintaxe | Emissão |
| :--- | :--- |
| `# Título` … `###### Título` | `<h1>` … `<h6>` |
| `##[id] Título` | Título com âncora fixa |
| `--- isolado` | `<hr>` (fora do Offset 0) |
| `|[ Legenda ]|` | Legenda de tabela |

### Texto
| Sintaxe | Emissão |
| :--- | :--- |
| `**negrito**` | `<strong>` |
| `~~itálico~~` | `<em>` |
| `==destaque==` | `<mark>` |
| `--riscado--` | `<del>` |
| `++inserido++` | `<ins>` |
| `^^sobrescrito^^` | `<sup>` |
| `__subscrito__` | `<sub>` |
| ` ``código`` ` | `<code>` |

### Caixas
| Sintaxe | Emissão |
| :--- | :--- |
| `[[url]]` | Link direto |
| `[[texto][url]]` | Link com rótulo |
| `[[texto][dica][url]]` | Link com tooltip |
| `{[src]}` | Imagem |
| `{[alt][src]}` | Imagem com alt |
| `{[alt][legenda][src]}` | Figura com legenda |
| `[#id]` | Salto de âncora |
| `[^ref]` | Chamada de rodapé |
| `([ref][Título]...)` | Definição de rodapé |

### Blocos
| Sintaxe | Emissão |
| :--- | :--- |
| `- item` | Lista não-ordenada |
| `1. item` | Lista ordenada |
| `-[termo] def` | Lista de definição |
| `- (_) / (#)` | Checkbox desmarcado / marcado |
| `> citação` | Citação em bloco |
| `<[!] Título` | Aviso (5 tipos: `-`, `#`, `?`, `!`, `+`) |
| `:[-] Título` | Bloco recolhível |
| `:[+] Título` | Bloco expandido |
| `` ```[lang] `` | Cerca de código |
| `===` | Injeção crua |
| `.([.classe][#id])` | Atributos (inline ou bloco) |

### Inline Especial
| Sintaxe | Emissão |
| :--- | :--- |
| `!([tecla])` | Tecla (`<kbd>`) |
| `?([sigla][nome])` | Abreviação (`<abbr>`) |
| `:([termo])` | Definição (`<dfn>`) |
| `$([expr])` | Fórmula matemática |
| `#([tag])` | Tag |
| `@([user])` | Menção |
| `&([fonte])` | Citação bibliográfica |
| `^([texto])` | Citação curta |
| `~([termo])` | Texto idiomático |
| `*([termo])` | Destaque visual |
| `-([termo])` | Texto menor |
| `%([iso][visual])` | Data/hora |

### Controle de Linha
| Sintaxe | Emissão |
| :--- | :--- |
| `\n` simples | Continuação suave (espaço) |
| `\` + `\n` | Quebra forçada (`<br>`) |
| `\n\n` | Fim de bloco |
| `\C` | Escape de caractere |
| ` . ` | Cola léxica |
| `/* ... */` | Comentário (removido) |


---

## Parte 0: Metadados do Documento (Frontmatter)

### 0.1 Invariante do Offset Zero
O cabeçalho de metadados do documento é delimitado por cercas de triplo hífen (`---`), com a exigência estrita de iniciar exatamente no **Byte 0 (`Offset 0`, linha 1, coluna 1)** do arquivo:

```text
---
title: Especificação do LambdaType
author: Equipe de Arquitetura
version: 1.1.0
tags: [compiladores, tipografia, lsp]
---
```

#### 0.1.1 Suporte a Formatos via Caixa de Identidade
Por padrão, o formato assumido entre as cercas `---` é **YAML**. Para utilizar formatos alternativos, anexa-se a Caixa de Identidade com o nome do formato colado aos hífens de abertura:

| Delimitador de Abertura (Offset 0) | Formato Assumido | Delimitador de Fechamento | Emissão / Destino |
| :--- | :---: | :---: | :--- |
| `---` | **YAML** (Padrão) | `---` | Objeto `AST.frontmatter` (`raw` string para plugins) |
| `---[toml]` | **TOML** | `---` | Objeto `AST.frontmatter` (`raw` string para plugins) |
| `---[json]` | **JSON** | `---` | Objeto `AST.frontmatter` (`raw` string para plugins) |
| `---[custom]` | **Personalizado** | `---` | Objeto `AST.frontmatter` (`raw` string para plugins) |

#### 0.1.2 Arquitetura do Compilador, Tolerância e Normalização
- **Processamento no Núcleo ($O(n)$ sem Dependências):** O compilador central do LambdaType não embute parsers pesados de YAML ou TOML no binário principal. O lexer extrai o bloco de texto bruto (`raw`) e o associa à raiz da AST. O parsing estruturado do conteúdo é delegado a plugins downstream via gancho de emissão `on_frontmatter_emit` ou ferramentas de SSG (Static Site Generators).
- **Tolerância a Arquivo Incompleto (Lossless EOF Fallback):** Caso o documento termine sem a cerca `---` de fechamento, o compilador não interrompe a execução: degrada graciosamente o bloco para texto comum inicial do corpo e emite um `Warning` educativo no LSP:
  > `[Warning L1:C1] Bloco de Frontmatter não finalizado. Esperado delimitador '---' de fechamento.`
- **Tolerância a UTF-8 BOM:** Arquivos codificados com a marca de ordem de byte UTF-8 (BOM: `0xEF, 0xBB, 0xBF` / `U+FEFF`), comuns no Windows, têm o BOM automaticamente descartado pelo pré-processador antes de validar o Byte 0, preservando a detecção do Frontmatter.
- **Normalização Canônica de Quebras de Linha:** Todas as sequências de quebra de linha física (`\r\n` CRLF ou `\r` legado) são normalizadas para o byte único `\n` (`0x0A`) na fase inicial de bufferização de entrada.

---

## Parte 1: Títulos e Seções

### 1.1 Tabela de Títulos e Cabeçalhos

| Elemento | Sintaxe LambdaType | Regra de Reconhecimento | Emissão Semântica (HTML) |
| :--- | :--- | :--- | :--- |
| **Título Nível 1** | `# Título` | Regra Léxica: 1 `#` no início da linha + espaço | `<h1>Título</h1>` |
| **Título Nível 2** | `## Título` | Regra Léxica: 2 `#` no início da linha + espaço | `<h2>Título</h2>` |
| **Título Nível 3** | `### Título` | Regra Léxica: 3 `#` no início da linha + espaço | `<h3>Título</h3>` |
| **Título Nível 4** | `#### Título` | Regra Léxica: 4 `#` no início da linha + espaço | `<h4>Título</h4>` |
| **Título Nível 5** | `##### Título` | Regra Léxica: 5 `#` no início da linha + espaço | `<h5>Título</h5>` |
| **Título Nível 6** | `###### Título` | Regra Léxica: 6 `#` no início da linha + espaço | `<h6>Título</h6>` |
| **Título com Âncora Fixa\*** | `##[id-fixo] Título` | Regra Sintática: `[id]` colado aos `#` + espaço | `<h2 id="id-fixo">Título</h2>` |

> **\* Regras de Conformidade e Sanitização do Identificador (`id`):**  
> - O conteúdo de `[id]` aceita nativamente apenas caracteres seguros para URI/DOM: `a-z`, `A-Z`, `0-9`, `-` e `.`.
> - Caracteres fora desse conjunto disparam um diagnóstico de `Warning` no LSP e são sanitizados deterministicamente pelo compilador:
>   1. **Símbolos ilegais:** Qualquer símbolo fora de `-` e `.` (ex.: `@`, `!`, `#`, `$`, `%`) é removido da string.
>   2. **Zonas contíguas de substituição:** Espaços (`0x20`) e/ou letras acentuadas/não-ASCII colapsam para **no máximo duplo hífen (`--`)** (ou hífen simples `-` se for apenas um espaço entre palavras).
>   3. **Desambiguação automática:** Em caso de IDs duplicados no mesmo documento, anexa-se um sufixo numérico sequencial (`-1`, `-2`).
> - **Degradação de Hashes Excedentes:** Linhas que iniciem com 7 ou mais hashes (`#######`) não constituem títulos válidos e degradam diretamente para texto de parágrafo comum em $O(1)$.

---

## Parte 2: Elementos Textuais Inline

### 2.1 Estilos Tipográficos Casados

| Elemento | Sintaxe LambdaType | Regra de Reconhecimento | Emissão Semântica (HTML) |
| :--- | :--- | :--- | :--- |
| **Importante / Negrito** | `**texto**` | Regra Léxica: Vizinhança assimétrica (colado nas pontas) | `<strong>texto</strong>` |
| **Ênfase / Modulação** | `~~texto~~` | Regra Léxica: Vizinhança assimétrica (colado nas pontas) | `<em>texto</em>` |
| **Realce / Destaque** | `==texto==` | Regra Léxica: Vizinhança assimétrica (colado nas pontas) | `<mark>texto</mark>` |
| **Removido / Riscado** | `--texto--` | Regra Léxica: Vizinhança assimétrica (colado nas pontas) | `<del>texto</del>` |
| **Inserido** | `++texto++` | Regra Léxica: Vizinhança assimétrica (colado nas pontas) | `<ins>texto</ins>` |
| **Sobrescrito** | `^^texto^^` | Regra Léxica: Vizinhança assimétrica (colado nas pontas) | `<sup>texto</sup>` |
| **Subscrito** | `__texto__` | Regra Léxica: Vizinhança assimétrica (colado nas pontas) | `<sub>texto</sub>` |
| **Código Inline** | ` ``código`` ` | Regra Léxica: Delimitador duplo de crases (conteúdo literal) | `<code>código</code>` |

#### 2.1.1 Regras de Imunidade Alfanumérica e Borda Estrita
Para evitar capturas acidentais em identificadores de programação, expressões matemáticas e termos técnicos:
- **Imunidade de Identificadores Snake_case e Dunders (`__init__`):** O delimitador de subscrito `__` não é ativado quando ambos os lados estiverem encostados em caracteres alfanuméricos (`[a-zA-Z0-9]__[a-zA-Z0-9]`). Casos como `__init__`, `__proto__` ou `variavel__privada` degradam para texto literal puro em $O(1)$.
- **Imunidade de Operadores de Programação (`C++`, `++i`, `i++`):** O delimitador de inserido `++` exige estritamente caractere não-espaço internamente e par casador na mesma linha física. Sequências como `C++` (precedido por identificador sem fechamento posterior) ou `++i` (precedido por espaço ou início de linha sem par casador) degradam deterministicamente para texto literal puro.
- **Desambiguação de Hífen Duplo (`--`):** O par `--` opera como travessão curto (*en-dash* `&ndash;`) quando aplicado entre espaços (` -- `) ou entre números e palavras (`1914--1918`, `word--other`). Ele só ativa `<del>` quando respeita a vizinhança assimétrica de estilo (colado no texto interno e separado no texto externo).

### 2.2 A Ontologia Semiótica das Três Caixas

Antes de estruturar hiperlinks, mídias e notas de rodapé, o LambdaType define uma divisão semiótica estrita e ortogonal entre três famílias de delimitadores primários:

| Caixa Primária | Papel Semiótico | Pergunta Respondida | Conteúdo Canônico |
| :---: | :--- | :---: | :--- |
| **`[ ... ]`** | **Identidade & Apresentação** | *O Quê?* | Conteúdo legível por humanos, rótulos textuais, textos alternativos e identificadores de ancoragem. |
| **`{ ... }`** | **Recurso & Ativo Externo** | *O Objeto?* | Arquivos pesados externos, imagens, vídeos, ativos de rede e fontes de mídia (`src`). |
| **`( ... )`** | **Operação & Sistema** | *Como?* | Chamadas funcionais do compilador, sigilos de sistema (`PREFIXO([ ... ])`), checkboxes (`(_)`, `(#)`) e comandos. |

#### 2.2.1 Princípio de Imunidade da Prosa Comum
- **Colchetes e Chaves isolados** em parágrafos comuns não capturam texto sem que haja uma composição explícita (`[[...]]`, `{[...]}` ou atalhos léxicos como `[#id]`, `[^ref]`).
- **Parênteses comuns (`(texto)`)** pertencem 100% à linguagem natural humana, avaliados como texto literal puro em $O(1)$. Eles só ativam o compilador quando precedidos por um sigilo operacional abraçando caixas (`PREFIXO([ ... ])`), em controles de formulário (`(_)`, `(#)`) ou em definições de rodapé isoladas em linha dedicada (`([ ... ])`). Expressões matemáticas (`(x + y) * 2`) e comentários parentéticos nunca são capturados.

#### 2.2.2 A Composição das Caixas
A partir dessa trindade fundamental, as estruturas avançadas derivam de composições previsíveis, com a regra universal de que **a URL ou endereço de rede/disco é SEMPRE o último argumento**:
1. **`[[ ... ]]` Navegação & Links:** Caixas de identidade dobradas (Aridade 1 a 3: `[[Destino]]`, `[[Rótulo][Destino]]`, `[[Rótulo][Tooltip][Destino]]`).
2. **`{[ ... ]}` Recursos de Mídia:** Caixa de recurso abraçando caixas de identidade (Aridade 1 a 3: `{[Origem]}`, `{[Alt][Origem]}`, `{[Alt][Legenda][Origem]}`).
3. **`([ ... ])` Metadados de Referência:** Caixa operacional abraçando caixas de identidade em linha dedicada (Aridade 2 a 5: `URL` sempre no fechamento).

### 2.3 Hiperlinks e Navegação Inline

| Elemento | Sintaxe LambdaType | Regra de Reconhecimento | Emissão Semântica (HTML) |
| :--- | :--- | :--- | :--- |
| **Salto de Âncora (Compacto)** | `[#id]` | Regra Léxica: 1 Caixa com `#` prefixado | `<a href="#id" class="anchor-link">#id</a>` |
| **Autolink** | `[[https://lambdatype.org]]` | Regra Sintática: Aridade 1 `[[Destino]]` | `<a href="https://lambdatype.org">https://lambdatype.org</a>` |
| **Link Canônico** | `[[Texto][https://lambdatype.org]]` | Regra Sintática: Aridade 2 `[[Rótulo][Destino]]` | `<a href="https://lambdatype.org">Texto</a>` |
| **Link com Tooltip** | `[[Texto][Dica][https://lambdatype.org]]` | Regra Sintática: Aridade 3 `[[Rótulo][Tooltip][Destino]]` | `<a href="https://lambdatype.org" title="Dica">Texto</a>` |

### 2.4 Recursos de Mídia e Figuras

| Elemento | Sintaxe LambdaType | Regra de Reconhecimento | Emissão Semântica (HTML) |
| :--- | :--- | :--- | :--- |
| **Mídia Direta** | `{[foto.png]}` | Regra Sintática: Aridade 1 `{[Origem]}` | `<img src="foto.png" alt="">` |
| **Mídia com Texto Alternativo** | `{[Foto][foto.png]}` | Regra Sintática: Aridade 2 `{[Alt][Origem]}` | `<img src="foto.png" alt="Foto">` |
| **Figura com Legenda** | `{[Foto][Legenda][foto.png]}` | Regra Sintática: Aridade 3 `{[Alt][Legenda][Origem]}` | `<figure><img src="foto.png" alt="Foto"><figcaption>Legenda</figcaption></figure>` |

#### 2.4.1 Reconhecimento Automático de Tipo de Mídia (Sniffing por Extensão)
A extensão do arquivo no argumento de destino determina a emissão da tag HTML adequada:
- **Vídeo (`.mp4`, `.webm`, `.ogg`):** Emite `<video controls src="..."></video>` (ou envolto em `<figure>` com `<figcaption>` se Aridade 3).
- **Áudio (`.mp3`, `.wav`):** Emite `<audio controls src="..."></audio>`.
- **Imagem (Padrão: `.png`, `.jpg`, `.jpeg`, `.svg`, `.webp`, etc.):** Emite `<img>`.

### 2.5 Notas de Rodapé e Referências Bibliográficas

As notas e citações utilizam a chamada inline compacta `[^ref]` no parágrafo e o bloco de definição `([ ... ])` obrigatoriamente isolado em linha dedicada:

| Elemento | Sintaxe LambdaType | Regra de Reconhecimento | Emissão Semântica (HTML no Rodapé) |
| :--- | :--- | :--- | :--- |
| **Chamada de Rodapé (Inline)** | `[^ref]` | Regra Léxica: 1 Caixa com `^` prefixado | `<sup id="fnref:ref"><a href="#fn:ref" class="footnote" rel="footnote" role="doc-noteref">N</a></sup>` |
| **Nota de Rodapé Simples** | `([ref][Título])` | Linha dedicada: Aridade 2 `([Ref][Título])` | `<li id="fn:ref"><p><cite>Título</cite> <a ...>↩</a></p></li>` |
| **Link Bibliográfico** | `([ref][Título][URL])` | Linha dedicada: Aridade 3 `([Ref][Título][URL])` | `<li id="fn:ref"><p><cite>Título</cite>. <a href="URL" class="footnote-url">URL</a> <a ...>↩</a></p></li>` |
| **Citação com Autoria** | `([ref][Título][Autor][URL])` | Linha dedicada: Aridade 4 `([Ref][Título][Autor][URL])` | `<li id="fn:ref"><p><cite>Título</cite> — <span class="footnote-author">Autor</span>. <a href="URL" class="footnote-url">URL</a> <a ...>↩</a></p></li>` |
| **Citação Completa** | `([ref][Título][Autor][Obs][URL])` | Linha dedicada: Aridade 5 `([Ref][Título][Autor][Obs][URL])` | `<li id="fn:ref"><p><cite>Título</cite> — <span class="footnote-author">Autor</span> (<em>Obs</em>). <a href="URL" class="footnote-url">URL</a> <a ...>↩</a></p></li>` |

> **Invariante de Linha Dedicada:** Definições `([ref][...])` devem ocupar sozinhas a sua respectiva linha física. Se declaradas inline no meio de uma frase, degradam integralmente para texto literal comum e emitem um `Warning` educativo no LSP.  
> **Numeração Sequencial ($N$):** O valor emitido no `<sup>` da chamada `[^ref]` é um número inteiro sequencial auto-incrementado ($1, 2, 3...$) baseado na ordem de aparição no documento.

#### 2.5.1 Estrutura Semântica HTML Emitida (Padrão W3C DPUB-ARIA)
O compilador consolida automaticamente as definições no rodapé do documento dentro de um container com papéis de acessibilidade e classes semânticas:

```html
<div class="footnotes" role="doc-endnotes">
  <ol>
    <li id="fn:1">
      <p><cite>Título da Obra</cite> — <span class="footnote-author">Nome do Autor</span> (<em>Observações</em>). <a href="https://..." class="footnote-url">https://...</a> <a href="#fnref:1" class="reversefootnote" role="doc-backlink">↩</a></p>
    </li>
    <li id="fn:44">
      <p><cite>Outra Referência</cite> <a href="#fnref:44" class="reversefootnote" role="doc-backlink">↩<sup>1</sup></a>&nbsp;<a href="#fnref:44:1" class="reversefootnote" role="doc-backlink">↩<sup>2</sup></a></p>
    </li>
  </ol>
</div>
```

> **Múltiplos Backlinks:** Se uma mesma nota `[^ref]` for citada mais de uma vez no texto, o item no rodapé gera links reversos numerados em sobrescrito (`↩<sup>1</sup>`, `↩<sup>2</sup>`).

### 2.6 Entidades Tipográficas e Símbolos

| Elemento | Sintaxe LambdaType | Regra de Reconhecimento | Emissão Semântica (HTML) |
| :--- | :--- | :--- | :--- |
| **Travessão (Em-dash)** | `---` | Regra Léxica: Vizinhança simétrica | `&mdash;` (—) |
| **Intervalo (En-dash)** | `--` | Regra Léxica: Vizinhança simétrica | `&ndash;` (–) |
| **Reticências Tipográficas** | `...` | Regra Léxica: 3 pontos contíguos | `&hellip;` (…) |
| **Seta Direita** | `->` | Regra Léxica: Vizinhança simétrica | `&rarr;` (→) |
| **Seta Esquerda** | `<-` | Regra Léxica: Vizinhança simétrica | `&larr;` (←) |
| **Bicondicional** | `<->` | Regra Léxica: Vizinhança simétrica | `&harr;` (↔) |
| **Implicação** | `=>` | Regra Léxica: Vizinhança simétrica | `&rArr;` (⇒) |

### 2.7 Controles de Formulário Inline

| Elemento | Sintaxe LambdaType | Regra de Reconhecimento | Emissão Semântica (HTML) |
| :--- | :--- | :--- | :--- |
| **Checkbox Desmarcado** | `(_)` | Regra Léxica: Sequência atômica de 3 caracteres | `<input type="checkbox" disabled>` |
| **Checkbox Marcado** | `(#)` | Regra Léxica: Sequência atômica de 3 caracteres | `<input type="checkbox" checked disabled>` |

### 2.8 Controle de Linha, Escape e Operadores

| Elemento | Sintaxe LambdaType | Regra de Reconhecimento | Emissão Semântica (HTML) |
| :--- | :--- | :--- | :--- |
| **Continuação Suave** | `\n` | Linha física simples no mesmo parágrafo | Concatena texto no mesmo nó com espaço |
| **Quebra Forçada (Hard Break)** | `\` + `\n` | Barra invertida no final da linha física | `<br>` |
| **Quebra de Bloco** | `\n\n` | Regra Léxica: Linha física em branco | Encerra container e abre nova tag de bloco |
| **Escape de Caractere** | `\C` | Regra Léxica: Barra invertida prefixando caractere | Emite caractere literal puro |
| **Cola Léxica (Concat)** | ` . ` | Regra Léxica: Ponto com espaço estrito bilateral | Remove operador e funde nós adjacentes |

### 2.9 Comentários no Código (`/* ... */`)

O LambdaType suporta comentários de bloco para anotações do autor, to-dos e rascunhos, completamente excluídos do pipeline de renderização (zero nós na AST, zero bytes no HTML):

#### 2.9.1 Comentário de Bloco (`/* ... */`)
- **Delimitação:** Inicia compulsoriamente com `/*` e consome todos os bytes (inclusive múltiplas linhas) até o correspondente `*/`.
- **Uso Inline ou Multilinha:** Pode ser inserido no meio de frases ou cobrir múltiplos parágrafos.
- **Zona de Supressão:** O conteúdo interno é completamente descartado no lexing.
- **Tolerância a Arquivo Incompleto (Lossless EOF Fallback):** Caso o documento termine sem o delimitador `*/` de fechamento, o compilador adota a política de perda zero (*Lossless*): **não descarta o restante do documento**. Em vez disso, degrada o token `/*` para texto literal comum e emite compulsoriamente um `Warning` educativo no LSP:
  > `[Warning L25:C10] Comentário '/*' não finalizado antes do fim do arquivo. Conteúdo preservado como texto literal.`

```text
O projeto LambdaType /* nota interna: revisar benchmark */ foca em previsibilidade.
```

*HTML Resultante:*
```html
<p>O projeto LambdaType foca em previsibilidade.</p>
```

> **Eliminação de Comentários de Linha (`//`):** Para garantir imunidade matemática e absoluta contra colisões com URLs (`https://`), caminhos de rede UNC (`//server/share`), CDNs e código embutido, o LambdaType não utiliza `//`. Apenas `/* ... */` é reconhecido.

### 2.10 Atributos e Classes Inline (`.([ ... ])`)

Permite injetar classes CSS, identificadores (`id`) e atributos semânticos arbitrários diretamente em nós inlines, preservando a imutabilidade da prosa comum:

#### 2.10.1 Posição e Assinatura Canônica
- **Posição Obrigatória:** Deve estar colado imediatamente após o delimitador de fechamento de um elemento inline válido (ex.: `**`, `~~`, `]]`, `{[...]}`, ` `` `).
- **Assinatura:** Utiliza o sigilo `.` colado à Caixa Operacional abraçando Caixas de Identidade estruturadas: `.([.classe1 .classe2][#id][attr=valor])`.

```text
Acesse o [[Painel Principal][/app]].([.btn .btn-primary]) para iniciar.
Este termo é **crítico**.([.alerta-texto][#aviso-seguranca]).
Consulte a função ``sys_init()``.([#func-init]).
```

*HTML Resultante:*
```html
<p>Acesse o <a href="/app" class="btn btn-primary">Painel Principal</a> para iniciar.<br>
Este termo é <strong class="alerta-texto" id="aviso-seguranca">crítico</strong>.<br>
Consulte a função <code id="func-init">sys_init()</code>.</p>
```

#### 2.10.2 Invariante de Borda e Degradação Graciosa
- **Precedência de Elemento Inline:** Se o `.([ ... ])` for precedido por texto comum da linguagem natural (ex.: `apartamento.([#casa])` ou `etc.([.nota])`), a ausência de um fechamento inline imediatamente anterior faz com que o token degrade integralmente para texto literal puro em $O(1)$.
- **Diagnóstico Educativo:** Dispara um `Warning` no LSP alertando que atributos inline exigem um elemento formatado antecedente.

### 2.11 Sanitização Universal e Injeção de Código Cru Inline (`=N( ... )N=`)

O LambdaType adota o princípio de segurança por padrão (*safe-by-default*), garantindo que todo texto inserido flua como conteúdo seguro sem risco de injeções arbitrárias no DOM:

#### 2.11.1 O Quinteto de Sanitização Segura
Qualquer texto de prosa comum tem os seguintes 5 caracteres automaticamente escapados para suas entidades HTML correspondentes:

| Caractere Reservado | Entidade HTML Emitida | Motivo de Proteção |
| :---: | :---: | :--- |
| **`&`** | `&amp;` | Impede injeção acidental ou truncamento de entidades |
| **`<`** | `&lt;` | Impede abertura de tags HTML em texto comum |
| **`>`** | `&gt;` | Impede fechamento antecipado de tags ou atributos |
| **`"`** | `&quot;` | Evita quebra de atributos envolvidos por aspas duplas |
| **`'`** | `&#39;` | Evita quebra de atributos envolvidos por aspas simples |

> **Imunidade de Diacríticos e UTF-8:** Letras acentuadas (`ç`, `ã`, `é`, etc.) e caracteres internacionais pertencem nativamente ao padrão UTF-8 e **nunca são convertidos para entidades**, garantindo máxima velocidade de compilação, SEO e menor tamanho de arquivo.

#### 2.11.2 Injeção de Código Cru Inline (`=N( ... )N=`)
Quando o autor precisa deliberadamente injetar um fragmento de código cru no meio da prosa sem que o quinteto seja sanitizado:
- **Assinatura Flexível contra Colisão:** `=N( conteúdo )N=` com $N \ge 1$ (ex.: `=( ... )=`, `==( ... )==`, `===( ... )===`).
- **Resolução de Fencing Breakout:** A exigência de correspondência exata no fechamento ($N_{abertura} = N_{fechamento}$) permite injetar códigos contendo sequências arbitrárias de `]` ou `)` (como arrays JavaScript `foo([1, 2])`) sem terminação prematura, bastando expandir $N$ (ex.: `==( let x = [10]; )==`).
- **Comportamento Direto:** No modo confiável (`Trusted`), despeja os bytes internos diretamente no fluxo de saída sem escapar o quinteto OWASP.
- **Fechamento Obrigatório na Mesma Linha:** A injeção inline respeita a Barreira Dura de Linha: não cruza `\n`. Se não for fechada antes do fim da linha, degrada para texto literal.

```text
Aqui temos um caso de =(<mark class="custom">destaque cru</mark>)= no texto.
Código JS inline cru: ==( if (a[0] && b) alert(1); )== continua normalmente.
```

*HTML Resultante (em CSC Trusted):*
```html
<p>Aqui temos um caso de <mark class="custom">destaque cru</mark> no texto.<br>
Código JS inline cru: if (a[0] && b) alert(1); continua normalmente.</p>
```

---

## Parte 3: Estruturas de Bloco (Containers)

Diferente dos elementos inline, os blocos são estruturas multilinhas que delimitam containers com regras de isolamento, continuação e profundidade.

### 3.1 Listas e Coleções Estruturadas
O LambdaType possui três formas canônicas de listas, todas governadas estritamente pela tabulação (`\t`):

#### 3.1.1 Lista Não-Ordenada
Inicia com hífen seguido compulsoriamente de espaço (`- `):
```text
- Primeiro item da lista
- Segundo item da lista
```

*HTML Resultante:*
```html
<ul>
  <li>Primeiro item da lista</li>
  <li>Segundo item da lista</li>
</ul>
```

#### 3.1.2 Lista Ordenada
Inicia com dígitos numéricos seguidos de ponto e espaço (`1. `):
```text
1. Primeiro passo da rotina
2. Segundo passo da rotina
```

*HTML Resultante:*
```html
<ol>
  <li>Primeiro passo da rotina</li>
  <li>Segundo passo da rotina</li>
</ol>
```

> **Numeração Inicial Arbitrária (`start="N"`):** Se uma lista ordenada iniciar com um número $N > 1$ (ex.: `5. Quinto item`), o compilador preserva o valor ordinal emitindo `<ol start="5">`.

#### 3.1.3 Lista de Definição
Inicia com hífen colado imediatamente à Caixa de Identidade `[termo]`, seguido de espaço e a definição textual:
```text
-[API] Interface de Programação de Aplicação.
-[DOM] Modelo de Objeto de Documento para manipulação web.
```

*HTML Resultante:*
```html
<dl>
  <dt>API</dt>
  <dd>Interface de Programação de Aplicação.</dd>
  <dt>DOM</dt>
  <dd>Modelo de Objeto de Documento para manipulação web.</dd>
</dl>
```

> **Transição Imediata de Marcador:** Mudar o tipo de lista (ex.: de `- ` para `1. `) encerra o container anterior e inicia o novo imediatamente, sem exigir linha em branco intermediária.

#### 3.1.4 Listas de Tarefas (Task Lists / Checklists)
Composição ortogonal dos marcadores de lista (`- `) com os controles de formulário inline (`(_)` e `(#)` da seção `2.7`):

```text
- (_) Implementar analisador léxico
- (#) Definir especificação da gramática
```

*HTML Resultante:*
```html
<ul class="task-list">
  <li><input type="checkbox" disabled> Implementar analisador léxico</li>
  <li><input type="checkbox" checked disabled> Definir especificação da gramática</li>
</ul>
```

#### 3.1.5 Múltiplos Parágrafos no Mesmo Item (Indentação por `\t`)
Para estender um item de lista com múltiplos parágrafos, as linhas subsequentes devem ser indentadas com `\t` (Eixo 1 da seção `5.10`):

```text
1. Primeiro passo da rotina:
	Este parágrafo continua dentro do primeiro passo porque está indentado com tabulação.
2. Segundo passo da rotina.
```

*HTML Resultante:*
```html
<ol>
  <li>
    <p>Primeiro passo da rotina:</p>
    <p>Este parágrafo continua dentro do primeiro passo porque está indentado com tabulação.</p>
  </li>
  <li>
    <p>Segundo passo da rotina.</p>
  </li>
</ol>
```

---

### 3.2 Texto Pré-Formatado e Cercas de Código (Code Fences)

#### 3.2.1 Texto Pré-Formatado Genérico
Delimitado por 3 ou mais crases isoladas em sua respectiva linha:
````text
```
Texto exibido exatamente como digitado,
com espaçamentos e quebras preservadas.
```
````

*HTML Resultante:*
```html
<pre><code>Texto exibido exatamente como digitado,
com espaçamentos e quebras preservadas.</code></pre>
```

#### 3.2.2 Cerca de Código com Linguagem (Syntax Highlighting)
Delimitada pela Caixa de Identidade `[lang]` colada às crases de abertura:
````text
```[rust]
fn main() {
    println!("Olá, LambdaType!");
}
```
````

*HTML Resultante:*
```html
<pre><code class="language-rust">fn main() {
    println!("Olá, LambdaType!");
}</code></pre>
```

#### 3.2.3 Embutimento de Cercas (Cercas de Vários Tamanhos)
Para documentar um trecho que já contenha cercas de código internas (ex.: um arquivo `.lt` demonstrando blocos de 3 crases), basta abrir o bloco externo com uma quantidade maior de crases (ex.: 4 crases ````[md]````):

`````text
````[markdown]
Este é um exemplo de LambdaType demonstrando código:

```rust
fn main() {
    println!("Olá, LambdaType!");
}
```

O bloco externo só encerra ao encontrar exatamente 4 crases isoladas:
````
`````

*HTML Resultante:*
```html
<pre><code class="language-markdown">Este é um exemplo de LambdaType demonstrando código:

```rust
fn main() {
    println!("Olá, LambdaType!");
}
```

O bloco externo só encerra ao encontrar exatamente 4 crases isoladas:</code></pre>
```

* **Invariante da Linha Isolada:** As cercas de abertura e de fechamento devem ocupar sua respectiva linha inteira sozinhas.
* **Zona de Exclusão Absoluta:** O conteúdo interno da cerca é imune a todas as regras léxicas, inlines, sigilos e escapes por `\`.
* **Correspondência Exata de Fechamento:** A cerca de abertura pode ter comprimento arbitrário a partir do mínimo de 3 crases ($N \ge 3$), com ou sem a Caixa de Identidade `[lang]`. Para encerrar o bloco, a cerca de fechamento deve conter **exatamente a mesma quantidade de crases da abertura** ($M = N$).
* **Regra do Envelope Estrito ($N_{externo} > N_{interno}$):** Para embutir cercas de código como conteúdo literal sem disparos acidentais de fechamento (*mishaps*), a cerca externa **DEVE ser estritamente maior** que qualquer cerca contida em seu interior ($N_{externo} \ge N_{interno} + 1$).
* **Tolerância a Arquivo Incompleto (Auto-Close em EOF):** Se o documento terminar sem a cerca de fechamento correspondente, o compilador fecha compulsoriamente a cerca no fim do arquivo (EOF), preservando todo o código capturado sem corromper a árvore AST, e emite um `Warning` educativo no LSP:
  > `[Warning L40:C1] Cerca de código não finalizada antes do fim do arquivo. Auto-fechada em EOF.`

---

### 3.3 Citações em Bloco

Utilizadas para citar fontes externas, falas ou passagens textuais.

#### 3.3.1 Citação Simples
Toda linha pertencente à citação exige o prefixo `> ` (maior que seguido de espaço):
```text
> A simplicidade é o último grau de sofisticação.
> Leonardo da Vinci
```

*HTML Resultante:*
```html
<blockquote>
  <p>A simplicidade é o último grau de sofisticação.<br>
  Leonardo da Vinci</p>
</blockquote>
```

#### 3.3.2 Aninhamento de Citações
Citações internas são declaradas pela repetição do símbolo separado por espaço:
```text
> Citação de nível 1 externa.
> > Citação de nível 2 aninhada internamente.
```

*HTML Resultante:*
```html
<blockquote>
  <p>Citação de nível 1 externa.</p>
  <blockquote>
    <p>Citação de nível 2 aninhada internamente.</p>
  </blockquote>
</blockquote>
```

#### 3.3.3 Continuação e Parágrafos Internos
- **Múltiplos Parágrafos na Citação:** Uma linha contendo apenas `>` (ou `> ` sem texto) encerra o parágrafo atual e abre um novo parágrafo dentro da mesma citação:
  ```text
  > Primeiro parágrafo da citação.
  >
  > Segundo parágrafo dentro do mesmo bloco citado.
  ```

  *HTML Resultante:*
  ```html
  <blockquote>
    <p>Primeiro parágrafo da citação.</p>
    <p>Segundo parágrafo dentro do mesmo bloco citado.</p>
  </blockquote>
  ```
- **Quebra Forçada na Citação (`\`):** Por padrão, linhas físicas consecutivas continuam o mesmo parágrafo fluidamente com espaço. Para forçar uma quebra de linha visual (`<br>`) dentro da citação, utiliza-se a barra invertida `\` no final da linha física:
  ```text
  > Primeira linha do verso \
  > Segunda linha logo abaixo com quebra visual.
  ```
- **Invariante de Não-Preguiça (No-Lazy-Continuation):** Linhas sem o prefixo `> ` encerram imediatamente a citação.

---

### 3.4 Avisos Tipados (Admonitions)

Os Admonitions são caixas de destaque especializadas para alertas, avisos e notas. 

#### 3.4.1 O Quinteto Canônico de Abertura
A abertura exige o caractere `<` colado à Caixa de Identidade contendo exatamente um dos **5 símbolos canônicos**, seguido de espaço e título opcional:

| Sintaxe de Abertura | Tipo Semântico | Emissão Semântica (HTML e ARIA) |
| :--- | :---: | :--- |
| `<[-] Título` | **NOTE** | `<div class="admonition admonition-note" role="note">` |
| `<[#] Título` | **TIP** | `<div class="admonition admonition-tip" role="note">` |
| `<[?] Título` | **IMPORTANT** | `<div class="admonition admonition-important" role="alert">` |
| `<[!] Título` | **WARNING** | `<div class="admonition admonition-warning" role="alert">` |
| `<[+] Título` | **CAUTION** | `<div class="admonition admonition-caution" role="alert">` |

> **Título Opcional:** Se o título for omitido (ex.: `<[!]` isolado na primeira linha), o bloco é renderizado normalmente sem a emissão de uma tag `<p class="admonition-title">` vazia.

#### 3.4.2 Regras de Corpo e Continuação Estrita
- **Prefixo Obrigatório de Corpo:** Todas as linhas subsequentes do corpo devem começar compulsoriamente com o prefixo `< ` (menor que seguido de espaço):
  ```text
  <[!] Cuidado com a Operação
  < Este procedimento recria as tabelas do banco de dados.
  < Certifique-se de realizar o backup antes de confirmar.
  ```

  *HTML Resultante:*
  ```html
  <div class="admonition admonition-warning" role="alert">
    <p class="admonition-title">Cuidado com a Operação</p>
    <p>Este procedimento recria as tabelas do banco de dados. Certifique-se de realizar o backup antes de confirmar.</p>
  </div>
  ```
- **Múltiplos Parágrafos no Admonition:** Uma linha contendo apenas `<` (ou `< ` sem texto) encerra o parágrafo atual e abre um novo parágrafo dentro do mesmo alerta:
  ```text
  <[#] Dica de Otimização
  < Primeiro parágrafo com as instruções básicas.
  <
  < Segundo parágrafo detalhando o ganho de performance.
  ```
- **Quebra Forçada no Admonition (`\`):** Para forçar uma quebra de linha visual (`<br>`) no corpo do aviso, utiliza-se a barra invertida `\` no final da linha física:
  ```text
  <[-] Informação Relevante
  < Linha com quebra forçada \
  < Linha seguinte logo abaixo.
  ```

#### 3.4.3 Composição, Contenção e Aninhamento de Admonitions
Em total conformidade com o Modelo Dual de Hierarquia (`5.10`), os Admonitions participam universalmente do sistema de contenção:
- **Contenção Passiva (Admonitions dentro de outros blocos):** Podem ser aninhados como filhos de itens de lista (`\t<[!]`), citações (`> <[!]`) e blocos expansíveis (`: <[!]`):
  ```text
  1. Execute o comando de compilação:
  	```[bash]
  	cargo build --release
  	```
  	<[!] Atenção ao Ambiente
  	< Certifique-se de configurar as variáveis de produção.
  2. Prossiga para o próximo passo.
  ```

  *HTML Resultante:*
  ```html
  <ol>
    <li>
      <p>Execute o comando de compilação:</p>
      <pre><code class="language-bash">cargo build --release</code></pre>
      <div class="admonition admonition-warning" role="alert">
        <p class="admonition-title">Atenção ao Ambiente</p>
        <p>Certifique-se de configurar as variáveis de produção.</p>
      </div>
    </li>
    <li>
      <p>Prossiga para o próximo passo.</p>
    </li>
  </ol>
  ```
- **Contenção Ativa (Blocos dentro do Admonition):** O corpo do admonition pode conter listas, citações, tabelas e cercas de código, bastando prefixá-los com `< `:
  ```text
  <[!] Checklist Obrigatório
  < - Verificar credenciais de acesso
  < - Confirmar variável de ambiente
  < - Executar testes de integração
  ```

  *HTML Resultante:*
  ```html
  <div class="admonition admonition-warning" role="alert">
    <p class="admonition-title">Checklist Obrigatório</p>
    <ul>
      <li>Verificar credenciais de acesso</li>
      <li>Confirmar variável de ambiente</li>
      <li>Executar testes de integração</li>
    </ul>
  </div>
  ```
- **Encerramento:** Qualquer linha que não inicie com `< ` (ou com a cadeia de prefixos do container pai) encerra imediatamente o admonition.

---

### 3.5 Tabelas Matriciais com Vetores de Absorção

As tabelas no LambdaType operam com delimitação por barras verticais (`|`), suportando legendas canônicas, separadores explícitos de seções (`<thead>`, `<tbody>`, `<tfoot>`), e fusão bidimensional de células via vetores de absorção unidirecionais (`<` e `^`).

#### 3.5.1 Tabela Básica com Alinhamento
O alinhamento é definido na linha divisória com hífen (`-`) através da posição dos dois-pontos (`:`), emitindo classes CSS semânticas:

```text
| Nome | Preço | Quantidade |
| :--- | :---: | ---: |
| Maçã | 3.50 | 10 |
| Pera | 4.20 | 5 |
```

*HTML Resultante:*
```html
<table>
  <thead>
    <tr>
      <th class="align-left">Nome</th>
      <th class="align-center">Preço</th>
      <th class="align-right">Quantidade</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="align-left">Maçã</td>
      <td class="align-center">3.50</td>
      <td class="align-right">10</td>
    </tr>
    <tr>
      <td class="align-left">Pera</td>
      <td class="align-center">4.20</td>
      <td class="align-right">5</td>
    </tr>
  </tbody>
</table>
```

#### 3.5.2 Legenda de Tabela (`|[ ... ]|`)
Se a primeira linha da tabela for uma Caixa de Identidade delimitada por barras verticais `|[ Legenda ]|`, ela é compilada como legenda semântica `<caption>`:

```text
|[ Tabela de Frutas em Estoque ]|
| Item | Quantidade |
| :--- | ---: |
| Uva  | 20 |
```

*HTML Resultante:*
```html
<table>
  <caption>Tabela de Frutas em Estoque</caption>
  <thead>
    <tr>
      <th class="align-left">Item</th>
      <th class="align-right">Quantidade</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="align-left">Uva</td>
      <td class="align-right">20</td>
    </tr>
  </tbody>
</table>
```

#### 3.5.3 Rodapé de Tabela (`<tfoot>`) via Divisória de Igualdade (`===`)
Linhas de totalização, somatórios ou notas de rodapé de tabela são precedidas pela linha divisória com sinal de igual (`===`), que fecha o `<tbody>` e abre o `<tfoot>`:

```text
|[ Tabela de Produtos ]|
| Produto | Quantidade | Preço |
| :--- | :---: | ---: |
| Maçã | 10 | 3.50 |
| Pera | 5 | 4.20 |
| :=== | :===: | ===: |
| Total Geral | 15 | 7.70 |
```

*HTML Resultante:*
```html
<table>
  <caption>Tabela de Produtos</caption>
  <thead>
    <tr>
      <th class="align-left">Produto</th>
      <th class="align-center">Quantidade</th>
      <th class="align-right">Preço</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="align-left">Maçã</td>
      <td class="align-center">10</td>
      <td class="align-right">3.50</td>
    </tr>
    <tr>
      <td class="align-left">Pera</td>
      <td class="align-center">5</td>
      <td class="align-right">4.20</td>
    </tr>
  </tbody>
  <tfoot>
    <tr>
      <td class="align-left">Total Geral</td>
      <td class="align-center">15</td>
      <td class="align-right">7.70</td>
    </tr>
  </tfoot>
</table>
```

#### 3.5.4 Vetor de Absorção Horizontal — Colspan (`<`)
Uma célula contendo unicamente `<` aponta para a célula à sua esquerda na mesma linha física, fundindo-se a ela:

```text
| Item e Descrição Completa | < |
| :--- | :--- |
| Maçã Fuji | Doce e crocante |
```

*HTML Resultante:*
```html
<table>
  <thead>
    <tr>
      <th colspan="2" class="align-left">Item e Descrição Completa</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="align-left">Maçã Fuji</td>
      <td class="align-left">Doce e crocante</td>
    </tr>
  </tbody>
</table>
```

#### 3.5.5 Vetor de Absorção Vertical — Rowspan (`^`)
Uma célula contendo unicamente `^` aponta para a célula imediatamente acima na mesma coluna, fundindo-se a ela:

```text
| Categoria | Produto |
| :--- | :--- |
| Fruta | Laranja |
| ^ | Limão |
```

*HTML Resultante:*
```html
<table>
  <thead>
    <tr>
      <th class="align-left">Categoria</th>
      <th class="align-left">Produto</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="2" class="align-left">Fruta</td>
      <td class="align-left">Laranja</td>
    </tr>
    <tr>
      <td class="align-left">Limão</td>
    </tr>
  </tbody>
</table>
```

#### 3.5.6 Absorção Bidimensional: Modelo da Caixa Delimitadora da Âncora (Anchor-Driven Bounding Box)
A fusão de células em blocos bidimensionais retangulares ($W \times H$) segue a regra determinística da **Decomposição Ortogonal do Bloco Mestre**:

1. **Célula Âncora (Raiz):** A célula superior esquerda do bloco fundido é a raiz de conteúdo.
2. **Cálculo da Largura ($W$ - Colspan):** Determinada estritamente pela contagem de marcadores `<` contíguos na linha da raiz (ou apontando para ela).
3. **Cálculo da Altura ($H$ - Rowspan):** Determinada estritamente pela contagem de marcadores `^` contíguos na coluna da raiz (ou apontando para ela).
4. **Equivalência Semântica das Células Interiores:** As células internas ao retângulo delimitador $[W \times H]$ que não estejam na linha ou coluna da raiz são células de preenchimento meramente cosmético/visual. Elas podem conter indiferentemente `<` ou `^` (ambos são válidos e silenciosos, sem gerar warnings), pois a geometria já está unicamente fixada pelos eixos ortogonais da raiz.

```text
| Bloco 2x2 | < | Terceira Coluna |
| ^         | < | Dado C          |
```

*Equivalente a:*
```text
| Bloco 2x2 | < | Terceira Coluna |
| ^         | ^ | Dado C          |
```

*HTML Resultante:*
```html
<table>
  <tbody>
    <tr>
      <td colspan="2" rowspan="2" class="align-left">Bloco 2x2</td>
      <td class="align-left">Terceira Coluna</td>
    </tr>
    <tr>
      <td class="align-left">Dado C</td>
    </tr>
  </tbody>
</table>
```

#### 3.5.7 Invariantes Estruturais, Fronteiras e Degradação de Vetores
Para garantir conformidade com a especificação HTML5 do W3C e manter parsing $O(n)$ sem ciclos:
- **Vetor `<` na Coluna 0:** Se a primeira célula de uma linha contiver `<` (sem célula à esquerda), o vetor não possui âncora; degrada para caractere literal puro em $O(1)$ e dispara compulsoriamente um `Warning` no LSP:
  > `[Warning L10:C1] Vetor de absorção horizontal '<' na primeira coluna não possui célula à esquerda. Degradado para literal.`
- **Vetor `^` na Linha 0:** Se uma célula na primeira linha da tabela (ou primeira linha do `<tbody>`) contiver `^` (sem célula acima), degrada para caractere literal e emite `Warning` no LSP:
  > `[Warning L10:C3] Vetor de absorção vertical '^' no topo da seção não possui célula acima. Degradado para literal.`
- **Proibição de Cruzamento de Fronteiras Estruturais:** Conforme a especificação W3C HTML5, células de tabela não podem atravessar fronteiras entre seções (`<thead>`, `<tbody>`, `<tfoot>`). Se um vetor `^` tentar absorver uma célula pertencente a outra seção (ex.: `^` na primeira linha do `<tbody>` apontando para o `<thead>`), o vetor é truncado, degradando para texto literal com `Warning` no LSP.
- **Geometrias Não-Retangulares ("L-Shapes"):** O compilador do LambdaType aceita exclusivamente matrizes retangulares de fusão. Qualquer vetor isolado que forme figuras côncavas ou em formato de "L" fora do bounding box da âncora dispara um `Warning` no LSP e as células anômalas degradam graciosamente para texto literal.

#### 3.5.8 Definição e Agrupamento de Colunas (`<colgroup>` e `<col span>`)
A linha de colunas `|{ ... }|` posiciona-se logo após a legenda e antes do cabeçalho, permitindo nomear classes de coluna, estender colunas via vetor `<` (`span`), e separar grupos lógicos via pipe duplo `||`:

```text
|[ Tabela de Produtos ]|
|{ c1 | c2 | < || c3 }|
| ISBN | Title | Author | Price |
| :--- | :---  | :---   | ---:  |
| 3476896 | Maria Joaquina | My first HTML | $53 |
| 5869207 | Ze da Silva | My first CSS | $49 |
| :=== | :===  | :===   | ===:  |
| **Total** | < | < | **$102** |
```

*HTML Resultante:*
```html
<table>
  <caption>Tabela de Produtos</caption>
  <colgroup>
    <col class="c1">
    <col class="c2" span="2">
  </colgroup>
  <colgroup>
    <col class="c3">
  </colgroup>
  <thead>
    <tr>
      <th class="align-left">ISBN</th>
      <th class="align-left">Title</th>
      <th class="align-left">Author</th>
      <th class="align-right">Price</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="align-left">3476896</td>
      <td class="align-left">Maria Joaquina</td>
      <td class="align-left">My first HTML</td>
      <td class="align-right">$53</td>
    </tr>
    <tr>
      <td class="align-left">5869207</td>
      <td class="align-left">Ze da Silva</td>
      <td class="align-left">My first CSS</td>
      <td class="align-right">$49</td>
    </tr>
  </tbody>
  <tfoot>
    <tr>
      <td colspan="3" class="align-left"><strong>Total</strong></td>
      <td class="align-right"><strong>$102</strong></td>
    </tr>
  </tfoot>
</table>
```

#### 3.5.9 Regras de Espaçamento, Degradação Graciosa e Diagnósticos (LSP Warnings)
Para preservar a clareza visual e o princípio *"Pure typography"*, as tabelas seguem convenções estritas de higiene léxica:

- **Espaçamento Canônico Obrigatório:** Toda célula de tabela deve conter compulsoriamente pelo menos **1 espaço** entre os delimitadores (`|` ou `||`) e o conteúdo (`| conteúdo |`).
- **Tolerância a Padding Cosmético e Alinhamento Visual:** O alinhamento visual de colunas via múltiplos espaços (ex.: `| ISBN      | Title          |`) é 100% transparente para o compilador. Qualquer quantidade de espaços adicionais além do mínimo obrigatório ($N_{\text{espaços}} \ge 1$) é tratada como padding cosmético e descartada na normalização (`trim()`), sem disparar nenhum aviso no LSP. Formatadores de código e extensões de IDE podem alinhar visualmente colunas livremente.
- **Divisórias Extensíveis ($N \ge 3$):** Linhas divisórias de cabeçalho (`-`) ou de rodapé (`=`) exigem no mínimo 3 caracteres repetidos, com ou sem colons de alinhamento (`| --- | :--- | ---: | :---: |`), aceitando qualquer comprimento para acompanhar a largura das colunas (ex.: `:--------` e `:========` têm paridade sintática idêntica a `:---` e `:===`). A ausência de colons (`---`) adota o alinhamento padrão à esquerda (`align-left`) silenciosamente.
- **Células Vazias:** Canonicamente, uma célula sem conteúdo deve conter pelo menos **1 espaço** (`| |` ou com padding `|           |`). Ambas são plenamente válidas e silenciosas.
- **Degradação de Espaços Faltantes:** A ausência de espaços (ex.: `|conteúdo|`) é tolerada pelo compilador via `trim()`, e a borda `|` atua como barreira lexical dura para permitir estilos casados (ex.: `|**negrito**|`), mas emite compulsoriamente um `Warning` educativo no LSP:
  > `[Warning L12:C1] Ausência de espaçamento canônico na célula '|conteúdo|'. Use '| conteúdo |' para manter a legibilidade.`
- **Pipes Consecutivos fora de Colgroup (`||`):** A ocorrência de barras coladas sem espaço (ex.: `||||` ou `| a || b |`) fora da linha de `<colgroup>` degrada graciosamente para células vazias, mas emite compulsoriamente um `Warning` no LSP alertando sobre ambiguidade com o operador de grupo:
  > `[Warning L15:C8] Uso ambíguo de '||' fora da declaração de colgroup. Para células vazias, use '| |'.`
- **Linhas Assimétricas e Auto-Padding:** Caso o autor cometa um descuido e escreva uma linha com menos células que a largura $C$ da tabela (ex.: 2 células numa tabela de 3 colunas), o compilador não quebra a estrutura: preenche automaticamente a linha com células vazias `<td></td>` até atingir $C$, emitindo um `Warning` educativo no LSP:
  > `[Warning L18:C1] Linha com quantidade insuficiente de células (esperado 3, encontrado 2). Célula vazia adicionada automaticamente.`
  Se uma linha contiver células excedentes além de $C$, o compilador emite as células extras no HTML e emite um `Warning` alertando sobre a assimetria da matriz.
- **Obrigatoriedade Estrita de Pipes de Borda (`| ... |`):** Toda linha de tabela DEVE obrigatoriamente iniciar com `|` e terminar com `|`. Linhas sem pipes nas extremidades (ex.: `Item | Preço` ou `a | b | c`) são expressamente rejeitadas como tabela pelo analisador e degradam diretamente para texto de parágrafo comum. Isso garante determinismo léxico $O(1)$ absoluto e imunidade total contra o sequestro predatório de expressões técnicas e comandos shell que utilizem pipes (ex.: `cat file | grep foo`).
- **Posicionamento de Metadados (`Caption` e `ColGroup`):** Metadados estruturais de tabela PRECISAM ocorrer estritamente antes do início dos dados (`<thead>` ou primeira linha física de dados). Qualquer ocorrência de `|[` ou `|{` após o início dos dados é interpretada como célula de dados comum degradada graciosamente, com disparo de `Warning` no LSP.
- **Ordem Canônica de Precedência:** A ordem canônica obrigatória no topo da tabela é `Caption` seguido de `ColGroup`:
  1. `|[ Legenda ]|`
  2. `|{ colgroup }|`
  3. Dados da tabela (`<thead>`, `<tbody>`, `<tfoot>`)
  - *Degradação por Ordem Invertida:* Declarar `ColGroup` antes de `Caption` compila com sucesso (o compilador reordena no HTML semântico com `<caption>` primeiro), mas emite compulsoriamente um `Warning` no LSP:
    > `[Warning L1:C1] Ordem invertida: 'ColGroup' declarado antes de 'Caption'. A ordem canônica recomendada é 'Caption' seguido de 'ColGroup'.`

#### 3.5.10 Quebra de Linha Intra-Célula (`\\`) e Desambiguação de Contrabarra
- **Natureza Estritamente Full-Inline:** Células de tabela são puramente inlines. É expressamente proibido embutir blocos estruturais (listas, cercas de código, citações, outras tabelas) no interior de células.
- **Quebra de Linha Visual Canônica (` \\ ` com Espaços):** Como o byte `\n` físico delimita rigorosamente as linhas de grade da tabela, a quebra de linha visual dentro de uma célula é expressa pela sequência canônica de dupla barra invertida delimitada compulsoriamente por espaço antes E depois (` \\ `), que compila para `<br>` no interior do nó da célula:
  ```text
  | Linha 1 \\ Linha 2 | Valor Único |
  ```
- **Degradação Elegante de `\\` sem Espaçamento (Suporte Nativo a Windows Paths):** Se a sequência `\\` **não** estiver cercada por espaços em ambos os lados (ex.: `C:\\arquivo.txt`, `\\servidor\share`, `chave\\valor`), ela **NÃO** atua como quebra de linha; ela degrada automaticamente para o escape canônico de contrabarra, emitindo uma única barra invertida literal `\` (produzindo `C:\arquivo.txt` e `\servidor\share`). Isso permite ao autor documentar livremente caminhos de arquivos, chaves de registro ou sintaxes com contrabarra dentro de tabelas sem quebrar a linha acidentalmente e sem exigir código inline ou quádrupla barra.
- **Escape de Pipe (`\|`):** Para inserir um caractere de barra vertical literal dentro do conteúdo de uma célula sem que o lexer o reconheça como divisor de coluna, utiliza-se o escape por barra invertida `\|`. Códigos inlines contendo pipes (ex.: `| ``a | b`` |`) têm seus pipes internos protegidos nativamente pelo lexer.

#### 3.5.11 Tabela sem Cabeçalho e Tabela Mínima (Headerless Tables)
Em formulários, pares de chave-valor ou matrizes puras de dados que não exigem cabeçalho formal (`<thead>`):

##### 1. Tabela sem Cabeçalho com Alinhamento (Divisória no Topo)
Se a linha divisória com hífens (`:---`) for declarada imediatamente antes da primeira linha de dados (após `caption` e `colgroup` opcionais), ela define o alinhamento das colunas mas **não emite `<thead>`**, enviando todas as linhas diretamente para o `<tbody>`:

```text
|[ Parâmetros de Configuração ]|
| :--- | ---: |
| Timeout | 5000ms |
| Retries | 3 |
```

*HTML Resultante:*
```html
<div class="table-container">
  <table>
    <caption>Parâmetros de Configuração</caption>
    <tbody>
      <tr>
        <td class="align-left">Timeout</td>
        <td class="align-right">5000ms</td>
      </tr>
      <tr>
        <td class="align-left">Retries</td>
        <td class="align-right">3</td>
      </tr>
    </tbody>
  </table>
</div>
```

##### 2. Tabela Mínima (Sem Divisória) e Desambiguação de Prosa
Se uma tabela for declarada sem nenhuma linha divisória de hífens (`:---`), todas as linhas físicas são compiladas diretamente no `<tbody>` e o alinhamento de todas as colunas assume o padrão à esquerda (`align-left`):

```text
| Chave 1 | Valor 1 |
| Chave 2 | Valor 2 |
```

*HTML Resultante:*
```html
<div class="table-container">
  <table>
    <tbody>
      <tr>
        <td class="align-left">Chave 1</td>
        <td class="align-left">Valor 1</td>
      </tr>
      <tr>
        <td class="align-left">Chave 2</td>
        <td class="align-left">Valor 2</td>
      </tr>
    </tbody>
  </table>
</div>
```

> **Invariante de Desambiguação com Prosa Técnica:** Para evitar a captura acidental de expressões com barras em texto comum (ex.: `escolha A | B | C`), uma tabela mínima sem divisória exige compulsoriamente no mínimo 2 linhas consecutivas com o mesmo número de colunas delimitadas por `|`, ou pipes de fechamento de linha (`| ... |`). Linhas isoladas contendo `|` sem divisória e sem múltiplas linhas correlacionadas degradam diretamente para texto de parágrafo comum.

#### 3.5.12 Mecânica de Renderização HTML/CSS de Tabelas Full-Inline
Para responder integralmente a como os motores de navegação web (Blink, Gecko, WebKit) processam as tabelas geradas pelo LambdaType:

1. **Quebra Automática Suave (*Word Wrapping*):**  
   Por padrão no HTML5, a tag `<td>` possui `white-space: normal`. Isso significa que **textos descritivos longos dentro de uma célula quebram automaticamente de linha** nos espaços em branco para se ajustar à largura da coluna. O autor técnico não precisa quebrar frases longas manualmente; basta escrever o texto contínuo na célula e o navegador realiza o wrap perfeito.
2. **Impacto do `<br>` (` \\ `) na Altura da Linha:**  
   Quando o autor usa ` \\ `, o `<br>` inserido força uma nova linha no contexto inline da célula. A altura da linha `<tr>` inteira expande-se automaticamente para acomodar a célula mais alta. Para manter harmonia visual profissional, o padrão de CSS recomendado para o LambdaType é `vertical-align: top;` em todas as células.
3. **Controle de Larguras e Performance (`table-layout`):**  
   - Em tabelas dinâmicas, os navegadores usam `table-layout: auto`, calculando a largura das colunas a partir do maior conteúdo inquebrável (*min-content*) e da frase completa (*max-content*).  
   - Com a declaração de `<colgroup>` do LambdaType (`|{ c1 | c2 | c3 }|`), o desenvolvedor pode ativar no CSS `table-layout: fixed;`, que determina as larguras instantaneamente em $O(1)$ a partir da primeira linha ou das classes de coluna, evitando repasses caros de renderização.
4. **Palavras Gigantes e URLs Inquebráveis:**  
   Para evitar que URLs longas ou identificadores sem espaços ultrapassem a largura da coluna no navegador, o CSS base do LambdaType deve sempre conter `overflow-wrap: break-word;`.
5. **Encapsulamento Padrão em Container Responsivo (`.table-container`):**  
   Por padrão na emissão HTML5, **toda tabela gerada pelo compilador LambdaType é encapsulada em um container** `<div class="table-container">\n  <table>...</table>\n</div>`. Isso fornece um gancho de estilo direto para que o CSS aplique `overflow-x: auto; -webkit-overflow-scrolling: touch;`, viabilizando rolagem horizontal tanto em smartphones quanto em monitores desktop exibindo tabelas com dezenas de colunas, preservando integralmente o layout da página.



---

### 3.6 Blocos Expansíveis (Details / Summary)

Os Blocos Expansíveis encapsulam seções colapsáveis nativas (`<details>` e `<summary>`) para respostas de exercícios, logs ou notas extensas:

#### 3.6.1 Assinaturas Canônicas de Abertura
A abertura exige o caractere `:` colado à Caixa de Identidade contendo compulsoriamente o sinal `-` (recolhido) ou `+` (expandido), seguido de espaço e o título do sumário:

| Sintaxe de Abertura | Estado Inicial | Emissão Semântica (HTML) |
| :--- | :---: | :--- |
| `:[-] Título` | **Recolhido / Fechado** | `<details><summary>Título</summary>` |
| `:[+] Título` | **Expandido / Aberto** | `<details open><summary>Título</summary>` |

> **Invariante do Sinal Obrigatório:** A presença de `[-]` ou `[+]` é **estritamente obrigatória**. Qualquer declaração sem o sinal (ex.: `:[Título]`) não é reconhecida como bloco expansível, degradando para texto literal com `Warning` no LSP.

#### 3.6.2 Regras de Linha e Prefixo de Corpo (`: `)
- **Natureza do `<summary>`:** O título do sumário é estritamente **inline** (aceita negrito, ênfase, código e links, mas não aceita blocos internos). Ele ocupa uma única linha física; se for longo, utiliza continuação suave via barra invertida (`\` + `\n`).
- **Prefixo Obrigatório de Corpo:** Todas as linhas subsequentes do corpo expansível devem iniciar compulsoriamente com o prefixo `: ` (dois-pontos seguido de espaço).
- **Múltiplos Parágrafos e Blocos Filhos:** Uma linha contendo apenas `:` (ou `: ` sem texto) encerra o parágrafo atual e abre um novo parágrafo. O corpo suporta listas, citações, tabelas e cercas de código devidamente prefixadas com `: `.

#### 3.6.3 Exemplos e Emissão

##### 1. Bloco Recolhido por Padrão (`:[-]`)
```text
:[-] Resposta do Exercício 1
: Este procedimento recria as tabelas do banco de dados.
: Certifique-se de realizar o backup antes de confirmar.
```

*HTML Resultante:*
```html
<details>
  <summary>Resposta do Exercício 1</summary>
  <p>Este procedimento recria as tabelas do banco de dados.<br>
  Certifique-se de realizar o backup antes de confirmar.</p>
</details>
```

##### 2. Bloco Expandido por Padrão (`:[+]`)
```text
:[+] Índice Detalhado
: 1. Introdução
: 2. Metodologia
: 3. Conclusão
```

*HTML Resultante:*
```html
<details open>
  <summary>Índice Detalhado</summary>
  <ol>
    <li>Introdução</li>
    <li>Metodologia</li>
    <li>Conclusão</li>
  </ol>
</details>
```

---

### 3.7 Divisor Temático (Thematic Break — `<hr>`)

Utilizado para representar uma separação temática ou mudança de cena em nível de parágrafo:

#### 3.7.1 Regra de Reconhecimento
- **Sintaxe Canônica:** Uma sequência de **3 ou mais hífens consecutivos** (`---`, `----`, etc.) isolada em sua respectiva linha (fora do Offset 0 do documento):
  ```text
  Primeiro parágrafo do capítulo.

  ---

  Novo parágrafo após a quebra temática.
  ```

*HTML Resultante:*
```html
<p>Primeiro parágrafo do capítulo.</p>
<hr>
<p>Novo parágrafo após a quebra temática.</p>
```

#### 3.7.2 Regras de Borda e Degradação
- **Isolamento Estrito:** A linha do divisor aceita apenas espaços opcionais em branco antes ou depois dos hífens (`[ ]*---[ ]*\n`).
- **Degradação:** Se houver qualquer outro caractere na linha (como letras, pontuações ou tabulações `\t`), a linha perde a condição de divisor e é avaliada conforme a regra de travessão tipográfico `---` (`&mdash;`) ou hífen literal contíguo (`5.5`).
- **Offset Zero:** Se ocorrer exatamente no Byte 0 do arquivo, ativa o cabeçalho de Frontmatter (`0.1`), nunca `<hr>`.

---

### 3.8 Atributos e Classes de Bloco (`.([ ... ])`)

Permite injetar classes CSS, identificadores (`id`) e atributos semânticos arbitrários em qualquer elemento estrutural de bloco (parágrafos, títulos, tabelas, citações, listas), unificando a sintaxe com os atributos inlines:

#### 3.8.1 Posição e Assinatura Canônica
- **Posição Obrigatória:** Ocupa a linha física imediatamente anterior ao bloco alvo, sem linha em branco intermediária.
- **Assinatura:** Utiliza o sigilo `.` colado à Caixa Operacional abraçando Caixas de Identidade estruturadas: `.([.classe1 .classe2][#id][attr=valor])`.

#### 3.8.2 Exemplos e Emissão

##### 1. Em Parágrafo
```text
.([.lead][#intro])
Este é o parágrafo de destaque no topo da página.
```

*HTML Resultante:*
```html
<p class="lead" id="intro">Este é o parágrafo de destaque no topo da página.</p>
```

##### 2. Em Tabela
```text
.([.table-striped .table-hover])
| Produto | Preço |
| :--- | ---: |
| Livro | 45.00 |
```

*HTML Resultante:*
```html
<table class="table-striped table-hover">
  <thead>
    <tr>
      <th class="align-left">Produto</th>
      <th class="align-right">Preço</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="align-left">Livro</td>
      <td class="align-right">45.00</td>
    </tr>
  </tbody>
</table>
```

##### 3. Em Citação
```text
.([.epigraph])
> A simplicidade é o último grau de sofisticação.
> Leonardo da Vinci
```

*HTML Resultante:*
```html
<blockquote class="epigraph">
  <p>A simplicidade é o último grau de sofisticação.<br>
  Leonardo da Vinci</p>
</blockquote>
```

#### 3.8.3 Invariante e Degradação
- Se a linha seguinte ao `.([ ... ])` for uma linha em branco (`\n\n`) ou outro token sem bloco válido, o `.([ ... ])` perde seu bloco-alvo e degrada para texto comum com disparo de `Warning` no LSP.

---

### 3.9 Blocos de Injeção Crua e Polimorfismo Multi-Alvo (`===`)

Os Blocos de Injeção Crua (`===`) transmitem código nativo diretamente para o documento de saída sem sanitização de caracteres (`<`, `>`, `&`, `"`, `'`), operando em dois modos distintos:

#### 3.9.1 Bloco Cru Incondicional (Sem Alvo)
Quando iniciado por `===` puro (sem Caixa de Identidade):
- **Injeção Incondicional Direta:** O compilador não assume, valida ou amarra o conteúdo a nenhuma linguagem. Ele simplesmente despeja o texto interno diretamente no fluxo de saída sem escapar nada, em qualquer backend de compilação.
- **Invariante de Não-Ramificação:** Um bloco que inicia com `===` puro **não aceita ramificações**. Ele termina estritamente no próximo `===` isolado.
- **Emissão Semântica Direta:** Todo o conteúdo interno é emitido verbatim no documento sem tags `<pre><code>` e sem sanitização.

```text
===
<div class="custom-embed">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/dQw4w9WgXcQ" allowfullscreen></iframe>
</div>
===
```

*HTML Resultante:*
```html
<div class="custom-embed">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/dQw4w9WgXcQ" allowfullscreen></iframe>
</div>
```

#### 3.9.2 Bloco Cru Condicional / Polimórfico (`===[alvo]`)
Quando iniciado com uma Caixa de Identidade explícita `===[alvo]`:
- **Entrega Condicional por Gatilho:** O conteúdo de um ramo `===[alvo]` **só é entregue se o compilador tiver um backend ou gatilho ativo para aquele alvo específico**. Se o alvo não estiver configurado ou não coincidir com a saída atual, o compilador **simplesmente não entrega o código**, ignorando a injeção e emitindo zero bytes.
- **Permissão de Ramificação:** Permite declarar múltiplos blocos alternativos de código cru, cada um direcionado a um backend específico (ex.: `html`, `latex`, `typst`, `markdown`).
- **Obrigatoriedade de Rótulo:** Todas as ramificações subsequentes devem conter compulsoriamente sua Caixa de Identidade (ex.: `===[latex]`).
- **Encerramento Global:** O bloco polimórfico encerra com `===` isolado (sem colchetes).

```text
===[html]
<div class="destaque"><em>Texto específico para Web</em></div>
===[latex]
\begin{tcolorbox}\textit{Texto específico para PDF/LaTeX}\end{tcolorbox}
===[typst]
#box(fill: luma(240))[_Texto específico para Typst_]
===
```

*HTML Resultante (ao compilar para HTML):*
```html
<div class="destaque"><em>Texto específico para Web</em></div>
```

*(Os ramos `latex` e `typst` são ignorados na emissão HTML).*

#### 3.9.3 Regra da Correspondência Exata ($M = N$) e Envelope ($N \ge 3$)
- **Paridade Rígida:** Se a cerca abrir com 3 iguais (`===[html]`), os ramos intermediários (`===[latex]`) e o fechamento final (`===`) devem conter rigorosamente 3 iguais.
- **Envelope Estrito:** Caso o conteúdo interno precise conter sequências de `===`, a cerca externa deve ser aberta com um comprimento estritamente maior (ex.: `====[html] ... ====[latex] ... ====`).
- **Tolerância a Arquivo Incompleto (Auto-Close em EOF):** Se um bloco cru (`===` ou `===[alvo]`) não for fechado antes do fim do arquivo, o compilador fecha compulsoriamente a cerca no fim do arquivo (EOF), preservando o conteúdo capturado, e emite um `Warning` educativo no LSP:
  > `[Warning L50:C1] Bloco de injeção crua '===' não finalizado antes do fim do arquivo. Auto-fechado em EOF.`
- **Governança por Contexto de Segurança (CSC):** Em ambientes restritos (modo `Sanitized`), injeções de código cru são neutralizadas ou sanitizadas via quinteto OWASP para impedir Stored XSS.

---

## Parte 4: Funções de Sistema (Sigilos)

As Funções de Sistema fornecem extensões semânticas pontuais para anotações e taxonomias técnicas, operando através da Caixa Operacional abraçando Caixas de Identidade estruturadas: `PREFIXO([ ... ])`.

### 4.1 Tabela de Funções de Sistema

| Sigilo | Função Semântica | Assinatura Canônica | Regra de Reconhecimento | Emissão Semântica (HTML) |
| :---: | :--- | :--- | :--- | :--- |
| **`!`** | **Entrada de Teclado** | `!([Ctrl+C])` | Regra Léxica: Unário em caixa `!([tecla])` | `<kbd>Ctrl+C</kbd>` |
| **`?`** | **Abreviação / Glossário** | `?([HTTP][Hypertext Transfer Protocol])` | Regra Sintática: Estruturado `?([sigla][significado])` | `<abbr title="Hypertext Transfer Protocol">HTTP</abbr>` |
| **`:`** | **Definição de Termo** | `:([termo])` | Regra Léxica: Unário em caixa `:([termo])` | `<dfn>termo</dfn>` |
| **`$`** | **Fórmula Matemática** | `$([E=mc^2])` | Regra Léxica: Unário em caixa `$([fórmula])` | `<math>...</math>` ou `<span class="math">` |
| **`#`** | **Tag / Taxonomia** | `#([backend])` | Regra Léxica: Unário em caixa `#([tag])` | `<a href="?tag=backend" class="tag">#backend</a>` |
| **`@`** | **Menção de Usuário** | `@([nome])` | Regra Léxica: Unário em caixa `@([nome])` | `<a href="/user/nome" class="mention">@nome</a>` |
| **`&`** | **Citação Bibliográfica** | `&([fonte])` | Regra Léxica: Unário em caixa `&([fonte])` | `<cite class="bib-ref">fonte</cite>` |
| **`^`** | **Citação Curta Inline** | `^([texto])` | Regra Léxica: Unário em caixa `^([texto])` | `<q>texto</q>` |
| **`~`** | **Texto Idiomático** | `~([termo])` | Regra Léxica: Unário em caixa `~([termo])` | `<i class="idiomatic">termo</i>` |
| **`*`** | **Destaque Visual Puro** | `*([termo])` | Regra Léxica: Unário em caixa `*([termo])` | `<b>termo</b>` |
| **`-`** | **Texto Secundário / Menor** | `-([termo])` | Regra Léxica: Unário em caixa `-([termo])` | `<small>termo</small>` |
| **`%`** | **Data / Timestamp** | `%([2026-09-26][26/09/2026])` | Regra Sintática: Estruturado `%([iso][visual])` | `<time datetime="2026-09-26">26/09/2026</time>` |

### 4.2 O Princípio da Caixa Unificada para Sigilos (`PREFIXO([ ... ])`)
Para eliminar 100% dos conflitos com a sintaxe de linguagens de programação, equações matemáticas e pontuação da linguagem humana, o LambdaType adota o paradigma universal da caixa unificada:
- **Imunidade Matemática Absoluta:** Expressões como `2^(x+y)`, `-(a+b)`, `$(x)` ou `x*(y+z)` **nunca colidem** com sigilos de citação, texto menor ou destaque visual, pois os sigilos exigem estritamente a Caixa de Identidade `[` colada ao parêntese de abertura (`PREFIXO([`).
- **Imunidade de Ponteiros e Código C/Rust:** Construções como `void *(malloc(size))` ou `int *(*fn)()` são processadas como texto ou código sem nenhuma interpretação espúria.
- **Aridade 1 (Modo Unário):** Consiste em um único argumento posicional delimitado por colchetes dentro dos parênteses: `PREFIXO([conteúdo])` (ex.: `!([Ctrl+V])`, `*([palavra])`).
- **Aridade Múltipla (Modo Estruturado):** Aceita múltiplos colchetes posicionais contíguos: `PREFIXO([arg1][arg2])` (ex.: `?([SGBD][Sistema de Gerenciamento de Banco de Dados])`).
- **Invariante de Isolamento:** Funções de sistema nunca se aninham umas dentro das outras. Tentativas de aninhamento disparam `Warning` no LSP e o sigilo interno degrada para texto literal.

---

## Parte 5: Mecânica do Compilador & Regras Formais

### 5.1 Modelo de Linha e Quebras
- **`\n` Simples (Continuação Suave em Prosa):** Concatena linhas físicas consecutivas pertencentes ao mesmo parágrafo com um espaço (`0x20`), proporcionando fluxo natural e legível sem forçar quebras no HTML.
- **`\` + `\n` (Hard Break Explícito):** A barra invertida antes de `\n` força a inserção de uma quebra de linha visual (`<br>`) no interior de parágrafos, citações e admonitions.
- **Quebra Intra-Célula de Tabela (`\\`):** Em células de tabela, o byte físico `\n` encerra a linha da grade; quebras de linha visuais internas utilizam obrigatoriamente a sequência `\\` (`<br>`).
- **`\n\n` Duplo (Block Break):** Linhas vazias encerram o container atual e abrem um novo bloco.
- **Barreira Dura de Bloco (Invariante Universal dos Inlines):** Os escopos inline (`**`, `~~`, `==`, `--`, `++`, `^^`, `__`, ` `` `, links, mídias, sigilos) fluem através de quebras simples `\n` dentro do mesmo bloco folha (parágrafo, título, célula de tabela, item de lista). O escopo encerra compulsoriamente em: linha em branco (`\n\n`), início de novo bloco estrutural ou fim de arquivo (EOF). Qualquer delimitador que permanecer órfão ao final do bloco degrada deterministicamente para texto literal.

### 5.2 Delimitadores Casados vs. Vizinhança Assimétrica
Tokens de formatação de texto exigem vizinhança assimétrica para abrir e fechar escopos:
- `espaço SÍMBOLO texto` $\rightarrow$ Abertura de escopo de estilo.
- `texto SÍMBOLO espaço` $\rightarrow$ Fechamento de escopo de estilo.
- Delimitadores não cruzam quebras de bloco (`\n\n`).

### 5.3 Princípio da Simetria de Vizinhança (Entidades Tipográficas)
Símbolos tipográficos simples (`--`, `---`, `->`, etc.) não criam escopos e exigem vizinhança simétrica:
- **Modo Espaçado:** `espaço SÍMBOLO espaço` (ex.: `A -> B`, `2 -- 3`).
- **Modo Compacto:** `[alfanumérico]SÍMBOLO[alfanumérico]` (ex.: `4--6`, `1914--1918`).

### 5.4 Operador de Cola Léxica (` . `) e Escape (`\`)
- **Cola Léxica:** A sequência estrita ` espaço . espaço ` remove a si mesma e os espaços vizinhos, colando os nós adjacentes na saída (`**termo** . s` $\rightarrow$ `termos` com "termo" em negrito).
- **Escape de Caractere (`\C`):** A barra invertida imediatamente antes de qualquer símbolo suprime sua função sintática, tratando-o como caractere literal puro.

### 5.5 Regra dos Hífens Contíguos
Qualquer sequência de **4 ou mais hífens consecutivos** (`----`, `------`) não é decomposta pelo lexer, degradando para texto literal (ou divisor temático se isolada na linha inteira).

### 5.6 A Ontologia Semiótica das Três Caixas

Ver 2.2 para a definição canônica da divisão semiótica entre as três caixas primárias ([ ... ] Identidade, { ... } Recurso, ( ... ) Operação).

#### 5.6.1 Fechamentos Compostos (A Trindade de Caixas)
O casamento das caixas externas com Caixas de Identidade internas resolve de forma ortogonal e semântica as três maiores necessidades de estruturação do documento, com a regra universal de que **a URL/Origem técnica é sempre o último argumento**:
1. **`[[ ... ]]` Navegação & Links:** Aridade 1 a 3 (`[Destino]`, `[Rótulo][Destino]`, `[Rótulo][Tooltip][Destino]`).
2. **`{[ ... ]}` Mídia & Recursos:** Aridade 1 a 3 (`[Origem]`, `[Alt][Origem]`, `[Alt][Legenda][Origem]`).
3. **`([ ... ])` Operação & Notas:** Aridade 2 a 5 em linha dedicada (`[Ref][Título]`, `[Ref][Título][URL]`, `[Ref][Título][Autor][URL]`, `[Ref][Título][Autor][Obs][URL]`), alimentando as chamadas inline `[^ref]`.

### 5.7 Filosofia de Tolerância: Degradação Graciosa Universal + Warnings
Como linguagem focada em comunicação e documentação técnica humana, o LambdaType prioriza a integridade e a preservação total do texto:
- **Lossless Fault-Tolerant por Padrão:** O compilador nunca descarta texto nem interrompe o processamento do arquivo por anomalias sintáticas.
- **Degradação Graciosa Automática:** Delimitadores incompletos, símbolos órfãos ou formatações incorretas degradam graciosamente para texto literal legível na saída.
- **Coleção de Diagnósticos (Warnings Educativos):** Paralelamente à AST, o compilador gera avisos detalhados (linha, coluna e causa) para orientar o LSP no editor sem quebrar a renderização.
- **CI/CD Confiável:** Por padrão, a compilação retorna código de sucesso (`exit code 0`). A flag opcional `--warnings-as-errors` pode ser ativada para transformar avisos em erros estritos em pipelines corporativos.

### 5.8 Transição Contígua de Blocos e Diagnósticos de Higiene (LSP Warnings)
A gramática do LambdaType resolve a transição entre blocos contíguos de forma inequívoca em $O(1)$:
- **Execução do Compilador:** Como cada bloco possui regras estritas de borda e fecho, blocos colados entre si ou colados diretamente em parágrafos de texto compilam com total sucesso e geram nós irmãos independentes na AST.
- **Diagnóstico Educativo de Higiene (Regra Geral):** É uma má prática de legibilidade colar blocos entre si ou colar blocos estruturais em parágrafos de texto sem separação por linha em branco física (`\n\n`). O compilador emite compulsoriamente um `Warning` no LSP em qualquer ocorrência de blocos contíguos não arejados.
- **Exceção Amigável Única:** Títulos imediatamente seguidos de seu primeiro parágrafo de texto (`# Título\nTexto`) são a única transição contígua tolerada sem disparo de aviso.

### 5.9 Pós-Processamento e Extensibilidade Semântica (Pipeline de Emissão)
O compilador LambdaType separa estritamente o parsing ($O(n)$ semântico) da camada de emissão de saída, garantindo extensibilidade total sem onerar a velocidade do núcleo:
- **Garantia de Tipagem na AST:** Cada elemento sintático (notas de rodapé, siglas, citações, mídias, tabelas) gera nós com propriedades e argumentos perfeitamente discriminados na árvore sintática abstrata.
- **Gatilhos de Emissão (AST Hooks / Triggers):** A camada de renderização fornece ganchos para que plugins e pipelines de publicação interceptem nós antes de gerar o HTML:
  - `on_footnote_emit`: Permite injetar atributos semânticos ricos como W3C DPUB-ARIA (`role="doc-endnotes"`, `role="doc-backlink"`), Microdata ou Schema.org (`itemprop="citation"`, `itemscope`, `itemtype="https://schema.org/DigitalDocument"`).
  - `on_media_emit`: Permite enriquecer imagens com metadados `schema.org/ImageObject` ou atributos responsivos (`srcset`, `loading="lazy"`).
  - `on_code_emit`: Permite acoplar destacadores de sintaxe de servidor (Tree-sitter, Syntect) ou injetar classes para bibliotecas client-side (Prism, Shiki).
- **Filosofia do Núcleo:** O gerador padrão emite HTML5 semântico limpo, rápido e universalmente compatível, delegando atributos e metadados verbosos a esses gatilhos dedicados.

### 5.10 Modelo Formal de Hierarquia e Contenção (Dual-Axis Nesting)

O LambdaType governa o aninhamento e a hierarquia entre blocos através de um **Modelo Dual Ortogonal**, garantindo parsing estritamente determinístico em $O(n)$ sem necessidade de backtracking:

#### 5.10.1 Eixo 1: Profundidade Estrutural por Tabulação (`\t`)
- **Domínio:** Governa exclusivamente Listas (`3.1`) e blocos subordinados a itens de lista (sublistas, cercas de código, tabelas, etc.).
- **Invariante do Byte Único:** Apenas o caractere `\t` (`0x09`) incrementa ou decrementa a profundidade estrutural na AST. Espaços comuns (`0x20`) nunca alteram a profundidade da pilha.
- **Pilha de Containers (Stack em $O(1)$):**
  - Cada `\t` contíguo no início da linha física adiciona exatamente 1 nível de profundidade (`ItemBody`).
  - Ao encontrar menos tabulações que o nível corrente, o compilador desempilha imediatamente os containers necessários até restaurar o nível correspondente.

#### 5.10.2 Eixo 2: Contexto de Envelope por Prefixos (`>`, `:`, `<`)
- **Domínio:** Governa containers de bloco fechado (Citações `3.3`, Blocos Expansíveis `3.6` e Avisos `3.4`).
- **Repetição de Tokens:**
  - Citações (`> `): A profundidade é dada pela repetição de `>` (`> > ` $\rightarrow$ citação aninhada).
  - Blocos Expansíveis (`: `): Um details dentro de outro repete o prefixo (`: :[-] Título`).
- **Invariante de Não-Preguiça (No-Lazy-Continuation):** Toda linha pertencente a um container de envelope deve conter compulsoriamente seu prefixo ativo. Linhas sem o prefixo encerram o container imediatamente.

#### 5.10.3 O Princípio do Descascamento Guiado por Pilha (Stack-Driven Peeling)
Ao processar uma linha física, o compilador consome os tokens de contenção combinando-os sequencialmente contra uma **pilha ativa de quadros de container** (*Frame Stack*):
1. **Quadro de Container (Frame):** Cada container aberto registra seu tipo no topo da pilha com o sigilo esperado de continuação (`> `, `: `, `< ` ou `\t`).
2. **Consumo Casado com a Pilha:** O scanner consome prefixos e tabulações na ordem em que os containers foram aninhados:
   - Se um Admonition for aberto dentro de um item de lista (`1. \n\t<[!]`), a pilha contém `[ListItem, ItemBody, Admonition]`. As linhas seguintes consom primeiro `\t` e depois `< `.
   - Se uma sublista for aberta dentro de uma citação (`> \n> \t- `), consome primeiro `> ` e depois `\t`.
3. **Desempilhamento em Mismatch:** Se um token esperado não casar, o compilador desempilha o topo sem avançar a posição do cursor na linha física, reavaliando o caractere contra o container pai restante.

#### 5.10.4 Invariantes de Contenção por Convenção Arquitetural
Para preservar a clareza tipográfica (*"Pure typography"*) e a previsibilidade $O(n)$, o LambdaType estabelece as seguintes convenções arquiteturais obrigatórias:
1. **Células de Tabela são Estritamente Inlines:** Células de tabela (`| conteúdo |`) aceitam exclusivamente nós inlines (estilos casados, links, quebra intra-célula `\\`, código inline, checkboxes). É proibido embutir blocos estruturais (listas, cercas, tabelas aninhadas) dentro de células. A quebra de linha visual dentro de uma célula é resolvida por `\\`.
2. **Universalidade de Contenção de Admonitions:** Avisos tipados (`<[-]` a `<[+]`) obedecem integralmente à pilha de contenção, podendo viver como filhos de itens de lista (`\t<[!]`), citações (`> <[!]`) e details (`: <[!]`), além de abrigar qualquer bloco filho em seu próprio corpo (`< `).

---

### 5.11 Classificação Formal do Compilador e Autômatos

A arquitetura formal do analisador LambdaType é estritamente determinística e divide-se em um modelo híbrido de dois níveis:

#### 5.11.1 Nível de Bloco: Streaming Deterministic Pushdown Transducer com Buffer de 2 Linhas
O nível estrutural de blocos opera como um transdutor de pilha determinístico com lookahead restrito a 2 linhas físicas ($k=2$):
- **Buffer de 2 Linhas:** O compilador mantém em janela deslizante a linha física corrente e a próxima linha física (`Lookahead(1 linha)`). Isso permite desambiguar em $O(1)$:
  1. Tabelas mínimas sem divisória vs. prosa contendo barras (`|`).
  2. Transições de blocos contíguos sem linha em branco intermediária.
  3. Atributos de bloco `.([ ... ])` precedendo o container alvo.
- **Pilha de Quadros (Frame Stack):** O descascamento consome prefixos e tabulações em tempo linear proporcional ao comprimento da linha ($O(L)$), garantindo processamento de blocos em tempo global estritamente linear $O(n)$ e espaço $O(d)$, onde $d$ é a profundidade de aninhamento.

#### 5.11.2 Nível Inline: DPDA-LIFO Bounded por Bloco Folha
O nível de prosa e elementos inlines opera como um **Autômato com Pilha LIFO de Delimitadores Determinístico**:
- **Escopo Confinado por Bloco Folha:** Todos os delimitadores de estilo (`**`, `~~`, `==`, `--`, `++`, `^^`, `__`, ` `` `, links, mídias, sigilos) fluem através de `\n` simples dentro do parágrafo ou bloco folha, e têm seu ciclo de vida confinado pelo término do bloco (`\n\n`, início de novo bloco ou EOF).
- **Sem Varredura Retroativa ($O(n)$ real):** Delimitadores empilham seus índices no buffer. Ao fechar o bloco, qualquer delimitador órfão remanescente na pilha é desempilhado e emitido como texto literal puro sem rescan.
- **Passada Linear Única:** O scanner inline percorre o span residual do bloco folha em uma única passada determinística.

---

### 5.12 Contexto de Segurança de Compilação (CompilationSecurityContext - CSC)

Para atender tanto a documentação técnica avançada quanto aplicações web públicas e multi-tenant (CMS, fóruns, blogs), o compilador LambdaType expõe dois perfis formais de segurança:

| Modo de Segurança | Gatilhos e Comportamento de Injeção | Tratamento de URLs e Mídias | Sanitização de Atributos |
| :--- | :--- | :--- | :--- |
| **`Trusted`** (Padrão CLI / Dev) | Injeções cruas `===` e `=N( ... )N=` são emitidas diretamente sem modificação. | Esquemas de URL arbitrários permitidos. | Classes e atributos `.([ ... ])` emitidos integralmente. |
| **`Sanitized`** (Web / CMS Aberto) | Injeções cruas são neutralizadas pelo quinteto OWASP ou suprimidas conforme política corporativa. | Protocolos perigosos (`javascript:`, `data:`, `vbscript:`) são sanitizados para `about:blank`. | Atributos de eventos (`onclick`, `onload`, `onerror`) são expurgados com LSP Warning. |
