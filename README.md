# LambdaType (`.lt`)

> *"Less ambiguity. Pure typography."*

Linguagem de marcação determinística com tipografia pura, polimorfismo multi-alvo e degradação graciosa universal.

## Estado

Em desenvolvimento ativo. Fase 0 concluída — infraestrutura base do parser.

## Estrutura do repositório

| Arquivo | Função |
| :--- | :--- |
| `lambdatype_referencia.md` | Especificação normativa da linguagem |
| `lambdatype.peg.txt` | Gramática PEG (alvo normativo para implementação) |
| `implementacao.md` | Diário de implementação e decisões arquiteturais |

## Sobre

O LambdaType é uma linguagem de marcação pessoal, sem pretensão de substituir Markdown. O objetivo é ter uma gramática determinística, tipografia limpa e um pipeline de compilação com separação estrita entre parsing, semântica e emissão.

A especificação em `lambdatype_referencia.md` é a fonte da verdade. A gramática em `lambdatype.peg.txt` é o alvo normativo para quem for implementar.