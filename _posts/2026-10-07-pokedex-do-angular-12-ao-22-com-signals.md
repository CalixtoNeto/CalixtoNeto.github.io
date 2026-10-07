---
layout: post
title: "Do Angular 12 ao 22 com signals: atualizando um projeto de estudo com testes primeiro"
description: "Passo a passo da atualização de uma Pokédex em Angular 12 para o Angular 22, com componentes standalone, signals e sem zone.js, usando testes ponta a ponta como trava de segurança."
---
Em 2021 fiz uma Pokédex em Angular 12 para estudar: uma lista de Pokémon por região, com busca por nome, consumindo a [PokeAPI](https://pokeapi.co/). Cinco anos depois, o Angular está na versão 22, e o jeito recomendado de escrever um componente mudou quase inteiro. Não existem mais NgModules, o estado fica em signals e o zone.js deixou de ser obrigatório.

Antes deste artigo, pedi a um agente de IA que fizesse a atualização. Ele abriu dois pull requests, e os dois estavam quebrados. Este texto mostra por que eles quebraram e como fiz a atualização de novo, do 12 ao 22, aplicando a mesma ideia do [extra do artigo dos mapas de votação]({{ '/blog/2026/10/mapas-de-voto-com-dados-abertos-e-llm/' | relative_url }}): **primeiro a trava de segurança, depois a mudança**.

O resultado está no [PR #3 do repositório](https://github.com/CalixtoNeto/angular-pokedex/pull/3), em 17 commits que dá para ler na ordem.

## Por que os dois PRs quebraram

Os dois PRs foram gerados pelo Codex, e as descrições deles contam a história:

- O **#1** modernizou o código (tipos, OnPush, `trackBy`), mas continuou no Angular 12. O agente não tinha acesso ao npm, a build só passava com um contorno de OpenSSL e os testes não rodaram porque não havia Chrome.
- O **#2** trocou as versões no `package.json` direto para o Angular 21 e **apagou o `package-lock.json`**. Sem acesso ao npm, ele não conseguiu rodar o `ng update`, então nenhuma migração automática foi aplicada. Build e testes também não rodaram.

O deploy de preview da Vercel falhou nos dois. O problema não era o código em si: **os dois PRs nunca rodaram**. Um agente que não consegue se verificar entrega algo com cara de pronto, e quem descobre o resto é o deploy.

Fechei os dois e recomecei com uma regra: nenhuma mudança entra sem que build e testes rodem antes.

## Passo 0: reproduzir o estado atual

Antes de mudar qualquer coisa, tentei rodar o `main` como está, no Node 22:

```text
npm run build
Error: error:0308010C:digital envelope routines::unsupported
code: 'ERR_OSSL_EVP_UNSUPPORTED'
```

É o mesmo erro que o Codex encontrou. O webpack do Angular 12 usa um algoritmo de hash que o OpenSSL 3 do Node 17+ desligou. Com `NODE_OPTIONS=--openssl-legacy-provider` a build passa, com **442,72 kB** no bundle inicial.

E os testes unitários? `ng test` nem começa:

```text
ENOENT: no such file or directory, lstat 'src/styles.css'
```

O `angular.json` apontava para um `styles.css` que nunca existiu (o arquivo real é `styles.scss`). Os 5 specs eram os esqueletos `should create` gerados pelo CLI, e um deles procurava um texto que não existe na página. Ou seja, **o projeto não tinha nenhum teste funcionando**.

## Passo 1: a trava, por fora do Angular

A atualização ia trocar o app por dentro: versão do framework, builder, NgModule por standalone, RxJS por signals. Um teste que dependesse dessa estrutura teria que ser reescrito a cada passo e não protegeria nada.

Por isso a trava é um teste **ponta a ponta**: o Playwright abre o app compilado num Chromium de verdade e olha a tela como um usuário. Para não depender da PokeAPI (que, aliás, estava bloqueada na rede onde fiz isso), cada pedido é respondido por uma versão falsa, no mesmo formato da API real:

```ts
// e2e/fixtures/pokeapi.ts
const POKEMON = {
  // O primeiro da lista responde por último: a ordem final tem de seguir a Pokédex, não a rede.
  bulbasaur: { id: 1, tipos: ['grass', 'poison'], atrasoMs: 400 },
  ivysaur: { id: 2, tipos: ['grass', 'poison'], atrasoMs: 0 },
  charmander: { id: 4, tipos: ['fire'], atrasoMs: 100 },
  // ...
};

await page.route('https://pokeapi.co/api/v2/**', route => responder(route, falsa));
```

O atraso no Bulbasaur é proposital. O código original pedia um Pokémon por vez, então a ordem saía certa de graça. Se a nova versão pedisse em paralelo e esquecesse de reordenar, o teste pegaria.

São 6 testes, todos descrevendo comportamento:

1. as regiões aparecem como botões, com nome legível ("Original Johto");
2. o app abre em Kanto, na ordem da Pokédex;
3. cada card mostra os tipos com a cor certa e a imagem oficial com o número em 3 dígitos;
4. a busca filtra pelo nome, sem diferenciar maiúsculas;
5. trocar de região troca a lista;
6. voltar a uma região já vista não pede os Pokémon de novo à API (é o que o interceptor de cache garante).

Os 6 passaram no app em Angular 12. **Esse foi o primeiro commit.** Só depois dele o código começou a mudar.

## Passo 2: do 12 ao 22, uma versão por vez

O guia oficial do Angular é claro: atualize **uma versão principal por vez**, com `ng update`. Cada versão traz migrações automáticas que reescrevem o código para a API nova. Trocar o número no `package.json` direto, como fez o PR #2, pula todas elas.

Para cada versão, o ciclo foi:

```bash
npx -p @angular/cli@N ng update @angular/core@N @angular/cli@N
npm run build
npx playwright test     # os 6 testes ponta a ponta
git commit              # um commit por versão
```

Como são dez repetições, escrevi um script que compila, roda o e2e e só faz o commit se tudo passar. A tabela mostra o que cada versão mudou sozinha:

| Versão | O que a migração fez | Bundle inicial |
|---|---|---|
| 12 | (ponto de partida) | 442,72 kB |
| 13 | Removeu os polyfills do Internet Explorer | 433,73 kB |
| 14 | `defaultCollection` → `schematicCollections`, alvo ES2020 | 428,04 kB |
| 15 | Removeu o `.browserslistrc` igual ao padrão, limpou o `test.ts` | 430,26 kB |
| 16 | Nada a migrar | 437,68 kB |
| 17 | Opções obsoletas do `angular.json` | 439,78 kB |
| 18 | `HttpClientModule` → `provideHttpClient()` | 460,92 kB |
| 19 | `standalone: false` nos componentes (standalone virou o padrão) | 459,15 kB |
| 20 | `moduleResolution: bundler` no tsconfig | 462,76 kB |
| 21 | Templates para `@for`, `provideZoneChangeDetection()` | 460,98 kB |
| 22 | `ChangeDetectionStrategy.Eager`, `withXhr()` | 481,89 kB |

Os 6 testes ponta a ponta passaram em todas as versões. Repare em duas coisas da tabela.

A primeira: **as migrações preservam o comportamento antigo de propósito.** O Angular 21 deixou de usar zone.js por padrão, então a migração acrescentou `provideZoneChangeDetection()` para o app continuar funcionando como antes. O 22 passou a usar OnPush e `fetch` por padrão, então a migração marcou cada componente como `Eager` e acrescentou `withXhr()`. Atualizar a versão não é o mesmo que modernizar o código. A modernização vem depois, como decisão sua.

A segunda: **o bundle cresceu** ao longo das versões, de 442 kB para 482 kB, porque o framework ganhou recursos que o app ainda não usava. Isso se resolve nos passos seguintes.

### O que travou no caminho

Nenhum dos problemas foi de código. Foram todos de ambiente:

- **O OpenSSL** se resolveu sozinho no Angular 13. Daí em diante, a build roda no Node 22 sem contorno.
- **O npm 10** é mais rígido com dependências entre pacotes do que o npm da época do Angular 13. O `ng update` falhava com `ERESOLVE`. A saída foi usar `NPM_CONFIG_LEGACY_PEER_DEPS=true` só durante a atualização e conferir no fim que o `npm ci` funcionava sem ele.
- **O lockfile.** Adicionar o Playwright com o npm 10 deixou o `package-lock.json` fora de sincronia, e o `npm ci` recusou. Corrigi antes de enviar qualquer coisa.
- **O Node.** O `ng update` baixa o CLI mais recente para orquestrar a atualização, e o CLI 22 exige Node 22.22.3 ou mais. Para os passos intermediários, rodei o CLI da própria versão de destino (`npx -p @angular/cli@N`). Para o 22, foi preciso o Node 24.
- **O TypeScript 6**, que veio com o Angular 22, recusa `baseUrl` e `downlevelIteration`. O `baseUrl` só servia para imports como `'src/environments/environment'`, que viraram caminhos relativos.

## Passo 3: o builder novo

O Angular 22 ainda aceita o builder antigo, baseado em webpack, mas avisa que ele está obsoleto. A troca pelo `@angular/build:application` (esbuild) é uma migração opcional do próprio `ng update`:

```bash
npx ng update @angular/cli --name use-application-builder
```

A build caiu de cerca de 13 segundos para 4. Uma consequência que quase passou despercebida: a saída mudou de `dist/angular-pokedex` para `dist/angular-pokedex/browser`. Isso quebraria o deploy da Vercel, que publica a pasta antiga. Um `vercel.json` com o `outputDirectory` novo resolve.

## Passo 4: Vitest no lugar do Karma

O Karma roda os testes dentro de um Chrome. É por isso que os testes do Codex não rodaram: não havia Chrome no ambiente dele. O Angular 21 trocou o padrão para o **Vitest**, que roda no Node com jsdom, sem navegador. Para um agente, essa é a diferença entre conseguir verificar o próprio trabalho e não conseguir.

A troca trouxe dois tropeços:

- O builder de testes não aceitava o `src/polyfills.ts` (um arquivo que só importava o `zone.js`). Ele espera o `zone.js` declarado direto na lista de polyfills, como num projeto novo.
- O RxJS 6 não tem os exports ESM que o Node exige. A atualização para o RxJS 7.8 resolveu, junto com o `@types/node`, que ainda estava na versão 12.

Com isso, o `npm ci` voltou a funcionar sem `legacy-peer-deps`, e o `node_modules` caiu de **1.312 para 357 pacotes**.

## Passo 5: standalone, e o teste que salvou a atualização

A migração oficial converte tudo para componentes standalone e apaga o `AppModule`:

```bash
npx ng g @angular/core:standalone --mode=convert-to-standalone
npx ng g @angular/core:standalone --mode=prune-ng-modules
npx ng g @angular/core:standalone --mode=standalone-bootstrap
```

A build passou sem nenhum aviso. Os testes ponta a ponta, não:

```text
✘ mostra as regiões como botões, com o nome legível
✘ abre em Kanto, na ordem da Pokédex, mesmo que a rede responda fora de ordem
✘ cada card mostra os tipos e a imagem oficial com o número em 3 dígitos
...
6 failed
```

A tela abria vazia. A migração tinha importado o `provideZoneChangeDetection` no `main.ts`, mas **não o colocou na lista de providers**. No Angular 22, sem esse provider, o app fica sem zone.js. Como o código ainda guardava os dados em arrays comuns, nada avisava o Angular que a resposta da API tinha chegado, e a tela nunca era atualizada.

Uma migração oficial, sem nenhum erro de compilação, deixou o app inútil. Os testes unitários não pegariam isso, porque nenhum deles passa pelo `main.ts`. Só um teste que abre o app de verdade e olha a tela. Devolvi o provider, os 6 testes voltaram a passar e o commit registra o motivo.

## Passo 6: signals

Até aqui o app estava atualizado, mas escrito como em 2021. O coração dele era este método do `PokemonService`:

```ts
async fetchPokemons(pokedex: string) {
  await this.unsubscribeALL()
  this._pokemons = []
  this.subscription = this.getNext(pokedex).subscribe(pokemons => {
    const details = pokemons.pokemon_entries.map((pokemon: any) => this.get(pokemon.pokemon_species.name));
    this.subscription = concat(...details).subscribe((response: any) => {
      this._pokemons.push({ /* ... */ ...response });
    });
  })
}
```

Ele tem arrays mutáveis, `any` por toda parte e uma lista manual de `Subscription` para cancelar pedidos na troca de região. O `concat` pede um Pokémon por vez, então Kanto, com 151 Pokémon, faz 151 pedidos em fila.

Antes de escrever a versão nova, escrevi os testes dela, e eles falharam porque os arquivos ainda não existiam. O desenho ficou em três camadas:

- **`PokeApiClient`**, o acesso à API, tipado;
- **funções puras** para transformar a resposta (`toPokemon`) e filtrar a busca (`filterByName`), testadas sem Angular nenhum;
- **`PokedexStore`**, com o estado em signals.

```ts
@Injectable({ providedIn: 'root' })
export class PokedexStore {
  private readonly api = inject(PokeApiClient);

  readonly regions = toSignal(this.api.regions(), { initialValue: [] });
  readonly selectedRegion = signal(DEFAULT_REGION_URL);
  readonly search = signal('');

  // Trocar de região cancela os pedidos da anterior: o rxResource cancela o stream antigo sozinho.
  private readonly pokemonsOfRegion = rxResource({
    params: () => this.selectedRegion(),
    stream: ({ params }) => this.api.pokemonsOfRegion(params),
  });

  readonly pokemons = computed(() => (this.pokemonsOfRegion.hasValue() ? this.pokemonsOfRegion.value() : []));
  readonly visiblePokemons = computed(() => filterByName(this.pokemons(), this.search()));

  selectRegion(url: string) {
    this.selectedRegion.set(url);
  }
}
```

O `rxResource` substitui toda a gestão manual de subscriptions. Quando `selectedRegion` muda, ele cancela o pedido anterior e começa o novo. Há um teste unitário só para isso, que confere com o `HttpTestingController` que o pedido antigo foi de fato cancelado.

Os componentes ficaram quase sem lógica. O card recebe o Pokémon por um `input()` de signal:

```ts
export class PokemonCardComponent {
  readonly pokemon = input.required<Pokemon>();
}
```

A lista só lê o store. O pipe de busca virou um `computed`, e o `FormsModule` saiu porque o campo de busca escreve direto no signal. Com todo o estado em signals, o zone.js pôde sair de verdade: agora são os signals que avisam o Angular quando algo mudou.

### Um tropeço nos testes

Os primeiros testes do store falharam mesmo com o código certo: a lista vinha vazia. Em vez de mexer no código até passar, inspecionei o estado do `rxResource` em três momentos. Ele publica o valor **uma volta do event loop depois** da resposta HTTP. O teste é que precisava esperar:

```ts
// O rxResource publica o valor uma volta do event loop depois da resposta.
const settle = async () => {
  await new Promise(resolve => setTimeout(resolve));
  TestBed.tick();
};
```

### O que mudou de comportamento, de propósito

Uma refatoração não deveria mudar comportamento, mas três mudanças valeram a pena. Cada uma está registrada no commit e coberta por teste:

- **Os detalhes dos Pokémon são pedidos 6 de cada vez**, e a lista continua na ordem da Pokédex. O teste ponta a ponta com o Bulbasaur atrasado garante a ordem.
- **Uma espécie sem Pokémon de mesmo nome fica de fora.** O Deoxys, por exemplo, é `deoxys-normal` na API. No código antigo, o 404 interrompia o `concat` e o resto da lista nunca carregava. Era um bug que ninguém tinha visto, porque Kanto não tem nenhum caso assim.
- **As imagens carregam só quando chegam perto da tela** (`loading="lazy"`) e ganharam um `alt` com o nome.

## Passo 7: o que sobrou de peso

Com o zone.js fora, o bundle caiu de 482 kB para 404 kB. Uma última olhada mostrou o `bootstrap.bundle.min.js`, com 80 kB, carregado em toda página. O app só usa as classes CSS do Bootstrap: nenhum `data-bs-*`, modal ou dropdown. Ele saiu, e os testes confirmaram que nada mudou. O bundle final ficou em **323,58 kB**.

## A sabotagem

Como no artigo dos mapas, quebrei o código de propósito para ver se a trava pegava. Fiz a lista sair na ordem em que a rede responde, e não na ordem da Pokédex:

```text
× abre em Kanto e mantém a ordem da Pokédex mesmo com respostas fora de ordem   (unitário)
✘ abre em Kanto, na ordem da Pokédex, mesmo que a rede responda fora de ordem   (ponta a ponta)
```

4 testes unitários e 2 ponta a ponta falharam. As duas camadas se complementam: o unitário diz exatamente qual regra quebrou, e o ponta a ponta confirma o que o usuário veria.

## O agente precisa saber como se verificar

O repositório ganhou um `CLAUDE.md` com o ciclo de verificação e uma regra sobre os testes de fora:

```markdown
Os testes em `e2e/` descrevem o comportamento visto de fora e não dependem de como o app é escrito.
Se um deles falhar depois de uma refatoração, o comportamento mudou: corrija o código, não o teste.
```

E um workflow no GitHub Actions roda os testes unitários, a build e os ponta a ponta em todo push e pull request. Um próximo PR que não compile vai aparecer vermelho antes de chegar à Vercel.

## Antes e depois

| | Angular 12 (`main`) | Angular 22 (PR #3) |
|---|---|---|
| Bundle inicial | 442,72 kB | 323,58 kB |
| Transferência estimada (gzip) | não informado | 66,36 kB |
| Tempo de build | ~13 s, só com contorno de OpenSSL | ~4 s |
| Testes unitários | não compilam | 17, Vitest, sem navegador |
| Testes ponta a ponta | nenhum | 6, com a PokeAPI simulada |
| Pacotes no `node_modules` | 1.312 | 357 |
| Pedidos de detalhe | um por vez | 6 em paralelo |
| zone.js | sim | não |

## Como repetir no seu projeto

1. **Rode o projeto como está antes de mudar qualquer coisa.** Anote o que funciona, o que não funciona e quanto pesa.
2. **Escreva testes ponta a ponta contra a versão atual**, com a API simulada. Eles precisam passar antes da primeira mudança.
3. **Atualize uma versão principal por vez, com `ng update`.** Nunca troque o número no `package.json` direto.
4. **Compile e rode os testes a cada versão**, e faça um commit por passo. Se algo quebrar, você sabe exatamente qual versão foi.
5. **Atualize primeiro, modernize depois.** As migrações preservam o comportamento antigo; standalone, signals e zoneless são passos seus, um de cada vez.
6. **Desconfie das migrações automáticas também.** A de standalone passou na build e deixou a tela vazia.
7. **Deixe por escrito como verificar o trabalho**, num `CLAUDE.md` e na CI, para o próximo agente (ou a próxima pessoa) não abrir um PR que nunca rodou.

## O que fica

A diferença entre os dois PRs quebrados e o PR novo não foi o modelo de IA nem a habilidade com Angular. Foi a possibilidade de **verificar**. Sem npm e sem navegador, o primeiro agente entregou código plausível e não testado. Com uma trava de 6 testes escrita antes de tudo, a atualização atravessou dez versões do Angular, uma migração que quebrou a tela em silêncio e uma reescrita completa do estado, sempre com a resposta certa à pergunta "isso ainda funciona?".
