# chamada

Automatização de chamada para registro de frequência em sala de aula.

Aplicativo de arquivo único (`index.html`). Não precisa de build nem de servidor próprio:
basta publicar o arquivo (GitHub Pages, por exemplo) e abrir a URL.

## Como funciona

O professor seleciona a turma e inicia a chamada. A tela de projeção exibe um QR code e um
código de 6 caracteres que gira a cada 2 minutos. Os alunos abrem o link, escolhem o curso e
digitam o nome completo. Ao encerrar, o app cruza os registros com a lista oficial da turma e
separa **presentes**, **faltosos**, **não identificados** e **duplicidades**.

## Onde ficam os dados

Tudo fica no Firebase Realtime Database, em dois espaços com proteções diferentes:

| Caminho | Conteúdo | Acesso |
|---|---|---|
| `cd_sessions`, `cd_att` | sessões e o que os alunos digitaram | leitura e escrita públicas — os alunos precisam gravar ali |
| `cd_priv/<chave>` | listas oficiais, cursos e histórico das chamadas | só quem tem a chave exata |

O `localStorage` do navegador guarda uma cópia de trabalho, para o app funcionar sem rede.

### ⚠ Regras do banco — obrigatório

As regras precisam **deixar de ser abertas na raiz**, senão a chave não protege nada. Em
*Realtime Database → Regras*, use:

```json
{
  "rules": {
    "cd_sessions": { ".read": true, ".write": true },
    "cd_att":      { ".read": true, ".write": true },
    "cd_priv": {
      "$chave": { ".read": true, ".write": true }
    }
  }
}
```

Como não há `.read` na raiz nem em `cd_priv`, ninguém consegue **listar** as chaves existentes:
só alcança os dados quem informa a chave inteira. São 32 caracteres aleatórios, o que torna a
adivinhação inviável — mas é proteção por segredo, não por autenticação. Quem receber a chave
(ou tiver acesso ao navegador do professor) alcança tudo.

`cd_att` continua público: nomes digitados pelos alunos, horários e a impressão digital do
aparelho ficam legíveis para quem tiver a URL do app. Isso já valia na versão anterior.

### Chave de sincronização

Gerada na primeira vez e visível em **Configurações**. Para usar outro computador: copie a
chave lá e cole em *Configurações → Usar outra chave*. Trate-a como uma senha.

**Turmas → Exportar backup (.json)** salva turmas, cursos, histórico e a chave num arquivo.
Guarde-o fora do repositório.

## Cadastrar turmas

Em **Turmas**, cole a lista copiada do sistema da UNI7. O parser aceita:

```
Semiótica (Publicidade e Propaganda)
Código: GSER032901
Turma: CSE0010204DNA
Carga Horária: 40h
Alunos (27):
   * Alicia Barros Silva (56054893)
   * Ana Beatriz Brito de Menezes (56054628)
```

Também aceita várias turmas de uma vez, listas simples de nomes (um por linha) e os formatos
`56054893 Nome`, `56054893;Nome` e `Nome;56054893`. Nada é gravado sem a sua confirmação na
prévia.

O identificador da turma é `Código-Turma` (ex.: `GSER032901-CSE0010204DNA`), porque o mesmo
código de turma se repete em disciplinas diferentes.

## Como os nomes são reconhecidos

Nomes são normalizados (maiúsculas, acentos, espaços e partículas como "de/da/dos" são
ignorados). A partir daí, em camadas:

| Situação | Resultado |
|---|---|
| O registro traz matrícula (formato antigo) e ela existe na lista | presente, automático — chave exata |
| Nome completo igual ao da lista | presente, automático |
| Nome abreviado, candidato único na turma, **mesmo primeiro nome e mesmo último sobrenome** | presente, marcado como "aproximado" e listado para conferência |
| Erro de digitação, nome fora de ordem, ou só iniciais | vai para decisão manual; o aluno segue faltoso até você confirmar |
| Corresponde a mais de um aluno | vai para decisão manual |
| Ninguém corresponde | não identificado |

Ninguém entra como presente por semelhança frouxa. Dois registros que apontem para o mesmo
aluno contam uma vez só; os extras aparecem em **duplicidades**.

## Turmas reunidas

Quando a aula junta duas turmas que o sistema da UNI7 mantém separadas — por exemplo
Jornalismo e Publicidade em Atividades Práticas —, marque **as duas** ao iniciar a chamada, ou
use *Vincular turmas* numa chamada antiga.

O app cruza os nomes com as duas listas juntas e depois separa os faltosos por turma, cada bloco
com seu próprio botão de copiar — que é como a chamada é lançada no sistema. Ninguém precisa
informar o curso: a turma de cada aluno vem da lista oficial, não do que ele digita.

### Grupos salvos

Em *Turmas → Turmas reunidas → Novo grupo* você salva a combinação com um nome. Ela passa a
aparecer como um botão único ao iniciar a chamada e ao vincular uma chamada antiga: um clique
marca todas as turmas do grupo. Ao salvar, o app avisa se houver nomes completos repetidos
entre as listas, porque esses alunos cairiam sempre na conferência manual.

Vale conferir que a união não cria nomes ambíguos. Nas turmas reunidas de 2026.2 não cria: nenhum
nome completo se repete entre as listas e nenhum aluno está nas duas.

## Aluno que assiste em outra turma

Em *Turmas → Exceções* você marca, aluno a aluno, a turma onde ele de fato assiste — o caso do
aluno matriculado à noite que, por acordo com a coordenação, frequenta a turma da manhã.

Efeito: ele sai da conferência da turma de matrícula e passa a ser reconhecido na turma onde
assiste, sem nunca contar falta na que não frequenta. A presença continua sendo atribuída à
**turma de matrícula**, porque é lá que a chamada é lançada no sistema da UNI7. Na tela de
Frequência ele aparece com a marca *outra turma*, e o percentual dele é calculado sobre as
chamadas da turma que ele assiste.

Reimportar a lista da UNI7 não apaga essas exceções.

## Frequência acumulada

A aba **Frequência** soma todas as chamadas salvas de uma turma e mostra, por aluno, presenças,
faltas, as datas em que faltou e o percentual. O limite mínimo é configurável na própria tela
(padrão 75%) e quem fica abaixo aparece marcado, com botão para copiar a lista e exportar uma
planilha em CSV — uma coluna por chamada, com P ou F.

Só entram no cálculo as chamadas que você **salvou no histórico** na tela de conferência.

## Recuperar aulas antigas

Sessões feitas antes desta versão não têm turma vinculada. No painel, elas trazem o botão
**"Vincular turmas"**: marque uma ou mais turmas e o app cruza os registros já gravados com as
listas oficiais, retroativamente.

Esses registros antigos guardavam a matrícula do aluno, e a matrícula é chave exata — a
conciliação deles é mais confiável que a por nome. Depois de vincular, confira e salve no
histórico para a aula entrar na frequência.

## Documentos nos registros antigos

Enquanto o formulário pedia matrícula, muitos alunos digitaram o **CPF** — e esses números
ficaram no `cd_att`, que é público. Em *Configurações → Manutenção* há uma ferramenta que
varre todos os registros, separa os números que não correspondem a nenhuma matrícula das suas
turmas, baixa um backup em CSV para o seu computador e então os apaga do Firebase. Nomes e
horários dos registros continuam lá.

Importe todas as turmas antes de rodar: o que define um número como legítimo é ele bater com
alguma matrícula oficial. Registros de turmas não importadas seriam considerados estranhos.

A versão atual não pede documento nenhum, então o problema não volta a acontecer.

## Configurações

- **Cursos** do formulário do aluno: lista editável, um por linha.
- **Senha do professor**: padrão `chamada`, guardada apenas neste navegador.
- **Firebase**: o projeto padrão está embutido em `A.init()`; para trocar, edite o objeto `cfg`.

## Limitações conhecidas

- O bloqueio de registro duplicado é por navegador, não por aparelho: quem limpar os dados do
  site ou abrir uma aba anônima consegue registrar de novo. A duplicidade continua sendo pega no
  cruzamento com a lista oficial, que é onde ela importa.
- O Firebase não guarda listas vazias: uma chamada sem "não identificados" volta do banco sem
  esse campo. Toda leitura passa por `comoLista()` — se você acrescentar campos de lista novos,
  normalize-os também, ou o histórico quebra ao recarregar.
- A senha do professor é verificada no navegador; não protege contra quem inspecionar o código.
  Ela não guarda relação com a chave de sincronização — trocar uma não troca a outra.
- `cd_att` é público por necessidade: os alunos precisam escrever ali sem autenticação.
- As listas oficiais dependem da chave permanecer secreta e das regras acima estarem aplicadas.
- A identificação é por nome. Duas pessoas com o mesmo primeiro nome e o mesmo último sobrenome
  são indistinguíveis — por isso as presenças "aproximadas" ficam sempre visíveis para conferir.
