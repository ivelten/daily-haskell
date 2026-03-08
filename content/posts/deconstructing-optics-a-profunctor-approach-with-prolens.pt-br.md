+++
title = "Desconstruindo Ópticas: Uma Abordagem com Profunctores no Prolens"
date = 2026-03-08T02:28:04Z
draft = false
slug = "deconstructing-optics-a-profunctor-approach-with-prolens"
tags = ["haskell", "functional-programming", "optics"]
+++


Gerenciar estado imutável profundamente aninhado em C# frequentemente nos leva ao "inferno do aninhamento", onde atualizar uma propriedade dentro de um objeto complexo exige uma cascata de clonagens manuais ou o uso de bibliotecas pesadas como `AutoMapper` ou `DeepCopy`. Em Haskell, resolvemos isso com ópticas — especificamente lentes — mas a vasta biblioteca `lens` pode ser intimidadora para equipes que migram de backgrounds orientados a objetos. O pacote `prolens` oferece uma alternativa minimalista, de alto desempenho e educacional, que reduz as ópticas à sua essência teórico-categórica: o profunctor.

### O Núcleo do Profunctor

No centro de tudo, uma lente é uma forma de focar em uma parte de um dado dentro de uma estrutura maior. Se você vem do C#, pense em uma lente como um acessor de propriedade bidirecional. No `prolens`, definimos uma lente usando as classes de tipos `Profunctor` e `Strong`.

```haskell
-- Uma representação simplificada do tipo de lente profunctor
type Lens s t a b = forall p. Strong p => p a b -> p s t
```

Aqui, `s` é a estrutura, `t` é a estrutura modificada, `a` é a parte focada e `b` é a parte modificada. Ao usar `Strong`, ganhamos a habilidade de distribuir a lente sobre uma tupla, permitindo-nos, efetivamente, "manter" o resto da estrutura enquanto transformamos o foco.

### Composição na Prática

O poder das lentes reside na composição. Em uma aplicação C# típica, acessar uma propriedade aninhada parece algo como `user.Address.Street`. Em Haskell, se definirmos lentes para esses campos, podemos compô-los usando o operador `.` (ponto).

```haskell
import Prolens

data User = User { _name :: String, _address :: Address }
data Address = Address { _street :: String }

-- Usando padrões de criação de lentes
addressL :: Lens' User Address
addressL = lens _address (\u a -> u { _address = a })

streetL :: Lens' Address String
streetL = lens _street (\a s -> a { _street = s })

-- Composição: Focar diretamente na rua
userStreetL :: Lens' User String
userStreetL = addressL . streetL
```

Diferente dos acessores de propriedade do C#, que são fortemente acoplados à instância do objeto, essas lentes são funções de primeira classe. Você pode passá-las como argumentos, armazená-las em listas ou compô-las dinamicamente em tempo de execução.

### Por que Profunctores?

A escolha de profunctores em vez da codificação tradicional Van Laarhoven (baseada em CPS) usada na biblioteca `lens` é uma escolha estratégica. As lentes Van Laarhoven dependem dos functores `Const` e `Identity` para enganar o sistema de tipos e realizar operações de `get` e `set`. Embora elegantes, elas podem produzir assinaturas de tipos massivas, difíceis de depurar para iniciantes.

Ópticas baseadas em profunctores, conforme implementadas no `prolens`, são mais "honestas". Elas mapeiam diretamente para a transformação de uma função. Um `Profunctor p` é um mapeamento que suporta `dimap :: (a' -> a) -> (b -> b') -> p a b -> p a' b'`. Isso torna a lógica de "aproximar" e "afastar" explícita nos tipos.

### Armadilhas Comuns e Desempenho

Uma armadilha comum para quem transita do C# é tratar lentes como "ponteiros". No C#, se você obtém uma referência para uma propriedade, pode mutá-la no local. Em Haskell, lentes não mutam; elas retornam uma nova cópia da estrutura. Ao lidar com registros grandes, isso pode parecer ineficiente. No entanto, como Haskell usa compartilhamento estrutural, atualizar um único campo em um registro grande não copia toda a árvore de objetos — ele apenas copia o "espinho" do registro que leva ao campo modificado.

Um caso de borda a observar é a "fragmentação de lentes". Se você criar lentes granulares demais, pode acabar lutando com a inferência de tipos. Prefira sempre `Lens'` (uma lente onde a estrutura não muda de tipo) em vez da `Lens` mais geral, a menos que você esteja realizando transformações em nível de tipo.

### Trade-offs

O `prolens` não é um substituto imediato para a biblioteca `lens` completa. Ele carece da enorme suíte de operadores (`^.`, `.~`, `%~`, `%%~`) e do suporte avançado para traversals e isomorfismos. No entanto, para um engenheiro sênior, isso é uma funcionalidade, não um bug. Ao usar `prolens`, você força sua equipe a entender *como* as ópticas funcionam, em vez de depender de uma biblioteca "mágica" que lida com atualizações de estado complexas de forma implícita.

Se o seu projeto exige ópticas de alto desempenho com o mínimo de sobrecarga em tempo de compilação, o `prolens` é uma excelente escolha. Ele fornece os primitivos necessários para construir exatamente o que você precisa, sem a carga cognitiva de uma dependência de 50.000 linhas.

## Further Reading
- [Repositório GitHub Kowainik/prolens](https://github.com/kowainik/prolens)
- [Profunctor Optics: The Essence of Lenses](https://www.cs.ru.nl/~jmh/papers/profunctor-optics.pdf)
- [A biblioteca lens vs. ópticas baseadas em profunctores](https://hackage.haskell.org/package/lens)
