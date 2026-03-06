+++
title = "Como Construí um Blog Cyborg em 4 Dias com Haskell, IA e um Bot no Discord"
date = "2026-03-05"
tags = ["haskell", "ia", "github-copilot", "hugo", "discord", "gitops", "meta"]
+++

Quatro dias atrás, eu tinha um blog sem tema, sem posts e sem pipeline. Hoje, tenho uma fábrica de conteúdo completamente operacional: um motor de orquestração em Haskell que descobre tópicos, elabora posts usando a API do Gemini, os envia para revisão no Discord e os publica automaticamente no GitHub Pages. Esta é a história de como cheguei até aqui, o que usei, e por que acredito que a abordagem importa muito além deste projeto específico.

## A Premissa: Um Cyborg, Não um Robô

Quero ser preciso sobre os objetivos desde o início, porque eles guiam cada decisão que tomei.

Eu não queria um blog de IA. Não queria um sistema que publicasse tudo o que um modelo de linguagem produz, sem revisão, como se meu nome não estivesse associado. Tenho opiniões sobre qualidade. Tenho uma voz própria. E, sinceramente, tenho experiência suficiente para saber quando um rascunho está errado, mesmo que a gramática seja impecável.

O que eu queria era um fluxo de trabalho *cyborg* — uma arquitetura em que permaneço firmemente no loop, mas a sobrecarga mecânica do processo de publicação é automatizada. A IA faz a pesquisa e o primeiro rascunho. Eu faço o julgamento. A pipeline faz o restante.

Essa distinção não é postura filosófica. É a restrição de design que moldou toda a arquitetura.

## Dia 1: Estruturando o Projeto Haskell

Comecei com um projeto Cabal em branco e um devcontainer. A escolha do toolchain — Haskell, GHC 9.6, PostgreSQL — foi deliberada. Passei boa parte dos últimos dois anos me convencendo de que Haskell é uma boa linguagem para construir sistemas backend confiáveis, e usá-lo para construir a infraestrutura do meu blog pareceu o tipo certo de prova.

E desde esse primeiro commit, o GitHub Copilot com Claude Sonnet 4.6 foi meu par de programação em todo o projeto Jarvis — não apenas no blog. Cada módulo, cada tipo de dado, cada migração SQL, cada teste. Eu desenhei cada funcionalidade, revisei cada etapa gerada e decidi o que era suficientemente bom para commitar. A IA escrevia código numa velocidade que eu não conseguiria sozinho. Eu fornecia o julgamento que nenhum modelo pode substituir.

O projeto é estruturado como um único projeto Cabal com uma biblioteca (`Orchestrator`) e um executável (`Main`). A biblioteca é organizada em módulos com responsabilidades bem definidas:

```text
src/Orchestrator/
├── AI/          -- Cliente da API Gemini
├── Database/    -- Esquema Persistent e pool de conexões
├── Discord/     -- Bot discord-haskell
├── GitHub/      -- Cliente REST da API do GitHub
├── Posts/       -- Renderizador Markdown para Hugo
├── Topics/      -- Seletor de conteúdo e lógica de ingestão
├── Pipeline.hs  -- Orquestração de alto nível
└── TextUtils.hs -- Geração de slugs, extração de títulos, truncamento de texto
```

Cada módulo tem uma responsabilidade única e clara. A fronteira entre eles é um tipo de dado Haskell simples. O módulo `Pipeline.hs` os conecta. Nada vaza.

## Dia 2: O Modelo de Concorrência

A decisão de design mais interessante do projeto inteiro é o modelo de concorrência. Veja o núcleo do `Main.hs`:

```haskell
main :: IO ()
main = do
  -- ... configuração omitida ...

  _ <- forkIO $
    scheduledLoop "[Discovery]"
      (cfgDiscoveryIntervalSecs cfg)
      (runDiscovery pipeEnv)

  _ <- forkIO $
    scheduledLoop "[Drafts]"
      (cfgDraftIntervalSecs cfg)
      (runDraftGeneration pipeEnv)

  putStrLn "[Jarvis] All workers started. Discord bot running..."
  startBot dcCfg
```

Dois workers são iniciados em background com `forkIO`. A thread principal bloqueia em `startBot`, que executa o loop de eventos do Discord. Esta é a concorrência idiomática em Haskell: threads leves, sem estado mutável compartilhado entre os workers (cada um lê do banco de dados de forma independente), e uma separação clara entre o trabalho agendado e o tratamento reativo de eventos.

A função `scheduledLoop` merece destaque:

```haskell
scheduledLoop :: String -> Int -> IO () -> IO ()
scheduledLoop prefix intervalSecs action = do
  result <- try action :: IO (Either SomeException ())
  case result of
    Left ex  -> putStrLn $ prefix <> " Error: " <> displayException ex
    Right () -> pure ()
  putStrLn $ prefix <> " Sleeping for " <> show intervalSecs <> "s."
  sleepSecs intervalSecs
  scheduledLoop prefix intervalSecs action
```

Exceções são capturadas, registradas e descartadas. O loop continua. Esse é exatamente o comportamento correto para um worker em background: uma falha na descoberta de conteúdo não deve derrubar o bot do Discord que está gerenciando uma revisão naquele momento. O sistema degrada de forma graciosa.

Uma simplificação deliberada que vale nomear: as chamadas a `forkIO` não têm nenhum mecanismo para sinalizar aos workers que terminem de forma limpa quando o processo é encerrado. Não há handshake com `MVar`, nem `async`/`wait`, nem protocolo de desligamento gracioso. Isso é intencional por enquanto — e o raciocínio é direto. Toda mudança de estado significativa no sistema é persistida no PostgreSQL *antes* de qualquer chamada externa ser feita. Se o processo for encerrado no meio de uma operação, o banco de dados é a fonte da verdade, e a próxima execução retomará a partir de um estado consistente. Não há trabalho em memória pelo qual valha esperar. Aguardar os workers drenarem antes de sair adicionaria complexidade sem adicionar segurança.

## Dia 3: O Fluxo de Revisão

O fluxo de revisão é a parte operacionalmente mais interessante do sistema. Quando um rascunho está pronto, `processDraft` em `Pipeline.hs` faz o seguinte:

```haskell
processDraft env rcKey rc = do
  draft <- generateDraft pipeAiCfg [rcToDiscovered rc]
  now   <- getCurrentTime
  markAsDrafted env rcKey now
  postDraftKey <- persistInitialDraft env rcKey rc draft now
  rr <- mkReviewRequest env rcKey postDraftKey now draft
  registerForReview pipeDcCfg rr
```

A sequência de etapas importa: o banco de dados é atualizado *antes* que a mensagem do Discord seja enviada. Se a chamada ao Discord falhar, o item já está marcado como `ContentDrafted`, portanto não será selecionado novamente no próximo ciclo. A atomicidade é garantida no limite do processo, não apenas no nível do banco de dados.

A função `mkReviewRequest` é particularmente elegante na forma como lida com o problema do conteúdo bilíngue. O corpo em português brasileiro é rastreado via um `IORef`, *sem* ser exposto no registro `ReviewRequest` que o bot do Discord vê. O corpo em inglês é o que o revisor lê e refina. Quando a aprovação é disparada, o corpo em pt-br é recuperado do `IORef` e publicado junto com a versão em inglês:

```haskell
approve ptBrRef finalBodyEn tags = do
  finalBodyPtBr <- readIORef ptBrRef
  publishDraft env rcKey postDraftKey createdAt
    finalBodyEn finalBodyPtBr tags
```

Isso mantém a interface do Discord limpa — os revisores interagem apenas com o inglês legível — enquanto garante que ambas as versões de idioma estejam sempre sincronizadas. A tradução para pt-br é revisada em conjunto sempre que o revisor solicita uma alteração.

## Dia 4: O Blog em Si

No quarto dia, construí o site Hugo — também com o GitHub Copilot e Claude Sonnet 4.6. Aqui a abordagem cyborg se torna reflexiva: usei um assistente de IA para construir o blog que um assistente de IA vai popular. O mesmo fluxo de trabalho se aplicou: descrevi o que queria, revisei o que foi gerado, ajustei a direção quando necessário e segui em frente. A conversa inteira que produziu este blog é, ela própria, um registro desse processo.

O tema é o [hugo-coder](https://github.com/luizdepra/hugo-coder), adicionado como submódulo git. O site suporta inglês e português desde o início. Cada post é publicado em dois arquivos com sufixos de idioma — `my-post.en.md` e `my-post.pt-br.md` — e o modo multilíngue do Hugo cuida do restante.

A pipeline de deploy é um workflow padrão do GitHub Actions que executa `hugo --minify` e publica no branch `gh-pages`. O Jarvis o aciona via API REST do GitHub após cada commit bem-sucedido.

## Por Que Haskell?

Esta é a parte em que quero me demorar, porque é a resposta honesta para a pergunta que espero receber.

Escolhi Haskell porque acredito que ele é excepcionalmente adequado para o desenvolvimento assistido por IA, e queria testar essa convicção em um projeto real. Aqui está o argumento central:

Código funcional expressa *o que* um cálculo significa, não *como* executá-lo mecanicamente. A lógica de negócio se torna uma composição de funções puras. Efeitos colaterais são explícitos, declarados nos tipos. Os dados fluem por pipelines em vez de serem mutados no lugar. Quando peço ao GitHub Copilot que implemente uma função em Haskell, ele trabalha dentro de um sistema que *exige* precisão: os tipos precisam se encaixar, os efeitos precisam estar corretos, a estrutura precisa ser consistente. A linguagem é uma restrição, e restrições fazem bem para a colaboração com IA.

O contraargumento — de que a curva de aprendizado do Haskell o torna impraticável — é real, mas é menos relevante do que costumava ser. Com as ferramentas de IA atuais, os aspectos mecânicos do Haskell (importações, precedência de operadores, empilhamento de monad transformers) são amplamente automatizados. O que resta é a clareza conceitual que torna o Haskell valioso em primeiro lugar. Você ainda precisa entender o que está construindo. A máquina ajuda a escrever.

Um aviso necessário: não estou afirmando que Haskell é a única linguagem que vale a pena usar, nem que é universalmente a escolha certa. Não acredito em balas de prata. Esta é uma decisão pessoal deliberada e um teste de conceito — queria validar minha hipótese em um projeto real, com restrições reais, sob pressão de tempo real. As conclusões que tiro aqui são observações honestas desse experimento, não um mandato.

## A Disciplina de Testes

Um ponto sobre o qual quero ser explícito: cada funcionalidade gerada com assistência de IA foi testada com testes unitários antes de ser considerada concluída. Não como uma reflexão tardia — como a condição para avançar para a próxima etapa.

A suite de testes do Jarvis tem duas camadas. Os testes unitários — cobrindo lógica pura como geração de slugs, utilitários de texto e etapas de orquestração da pipeline — simulam todas as dependências externas em processo: sem Discord, sem Gemini, sem API do GitHub. Esses rodam rápido e rodam em qualquer lugar. Os testes de integração, porém, exercitam o banco de dados real: eles testam as migrações do esquema Persistent e a camada de banco contra uma instância real de PostgreSQL. Isso é intencional — migrações de banco de dados são exatamente o tipo de coisa que você não quer simular.

Ambas as camadas estão integradas ao devcontainer. O ambiente de desenvolvimento é um [VS Code Dev Container](https://code.visualstudio.com/docs/devcontainers/containers) que sobe o container da aplicação junto com um sidecar PostgreSQL automaticamente via `docker-compose`. Uma vez dentro do container, GHC 9.6 e Cabal estão no `PATH`, o banco está rodando e `cabal test` simplesmente funciona. Sem configuração manual, sem variáveis de ambiente para ajustar no desenvolvimento local. Esta foi mais uma área onde a abordagem cyborg se pagou — toda a configuração do devcontainer também foi construída com o Copilot, e funciona de forma confiável.

Serei honesto: minhas mensagens de commit durante esses quatro dias não são um modelo de clareza. Eu estava me movendo rapidamente, e a disciplina que apliquei ao código em si nem sempre chegou ao log de commits. Isso é algo que pretendo melhorar conforme o projeto amadurece. Mas a substância nunca esteve em dúvida — cada etapa foi revisada, cada novo módulo foi testado, e a pipeline foi executada de ponta a ponta antes de eu declarar qualquer coisa como concluída.

Essa é a disciplina que torna o desenvolvimento assistido por IA confiável em vez de imprudente. A velocidade é real. A responsabilidade também precisa ser.

## O Que Aprendi

Quatro dias. Uma aplicação Haskell. Um blog Hugo. Um MVP funcional.

A lição mais importante é que o modelo cyborg não é um compromisso — é uma melhoria genuína em relação às alternativas totalmente manuais e totalmente automatizadas. Obtenho a eficiência dos rascunhos de IA sem abrir mão do julgamento editorial. O sistema escala com minha atenção: se estou ocupado, os rascunhos são enfileirados; quando estou disponível, reviso e aprovo pelo Discord em minutos.

A segunda lição é sobre o próprio modelo de colaboração. O GitHub Copilot com Claude Sonnet 4.6 não foi um gerador de código que apontei para um arquivo em branco. Foi um par de programação com quem trabalhei iterativamente — descrevendo a intenção, revisando o resultado, questionando tudo que não atingia o padrão esperado, e avançando apenas quando estava satisfeito. Os quatro dias foram intensos, mas não foram imprudentes.

Acredito fortemente que este é o fluxo de trabalho ideal para o momento atual: tratar o agente de IA como seu par em um modelo de pair programming do Extreme Programming. Não uma ferramenta que você comanda. Um parceiro com quem você pensa junto. No XP clássico, há um driver e um navigator — um escreve, o outro observa e questiona. Com IA, os papéis são naturais: a IA dirige com velocidade e habilidade enciclopedista no teclado; você navega com expertise de domínio, julgamento arquitetural e responsabilidade pelo resultado. Você segura o volante. A IA é o motor.

O que faz esse modelo funcionar é a disciplina que ele exige de *você*. Você não pode desligar. Precisa entender cada linha antes de fazer o commit. Precisa saber por que uma decisão de design foi tomada, porque será você quem a defenderá, estenderá e depurará daqui a seis meses. A IA dá a você alavancagem. A expertise é o que torna essa alavancagem segura.

A terceira lição é sobre Haskell especificamente. A disciplina que a linguagem impõe — a explicitude, os tipos, a separação entre código puro e com efeitos — se traduziu diretamente em um sistema mais fácil de raciocinar, mais fácil de estender e mais difícil de quebrar acidentalmente.

O prazo de quatro dias não teria sido possível sem as ferramentas de IA. Mas a confiabilidade do resultado não é produto de nenhum fator isolado — é a soma da disciplina do modelo cyborg, das decisões arquiteturais tomadas ao longo do caminho, e da solidez que Haskell impõe por padrão.

## Próximos Passos

O MVP está pronto. Mas ainda não está terminado — há duas áreas que quero refinar.

A primeira é o **processo de revisão**. O bot do Discord atual lida bem com o caminho feliz, mas os casos extremos da interação do revisor — frases de aprovação parcialmente digitadas, reações rápidas de emoji, feedback concorrente na thread, formatos de mensagem inesperados — ainda não foram todos deliberadamente exercitados. Quero fortalecer o tratamento de eventos do bot com testes mais direcionados e testes de caos intencional da camada de interação com o Discord antes de deixar tudo rodar sem supervisão.

A segunda é a **infraestrutura**. Por enquanto, o Jarvis roda localmente. Isso significa que o worker de descoberta, o worker de rascunhos e o bot do Discord só estão vivos quando meu laptop está aberto. O próximo passo óbvio é implantá-lo em uma máquina persistente — uma instância pequena no [Hetzner Cloud](https://www.hetzner.com/cloud/) é o plano. A configuração é direta: um único VPS rodando Docker Compose com o executável do Jarvis e um container PostgreSQL, gerenciado com um serviço `systemd` simples ou uma política de restart do Compose. O Hetzner oferece boa relação custo-benefício para o caso de uso: custo baixo, infraestrutura europeia confiável e poder computacional suficiente para um orquestrador leve sempre ativo.

Com isso em funcionamento, a pipeline roda sem que eu precise pensar nela. O Jarvis descobre tópicos, enfileira rascunhos e me avisa no Discord quando uma revisão está pronta. Eu aprovo ou dô feedback pelo celular. O post vai ao ar. Esse é o objetivo.

Este blog existe para documentar essa jornada. O [Jarvis](https://github.com/ivelten/jarvis) existe como a infraestrutura que o mantém funcionando. Estou genuinamente animado com o que vem por aí.
