# Multiplayer Local

**CORRIGIR ACENTOS EM TECLADO COMPATIVEL**
Primeiro passo para entendermos um jogo multiplayer eh criarmos as regras para o jogo executar corretamente de modo multiplayer, podemos fazer isso adicionando mais um player de forma local. O suporte ao multiplayer exige uma pequena refatoracao, que sera por onde comecaremos.

## Refatorando

Com a atualizacao do Rust para versao `1.66`, o linter do Rust sugeriu algumas refatoracoes mais simples, mas muito bem observadas no modulo ` grid`. A primeira eh na funcao `convert` e na funcao `translate_position` que possuia um casting desnecessario 