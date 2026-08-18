# Banco de Horas — Zukkin · GitHub + Firebase · Passo a passo

Este pacote contém:

- `index.html` — a aplicação completa (dashboard, 7 setores, histórico, relatórios com confirmação manual de envio, e-mails, configurações). Sem tela de login — quem abre o link entra direto.
- `firebase.json`, `firestore.rules`, `firestore.indexes.json` — configuração do banco de dados (Firestore).
- `SETUP.md` — este guia.

O site é hospedado no **GitHub Pages** (arquivo estático, sem servidor próprio) e os dados (colaboradores, histórico semanal, gestores/e-mails cadastrados, configurações) ficam salvos num banco de dados real no **Firebase (Firestore)** — assim qualquer computador que abrir o link vê e edita os mesmos dados, em vez de cada navegador guardar sua própria cópia.

A planilha "BH - Zukkin" continua no seu Google Drive normalmente. O painel busca os dados dela automaticamente a cada abertura, sem você precisar importar nada — isso só é possível porque a planilha está compartilhada como **"Qualquer pessoa com o link pode visualizar"** (já está assim hoje). Isso significa que qualquer pessoa que tiver o link da planilha (não o link do painel — o link da própria planilha) também conseguiria abri-la direto no Google Sheets. Se em algum momento isso deixar de ser aceitável, dá pra voltar para importação manual (o painel já tem essa opção em Configurações) e tornar a planilha privada de novo.

---

## 1. Criar o projeto no Firebase (o banco de dados)

1. Acesse [console.firebase.google.com](https://console.firebase.google.com) → **Adicionar projeto**.
2. Dê um nome (ex: `zukkin-banco-de-horas`) e conclua. O plano gratuito **Spark** é suficiente.
3. **Compilação → Authentication** → **Vamos começar** → habilite o provedor **Anônimo** ("Anonymous"). Isso é só para as regras do Firestore funcionarem — ninguém vê tela de login.
4. **Compilação → Firestore Database** → **Criar banco de dados** → escolha uma região (ex: `southamerica-east1`) → **modo de produção** (as regras deste pacote já cuidam da segurança).

## 2. Pegar as credenciais do app (`firebaseConfig`)

1. ⚙️ **Configurações do projeto** → em **Seus apps**, clique no ícone `</>` (Web) para registrar um app (não precisa marcar Hosting).
2. Copie o objeto `firebaseConfig` mostrado na tela.
3. Abra `index.html`, localize o bloco perto do topo:

   ```js
   const firebaseConfig = {
     apiKey: "SUA_API_KEY_AQUI",
     authDomain: "seu-projeto.firebaseapp.com",
     projectId: "seu-projeto",
     ...
   };
   ```

   e substitua pelos valores reais.

## 3. Publicar as regras do Firestore

Requer [Node.js](https://nodejs.org) instalado.

```bash
npm install -g firebase-tools
firebase login
cd caminho/da/pasta/deste/pacote
firebase use --add        # selecione o projeto criado no passo 1
firebase deploy --only firestore:rules,firestore:indexes
```

## 4. Criar o repositório no GitHub e publicar (GitHub Pages)

Eu não tenho acesso à sua conta do GitHub, então esta parte é com você (ou peça pra alguém do time de TI):

1. Crie um repositório novo no GitHub (pode ser público — o GitHub Pages gratuito exige repositório público).
2. Suba os arquivos deste pacote (`index.html`, `firebase.json`, `firestore.rules`, `firestore.indexes.json`) para o repositório — não precisa de nenhuma etapa de build, é um site estático simples:

   ```bash
   cd caminho/da/pasta/deste/pacote
   git init
   git add .
   git commit -m "Painel de banco de horas"
   git branch -M main
   git remote add origin https://github.com/SEU-USUARIO/SEU-REPOSITORIO.git
   git push -u origin main
   ```

3. No GitHub: **Settings → Pages** → em "Build and deployment", selecione **Deploy from a branch** → branch **main**, pasta **/ (root)** → **Save**.
4. Em alguns minutos o GitHub mostra a URL pública (algo como `https://seu-usuario.github.io/seu-repositorio/`). É esse link que o RH vai acessar — trate-o como um link "só para RH", do mesmo jeito que o link de uma planilha interna.

## 5. Cadastrar os gestores dos 7 setores

No painel, vá em **E-mails** e cadastre nome + e-mail do gestor de cada setor (Qualidade, Operações, Projetos, RH, Desenvolvimento, Zpromo, Zrobot). Isso fica salvo no Firestore — qualquer computador que abrir o painel vê e pode editar.

## 6. Configurar o envio de e-mail (Power Automate)

Igual à versão anterior: crie um fluxo de nuvem instantâneo no Power Automate com o gatilho **"Quando uma solicitação HTTP for recebida"**, aceitando este corpo:

```json
{
  "setor": "string",
  "semana": "string",
  "modoTeste": true,
  "destinatarios": ["string"],
  "geradoEm": "string",
  "colaboradores": [
    { "nome": "string", "cargo": "string", "saldoAtual": 0 }
  ]
}
```

Adicione a ação **"Enviar um e-mail (V2)"** usando `destinatarios` e percorrendo `colaboradores` para montar o corpo. Copie a URL HTTP POST gerada e cole em **Configurações → URL do fluxo HTTP** no painel.

**Nenhum e-mail é enviado automaticamente.** Ao clicar em "Enviar relatório de `<Setor>`" na tela Relatórios, o painel sempre abre uma tela de confirmação mostrando os destinatários exatos e a tabela completa do e-mail — só depois de clicar em "Confirmar e enviar" a chamada HTTP é feita.

### Seu e-mail e o modo de teste

Em **Configurações**, cadastre "Seu e-mail (RH)". Marcando **Modo de teste**, todo envio é redirecionado para esse e-mail em vez do gestor real — útil para conferir o formato antes de ativar de verdade.

## 7. Testar

1. Acesse a URL do GitHub Pages — o painel abre direto, sem login, e já começa a buscar a planilha automaticamente.
2. Confira em **Dashboard** e nos **Setores** se os saldos aparecem certos.
3. Em **Relatórios**, marque **Modo de teste**, clique em "Enviar relatório de `<Setor>`", confirme os destinatários na tela que abrir e clique em "Confirmar e enviar" — o e-mail deve chegar só no seu e-mail de teste.
4. Abra o painel de outro computador (ou peça para outra pessoa do RH abrir) e confirme que os gestores/e-mails cadastrados aparecem lá também — é o sinal de que o Firestore está funcionando como banco de dados compartilhado.

---

## Sobre a importação automática

A cada abertura do painel (e ao clicar em "Atualizar agora"), o navegador baixa a exportação pública em `.xlsx` da planilha "BH - Zukkin" (`https://docs.google.com/spreadsheets/d/.../export?format=xlsx`) e faz o parsing das mini-tabelas "Banco de Horas - `<Setor>`" em cada aba (Qualidade, Operações, Produtos [que contém Zpromo e Zrobot lado a lado], Projetos, Desenvolvimento, RH). Já testei esse parser contra a planilha real: reconhece corretamente os 60 colaboradores nos 7 setores, incluindo saldos negativos e as duas colunas mescladas (Desenvolvimento e RH têm coluna "Cargo").

Se por algum motivo a planilha deixar de estar acessível publicamente (ex.: você reverte o compartilhamento), a atualização automática vai falhar com um aviso — nesse caso, use o campo "importar arquivo manualmente" em **Configurações**: baixe a planilha do Google Sheets como `.xlsx` (Arquivo → Fazer download → Microsoft Excel) e selecione o arquivo ali.

Apenas os 7 setores reais são reconhecidos: **Qualidade, Operações, Projetos, RH, Desenvolvimento, Zpromo, Zrobot**. Qualquer bloco com um nome de setor diferente é ignorado e aparece como aviso.
