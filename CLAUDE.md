# Monitor — site do projeto Rastreio

Este repositório é só a metade "site" de um projeto de duas metades. A outra
é o `rastreio` (app Android + Supabase), clonado ao lado como repositório
irmão. **As decisões de produto, o histórico e o contexto de negócio moram
lá, não aqui:**

- `CLAUDE.md` do `rastreio` — a especificação: os oito itens que definem
  traçado, gravação, sinal, farol, monitor, timeline, cercas e os campos da
  lista. É o que este site implementa do lado do navegador.
- `HISTORICO.md` do `rastreio` — arquitetura, diagnósticos já resolvidos,
  modelo de segurança, mapa do código do app.
- `CONTEXTO-VTU.md` do `rastreio` — para que o projeto existe (pátio de
  manobras da VLI em Vitória) e a natureza de piloto da infraestrutura.

Leia os três antes de propor qualquer mudança aqui. Uma alteração que faz
sentido olhando só este arquivo pode contradizer uma decisão de produto que
está registrada do outro lado.

## O que tem neste repositório

Um arquivo só: `index.html`, ~2100 linhas, publicado direto no GitHub Pages.
Sem `package.json`, sem empacotador, sem build. É por isso que o nome
precisa continuar exatamente `index.html`, minúsculo, na raiz — GitHub Pages
dá 404 em qualquer outra grafia.

Por dentro, `index.html` tem três partes que não devem ser confundidas:

1. **Cabeçalho e CSS** (topo do arquivo) — fontes, cores (`--vivo`, `--morno`,
   `--frio`, `--parado`, `--andando`), a tela de acesso (`#acesso`).
2. **Bloco `NUCLEO:INICIO` … `NUCLEO:FIM`**, dentro do primeiro `<script>` —
   gerado, **nunca edite aqui**. Ver seção seguinte.
3. **Segundo `<script>`**, depois do núcleo — código específico do site:
   `SUPABASE_URL`/`SUPABASE_KEY`, estado em memória (`aparelhos`, `historico`,
   `cercas`, `vinculos`, `instante` da timeline), a função `rpc()`, montagem
   do mapa Leaflet, pins, balões, desenho de cerca, timeline arrastável,
   painel de parede. É aqui que qualquer mudança de comportamento do *site*
   (e só do site) acontece.

## O núcleo compartilhado — regra que não se contorna

As regras de traçado, farol, sinal, status, fuso horário e cercas (itens 1,
3, 4, 7 e 8 do `CLAUDE.md` do `rastreio`) precisam dar a MESMA resposta no
app e no site. Por isso `src/trilha.js`, `src/tempo.js`, `src/paradas.js`,
`src/vinculos.js` e `src/cercas.js` são escritos **no repositório `rastreio`**
e chegam até aqui via `node ferramentas/embutir.js`, que despeja os cinco
arquivos entre os marcadores `NUCLEO:INICIO` / `NUCLEO:FIM` deste
`index.html`.

**Se a mudança é em traçado, farol, sinal, status, fuso ou cerca: o trabalho
começa no `rastreio`, não aqui.** Editar dentro dos marcadores à mão resolve
na hora e se perde na próxima geração — ninguém vai lembrar de reaplicar.

Sequência completa, rodada de dentro do repositório `rastreio` (que precisa
estar clonado ao lado deste, como `../monitor`):

    node ferramentas/embutir.js    # leva o núcleo para cá
    node ferramentas/conferir.js   # barra nome repetido (tela em branco sem erro)
    node ferramentas/prova.js      # roda as contas com dados de teste
    node ferramentas/render.js     # abre a página num navegador e olha se pintou

Só depois disso o `index.html` atualizado é commitado **neste** repositório.
`conferir.js` existe porque o trecho embutido vira um `<script>` no mesmo
escopo global do resto da página: um `const` com nome repetido nos dois lados
derruba a página inteira, sem erro visível além da tela em branco.

Se a mudança é só de UI do site — cor, layout, texto de balão, comportamento
da timeline, desenho da cerca no mapa — ela é local, no segundo `<script>`
deste `index.html`, e não precisa do `rastreio` nem do `embutir.js`.

## `render.js`: o único que abre a página

`conferir.js` e `prova.js` olham o arquivo como TEXTO. Nenhum dos dois carrega
a página, e por isso os dois passam com folga num `index.html` que não pinta
nada. Foi o que aconteceu em 10/08/2026: dois commits passaram nos dois e
deixaram o monitor sem um único pin **no ar**, porque publicar na `main` é
publicar em produção.

`render.js` carrega o `index.html` num Chromium de verdade, com dados
inventados servidos no lugar do Supabase, e confere três coisas: apareceu
ativo na lista, apareceu pin no mapa, e a localidade saiu pela cerca **menor**.
Qualquer erro de JavaScript vira falha — inclusive os que só acontecem no
`setInterval`, que é onde o painel de parede morre calado.

**Ele roda a página DUAS vezes, e a segunda é a que importa:** uma fingindo um
servidor completo, e outra fingindo um servidor **sem** `listar_contatos` —
que é o estado real sempre que um script de `sql/` foi escrito e ainda não foi
colado no painel do Supabase. Foi exatamente esse o defeito de 10/08: a
variável `contatoDe` só nascia dentro de um `try/catch` silencioso, e com a RPC
faltando ela nunca nascia.

**A regra que fica:** o servidor sempre vai atrás do site. Se o painel quebra
quando uma RPC nova ainda não existe lá, ele está quebrado — e a prova precisa
cobrir a ausência dela, não só a presença.

Precisa do `playwright` instalado. Numa máquina que já tem o Chromium por fora
(o contêiner do Claude Code tem), aponte com `CHROMIUM=/caminho/do/chrome`. O
Leaflet é baixado uma vez e guardado na pasta temporária, então depois da
primeira execução a prova roda offline. Se não conseguir o Leaflet, ela falha
dizendo que NÃO rodou — nunca finge que passou.

## Deploy

Não existe build. Um `git push` para `main` publica sozinho, via GitHub
Pages — normalmente em menos de um minuto. Diferente do app (que exige
reinstalar o APK ou publicar update), o site atualiza para todo mundo assim
que o navegador recarrega a página. Não há ambiente de teste separado:
publicar em `main` é publicar em produção. Por isso o passo de conferência
acima (`conferir.js`, `prova.js`) é o que existe no lugar de testes
automatizados de CI.

## O cache do navegador esconde a publicação

Publicar em `main` põe o arquivo novo no ar em menos de um minuto — mas o
navegador de quem já visitou continua servindo a versão guardada, e
`Ctrl+Shift+R` nem sempre vence.

Aconteceu em 10/08/2026: uma versão com defeito foi publicada, revertida em
seguida, e o dono continuou vendo o painel vazio por um bom tempo. O
arquivo no ar já era o certo — conferido por md5 contra o `main` — e o que
ele via era o antigo, do cache.

**Como confirmar que é cache, e não o site:** abrir numa janela anônima.
Ela ignora o cache por completo. Se funcionar ali, é cache; se não, é o
site de verdade.

**Por que isto importa mais do que parece:** o painel de parede
(`CONTEXTO-VTU` 5.4) fica aberto o dia inteiro, sem ninguém recarregando.
Uma correção publicada pode simplesmente não chegar nele até alguém fechar
e abrir o navegador — e ninguém vai desconfiar disso, porque a página
"está funcionando".

## Byte nulo: a armadilha que já aconteceu neste arquivo

`chaveDe`/`nomeDe`/`grupoDe`, perto do topo do segundo `<script>`, separam
grupo e aparelho com `"\u0000"` — de propósito, porque nome de aparelho pode
ter espaço. **Esse byte precisa ficar escrito como o escape de seis
caracteres `\u0000`, nunca como byte cru.** Byte cru faz o arquivo ser
classificado como binário: `grep` responde "nenhuma ocorrência" em vez de dar
erro, e o byte some numa cópia entre sistemas sem avisar — já aconteceu neste
`index.html` antes. Se uma ferramenta (ou um editor automatizado) gravar o
byte cru por engano, `node ferramentas/corrigir-nulo.js index.html`, no
repositório `rastreio`, troca de volta pelo escape.

## Chave do Supabase exposta no código — é intencional

`SUPABASE_URL` e `SUPABASE_KEY` aparecem em texto puro nas primeiras linhas
do segundo `<script>`. Não é descuido: é a chave `anon public`, feita para
ficar em código de cliente. Quem protege o dado são as tabelas trancadas com
RLS e as funções RPC security-definer que conferem senha do lado do banco —
ver "Modelo de segurança" no `HISTORICO.md` do `rastreio`. Não troque essa
chave por variável de ambiente nem tente escondê-la: não existe processo de
build aqui para injetar segredo, e a proteção real está no banco, não no
sigilo da chave.

## Como trabalhar neste projeto

Mesmo dono, mesma forma de trabalhar do repositório `rastreio`: ele decide o
que fazer e confere o resultado como PO, não é desenvolvedor e fala
português.

**Nunca trabalhe direto na `main`** — aqui isso vale ainda mais que no app,
porque não há build nem tela de teste separada: um push errado em `main` fica
no ar imediatamente para quem estiver olhando o painel. Um branch por passo.
Commite a cada mudança coerente, mensagem em português dizendo **o que
mudou e por quê**.

**O merge é seu, não dele** (decidido pelo dono em 11/08/2026, substituindo a
regra anterior, que exigia ele confirmar ter testado o site antes). Terminou e
passou nas provas? Mergeia — nada de branch de pé esperando autorização.

Aqui isso pesa mais que no app, porque **merge é publicação imediata**: em
menos de um minuto está no ar para quem estiver olhando o painel. Por isso as
duas obrigações que substituem o portão humano:

- **`node ferramentas/render.js` antes de todo merge**, sem exceção. É a única
  prova que abre a página de verdade; `conferir.js` e `prova.js` passam com
  folga num `index.html` que não pinta nada. Sem o portão do dono, essa prova
  é a única rede que sobrou.
- **Dizer na resposta o que mudou na tela dele.** Ele descobre a mudança
  abrindo o painel, não lendo o diff.

Toda resposta termina com um resumo curto separando o que está decidido do
que depende dele: decisão pendente, comando para rodar, ou coisa que ficou
fora do escopo.
