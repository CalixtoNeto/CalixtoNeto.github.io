---
layout: post
title: "Fronteiras e contratos fortes no frontend Angular: evitando o caos dos componentes gerados por IA"
description: "Repositório abstrato, signals, TypeScript estrito, ESLint próprio e fronteiras do Nx aplicados a uma Pokédex real, para que a IA não consiga criar God Components nem acoplar a interface à API."
serie: "Clean Architecture & IA"
ordem: 3
---
Peça a um agente de IA "um card que mostre os stats do Pokémon" e a resposta mais provável é um componente que faz tudo: injeta o `HttpClient`, monta a URL da PokeAPI, transforma o JSON, guarda o estado e desenha a tela. Funciona na primeira vez. Na décima mudança, ninguém mais sabe onde termina a interface e onde começa a API.

Este artigo mostra como impedir isso **com barreiras que o código não consegue atravessar**, e não com boas intenções. O laboratório é a mesma Pokédex do [artigo anterior]({{ '/blog/2026/10/pokedex-do-angular-12-ao-22-com-signals/' | relative_url }}). Ela já estava em Angular 22 com signals e agora ganhou um detalhe completo de cada Pokémon: stats, cadeia de evolução, selo de raridade e fraquezas por tipo.

![Lista da Pokédex com cards coloridos pelo tipo]({{ '/assets/img/blog/pokedex-lista.png' | relative_url }})

O código está no [PR #4 do repositório](https://github.com/CalixtoNeto/angular-pokedex/pull/4).

## O problema não é a IA escrever código ruim

O problema é que, sem fronteiras, **o caminho mais curto é sempre acoplar**. Um modelo de linguagem otimiza para resolver o pedido que recebeu. Se o componente pode injetar o `HttpClient`, injetar o `HttpClient` é a solução mais curta. Uma revisão pode pegar isso uma vez, mas não em todos os PRs, para sempre.

A saída é tornar o atalho **impossível de compilar ou de passar no lint**. A partir daí, a IA continua livre para escrever o código que quiser, só que dentro de caixas.

## 1. A interface depende de um contrato, não da API

O primeiro corte é entre o que a tela precisa e de onde o dado vem. O domínio define um contrato, uma classe abstrata que o Angular também usa como token de injeção:

```ts
// @pokedex/domain
export abstract class PokemonRepository {
  abstract regions(): Observable<Region[]>;
  abstract pokemonsOfRegion(regionId: string): Observable<Pokemon[]>;
  abstract detail(name: string): Observable<PokemonDetail>;
}
```

`PokemonDetail` é um modelo do domínio, e não o JSON da PokeAPI. A API devolve altura em decímetros, `gender_rate` em oitavos de fêmea e a cadeia de evolução como uma árvore. O domínio quer `heightInMeters`, `{ male: 87.5, female: 12.5 }` e uma lista de etapas. A tradução fica do outro lado da fronteira:

```ts
// @pokedex/data-access: o único lugar que conhece a PokeAPI
export function providePokeApi() {
  return [
    provideHttpClient(withInterceptors([cacheInterceptor])),
    { provide: PokemonRepository, useClass: PokeApiPokemonRepository },
  ];
}
```

E quem escolhe a implementação é o app, uma vez só:

```ts
export const appConfig: ApplicationConfig = {
  providers: [providePokeApi(), provideRouter(routes, withComponentInputBinding())],
};
```

Do lado da tela, o estado fica em signals e depende só do contrato:

```ts
@Injectable({ providedIn: 'root' })
export class PokedexStore {
  private readonly repository = inject(PokemonRepository);

  readonly regions = toSignal(this.repository.regions(), { initialValue: [] });
  readonly selectedRegion = signal('kanto');
  readonly search = signal('');

  private readonly pokemonsOfRegion = rxResource({
    params: () => this.selectedRegion(),
    stream: ({ params }) => this.repository.pokemonsOfRegion(params),
  });

  readonly pokemons = computed(() => (this.pokemonsOfRegion.hasValue() ? this.pokemonsOfRegion.value() : []));
  readonly visiblePokemons = computed(() => filterByName(this.pokemons(), this.search()));
}
```

O ganho aparece nos testes. O store é testado com um repositório **em memória**, sem HTTP nenhum, e verifica o que importa: a lista atualiza quando o repositório emite, e trocar de região deixa de ouvir a região anterior. A ordem dos Pokémon, o 404 de uma espécie e o cancelamento de pedidos são testados uma vez só, no repositório da PokeAPI. Cada teste sabe de uma coisa.

Um detalhe que parece pequeno e não é: a região era identificada pela URL da PokeAPI (`https://pokeapi.co/api/v2/pokedex/2/`). Com o contrato, ela passou a ser identificada pelo nome (`kanto`), porque uma URL de API não tem lugar no estado da interface.

## 2. Fronteiras físicas com Nx

Contrato sem fiscalização é só sugestão. Por isso o projeto virou um monorepo Nx, com uma lib por camada e uma **tag** em cada uma:

| Lib | Tag | Pode importar |
|---|---|---|
| `@pokedex/domain` | `type:domain` | nada |
| `@pokedex/data-access` | `type:data-access` | domain |
| `@pokedex/ui`, `@pokedex/ui-detail` | `type:ui` | domain, ui |
| `@pokedex/feature-list`, `@pokedex/feature-detail` | `type:feature` | domain, ui |
| app | `type:app` | todas |

A regra `@nx/enforce-module-boundaries` transforma a tabela em lei:

```js
depConstraints: [
  { sourceTag: "type:app", onlyDependOnLibsWithTags: ["type:feature", "type:data-access", "type:ui", "type:domain"] },
  // Features (componentes smart) falam com o domínio e desenham com a ui; nunca com a API.
  { sourceTag: "type:feature", onlyDependOnLibsWithTags: ["type:domain", "type:ui"] },
  { sourceTag: "type:ui", onlyDependOnLibsWithTags: ["type:domain", "type:ui"] },
  { sourceTag: "type:data-access", onlyDependOnLibsWithTags: ["type:domain"] },
  { sourceTag: "type:domain", onlyDependOnLibsWithTags: ["type:domain"] },
]
```

Testei a barreira fazendo o que um agente apressado faria. Coloquei um import da `data-access` dentro da feature:

```text
1:1  error  A project tagged with "type:feature" can only depend on libs tagged
            with "type:domain", "type:ui"  @nx/enforce-module-boundaries
```

E um componente dumb importando o store da feature:

```text
14:1  error  Circular dependency between "pokedex-ui" and "pokedex-feature-list" detected
```

As duas mensagens dizem exatamente qual fronteira foi cruzada. Para um agente, isso vale mais que qualquer instrução em prompt: o erro aparece no mesmo ciclo em que ele tenta validar o próprio trabalho.

### Fronteiras também protegem o bundle

O detalhe do Pokémon é carregado sob demanda, só quando alguém abre um card. Mesmo assim, o bundle inicial cresceu mais do que deveria. Os componentes do detalhe (barras de stats, evolução, fraquezas) estavam na mesma lib do card da lista, e a lista e o detalhe importavam essa lib pelo mesmo `index.ts`. O empacotador põe o código compartilhado no pedaço inicial, e o carregamento sob demanda perdeu o sentido.

A correção foi outra fronteira: os componentes do detalhe foram para `@pokedex/ui-detail`, e o detalhe virou um pedaço de **12 kB** baixado só quando necessário. Dividir libs por área de carregamento também é contrato.

## 3. TypeScript estrito: menos lugares para o erro se esconder

O `tsconfig.base.json` ganhou as opções que o `strict: true` não liga:

```json
"noUncheckedIndexedAccess": true,
"exactOptionalPropertyTypes": true,
"noPropertyAccessFromIndexSignature": true,
"noImplicitOverride": true
```

A primeira já pegou algo no próprio teste do card: `selos[0].classList` passou a ser erro, porque `selos[0]` pode ser `undefined`. Código gerado por IA adora assumir que o array tem um elemento. Com essa opção, a suposição precisa ser explícita.

O ESLint completa o que o TypeScript não vê: `any` e `!` (asserção de não nulo) viraram erro, e cada arquivo tem limite de tamanho, de linhas por função e de complexidade:

```js
"max-lines": ["error", { max: 150, skipBlankLines: true, skipComments: true }],
"max-lines-per-function": ["error", { max: 30, skipBlankLines: true, skipComments: true }],
complexity: ["error", 8],
```

É o princípio do artigo dos mapas (arquivos e funções curtos como contrato de contexto), agora como regra que barra, e não como recomendação.

## 4. Uma regra de ESLint própria para componentes dumb

A tag `type:ui` impede um componente dumb de importar outra lib, mas não impede que ele chame `inject(HttpClient)` direto do Angular. Esse é o atalho que sobra, e nenhuma regra pronta fecha. Então escrevi uma, em `tools/lint-rules`:

```js
// Componentes de @pokedex/ui são dumb: recebem dados por input() e avisam por output().
export const dumbComponent = {
  meta: {
    messages: {
      noInject: 'Componente de ui não usa inject(). Receba o dado por input() e deixe a feature buscá-lo.',
      noConstructorInjection: 'Componente de ui não recebe dependências pelo construtor. Use input() e output().',
      noHttp: 'A ui não fala com HTTP. O acesso à API fica em @pokedex/data-access, atrás do PokemonRepository.',
    },
  },
  create(context) { /* ... */ },
};
```

A regra tem testes próprios, escritos antes dela, com o `RuleTester` do ESLint:

```js
ruleTester.run('dumb-component', dumbComponent, {
  valid: [{ code: `@Component({...}) export class Card { readonly pokemon = input.required<Pokemon>(); }` }],
  invalid: [
    { code: `@Component({...}) export class Card { private readonly store = inject(PokedexStore); }`,
      errors: [{ messageId: 'noInject' }] },
    // construtor com dependência e import de @angular/common/http
  ],
});
```

Repare nas mensagens: elas não dizem só "proibido", dizem **o que fazer em vez disso**. Quem lê o erro é muitas vezes um agente, e a mensagem do lint é o prompt mais preciso que ele vai receber.

As features ganharam uma barreira parecida, com uma regra pronta (`no-restricted-imports`): nenhuma delas importa `@angular/common/http`.

## 5. Smart e dumb, com os papéis escritos no código

A divisão entre componentes smart e dumb existe há anos. A diferença agora é que ela está imposta pelo lint:

| | Smart (`feature-*`) | Dumb (`ui`, `ui-detail`) |
|---|---|---|
| Sabe de onde vem o dado | Sim, pelo contrato | Não |
| Usa `inject()` | Sim | Proibido pela regra própria |
| Estado | Signals e `rxResource` | Só `input()` e `computed()` |
| Teste | Com repositório falso | Com `setInput`, sem nenhum provider |

A página de detalhe é smart. Ela lê o nome da rota, pede o detalhe ao repositório e distribui os pedaços:

```ts
export class PokemonDetailPageComponent {
  private readonly repository = inject(PokemonRepository);
  readonly name = input.required<string>();   // vem da rota, com withComponentInputBinding

  protected readonly request = rxResource({ params: () => this.name(), stream: ({ params }) => this.repository.detail(params) });
  protected readonly detail = computed(() => (this.request.hasValue() ? this.request.value() : null));
  protected readonly defenses = computed(() => typeDefenses(this.detail()?.types ?? []));
}
```

As barras de stats são dumb: recebem uma lista e desenham.

```ts
export class StatBarsComponent {
  readonly stats = input.required<BaseStat[]>();
  protected readonly total = computed(() => statTotal(this.stats()));
}
```

![Aba About do Bulbasaur]({{ '/assets/img/blog/pokedex-sobre.png' | relative_url }})

## 6. Quando a ferramenta automática afrouxa a barreira

Este foi o momento mais instrutivo do trabalho. Ao gerar duas libs novas com o próprio Nx, o gerador:

- criou um `eslint.base.config.mjs` com a regra de fronteiras **liberada** (`sourceTag: "*"` → `"*"`);
- fez as libs importarem esse arquivo, além do antigo;
- e **apagou** a linha que registrava o plugin da regra própria.

Desta vez o lint quebrou alto, com um import duplicado. Mas bastaria uma combinação um pouco diferente para as fronteiras ficarem abertas em silêncio. A lição vale para qualquer automação, gerador ou agente: **depois de mexer nos arquivos de configuração, repita as sabotagens**. Refiz as três (feature importando a `data-access`, feature importando HTTP, `inject()` num componente dumb), e as três voltaram a ser barradas.

## 7. Prompts que trabalham a favor das fronteiras

Com as barreiras no lugar, o prompt deixa de ser o lugar onde se explica arquitetura e passa a ser o lugar onde se aponta **a camada**. O `CLAUDE.md` do repositório tem a tabela de libs e uma regra curta:

```markdown
Nenhum desses gates se desliga para fazer uma mudança passar. Se um deles barrar, a mudança está errada ou o
teste que falta ainda não foi escrito. Não afrouxe eslint.base.config.mjs, nx.json (cobertura) nem
tsconfig.base.json sem que isso seja o objetivo da tarefa.
```

E um pedido de funcionalidade fica assim:

```text
Mostre os movimentos do Pokémon numa aba "Moves".
1. Escreva primeiro o teste ponta a ponta da aba, com a resposta real recortada do api-data.
2. Domínio: acrescente moves ao PokemonDetail. Data-access: mapeie de pokemon.moves, com teste de contrato.
3. Ui-detail: um componente dumb que recebe a lista por input(). Feature-detail: só ligue a aba.
Rode npm run lint, npm run test:ci, npm run build e npm run e2e antes de dizer que terminou.
```

O prompt não precisa dizer "não chame a API do componente". O lint já diz, e diz melhor.

## O que fica

Fronteiras no frontend não são burocracia: são a forma de dar liberdade à IA sem perder o controle do desenho. O contrato separa a tela da API. O Nx e o ESLint transformam essa separação em erro de lint. A regra própria fecha o atalho que nenhuma regra pronta fecha. E o `CLAUDE.md` diz ao agente onde está cada coisa e como provar que terminou.

O próximo artigo trata da outra metade da trava: os testes que provam que o código dentro das caixas faz o que deveria.
