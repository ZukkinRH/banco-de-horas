# Banco de Horas — Zukkin

Painel interno do RH da Zukkin para acompanhar o banco de horas dos colaboradores por
setor e enviar um relatório semanal por e-mail aos gestores de cada setor.

Usuário principal: **Rodrigo Higino** (RH, rodrigo.higino@zukkin.com). Ele não é
desenvolvedor — explique mudanças em linguagem simples e evite jargão desnecessário.

---

## 1. O que é este repositório

Um **único arquivo `index.html`** (~2.300 linhas) contendo HTML, CSS e JavaScript.
Não há build, bundler, `package.json`, dependências npm nem etapa de compilação.
Editar `index.html` e dar push é o deploy inteiro.

- **Repositório**: `ZukkinRH/banco-de-horas`
- **Site publicado** (GitHub Pages, branch `main`, raiz): https://zukkinrh.github.io/banco-de-horas/
- O Pages leva ~30–60s para refletir um push.

**Não** quebre isso em vários arquivos (CSS/JS separados) sem o Rodrigo pedir — a
escolha de arquivo único foi deliberada, para manter o deploy trivial.

## 2. Como os dados entram e saem

```
Planilha Google Sheets  ──(export xlsx, lido no navegador com SheetJS)──►  index.html
                                                                              │
                                       estado salvo ◄──── Firebase Firestore ─┤
                                                                              │
                             POST fire-and-forget ──► Google Apps Script ──► Gmail ──► gestores
```

1. **Origem dos dados**: uma planilha do Google Sheets pública por link, lida
   automaticamente toda vez que o painel abre. O RH mantém essa planilha; o painel
   apenas lê. Não há escrita de volta na planilha pelo site.
2. **Persistência**: Firebase Firestore guarda o estado do app (e-mails de gestores,
   configurações, histórico de semanas importadas) num documento único.
3. **Envio de e-mail**: um Web App do Google Apps Script recebe um POST com o payload
   do relatório e dispara o e-mail via `GmailApp`.

### Identificadores reais (já em produção)

| O quê | Valor |
|---|---|
| Google Sheets `fileId` | `14jGeC75wE64upU4rvOrI75zZoQi3AhSxtZEgyT1Muwk` (título "BH - Zukkin") |
| Firebase project | `zukkin-banco-de-horas` (região São Paulo) |
| Documento do Firestore | coleção `estado`, documento `main` |
| Apps Script — projeto | `1MvDDSTLKa4fS-SDDZR7rvre92zD2pfEWoFxU1hvb5obVhS5PLN54CIyQ` |
| Apps Script — URL do Web App | `https://script.google.com/macros/s/AKfycbzewCkowYfB1stBueYUGC7Y8EQXobZSGonAKpIIJG0k8srB5HlZ_cCRy1p9UJdVn3EZ/exec` |

A URL do Apps Script **não** fica hardcoded no código: o Rodrigo cola ela em
Configurações → "URL do fluxo HTTP", e ela é persistida no Firestore.

## 3. Arquitetura interna do `index.html`

O JS está dividido em módulos IIFE, cada um num bloco `<script>` próprio, nesta ordem:

| Módulo | Linhas aprox. | Responsabilidade |
|---|---|---|
| `Data` | 453–560 | Estado global, setores, config, persistência no Firestore |
| `Calc` | 567–720 | Cálculos de saldo, variação e situação (positivo/negativo/atenção/zerado) |
| `History` | 731–776 | Navegação entre semanas |
| `Import` | 794–999 | Leitura e parsing da planilha (SheetJS) |
| `Report` | 1006–1113 | Montagem do relatório por setor, HTML do preview, export CSV |
| `EmailModule` | 1123–1183 | Chamada HTTP ao serviço de envio |
| `Charts` | 1190–1307 | Gráficos (Chart.js) |
| `App` | 1320–2304 | Roteamento de telas e toda a renderização |

Telas (renderizadas por `App`): Dashboard, Setor, Histórico, Comparador, Relatórios,
E-mails, Configurações.

**Setores reais (7, fixos)**: Qualidade, Operações, Projetos, RH, Desenvolvimento,
Zpromo, Zrobot. Estão hardcoded em `Data.SETORES`. Já houve um bug de assumir um
setor "Produtos" que não existe — não invente setores.

### Formato da planilha (o parser depende disso)

A planilha **não** é uma tabela única. São mini-tabelas empilhadas e às vezes lado a
lado, uma por setor:

```
Banco de Horas - Qualidade          Banco de Horas - Zpromo
Colaborador | Cargo | Saldo Atual   Colaborador | Saldo Atual
Fulano      | ...   | 01:46         Beltrano    | 00:23
Sicrano     | ...   | -00:12        (bloco pode acabar antes do outro)
(termina na primeira linha em branco)
```

O parser (`Import`) localiza cada bloco pelo título `Banco de Horas - <Setor>`, lê a
linha de cabeçalho seguinte e consome linhas até a primeira em branco. Blocos lado a
lado de tamanhos diferentes são suportados.

A planilha só traz o **saldo atual**. O sistema deriva o resto: saldo anterior = último
saldo conhecido daquele colaborador; gestor = gestor cadastrado do setor. A "Data Base"
que aparece em alguns blocos é **ignorada de propósito** (fica inconsistente entre
setores) — a semana de referência vem da escolha do usuário.

## 4. Armadilhas conhecidas (leia antes de mexer)

**`mode: 'no-cors'` no envio é obrigatório.** O Apps Script responde a um POST com um
redirect 302 para uma URL `script.googleusercontent.com/macros/echo?...` que só aceita
GET/HEAD. O navegador converte POST→GET ao seguir o redirect e a resposta final não
passa no CORS, causando `TypeError: Failed to fetch` — mesmo com o e-mail tendo sido
enviado com sucesso no servidor. Por isso a chamada é **fire-and-forget**: em `no-cors`
a resposta é opaca e qualquer chamada que não lance erro de rede é tratada como sucesso.
Não "conserte" isso voltando para `mode: 'cors'` ou lendo `resp.ok`.

**Não envie header `Content-Type: application/json`.** Isso dispara um preflight OPTIONS
que o Apps Script não trata. O corpo vai como `text/plain` e o servidor faz `JSON.parse`
normalmente.

**O campo se chama `powerAutomateUrl` mas é a URL do Apps Script.** Nome legado: a
primeira tentativa usou Power Automate, que exigia licença Premium que o tenant da
Zukkin não tem. Migrou-se para Apps Script (gratuito), e o nome do campo foi mantido de
propósito para minimizar o diff. Renomear exigiria migrar o documento do Firestore —
não faça sem combinar com o Rodrigo.

**Nada é enviado automaticamente.** Todo envio passa por uma tela de confirmação que
mostra destinatários e conteúdo exato. Essa garantia foi pedida explicitamente pelo
Rodrigo — preserve-a em qualquer refatoração.

**Modo de teste.** Quando ligado, todos os envios vão para o e-mail de teste configurado
em vez dos gestores reais. Deve continuar existindo e ficar visível na tela de
confirmação.

**Não há tela de login.** O app faz `signInAnonymously()` no Firebase apenas para as
regras do Firestore funcionarem. Ninguém digita senha. Isso foi uma decisão explícita.

## 5. Validar antes de dar push

Não existe suíte de testes. A verificação mínima obrigatória é conferir que todos os
blocos `<script>` continuam sintaticamente válidos:

```bash
node -e "
const fs=require('fs');
const html=fs.readFileSync('index.html','utf8');
const blocks=[...html.matchAll(/<script>([\s\S]*?)<\/script>/g)];
let ok=true;
blocks.forEach((b,i)=>{try{new Function(b[1])}catch(e){ok=false;console.log('ERRO no bloco',i,e.message)}});
console.log(blocks.length+' blocos —', ok?'ALL OK':'FALHOU');
"
```

Deve reportar **9 blocos — ALL OK**. Se o número de blocos mudar, algo foi adicionado
ou removido: confirme se foi intencional.

Depois do push, aguarde ~30–60s e confirme que o Pages atualizou antes de testar:

```bash
curl -s https://zukkinrh.github.io/banco-de-horas/ | grep -c "no-cors"
```

## 6. Fluxo operacional semanal (contexto, não código)

Toda segunda-feira:

1. Extrai-se o relatório "Banco de horas (período)" da plataforma **VR Ponto Mais**
   (`app2.pontomais.com.br` → Relatórios). Ele traz `Nome` e `Saldo BH` por colaborador.
2. Esses saldos alimentam a planilha do Google Sheets.
3. O painel lê a planilha automaticamente ao abrir.
4. O Rodrigo revisa em "Relatórios" e clica em "Enviar relatório de \<Setor\>" para cada
   setor, confirmando cada envio.

Os passos 1–2 e o aviso do passo 3 estão automatizados como tarefas agendadas no Cowork
(`vr-pontomais-extracao-semanal` às 8h05 e `banco-de-horas-aviso-semanal` às 9h de
segunda) — **fora deste repositório**. Claude Code não controla navegador; se a tarefa
envolver VR Ponto Mais, Firebase Console ou Google Sheets pela interface, oriente o
Rodrigo a fazer no Cowork.

## 7. Pendências conhecidas

- Cadastrar os e-mails reais dos gestores dos 7 setores na tela "E-mails" (hoje vazios)
  e só então desligar o Modo de teste.
- Existe um fluxo órfão no Power Automate ("Http -> Criar Tabela HTML,Enviar um email
  (V2)") que ficou inutilizável por falta de licença. Não é usado por nada; pode ser
  apagado pelo Rodrigo.

## 8. Convenções

- Toda a interface é em **português do Brasil**. Mantenha textos, rótulos e mensagens
  de erro em português.
- Comentários no código também em português.
- Saldos são exibidos no formato `+01h46` / `-00h12`; a planilha usa `01:46` / `-00:12`.
- Identidade visual Zukkin: cabeçalho escuro (`#17191c`), vermelho de destaque
  (`--red`), tipografia sem serifa. O e-mail enviado segue o mesmo padrão.
- Ao commitar, use mensagens descritivas em português, sem acentos (o histórico atual
  segue esse padrão).
