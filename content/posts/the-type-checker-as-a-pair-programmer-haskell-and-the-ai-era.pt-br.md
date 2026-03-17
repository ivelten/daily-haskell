+++
title = "O Verificador de Tipos como Pair Programmer: Haskell e a Era da IA"
date = 2026-03-08T06:47:54Z
draft = false
slug = "the-type-checker-as-a-pair-programmer-haskell-and-the-ai-era"
tags = ["haskell", "artificial-intelligence", "functional-programming"]
+++


A proliferação de Large Language Models (LLMs) mudou fundamentalmente o gargalo da engenharia de software: passamos de escrever sintaxe para verificar intenção. À medida que integramos ferramentas como GitHub Copilot, Claude Code e Cursor em nossos fluxos de trabalho diários, a escolha da linguagem de programação torna-se menos sobre "quão rápido eu consigo digitar isso" e mais sobre "quão confiavelmente a IA pode verificar sua própria saída". Haskell, com seu sistema de tipos rigoroso e ênfase em funções puras, ocupa uma posição única nesse novo paradigma.

## A Economia de Tokens vs. O Imposto da Correção

Uma análise recente feita por desenvolvedores testando o Claude Code sugere que Haskell pode estar em desvantagem no que diz respeito à "economia de tokens". Como Haskell é denso e expressivo, ele frequentemente requer menos tokens para representar um modelo de domínio complexo do que Java ou C#. No entanto, a tendência da IA de alucinar pode ser mais custosa em Haskell. Quando uma IA gera um script em Python, ela pode produzir um código que executa, mas que contém erros de lógica sutis. Quando ela gera Haskell, o compilador atua como um guardião implacável.

Se a IA falhar em satisfazer o verificador de tipos, o "custo" se manifesta como um ciclo de tokens de correção de erros. Você pode se ver em um loop onde a IA tenta corrigir uma incompatibilidade de tipos introduzindo um `unsafeCoerce` ainda mais complicado ou uma restrição `Typeable` equivocada.

```haskell
-- Uma alucinação típica de IA: tentando corrigir uma incompatibilidade de tipos 
-- envolvendo as coisas em newtypes desnecessários ou casts inseguros.
processData :: [a] -> Either String [b]
processData xs = Right (unsafeCoerce xs) -- IA ruim! Pare com isso.
```

O segredo para usar IA com Haskell é solicitar "Desenvolvimento Orientado a Tipos" (TDD). Em vez de pedir à IA para "escrever uma função para processar esses dados", peça para ela "definir os tipos de dados primeiro e, em seguida, implementar a lógica". Ao forçar a IA a definir os tipos, você a obriga a internalizar o modelo de domínio antes de tentar resolver a implementação, reduzindo significativamente a taxa de alucinação.

## Aproveitando o Compilador como um Agente

No contexto de agentes de IA, Haskell atua como uma "camada de verificação" natural. A maioria das ferramentas de IA modernas pode ler erros de compilador. Se você direcionar a saída do `ghc` de volta para o contexto do seu agente, você fornece a ele um loop de feedback objetivo e imediato.

Considere um cenário onde você está refatorando uma máquina de estados complexa. Um desenvolvedor C# pode confiar em testes unitários para capturar regressões. Um desenvolvedor Haskell, no entanto, pode aproveitar GADTs (Generalized Algebraic Data Types) para codificar a máquina de estados nos próprios tipos.

```haskell
{-# LANGUAGE GADTs #-}

-- Codificando transições de estado no sistema de tipos
data Idle
data Running
data Stopped

data ProcessState s where
    IdleState   :: ProcessState Idle
    RunningState :: Int -> ProcessState Running
    StoppedState :: ProcessState Stopped

-- A IA pode facilmente raciocinar sobre essa estrutura porque o 
-- compilador proíbe transições inválidas por design.
transition :: ProcessState Idle -> ProcessState Running
transition IdleState = RunningState 0
```

Quando a IA tenta modificar esse código, o feedback do compilador a força a respeitar as transições de estado. Isso cria um efeito de "guarda-corpo" (guardrail) que é significativamente mais forte do que em linguagens dinâmicas.

## Onde Haskell Falha: O Ecossistema de Bibliotecas

A principal desvantagem de usar IA com Haskell é o problema da "biblioteca alucinada". LLMs são treinados em vastas extensões da internet, incluindo tutoriais de Haskell obsoletos e pacotes do Hackage abandonados. Se você pedir a uma IA para usar uma biblioteca como `lens` ou `conduit`, ela frequentemente gerará código que é sintaticamente bonito, mas que depende de APIs obsoletas ou funções inexistentes.

Em C#, a superfície da API é vasta, mas geralmente estável e bem documentada nos dados de treinamento. Em Haskell, a dependência de versões específicas de `base`, `mtl` ou `aeson` significa que a IA geralmente precisa de acesso ao seu ambiente local de `cabal` ou `stack` para ser verdadeiramente eficaz. Sem uma RAG (Retrieval-Augmented Generation) que indexe as dependências específicas do seu projeto, a IA falhará frequentemente em produzir código idiomático.

## Trade-offs Estratégicos

Se você é um engenheiro sênior gerenciando uma equipe, a escolha de Haskell em um fluxo de trabalho assistido por IA resume-se a um trade-off:

1. **Alta Verificação, Alta Configuração:** Você gasta mais tempo configurando o contexto da IA (fornecendo definições de tipos, exportações de módulos e estrutura do projeto), mas termina com uma base de código que é matematicamente mais difícil de quebrar.
2. **O "Imposto da Correção":** Você gastará mais tokens na geração inicial porque o compilador é rigoroso. Embora a IA possa exigir mais passos de raciocínio para satisfazer o verificador de tipos, você ganha uma arquitetura legível e verificável que torna a manutenção e a refatoração a longo prazo significativamente mais seguras e previsíveis.

Em última análise, Haskell é a linguagem perfeita para a era dos "Agentes de IA" justamente porque é a linguagem mais difícil de "trapacear". Quando a IA escreve uma função, ela não precisa apenas parecer correta; ela precisa ser correta em termos de tipos. Ao tratar o compilador como o revisor de código principal, você transforma a IA em uma ferramenta poderosa para gerar sistemas robustos e de nível de produção.

## Leitura Adicional

- [Type-Driven Development with Idris and Haskell](https://www.manning.com/books/type-driven-development-with-idris)
- [The impact of language choice on AI code generation](https://dev.to/mame/which-programming-language-is-best-for-claude-code-508a)
