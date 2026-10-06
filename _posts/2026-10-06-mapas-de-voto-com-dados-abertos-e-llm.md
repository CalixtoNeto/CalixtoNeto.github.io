---
layout: post
title: "Dois mapas de votação, zero reais de infraestrutura: dados abertos + LLM + GitHub Pages"
description: "Passo a passo de como montei o Mapa do voto da Paraíba e de Bayeux com ajuda de IA, coletando dados do TSE e publicando tudo de graça."
---
Em 5 de outubro de 2026, um dia depois do primeiro turno, criei dois repositórios: o **Mapa do voto · Paraíba** e o **Mapa do voto · Bayeux**. Dois sites públicos, com votos de presidente a vereador (presidente na Paraíba, vereador em Bayeux), histórico de eleições desde 2012, comparação de um candidato entre eleições e votos por escola. Sem servidor, sem banco de dados e sem custo.

Fiz os dois com ajuda de um LLM. Este artigo mostra como o projeto é montado, de onde vêm os dados, como o mapa é desenhado e como tudo é publicado de graça. A ideia é que você consiga repetir para a sua cidade, o seu estado ou qualquer outro assunto com dados abertos.

- Paraíba: [site](https://calixtoneto.github.io/mapa-do-voto-pb/) · [código](https://github.com/CalixtoNeto/mapa-do-voto-pb)
- Bayeux: [site](https://calixtoneto.github.io/mapa-do-voto-bayeux/) · [código](https://github.com/CalixtoNeto/mapa-do-voto-bayeux)

![Mapa da Paraíba: escolhendo eleição, cargo e candidato, e vendo os votos por escola]({{ '/assets/img/blog/pb-uso.gif' | relative_url }})

## O que os sites fazem

- **Paraíba:** votos de presidente, governador, senador, deputado federal e deputado estadual nos 223 municípios. Ao clicar num município, aparecem os votos por escola (local de votação).
- **Bayeux:** votos de vereador, deputados, senador e governador **por bairro**, de 2012 a 2026.
- **Evolução do candidato:** quando a mesma pessoa disputou mais de uma eleição, o site mostra os votos em cada uma e, no mapa, onde ela ganhou e perdeu votos.

![Comparando a votação de um candidato entre eleições em Bayeux]({{ '/assets/img/blog/bayeux-comparar.gif' | relative_url }})

## A regra do jogo: nada que custe dinheiro

Antes de escrever código, defini restrições. Elas decidem toda a arquitetura:

| Peça | Solução | Custo |
|---|---|---|
| Fonte de dados | Portal de Dados Abertos do TSE, API de resultados do TSE, IBGE | R$ 0 |
| Banco de dados | Nenhum. Arquivos JSON commitados no repositório | R$ 0 |
| Backend | Nenhum. O site é 100% estático | R$ 0 |
| Atualização dos dados | GitHub Actions (disparo manual) | R$ 0 em repositório público |
| Hospedagem | GitHub Pages | R$ 0 |
| Bibliotecas | Preact + htm e TopoJSON via CDN, sem etapa de build | R$ 0 |

O ponto central é este: **o navegador de quem visita nunca chama o TSE**. Os dados são gerados uma vez, viram arquivos, e o site só lê arquivos. Isso elimina servidor, limite de requisições e dependência de uma API que pode cair no dia da eleição.

## Passo 1: de onde vêm os dados

Os resultados eleitorais são públicos, mas espalhados. O gerador (`scripts/gerar-dados.mjs`) combina três fontes, nesta ordem de preferência:

1. **CSV dos Dados Abertos do TSE**, com a votação nominal por município e zona. O presidente vem do arquivo nacional do mesmo .zip.
2. **API de resultados do TSE**, só para o que o CSV ainda não tem, como um segundo turno recém-apurado. A API guarda apenas o ciclo atual e o anterior, então ela não serve como fonte do histórico.
3. **CSV por seção eleitoral**, para o detalhe por local de votação.

Cada eleição vira um arquivo `ANO-tTURNO.json` em `public/data/eleicoes/`. Um `index.json` lista o que existe, e o site monta o menu de eleições a partir dele. Para ligar o mesmo candidato entre eleições, o gerador cria um `pessoas.json` usando o nome completo registrado no TSE.

Os arquivos por eleição ficam entre 1 e 2 MB. O detalhe por escola fica em arquivos separados (`ANO-tTURNO-locais.json`) que só são baixados quando o visitante clica num município. Assim a página inicial carrega rápido.

## Passo 2: coletar sem derrubar o servidor dos outros

Baixar dados públicos de um órgão do governo pede educação. O módulo `scripts/lib/tse.mjs` tem três cuidados simples:

```js
// poucas requisições em paralelo
export async function paralelo(itens, limite, fn) { /* ... */ }

// retentativa com espera crescente em erro de rede, 429 e 5xx
export async function getJson(url, { tentativas = 4, esperaMs = 1500 } = {}) { /* ... */ }

// arquivo grande vai para o disco e é reaproveitado nas próximas execuções
export async function baixar(url, destino) { /* ... */ }
```

O gerador também se identifica com um `User-Agent` próprio. Os .zip do TSE são lidos em streaming, linha a linha, com a biblioteca `fflate`, sem carregar tudo na memória. Os CSVs vêm em Windows-1252 com campos entre aspas e separados por ponto e vírgula, então o parser é feito à mão e sabe lidar com as aspas.

## Passo 3: a geometria do mapa

**Paraíba.** O contorno dos 223 municípios vem de um GeoJSON do IBGE (repositório `geodata-br`). Um script (`gerar-malha.mjs`) renomeia municípios cuja denominação mudou desde a malha de 2010 e passa tudo pelo `mapshaper`, que simplifica a geometria em 12% e exporta TopoJSON. O resultado é um arquivo pequeno, suficiente para desenhar o estado inteiro no navegador.

```js
execFileSync('npx', ['mapshaper', 'tmp/pb.geojson',
  '-simplify', '12%', 'keep-shapes',
  '-rename-layers', 'municipios',
  '-o', 'format=topojson', 'quantization=1e5', 'public/data/pb.topo.json']);
```

**Bayeux e a pergunta dos bairros.** Aqui apareceu o problema mais interessante: o TSE **não publica votos por bairro**. O que existe é:

- o voto por **seção eleitoral**;
- uma tabela "Eleitorado por local de votação", que diz, para cada seção, em qual escola ela funciona, em que bairro (`NM_BAIRRO`) e com quais coordenadas.

Juntando as duas, cada seção soma seus votos no bairro do local onde funciona. O site desenha um círculo por bairro, no centro dos locais de votação dele, com o tamanho proporcional aos votos do candidato. Por cima vai o contorno do município, vindo da API de malhas do IBGE.

Isso tem uma consequência que o site avisa e que vale repetir aqui: o mapa mostra **onde o voto foi depositado, não onde o eleitor mora**. É uma boa aproximação para comparar regiões da cidade, mas não é o endereço do eleitor.

## Passo 4: o front-end sem build

A pasta `public/` é o site. Não há Node em produção, nem webpack, nem etapa de build. O `index.html` carrega duas bibliotecas por CDN:

```html
<script src="https://cdn.jsdelivr.net/npm/htm@3.1.1/preact/standalone.umd.js"></script>
<script src="https://cdn.jsdelivr.net/npm/topojson-client@3.1.0/dist/topojson-client.min.js"></script>
<script src="js/app.js"></script>
```

O mapa é desenhado num `<canvas>` 2D. Para saber em qual município o visitante clicou, o código guarda o `Path2D` de cada um e testa o ponto com `isPointInPath`. Isso funciona bem para 223 polígonos e permite zoom e arrasto sem biblioteca de mapas.

Preact com `htm` dá a ergonomia de componentes sem JSX e sem compilar nada. Para um projeto desse tamanho (cerca de 600 linhas de JavaScript no app inteiro), é suficiente.

## Passo 5: publicar de graça

A publicação é um workflow de poucas linhas que envia a pasta `public/` para o GitHub Pages a cada `push` na `main`:

```yaml
- uses: actions/upload-pages-artifact@v3
  with:
    path: public
- uses: actions/deploy-pages@v4
```

Para as próximas eleições há um segundo workflow, de disparo manual: em **Actions → Atualizar dados de uma eleição → Run workflow**, informa-se o ano. Ele roda o gerador, commita os JSON novos e dispara a publicação. Em repositório público, o GitHub Actions e o Pages não cobram nada. O Pages tem limites (1 GB por site e uso moderado de banda), que um projeto como este, com cerca de 13 MB de dados, não chega perto de tocar.

## Como a IA entrou nisso

Eu não pedi "faça um mapa eleitoral" e esperei. O que funcionou foi trabalhar em passos pequenos e verificáveis, que o histórico de commits mostra bem:

1. um gerador que baixa e agrega os dados de **um** ano;
2. o mesmo gerador para vários anos;
3. o site lendo os arquivos gerados;
4. a lista de eleições montada sozinha, sem enviar arquivo;
5. votos por escola;
6. comparação entre eleições.

A divisão de trabalho que funcionou para mim: a IA escreve rápido a parte mecânica (parser de CSV, leitura de .zip, componentes, workflows), e eu decido o desenho, rodo, olho os números e volto com o erro ou a dúvida. O ciclo é curto: pedir, rodar, conferir, ajustar.

## O que a IA não resolve sozinha

Quase todos os problemas reais deste projeto estão nos dados, não no código. Os que apareceram:

- **Nomes de municípios mudaram** depois da malha do IBGE de 2010, então foi preciso um mapa de correção (`NOMES_ATUAIS`).
- **O CSV por seção vem com `#NULO#`** no nome de alguns locais em 2026, e o nome tem de ser buscado na tabela de locais.
- **Candidaturas anuladas** deixam votos que não devem entrar na soma por local.
- **O presidente não está no arquivo estadual**, só no nacional.
- **A API do TSE só guarda dois ciclos**, então o histórico precisa ser gerado e guardado por você.
- **Comparar cargos diferentes engana.** Em ano de dois senadores, cada eleitor vota duas vezes, então a variação de votos não mede crescimento de apoio. O site avisa quando você compara cargos ou turnos diferentes.

Nenhum desses pontos aparece se você só aceita o primeiro código gerado. Todos aparecem quando você confere o resultado contra os números oficiais. É por isso que a regra continua sendo a mesma de sempre: **LLM acelera, mas o resultado precisa ser verificável**. A verificação possível aqui é comparar totais por município e por bairro com o que o próprio TSE publica.

## Como repetir para a sua cidade

1. Escolha o recorte (município ou estado) e ache o código do TSE e o do IBGE dele.
2. Baixe um ano de votação por município e zona nos Dados Abertos do TSE e veja as colunas.
3. Escreva um gerador que transforme o CSV em JSON por eleição.
4. Baixe o contorno no IBGE e simplifique com `mapshaper`.
5. Faça uma página estática que leia os JSON e desenhe o mapa.
6. Publique a pasta no GitHub Pages.

Os dois repositórios estão abertos. Para uma nova cidade, o ponto de partida é o de Bayeux: troque o código do município e rode `npm run dados` e `npm run eleicoes`.

## O que fica

Há alguns anos, um projeto assim exigiria servidor, banco, deploy e uma conta mensal. Hoje cabe em uma pasta de arquivos estáticos, em um repositório público e em pouco mais de um dia de trabalho com um assistente de código. Dados abertos existem, hospedagem estática é gratuita, e um LLM encurta a distância entre ter a ideia e ter o site no ar. O trabalho que sobra, e que importa, é entender os dados e conferir o resultado.
