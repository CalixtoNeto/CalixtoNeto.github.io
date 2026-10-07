---
layout: post
title: "Testes automatizados estritos como quality gates para código de IA"
description: "Por que uma suíte de testes impiedosa é a única auditoria confiável para código gerado por agentes, e como o TDD vira o contrato que a IA precisa satisfazer. Com o passo a passo real de uma funcionalidade nova na Pokédex."
serie: "Clean Architecture & IA"
ordem: 4
---
O [artigo anterior]({{ '/blog/2026/10/fronteiras-e-contratos-no-frontend-angular/' | relative_url }}) colocou o código da Pokédex dentro de caixas: um contrato entre a tela e a API, fronteiras do Nx, uma regra de ESLint própria. As caixas garantem que cada coisa está **no lugar certo**. Elas não garantem que o código **faz o que deveria**. Essa é a parte dos testes.

A tese deste artigo é direta: com agentes escrevendo boa parte do código, a revisão linha por linha deixou de escalar, e o que sobra como auditoria confiável é **uma suíte de testes que barra a entrega**. Não é um relatório que alguém lê depois: é um portão (*quality gate*) na CI. O código não passa enquanto a suíte não passar.

O exemplo é real: o detalhe do Pokémon, com stats, evoluções, raridade e fraquezas, construído com teste antes do código em todas as camadas. Está no [PR #4](https://github.com/CalixtoNeto/angular-pokedex/pull/4).

![Aba de evolução do Pikachu]({{ '/assets/img/blog/pokedex-evolucao.png' | relative_url }})

## Por que os testes deixaram de ser opcionais

Um teste tem três papéis, e com IA os três mudam de peso:

1. **Especificação.** O teste diz o que o código deve fazer, antes do código existir. Para um agente, é o pedido mais preciso possível: não "mostre as fraquezas", mas "Bulbasaur toma ×2 de fogo, gelo, voador e psíquico, e ×¼ de planta".
2. **Verificação.** O agente roda os testes sozinho e sabe se terminou. Sem isso, "terminei" significa "escrevi algo plausível". Os dois PRs de IA quebrados do [artigo da atualização]({{ '/blog/2026/10/pokedex-do-angular-12-ao-22-com-signals/' | relative_url }}) eram exatamente isso.
3. **Memória.** Na próxima mudança, feita por outro agente ou outra pessoa, os testes lembram de cada regra que já foi combinada.

Um código que nenhum teste exercita é código que ninguém verificou. Com volume de IA, isso deixa de ser exceção e vira a regra, a não ser que um portão impeça.

## As camadas da suíte

Cada camada da arquitetura tem o tipo de teste que faz sentido para ela:

| Camada | Teste | O que prova |
|---|---|---|
| Tela inteira | Ponta a ponta (Playwright) | O que o usuário vê e faz, num navegador de verdade |
| `data-access` | Contrato, com respostas reais da PokeAPI | Que o mapeamento entende a API como ela é |
| `domain` | Unitário puro | As regras: tabela de tipos, soma de stats, filtro |
| `ui`, `ui-detail` | Componente com `setInput` | Que cada componente dumb desenha o que recebe |
| `feature-*` | Componente com repositório falso | Que a tela smart liga as peças certas |
| `tools/lint-rules` | `RuleTester` | Que a regra própria barra o que deve e deixa passar o resto |

No fim, foram **67 testes unitários, 5 da regra de lint e 14 ponta a ponta**. Todos rodam sem rede e sem precisar de um Chrome instalado, menos os ponta a ponta. Isso não é detalhe: é a diferença entre um agente que consegue se verificar e um que não consegue.

## O ciclo de TDD com IA, passo a passo

O fluxo que funcionou, em ordem:

### 1. O teste de fora, primeiro

Antes de qualquer código, 8 testes ponta a ponta descreviam o detalhe. Os números vêm das respostas reais da PokeAPI:

```ts
test('Base Stats mostra cada stat e o total', async ({ page }) => {
  await page.goto('/pokemon/bulbasaur');
  await aba(page, 'Base Stats').click();
  const linhas = page.getByRole('tabpanel').getByRole('row');
  await expect(linhas).toHaveText([/HP\s*45/, /Attack\s*49/, /Defense\s*49/, /Sp\. Atk\s*65/, /Sp\. Def\s*65/, /Speed\s*45/, /Total\s*318/]);
});

test('Defenses mostra fraquezas e resistências calculadas pelos tipos', async ({ page }) => {
  await page.goto('/pokemon/bulbasaur');
  await aba(page, 'Defenses').click();
  await expect(page.locator('[data-defense="weak"] li')).toHaveText(['Fire ×2', 'Ice ×2', 'Flying ×2', 'Psychic ×2']);
  await expect(page.locator('[data-defense="resistant"] li'))
    .toHaveText(['Water ×½', 'Electric ×½', 'Grass ×¼', 'Fighting ×½', 'Fairy ×½']);
});
```

Resultado: **8 falhando, 6 passando** (os 6 antigos, da lista). É o ponto de partida certo. Um teste que nunca falhou não provou nada.

### 2. Desça uma camada, de novo com o teste primeiro

Para as fraquezas, o teste do domínio veio antes da tabela de tipos. Os casos foram calculados à mão, e não copiados de uma implementação:

```ts
it('chega a ×4 e a imunidade quando os tipos se somam (Charizard: fogo e voador)', () => {
  const { weak, immune } = listed(['fire', 'flying']);
  expect(weak).toEqual(['water 2', 'electric 2', 'rock 4']);
  expect(immune).toEqual(['ground 0']);
});
```

É aqui que a IA mais ajuda e onde mais precisa de trava. Gerar a tabela 18×18 de efetividade é trabalho mecânico, que um modelo faz em segundos. Errar uma célula é fácil e silencioso. Os casos calculados à mão (Bulbasaur, Charizard, Gengar, um tipo só, tipo desconhecido) são o contrato que a tabela precisa satisfazer.

![Fraquezas e resistências do Bulbasaur]({{ '/assets/img/blog/pokedex-defesas.png' | relative_url }})

### 3. Contrato com dados reais, não com dados inventados

A PokeAPI estava bloqueada na rede onde fiz o trabalho, mas ela tem um espelho estático no GitHub, o [`PokeAPI/api-data`](https://github.com/PokeAPI/api-data). Baixei de lá as respostas reais de `pokemon`, `pokemon-species` e `evolution-chain` e recortei só os campos que o app usa: de 600 KB por Pokémon para 120 KB no total. Elas viraram fixtures de **teste de contrato**:

```ts
import pichu from '../testing/pokeapi/pokemon-pichu.json';
import pichuSpecies from '../testing/pokeapi/species-pichu.json';
import pikachuChain from '../testing/pokeapi/evolution-chain-10.json';

it('evolução por amizade e por item', () => {
  const stages = detail(pichu, pichuSpecies, pikachuChain).evolution.map(stage => stage.map(s => [s.name, s.condition]));
  expect(stages).toEqual([[['pichu', null]], [['pikachu', 'Friendship']], [['raichu', 'Thunder Stone']]]);
});
```

Fixtures inventadas testam o que você **acha** que a API devolve. As reais testam o que ela devolve: `gender_rate` em oitavos, alturas em decímetros, textos com `\f` e "POKéMON" vindos dos cartuchos antigos, URLs de espécie que só existem dentro de outra URL. A suíte da `data-access` (24 testes, com os de contrato) passou na primeira implementação, e é exatamente isso que um teste de contrato deve dar: confiança de que a implementação entende a API de verdade. Os testes ponta a ponta usam os mesmos arquivos.

### 4. Leia o motivo da falha antes de mexer no código

Três testes dos componentes falharam com a implementação já correta:

```text
expected [ 'HP45', 'Attack49', … ] to deeply equal [ 'HP 45', 'Attack 49', … ]
```

O Angular remove o espaço em branco entre elementos. O teste é que dependia de um detalhe que não importa. A correção foi no teste, lendo célula por célula. É a situação em que um agente sem critério "conserta" o código para o teste passar (por exemplo, enfiando um espaço no template). A regra que escrevi no `CLAUDE.md` vale para pessoas e para agentes: **descubra por que falhou antes de decidir o que mudar**.

### 5. Feche com o teste de fora

Com todas as camadas verdes, os testes ponta a ponta rodaram de novo. Dez passaram e quatro falharam. Uma falha era de fixture (faltava a cadeia de evolução do Charmander). As outras três vinham de **um bug de verdade**:

```text
<img class="artwork" alt="bulbasaur" …> from <header …> subtree intercepts pointer events
```

A arte do Pokémon, posicionada sobre a borda do cabeçalho, cobria as abas na largura de desktop e roubava o clique. No celular quase não aparecia. Nenhum teste unitário pegaria isso, porque só um navegador de verdade, clicando como uma pessoa, enxerga sobreposição.

## Cobertura como portão, e o ponto cego dela

A cobertura entrou como gate **por arquivo**, e não como média do projeto. Um arquivo sem teste não se esconde atrás dos outros:

```json
"configurations": {
  "ci": {
    "coverage": true,
    "coverageInclude": ["{projectRoot}/src/**/*.ts"],
    "coverageThresholds": { "perFile": true, "statements": 90, "branches": 85, "functions": 90, "lines": 90 }
  }
}
```

A linha `coverageInclude` tem história. Para testar o gate, acrescentei ao domínio uma função sem teste nenhum, que é o que acontece quando um agente gera código "a mais". **O gate passou.** A cobertura continuava 5/5 statements. O builder empacota o código dos testes, e o empacotador remove por *tree-shaking* tudo que nenhum teste importa. A função nova nem chegava ao relatório.

Com `coverageInclude`, todo arquivo da lib entra no relatório, testado ou não. A mesma sabotagem agora falha:

```text
ERROR: Coverage for lines (0%) does not meet global threshold (90%) for libs/pokedex/domain/src/lib/sort-by-name.ts
```

E o gate corrigido pagou a conta na hora. Ele apontou dois arquivos que só os testes ponta a ponta cobriam (a lista e `providePokeApi()`). Depois barrou o mapeamento do detalhe, com 78,6% dos *branches* cobertos, porque faltava testar os dados ausentes: Pokémon sem arte oficial e espécie sem texto em inglês, que são casos reais nas formas novas da PokeAPI.

Uma ressalva honesta: cobertura mede o que foi **executado**, não o que foi **verificado**. Um teste sem `expect` cobre 100% e não prova nada. Por isso a cobertura é um portão mínimo, e não o principal.

## Testar os testes: a sabotagem

A pergunta que fecha qualquer gate é "e se eu quebrar de propósito?". Fiz isso em cada camada:

| Sabotagem | Quem barrou |
|---|---|
| Lista na ordem da rede, não da Pokédex | 4 testes unitários e 2 ponta a ponta |
| Função nova sem teste | Cobertura por arquivo (depois do `coverageInclude`) |
| Feature importando a `data-access` | `@nx/enforce-module-boundaries` |
| Feature importando `HttpClient` | `no-restricted-imports` |
| `inject()` num componente dumb | Regra própria `pokedex/dumb-component` |
| Escolher a grafia menos frequente de um bairro (artigo dos mapas) | Golden master |

É uma versão manual de **teste de mutação**: alterar o código e conferir se algum teste percebe. A versão automática (Stryker) é o próximo passo natural. Ela faz isso milhares de vezes e mede quantas mutações sobrevivem. Mesmo manual, o exercício já achou o ponto cego da cobertura, que nenhum relatório mostraria.

## O portão na CI

Tudo isso só vale se roda sozinho, em todo PR:

```yaml
- name: Lint (fronteiras entre libs e regras do projeto)
  run: npm run lint
- name: Testes unitários com cobertura mínima por arquivo
  run: npm run test:ci
- name: Build de produção
  run: npm run build
- name: Testes ponta a ponta (PokeAPI simulada)
  run: npm run e2e
```

E o `CLAUDE.md` diz ao agente que esses são os quatro comandos que definem "terminei", e que nenhum deles se desliga para uma mudança passar.

![Selo de lendário no Mewtwo]({{ '/assets/img/blog/pokedex-raridade.png' | relative_url }})

## O que os testes não garantem

- **A API real.** Os testes rodam contra respostas reais, mas gravadas. Se a PokeAPI mudar amanhã, a CI continua verde. Um teste agendado contra a API de verdade, fora do caminho do PR, cobriria isso.
- **A aparência.** Nenhum teste aqui compara pixels. As capturas deste artigo foram conferidas por olho. Testes de regressão visual seriam a camada seguinte.
- **Intenção.** O teste prova que o código faz o que o teste pede. Se o teste pedir a coisa errada, os dois concordam no erro. Escrever o teste continua sendo trabalho de quem entende o problema.

## Como aplicar no seu projeto

1. **Ponta a ponta primeiro, com a API simulada.** É a trava que sobrevive a qualquer refatoração.
2. **Fixtures reais.** Grave respostas de verdade e recorte; não invente JSON.
3. **Teste antes do código, em cada camada.** Veja falhar pelo motivo certo.
4. **Cobertura por arquivo, com todos os arquivos no relatório.** E teste o gate com uma função sem teste.
5. **Sabote.** Quebre cada regra de propósito e confira quem barra.
6. **Tudo na CI e no `CLAUDE.md`.** O agente precisa saber o que significa "terminei" e não pode ter como pular.

## O que fica

A IA mudou o custo de escrever código: ficou barato. Ela não mudou o custo de código errado: continua caro, e agora chega em maior volume. Uma suíte de testes estrita é o que equilibra as duas coisas. O teste vem antes, como contrato. O agente implementa até satisfazê-lo. E a CI se recusa a aceitar menos que isso. O papel humano se desloca para onde ele não é substituível: decidir o que o teste deve pedir.
