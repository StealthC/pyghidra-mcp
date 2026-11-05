# Ghidra Server e projetos compartilhados: funcionamento e requisitos para o pyghidra-mcp

> Levantamento técnico baseado na implementação do Ghidra 12.1.2 disponível no ambiente\
> (`$GHIDRA_INSTALL_DIR`), em especial nas classes `GhidraProject`, `DefaultProjectManager`,\
> `DefaultProjectData`, `DomainFile`, `RepositoryAdapter`, `GhidraURLQuery`,\
> `TransientProjectManager`, `HeadlessAnalyzer` e `HeadlessClientAuthenticator`, além dos\
> documentos `server/svrREADME.md` e `support/analyzeHeadlessREADME.md`.

## 1. Sumário executivo

O Ghidra Server é um serviço Java RMI que armazena **repositórios versionados**. Um projeto\
compartilhado (`shared project`) não é um projeto que “fica no servidor”: ele continua sendo um\
projeto local (`.gpr` + `.rep`), mas o seu armazenamento versionado é remoto. O `.rep` guarda, em\
`project.prp`, o host, a porta e o nome do repositório; ao abrir o projeto, o cliente\
conecta-se ao servidor e usa um `RemoteFileSystem` como *versioned file system*.

O fluxo oficial é:

1. O administrador sobe o Ghidra Server e cadastra usuários.
2. Um usuário cria um **repositório** no servidor (quem cria vira `ADMIN` do repositório).
3. O usuário cria um **projeto compartilhado local** ligado a esse repositório, ou converte um\
   projeto local existente em compartilhado.
4. Binários importados ficam inicialmente **privados** no projeto local; para os demais usuários\
   vê-los, é preciso `addToVersionControl` (primeira versão) ou `checkin` (novas versões).
5. Os demais usuários “entram” no repositório criando um projeto compartilhado local vazio ligado\
   ao mesmo repositório; os arquivos versionados aparecem, mas só podem ser editados após\
   `checkout`.
6. O ciclo colaborativo é: `checkout` → editar/analisar → salvar → `checkin`; os outros fazem\
   `update`/`merge` para incorporar versões novas.

O `pyghidra-mcp` atual só conhece projetos locais: cria/abre `GhidraProject` por caminho de\
diretório e nome, importa binários como arquivos privados e nunca publica versões. Para suportar\
Ghidra Server é necessário adicionar uma camada de configuração/autenticação, um backend de\
projeto que aceite `Project`/`ProjectData` (e não apenas `GhidraProject`), ferramentas de\
checkout/checkin/sync e uma política explícita de publicação e de tratamento de conflitos.

**Recomendação arquitetural central:** separar o contexto em um `ProjectBackend`(`local`, `shared-local`, `shared-transient`) e fazer as ferramentas operarem sobre `ProjectData`e `DomainFile`. `GhidraProject` deve continuar sendo um detalhe do backend local, porque ele não\
sabe criar projetos compartilhados nem abrir URLs `ghidra://`.

## 2. Escopo e terminologia

Este documento usa os seguintes termos para evitar a ambiguidade de “importar projeto”:

| Termo | Significado neste documento |
| --- | --- |
| **Repositório** | Estrutura versionada no Ghidra Server, identificada por nome (ex.: `malware-lab`). |
| **Projeto compartilhado local** | Projeto `.gpr`/`.rep` local cujo armazenamento versionado aponta para um repositório remoto. |
| **Criar projeto compartilhado** | Criar repositório (se necessário) + criar o projeto local ligado a ele. |
| **Entrar/join no repositório** | Criar um projeto local vazio ligado a um repositório existente; os arquivos versionados passam a ser listados. |
| **Abrir projeto compartilhado** | Abrir um `.gpr` compartilhado já existente; o Ghidra lê `project.prp` e reconecta. |
| **Projeto transiente/remoto** | Acesso direto por `ghidra://host:port/repo[/pasta]`, sem `.gpr` local (usado pelo `analyzeHeadless`). |
| **Importar binário** | Carregar um arquivo executável em uma pasta do projeto (privado até ser publicado). |
| **Publicar** | `DomainFile.addToVersionControl(...)`: envia a primeira versão ao repositório. |
| **Check-out** | `DomainFile.checkout(...)`: cria uma cópia privada local para edição. |
| **Check-in** | `DomainFile.checkin(...)`: envia uma nova versão ao repositório. |
| **Update/sync** | Atualizar a árvore e incorporar a última versão no checkout local (`DomainFile.merge(...)`). |
| **Undo checkout** | `DomainFile.undoCheckout(...)`: descarta o checkout local (opcionalmente mantendo `.keep`). |

“Importar um projeto” pode ainda significar importar um arquivo `.gar` (Ghidra Project Archive).\
Esse é um fluxo diferente de “entrar em um repositório” e **não** faz parte do escopo mínimo\
proposto aqui; ver decisões abertas.

## 3. Como o Ghidra Server funciona

### 3.1 Arquitetura e portas

- O servidor é uma aplicação Java autônoma (`ghidra.server.remote.GhidraServer`), iniciada pelos\
  scripts `server/ghidraSvr` (Unix/macOS) ou `server/ghidraSvr.bat` (Windows), normalmente via\
  YAJSW como serviço.
- A comunicação é **Java RMI**, não HTTP. O cliente usa `ClientUtil.getRepositoryServer(host, port, ...)` e obtém um `RepositoryServerAdapter`; cada repositório é acessado por um\
  `RepositoryAdapter`.
- A porta base padrão é **13100**. O servidor utiliza **três portas TCP consecutivas**:\
  `base`, `base + 1` e `base + 2`. Firewalls precisam liberar as três.
- O servidor mantém estado em memória para todos os repositórios; a memória é configurada em\
  `server/server.conf` (`wrapper.java.maxmemory`). O README do servidor traz uma fórmula empírica\
  baseada na quantidade de arquivos e clientes.
- TLS é usado por padrão. Se `-Dghidra.keystore` não for configurado, o servidor gera um\
  certificado autoassinado temporário. Sem `-Dghidra.cacerts`, o cliente Ghidra usa\
  `DefaultTrustManagerFactory.OpenTrustManager` (trust-all), portanto aceita esse certificado\
  sem configuração extra (confirmado no spike). Em PKI/CA customizada, o servidor usa\
  `-Dghidra.cacerts` para validar certificados; esse é o modo em que a confiança precisa ser\
  configurada.
- O diretório de repositórios (`ghidra.repositories.dir`) deve ser dedicado, local e compatível\
  com o *filesystem watcher* do Ghidra. Ele contém os repositórios, o arquivo `users`, o arquivo\
  de ACL por repositório (`userAccess.acl`), o diretório `~ssh` e os logs (`server.log`). O\
  `wrapper.log` fica na instalação.

### 3.2 Configuração e inicialização

Os parâmetros ficam em `server/server.conf` e são passados ao processo Java via\
`wrapper.app.parameter.N`:

| Parâmetro | Efeito |
| --- | --- |
| `-a0` | Senha local do Ghidra (arquivo `users`; senha inicial `changeme`). |
| `-a1 -d<dominio>` | Active Directory via Kerberos. |
| `-a2` | Autenticação PKI (certificado de cliente). |
| `-a4 -jaas <arquivo>` | JAAS (LDAP, PAM, programa externo etc.). |
| `-e<dias>` | Expiração/reset de senha no modo `-a0`. |
| `-u` | Permite que o usuário informe o ID de login no modo `-a0`. |
| `-autoProvision` | Cria usuários automaticamente após autenticação (`-a1`/`-a4`). |
| `-anonymous` | Permite acesso anônimo; cada repositório ainda precisa permitir. |
| `-ssh` | Permite autenticação por chave SSH, somente em conjunto com `-a0` e para clientes headless. |
| `-ip`, `-ipAlt`, `-i`, `-p`, `-n` | Endereço publicado, nomes alternativos, interface, porta base e reverse DNS. |

**Ordem dos argumentos:** o `<repository_path>` deve ser o **último** argumento. Em\
`server.conf`, por exemplo: `wrapper.app.parameter.1=-a0`, `.2=-p<porta>`, `.3=-e0`,\
`.4=${ghidra.repositories.dir}`. Opções depois do path fazem o servidor sair com\
`Invalid usage!` (confirmado no spike).

O servidor deve ser iniciado antes de qualquer operação de projeto. O `pyghidra-mcp` **não deve**\
gerenciar o processo do servidor (instalação de serviço, certificados, AD/JAAS); ele apenas é\
cliente.

### 3.3 Usuários, autenticação e autorização

- Todo usuário precisa ser previamente cadastrado no servidor. O `svrAdmin` é a ferramenta\
  administrativa: `-add`, `-remove`, `-reset`, `-list`, `-users`, `-dn`, `-admin`, `-grant`,\
  `-revoke`, `-migrate`. Ele exige o servidor **em execução** (precisa de `<repos>/users` e\
  `<repos>/~admin`; caso contrário imprime `Invalid Ghidra server directory!`). Os comandos são\
  enfileirados (`Command queued.`) e processados pelo servidor. O cabeçalho do script não lista\
  `-grant`/`-revoke`, mas `ghidra.server.ServerAdmin` os aceita (confirmado).
- **Não existe comando** `svrAdmin` **para criar repositório.** Repositórios são criados por um\
  cliente autenticado via `RepositoryServerAdapter.createRepository(name)`. Na implementação,\
  qualquer usuário válido (não anônimo) pode criar um repositório, e o criador recebe\
  automaticamente permissão `ADMIN` naquele repositório.
- Permissões por repositório:
  - `READ_ONLY` (`+r`): apenas leitura; `User.isReadOnly()`; não pode fazer check-out nem\
    check-in.
  - `WRITE` (`+w`): check-out/check-in.
  - `ADMIN` (`+a`): inclui escrita e operações administrativas do repositório (gerenciar usuários,\
    terminar checkouts de terceiros etc.).
- O criador do repositório pode definir a lista de usuários com\
  `RepositoryAdapter.setUserList(User[], anonymousAccessAllowed)`. O wizard do Ghidra faz isso ao\
  criar um repositório.
- Usuário anônimo (`-anonymous` + repositório com acesso anônimo) é sempre somente leitura.
- O modo `-ssh` é suportado apenas pelo Headless Analyzer/cliente headless, somente com `-a0`.\
  A chave pública do usuário deve estar em `repositories/~ssh/<usuario>.pub`, e a chave privada\
  deve ser RSA em formato PEM (`ssh-keygen -m pem -t rsa -b 2048`). Formatos OpenSSH, ECDSA e\
  Ed25519 não são suportados por essa modalidade.

### 3.4 Modelo de versionamento

O repositório é um **filesystem versionado**. As operações são por `DomainFile`:

- `addToVersionControl(comment, keepCheckedOut, monitor)`: publica um arquivo privado (primeira\
  versão) e, opcionalmente, já o deixa em check-out.
- `checkout(exclusive, monitor)`: cria uma cópia privada local. Check-out **exclusivo** impede\
  outros check-outs; **não exclusivo** permite múltiplos check-outs e exige merge posterior.\
  Retorna `false` quando um check-out exclusivo não é possível por já existir check-out de outro\
  usuário.
- `checkin(CheckinHandler, monitor)`: publica nova versão. Requer:
  - conexão com o repositório;
  - projeto e repositório graváveis;
  - arquivo em check-out;
  - **sem alterações não salvas** (`isChanged() == false`);
  - **sem dados de recuperação** pendentes;
  - alteração desde o check-out (`modifiedSinceCheckout()`).\
    O `CheckinHandler` fornece comentário, `keepCheckedOut` e `createKeepFile`.\
    **Se o arquivo estiver aberto, o Ghidra força** `keepCheckedOut = true`.
- `undoCheckout(keep)`: descarta o check-out; `keep=true` renomeia a cópia privada para `.keep`.
- `merge(okToUpgrade, monitor)`: incorpora a versão mais recente do repositório no check-out\
  local. É a operação que o Ghidra chama de **Update**. Não é permitida em projeto transiente.\
  **Em modo headless, `merge` só funciona quando o check-out não tem alterações locais**\
  (`modifiedSinceCheckout() == false`): nesse caso ele apenas atualiza o check-out para a versão\
  mais recente. Com alterações locais, `GhidraFileData.merge` lança\
  `IOException: Merge failed, merge is not supported in headless mode`. Confirmado no spike.
- `getChangesByOthersSinceCheckout()`: retorna o `ChangeSet` com alterações de outros usuários\
  desde o check-out; útil para pré-visualizar conflitos.
- `getLatestVersion()`, `getVersion()`, `getVersionHistory()`: versão mais recente, versão local\
  (ou versão do check-out) e histórico (`Version[]` com usuário, data e comentário).
- `getReadOnlyDomainObject(consumer, version, monitor)` e\
  `getImmutableDomainObject(...)`: abrem uma versão específica sem editá-la.
- `getCheckouts()` / `getCheckoutStatus()`: check-outs de todos os usuários / check-out\
  correspondente a este projeto. `terminateCheckout(id)` permite que o dono ou um admin encerre\
  um check-out.
- `delete()` apaga o arquivo e **todas as versões**; `delete(version)` apaga uma versão\
  específica (mais antiga ou mais recente). Requer permissão e não pode haver uso/check-out\
  incompatível.

Observações importantes de semântica:

- Abrir um arquivo versionado que **não** está em check-out resulta em um objeto **read-only**(`DomainFile.getDomainObject` cai em `getReadOnlyObject` quando não há cópia privada). Para\
  editar é preciso fazer check-out e reabrir.
- O check-out é feito por **projeto local**. Dois clientes com projetos locais diferentes podem\
  fazer check-out do mesmo arquivo (se não exclusivo). O servidor controla os check-outs e o\
  autor do check-in.
- Um check-out “transiente” (`CheckoutType.TRANSIENT`) é usado em projetos transientes\
  (`ghidra://`), é sempre exclusivo e só persiste enquanto a conexão do cliente estiver viva.
- `ProjectData.findCheckedOutFiles(monitor)` lista os check-outs locais do projeto.
- `ProjectData.refresh(force)` e `DomainFolder.refresh(force)` sincronizam a árvore local com o\
  filesystem versionado; não fazem merge de conteúdo.
- O “commit” do `analyzeHeadless` não é uma operação única do framework: ele executa\
  `addToVersionControl` para arquivos novos e `checkin` para arquivos já versionados.

### 3.5 Limitações operacionais relevantes

- O projeto local é protegido por um **lock** (`.lock`). Dois processos não podem abrir o mesmo\
  `.gpr` simultaneamente. Para vários usuários, cada um precisa de seu próprio projeto local\
  (ou usar modo transiente).
- O servidor mantém estado em memória; muitos arquivos/versões aumentam o consumo e o tempo de\
  listagem. Operações de status não devem varrer versões/check-outs de todos os arquivos sem\
  necessidade.
- O servidor pode reiniciar ou a conexão cair; `RepositoryAdapter` tenta reconectar em operações\
  que dependem do servidor, mas check-outs transientes podem ser invalidados.
- O `DefaultClientAuthenticator` pode abrir diálogos Swing. Em modo headless é obrigatório\
  instalar um autenticador headless antes de qualquer conexão.
- `HeadlessClientAuthenticator` não recebe senha por parâmetro: sua assinatura é\
  `installHeadlessClientAuthenticator(user, sshKeyFile, allowPasswordPrompt)`. Com\
  `allowPasswordPrompt=false` ele falha o login; com `true` lê a senha da **stdin** (sem máscara)\
  — incompatível com o transporte `stdio` do MCP. Para `-a0`, use\
  `PasswordClientAuthenticator(user, password)` via `ClientUtil.setClientAuthenticator`.\
  Além disso, sem o servidor iniciado com `-u`, o cliente envia `SystemUtilities.getUserName()`\
  (cacheado de `user.name`), **não** o usuário do authenticator: é preciso iniciar a JVM com\
  `-Duser.name=<usuário>` antes do primeiro uso. Confirmado no spike.
- Em modo GUI, o projeto é do Ghidra; o MCP apenas o observa/opera. Em modo headless, o MCP é o\
  dono do `Project`/`ProjectData` e deve fechar corretamente.

## 4. Como projetos compartilhados são criados e importados

### 4.1 Estrutura de um projeto compartilhado

Um projeto compartilhado local tem a mesma estrutura de um projeto comum:

```
meu-projeto.gpr
meu-projeto.rep/
  project.prp     # XML: OWNER, SERVER, REPOSITORY_NAME, PORT_NUMBER
  ...
meu-projeto-pyghidra-mcp/
  chromadb/
  gzfs/
```

A diferença está em `project.prp`: quando `SERVER` e `REPOSITORY_NAME` estão presentes, o\
`DefaultProjectData` instancia um `RepositoryAdapter` e usa `RemoteFileSystem` como\
`versionedFileSystem`. O arquivo é XML, por exemplo:

```xml
<FILE_INFO>
  <BASIC_INFO>
    <STATE NAME="OWNER" TYPE="string" VALUE="alice" />
    <STATE NAME="SERVER" TYPE="string" VALUE="127.0.0.1" />
    <STATE NAME="REPOSITORY_NAME" TYPE="string" VALUE="spike-repo" />
    <STATE NAME="PORT_NUMBER" TYPE="int" VALUE="45000" />
  </BASIC_INFO>
</FILE_INFO>
```

O `ProjectLocator` continua sendo local; `ProjectData.getSharedProjectURL()` devolve a URL\
`ghidra://host:port/repo` correspondente.

### 4.2 Criar um repositório

Via API (equivalente ao wizard):

```java
RepositoryServerAdapter server = ClientUtil.getRepositoryServer(host, port, true);
RepositoryAdapter repo = server.createRepository("malware-lab");
repo.setUserList(users, /* anonymousAccessAllowed */ false);
```

Ou pelo helper `GhidraProject.getServerRepository(host, port, repoName, createIfNeeded)`, que\
procura o repositório e cria se não existir. Ele **não** configura a lista de usuários.

O criador recebe `ADMIN` no repositório. Para conceder acesso a outros usuários, é preciso\
`setUserList` (via cliente admin) ou `svrAdmin -grant`. Não há criação de repositório via\
`svrAdmin`.

### 4.3 Criar um projeto compartilhado novo

Há dois caminhos:

**A. Criar já compartilhado (recomendado para novos projetos)**

Usar o `DefaultProjectManager` (a classe `PyGhidraProjectManager` existe justamente para expor o\
construtor protegido):

```python
from ghidra.pyghidra import PyGhidraProjectManager
from ghidra.framework.model import ProjectLocator

manager = PyGhidraProjectManager()
locator = ProjectLocator(local_dir, project_name)   # diretório pai deve existir
# 3º argumento = "remember"/recent projects; o modo compartilhado vem do repo_adapter
project = manager.createProject(locator, repo_adapter, True)
```

Equivalente ao wizard “New Project → Shared Project → Create a new Repository” (ou “Join existing\
Repository”). O projeto local nasce vazio, mas já lista os arquivos versionados do repositório.

**B. Converter um projeto local existente**

Criar/abrir o projeto local normalmente e chamar:

```python
project.getProjectData().convertProjectToShared(repo_adapter, TaskMonitor.DUMMY)
```

A documentação do Ghidra recomenda **fechar e reabrir** o projeto após a conversão. A conversão\
transforma arquivos versionados em arquivos privados e troca o filesystem versionado local pelo\
remoto; portanto, depois dela os arquivos precisam ser publicados novamente com\
`addToVersionControl` se você quiser que apareçam para os outros usuários.

### 4.4 “Importar”/entrar em um projeto compartilhado existente

Há três modalidades, com semânticas diferentes:

1. **Abrir um** `.gpr` **compartilhado já existente**

   - `GhidraProject.openProject(dir, name, restore)` ou\
     `DefaultProjectManager.openProject(locator, restore, resetOwner)`.
   - O Ghidra lê `project.prp` e reconecta ao servidor/repositório. A autenticação precisa\
     estar instalada **antes** da abertura.
   - É o fluxo para quem já tem uma cópia local do projeto.

2. **Entrar/join em um repositório existente (projeto local novo)**

   - Criar um projeto local vazio ligado ao repositório:\
     `manager.createProject(locator, repo_adapter, True)`.
   - O repositório já existente é listado; os arquivos versionados aparecem, mas para editar é\
     necessário `checkout`.
   - É o fluxo “Join existing Repository” do wizard do Ghidra.

3. **Acessar sem projeto local (transiente)**

   - URL `ghidra://host[:port]/repo[/pasta]`, processada por `GhidraURLQuery` /\
     `GhidraURLConnection` / `TransientProjectManager`.
   - Não cria `.gpr`; o `ProjectLocator` é transiente e o `ProjectData` é mantido em cache no\
     processo.
   - O `analyzeHeadless` usa esse modo para importar/processar direto no repositório. Nele, o\
     commit é habilitado automaticamente quando há permissão de escrita e não se usa\
     `-readOnly`.
   - Limitações: check-out transiente é exclusivo e vinculado à conexão; `merge` não é permitido.\
     É adequado para leitura e importação pontual, não para edição colaborativa longa.

### 4.5 Importar binários e publicar

O importador do Ghidra (`ProgramLoader` ou o helper `GhidraProject.importProgram`) cria o\
programa no **filesystem privado local**. Em projeto compartilhado, isso significa:

1. Importar e salvar o programa na pasta desejada do projeto.
2. Analisar (opcional).
3. Salvar (`DomainFile.save` / `program.save`).
4. Publicar:
   - arquivo novo: `df.addToVersionControl(comment, keepCheckedOut, monitor)`;
   - arquivo já versionado e em check-out: `df.checkin(handler, monitor)`.

Se o arquivo estiver aberto no momento da publicação, o Ghidra força `keepCheckedOut = true`.\
Para liberar o check-out e permitir que outro usuário faça check-out exclusivo, é preciso fechar\
o programa antes do check-in (ou aceitar que ele continuará em check-out).

Em lote, o `analyzeHeadless` faz `addToVersionControl`/`checkin` por arquivo, com o comentário de\
commit configurado. O `pyghidra-mcp` deve seguir a mesma semântica, mas com uma diferença de\
segurança recomendada: **não publicar silenciosamente**. O padrão deve ser `publish=False` e o\
resultado deve informar que o arquivo ficou privado; a publicação deve ser explícita (ou\
configurada por `--publish-imports`).

### 4.6 Matriz de operações e APIs

| Operação | API principal | Observações |
| --- | --- | --- |
| Conectar ao servidor | `ClientUtil.getRepositoryServer` / `RepositoryServerAdapter.connect` | Requer autenticador instalado. |
| Criar repositório | `RepositoryServerAdapter.createRepository` | Criador vira `ADMIN`; pode falhar por nome duplicado. |
| Listar repositórios | `getRepositoryNames` | Retorna apenas os acessíveis ao usuário. |
| Gerenciar usuários do repo | `getUserList` / `setUserList` | Requer `ADMIN`; `User` tem `READ_ONLY`, `WRITE`, `ADMIN`. |
| Criar projeto compartilhado | `DefaultProjectManager.createProject(locator, repo, remember)` | Diretório pai precisa existir. |
| Converter projeto local | `ProjectData.convertProjectToShared(repo, monitor)` | Fechar/reabrir depois; arquivos viram privados. |
| Abrir projeto compartilhado | `DefaultProjectManager.openProject` / `GhidraProject.openProject` | Lê `project.prp` e conecta. |
| Acesso transiente | `GhidraURLQuery.queryRepositoryUrl` / `GhidraURLConnection` | Sem `.gpr`; cache `TransientProjectManager`. |
| Importar binário | `ProgramLoader` (recomendado) ou `GhidraProject.importProgram` | Cria arquivo privado. |
| Publicar arquivo novo | `DomainFile.addToVersionControl` | Primeira versão. |
| Check-out | `DomainFile.checkout` | `exclusive` controla concorrência. |
| Check-in | `DomainFile.checkin` | Exige salvo, modificado, conectado, sem recovery. |
| Undo checkout | `DomainFile.undoCheckout` | `keep` cria `.keep`. |
| Update/merge | `DomainFile.merge` | Não permitido em transiente. |
| Refresh/sync da árvore | `ProjectData.refresh` / `DomainFolder.refresh` | Não faz merge. |
| Histórico | `DomainFile.getVersionHistory` | `Version[]`: número, autor, data, comentário. |
| Check-outs | `DomainFile.getCheckouts` / `getCheckoutStatus` / `ProjectData.findCheckedOutFiles` | Status por arquivo/projeto. |
| Encerrar checkout de terceiro | `DomainFile.terminateCheckout` / `RepositoryAdapter.terminateCheckout` | Dono ou admin. |
| Apagar arquivo/versão | `DomainFile.delete()` / `delete(version)` | `delete()` apaga todas as versões. |

## 5. Estado atual do pyghidra-mcp (gap analysis)

### 5.1 Como o projeto é aberto hoje

`src/pyghidra_mcp/context.py`:

- `_get_or_create_project()` monta um `ProjectLocator` local e chama\
  `GhidraProject.openProject(dir, name, True)` ou `GhidraProject.createProject(dir, name, False)`.
- `_init_project_programs()` abre **todos** os arquivos de conteúdo `Program` com\
  `openProgram(..., readOnly=False)`.
- `import_binary()` importa, renomeia, faz `saveAs` e fecha/reabre o programa. Não chama\
  `addToVersionControl`.
- `save()` chama `GhidraProject.save(program)`; `close()` chama `save` e `close`.
- `delete_program()` chama `DomainFile.delete()` sem política para versões/check-outs.

`src/pyghidra_mcp/project_spec.py` só entende diretório + nome ou arquivo `.gpr`.\
`src/pyghidra_mcp/server.py` expõe `--project-path` como `click.Path(path_type=Path)` (sem\
`exists=True`), mas o valor é tratado como caminho local; URLs `ghidra://` não são suportadas e\
não há opções de servidor/autenticação.

### 5.2 O que já funciona

- Projetos locais e `.gpr` existentes, inclusive projetos que já sejam compartilhados: ao abrir,\
  o Ghidra tenta conectar usando `project.prp`. Falta, porém, configurar autenticação\
  headless antes da abertura.
- Importação, análise, indexação e ferramentas de leitura/escrita em programas locais.
- Modo GUI, que herda o suporte nativo do Ghidra a projetos compartilhados.

### 5.3 Gaps e riscos

| \# | Gap | Impacto |
| --- | --- | --- |
| G1 | Nenhuma instalação de autenticador headless (`HeadlessClientAuthenticator`/`PasswordClientAuthenticator`) | Abertura de projeto compartilhado falha ou tenta diálogo Swing; em `stdio` pode travar. |
| G2 | Nenhuma opção de servidor/repositório/credenciais | Impossível criar/entrar em repositório pela CLI/MCP. |
| G3 | `GhidraProject` não cria projetos compartilhados | Necessário usar `DefaultProjectManager`/`ProjectData`. |
| G4 | `_init_project_programs` abre tudo para escrita | Em arquivos versionados não checados, o Ghidra abre read-only; ferramentas de escrita falham de forma confusa. |
| G5 | Importação não publica | Binários importados em projeto compartilhado ficam invisíveis aos demais. |
| G6 | `delete_program` apaga todas as versões | Risco de perda irreversível; não considera check-outs de terceiros. |
| G7 | `save`/`close` não distinguem read-only/remoto | Podem falhar ou tentar salvar programas que não estão em check-out. |
| G8 | Nenhum status de versão/check-out | O agente não sabe se pode editar, se precisa de update ou se há conflito. |
| G9 | Nenhum sync/update/merge | Colaboração real impossível. |
| G10 | Indexação por nome/path, sem versão | Índice fica obsoleto após checkout/merge/checkin. |
| G11 | Análise concorrente em todos os arquivos | Check-outs e gravações precisam de serialização e política de conflito. |
| G12 | `--project-path` exige existência e não aceita URL | Modo transiente/`ghidra://` não é suportado. |
| G13 | Sem modo read-only de projeto local | Abrir projeto compartilhado somente leitura exige modo transiente ou permissões do repo. |
| G14 | Sem tratamento tipado de erros RMI/autenticação | Mensagens genéricas, sem ação corretiva, e risco de retry indevido. |

## 6. Requisitos

A prioridade usa MoSCoW: **M** = must, **S** = should, **C** = could.

### 6.1 Requisitos funcionais

#### Servidor e autenticação

| ID | Prio | Requisito |
| --- | --- | --- |
| FR-SRV-01 | M | Configurar host, porta (default 13100) e repositório por CLI/env/arquivo de configuração. |
| FR-SRV-02 | M | Validar conectividade e compatibilidade com o servidor (`ClientUtil.checkGhidraServer`) antes de abrir o projeto, com erro claro se indisponível/incompatível. |
| FR-SRV-03 | M | Instalar autenticador headless antes de qualquer conexão: senha (env/arquivo), chave SSH (PEM RSA) ou PKI (keystore). |
| FR-SRV-04 | M | Suportar autenticação anônima somente quando explicitamente configurada e quando o repositório permitir; tratar como read-only. |
| FR-SRV-05 | M | Nunca ler senha da stdin quando o transporte for `stdio`; falhar com instrução para usar env/arquivo/chave. |
| FR-SRV-06 | M | Redigir senhas, tokens e senhas de keystore de logs, mensagens de erro, traces e argumentos de processo. |
| FR-SRV-07 | M | Listar repositórios acessíveis ao usuário. |
| FR-SRV-08 | S | Criar repositório, com opção de lista inicial de usuários/permissões e acesso anônimo. |
| FR-SRV-09 | S | Listar/alterar usuários e permissões de um repositório (requer `ADMIN`). |
| FR-SRV-10 | S | Desconectar/reconectar explicitamente, limpando o cache de conexão (`ClientUtil.clearRepositoryAdapter`) quando necessário. |
| FR-SRV-11 | C | Exportar informações do servidor (versão, porta, usuário, read-only) para diagnóstico. |

#### Criação/abertura de projeto

| ID | Prio | Requisito |
| --- | --- | --- |
| FR-PRJ-01 | M | Criar projeto compartilhado novo em um diretório local, opcionalmente criando o repositório. |
| FR-PRJ-02 | M | Entrar em repositório existente criando um projeto local vazio ligado a ele. |
| FR-PRJ-03 | M | Abrir `.gpr` compartilhado existente e reconectar automaticamente. |
| FR-PRJ-04 | M | Converter projeto local existente em compartilhado, com aviso de que os arquivos precisam ser publicados novamente. |
| FR-PRJ-05 | S | Abrir repositório em modo transiente via `ghidra://host:port/repo[/pasta]`, sem `.gpr`, fechando o `ProjectData` exatamente uma vez ao encerrar. |
| FR-PRJ-06 | S | Modo read-only explícito para projeto/repositório, bloqueando ferramentas de escrita. |
| FR-PRJ-07 | M | Detectar projeto já compartilhado e validar que servidor/repositório configurados coincidem com `project.prp`. |
| FR-PRJ-08 | M | Falhar de forma clara se o projeto local estiver bloqueado por outro processo (`.lock`) ou se o diretório pai não existir. |
| FR-PRJ-09 | M | Perder o mínimo de trabalho possível ao fechar: salvar apenas programas graváveis; nunca fazer check-in automático no shutdown. |
| FR-PRJ-10 | C | Gerenciar múltiplos projetos/repositórios no mesmo processo (ver decisão aberta). |

#### Importação e publicação

| ID | Prio | Requisito |
| --- | --- | --- |
| FR-IMP-01 | M | Importar binário para pasta arbitrária do projeto, preservando a hierarquia e o comportamento atual de deduplicação por nome+hash. |
| FR-IMP-02 | M | Após importar, retornar status: `private`, `published`/`versioned`, versão e se está em check-out. |
| FR-IMP-03 | M | Publicar arquivo novo com `addToVersionControl(comment, keepCheckedOut)` quando `publish=True`; nunca publicar silenciosamente por padrão. |
| FR-IMP-04 | M | Garantir que o programa esteja salvo e sem recovery antes de publicar/check-in. |
| FR-IMP-05 | S | Suportar importação em lote com resultado por arquivo (sucesso, falha, publicado, versão). |
| FR-IMP-06 | S | Suportar `overwrite` apenas com política explícita e sem apagar versões por acidente. |
| FR-IMP-07 | C | Importar diretamente em modo transiente e publicar no repositório (equivalente ao `analyzeHeadless` URL). |

#### Check-out, check-in, undo e sync

| ID | Prio | Requisito |
| --- | --- | --- |
| FR-VCS-01 | M | Fazer check-out de um ou mais binários, com modo exclusivo (default) ou compartilhado. |
| FR-VCS-02 | M | Se o arquivo já estiver em check-out pelo mesmo projeto, tratar como no-op idempotente e retornar o status atual. |
| FR-VCS-03 | M | Se o check-out exclusivo falhar por check-out de terceiro, retornar `exclusive_checkout_unavailable` com usuários/check-outs e sugerir modo não exclusivo. |
| FR-VCS-04 | M | Fazer check-in com comentário, `keep_checked_out` e `create_keep_file`, respeitando a regra do Ghidra de forçar `keep_checked_out` quando o programa está aberto. |
| FR-VCS-05 | M | Antes do check-in, salvar o programa e validar `isChanged`, recovery e `modifiedSinceCheckout`; se não for possível, explicar exatamente o que falta. |
| FR-VCS-06 | M | Fazer undo checkout com opção `keep` (padrão seguro) e exigir confirmação explícita para descartar alterações. |
| FR-VCS-07 | M | Atualizar a árvore (`ProjectData.refresh`) e reportar arquivos cujo `latestVersion > version` (check-out desatualizado). |
| FR-VCS-08 | M | Fazer update/merge explícito de um check-out, retornando versão anterior/nova e conflitos. Em headless, o merge de check-out com alterações locais não é suportado pelo Ghidra; o produto deve limitar-se a update de check-out sem alterações e reportar bloqueio explícito. |
| FR-VCS-09 | S | Pré-visualizar alterações de terceiros (`getChangesByOthersSinceCheckout`) antes do merge. |
| FR-VCS-10 | M | Listar versões de um arquivo (número, autor, data, comentário), com paginação. |
| FR-VCS-11 | S | Abrir/ler uma versão específica sem alterar o check-out. |
| FR-VCS-12 | S | Listar check-outs do projeto e do repositório, com filtro por arquivo/usuário. |
| FR-VCS-13 | S | Permitir que admin encerre check-out de terceiro, com confirmação e auditoria. |
| FR-VCS-14 | S | Sync de projeto: refresh + lista de arquivos desatualizados; `merge` só quando solicitado. |
| FR-VCS-15 | C | Resolver conflitos de merge de forma assistida (o Ghidra oferece merge; conflitos finos continuam responsabilidade do usuário). |

#### Exclusão e operações destrutivas

| ID | Prio | Requisito |
| --- | --- | --- |
| FR-DEL-01 | M | Excluir binário privado local sem tocar no repositório. |
| FR-DEL-02 | M | Excluir binário versionado somente com confirmação explícita; avisar que `delete()` apaga todas as versões. |
| FR-DEL-03 | M | Bloquear exclusão se houver check-out de outro usuário ou se o arquivo estiver em uso, salvo operação admin explícita. |
| FR-DEL-04 | S | Permitir `dry_run` para operações destrutivas. |

### 6.2 Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| NFR-01 | **Compatibilidade**: funcionar com o cliente/servidor Ghidra suportados pelo projeto; detectar incompatibilidade de versão e orientar o usuário. A versão inspecionada usa interface de servidor 13. |
| NFR-02 | **Segurança**: TLS sempre que o servidor usar; suportar CA customizada (`-Dghidra.cacerts`). Atenção: sem `cacerts`, o cliente Ghidra usa trust-all (`OpenTrustManager`); o produto deve documentar isso e não tratar o default como verificação forte. |
| NFR-03 | **Segredos**: credenciais nunca em argumentos de ferramenta MCP, URLs, logs ou arquivos de projeto; preferir env, arquivo com permissão `0600` e, opcionalmente, keyring. |
| NFR-04 | **Isolamento**: um autenticador/identidade por processo JVM; não tentar usar credenciais diferentes para servidores diferentes no mesmo processo. |
| NFR-05 | **Concorrência**: leituras podem ser concorrentes; operações de escrita (checkout, checkin, undo, merge, delete, publish) devem ser serializadas por arquivo e, se necessário, por projeto. |
| NFR-06 | **Idempotência**: checkout e refresh devem ser idempotentes; check-in não é idempotente e não pode ser repetido cegamente após timeout. |
| NFR-07 | **Recuperação**: após falha/timeout de escrita, verificar o estado real (versão, histórico, check-out) antes de decidir repetir; retornar `unknown_outcome` se não for possível determinar. |
| NFR-08 | **Desempenho**: não abrir todos os arquivos versionados no startup; status e listagens devem ser lazy/paginados; evitar chamadas RMI por arquivo em loops grandes. |
| NFR-09 | **Escalabilidade**: suportar projetos grandes limitando indexação inicial e permitindo `--max-binaries`/paginação. |
| NFR-10 | **Confiabilidade**: shutdown gracioso salva apenas o que é gravável; não faz check-in automático; libera recursos Java e conexões. |
| NFR-11 | **Observabilidade**: logs estruturados com host, repo, usuário, arquivo, versão, check-out id e duração; auditoria local de operações de escrita. |
| NFR-12 | **Portabilidade**: normalizar separadores de path (`/` no Ghidra), nomes inválidos e diferenças Windows/Linux. |
| NFR-13 | **Compatibilidade retroativa**: projetos locais e testes existentes devem continuar funcionando sem novas flags. |
| NFR-14 | **Erros acionáveis**: mapear exceções Java/RMI para códigos estáveis com mensagem e próxima ação. |

### 6.3 Configuração CLI e ambiente

Opções propostas para `pyghidra-mcp` (nomes sujeitos a revisão):

```text
--server-host HOST                  # ou --server-url ghidra://host:port
--server-port PORT                  # default 13100
--server-user USER
--server-password-env VAR           # lê a senha de variável de ambiente
--server-password-file PATH         # arquivo 0600, uma linha
--server-ssh-key PATH               # PEM RSA 2048
--server-keystore PATH              # PKI
--server-keystore-password-env VAR
--server-anonymous                  # somente leitura
--server-cacerts PATH               # truststore customizada
--repository NAME
--create-repository                 # cria se não existir
--reuse-repository                  # permite usar existente com --create-repository
--join-repository                   # cria projeto local vazio ligado ao repo
--shared-project                    # projeto novo já compartilhado
--convert-to-shared                 # converte projeto local existente
--project-path PATH|ghidra://...    # aceitar URL para modo transiente
--read-only                         # modo somente leitura
--publish-imports                   # publica binários importados ao final
--publish-comment TEXT              # comentário de publicação/check-in
--checkout-mode exclusive|shared    # default exclusive
--server-timeout SECONDS
--allow-server-admin                # habilita ferramentas administrativas
```

Variáveis de ambiente recomendadas:

```text
GHIDRA_SERVER_HOST
GHIDRA_SERVER_PORT
GHIDRA_SERVER_USER
GHIDRA_SERVER_PASSWORD
GHIDRA_SERVER_SSH_KEY
GHIDRA_SERVER_KEYSTORE
GHIDRA_SERVER_KEYSTORE_PASSWORD
GHIDRA_SERVER_CACERTS
GHIDRA_SERVER_REPOSITORY
```

Regras de validação:

- `--repository` exige servidor; `--create-repository`/`--join-repository` exigem `--repository`.
- `--join-repository` e `--convert-to-shared` são mutuamente exclusivos e exigem projeto local.
- `--project-path ghidra://...` implica modo transiente; `--project-name` é ignorado.
- `--read-only` com `--publish-imports`/check-in deve falhar na inicialização.
- `--server-password-*` e `--server-ssh-key`/`--server-keystore` são mutuamente exclusivos\
  (escolher um método).
- Em `--transport stdio`, qualquer opção que implique prompt interativo deve ser rejeitada.

Exemplos:

```bash
# Criar repositório + projeto compartilhado e importar/publicar
pyghidra-mcp \
  --transport streamable-http \
  --server-host ghidra.example.org --server-port 13100 \
  --server-user alice --server-password-env GHIDRA_SERVER_PASSWORD \
  --repository malware-lab --create-repository \
  --project-path ~/ghidra/malware-lab --project-name malware-lab \
  --publish-imports \
  /samples/a.bin /samples/b.bin

# Entrar em um repositório existente
pyghidra-mcp \
  --transport streamable-http \
  --server-host ghidra.example.org \
  --server-user bob --server-ssh-key ~/.ssh/id_rsa \
  --repository malware-lab --join-repository \
  --project-path ~/ghidra/malware-lab --project-name malware-lab

# Abrir sem projeto local, somente leitura
pyghidra-mcp \
  --transport streamable-http \
  --server-host ghidra.example.org --server-user guest --server-anonymous \
  --project-path ghidra://ghidra.example.org:13100/malware-lab --read-only
```

### 6.4 Ferramentas MCP propostas

A criação/abertura de projeto muda o contexto inteiro do processo. Por isso, a recomendação é\
tratá-la como **operação de inicialização/CLI**, não como ferramenta MCP em tempo de execução.\
Ferramentas de repositório e de ciclo de vida de arquivos podem ser expostas normalmente.

#### Contexto (CLI/startup)

| Operação | Forma | Observação |
| --- | --- | --- |
| Criar repositório | `--create-repository` + `--repository` | Também pode ser ferramenta admin se o processo não estiver preso a um projeto. |
| Criar projeto compartilhado | `--shared-project` | Requer repo novo ou existente. |
| Entrar em repo existente | `--join-repository` | Cria `.gpr` local vazio. |
| Converter projeto local | `--convert-to-shared` | Fechar/reabrir; publicar arquivos depois. |
| Abrir `.gpr` compartilhado | `--project-path .../*.gpr` | Detecta shared por `project.prp`. |
| Abrir transiente | `--project-path ghidra://...` | Sem `.gpr`; sujeito às limitações de transiente. |

#### Ferramentas de runtime

| Tool | Prio | Entrada principal | Saída | API |
| --- | --- | --- | --- | --- |
| `server_status` | M | — | conexão, host/porta, usuário, read-only, repo, versão | `RepositoryServerAdapter`, `RepositoryAdapter` |
| `server_connect` | S | — | status | `connect()` |
| `server_disconnect` | S | — | status | `disconnect()` |
| `list_repositories` | M | — | nomes/perms | `getRepositoryNames` |
| `create_repository` | S | `name`, `users?`, `anonymous?` | repo/status | `createRepository`, `setUserList` |
| `get_repository_users` | S | repo? | usuários/perms | `getUserList` |
| `set_repository_users` | C | repo, users, anonymous | status | `setUserList` |
| `shared_project_status` | M | `include_files?` | modo, server, repo, conectado, resumo, arquivos | `ProjectData`, `Project`, `DomainFile` |
| `get_binary_shared_status` | M | `binary_name` | versioned, checked out, versão, latest, flags, checkouts | `DomainFile` |
| `checkout_binary` | M | `binary_name`/lista, `exclusive?` | status por item | `DomainFile.checkout` |
| `checkin_binary` | M | `binary_name`/lista, `comment?`, `keep_checked_out?`, `create_keep_file?` | versão criada | `DomainFile.checkin` |
| `undo_checkout_binary` | M | `binary_name`/lista, `keep?`, `confirm?` | status | `DomainFile.undoCheckout` |
| `update_binary` | M | `binary_name`/lista, `ok_to_upgrade?` | versão anterior/nova, conflito | `DomainFile.merge` |
| `sync_project` | M | `merge_checked_out?` | refresh + pendências | `ProjectData.refresh`, `DomainFile.merge` |
| `list_binary_versions` | M | `binary_name`, `offset`, `limit` | versões | `getVersionHistory` |
| `list_checkouts` | S | `binary_name?`, `user?` | check-outs | `getCheckouts`/`findCheckedOutFiles` |
| `terminate_checkout` | S | `binary_name`, `checkout_id`, `confirm` | status | `terminateCheckout` |
| `get_changes_since_checkout` | S | `binary_name` | resumo de mudanças | `getChangesByOthersSinceCheckout` |
| `import_binary` (estender) | M | \+ `folder?`, `publish?`, `comment?`, `keep_checked_out?` | \+ publicado/versão | `ProgramLoader`, `addToVersionControl` |
| `delete_project_binary` (estender) | M | \+ `delete_versions?`, `confirm?`, `dry_run?` | status | `DomainFile.delete` |

Regras de schema:

- Resultados de lote devem retornar `results[]` com `ok`, `error_code`, `message` por item; não\
  abortar o lote inteiro por uma falha.
- Operações destrutivas exigem `confirm=true` (ou `dry_run=true`).
- Check-in aceita `expected_latest_version` opcional para abortar se o arquivo mudou desde o\
  início da edição.
- Status deve retornar `next_actions` (ex.: `checkout`, `update`, `save`, `checkin`).

### 6.5 Modelos de dados propostos

```python
class ServerStatus(BaseModel):
    configured: bool
    connected: bool
    host: str | None = None
    port: int | None = None
    user: str | None = None
    read_only: bool = False
    repository: str | None = None
    server_version: str | None = None
    error: str | None = None

class CheckoutInfo(BaseModel):
    checkout_id: int
    user: str
    version: int
    exclusive: bool
    checkout_type: str          # NORMAL | EXCLUSIVE | TRANSIENT
    checkout_time: int
    project_path: str | None = None

class SharedFileStatus(BaseModel):
    path: str
    name: str
    content_type: str
    versioned: bool
    read_only: bool
    checked_out: bool
    checked_out_by_me: bool
    checkout_user: str | None = None
    checkout_id: int | None = None
    exclusive: bool = False
    current_version: int
    latest_version: int
    needs_update: bool
    modified_since_checkout: bool
    can_checkout: bool
    can_checkin: bool
    can_merge: bool
    checkouts: list[CheckoutInfo] = []

class SharedProjectStatus(BaseModel):
    mode: Literal["local", "shared-local", "shared-transient"]
    project_path: str | None
    server: ServerStatus
    repository: str | None
    connected: bool
    read_only: bool
    files: list[SharedFileStatus]
    pending_publish: list[str]
    dirty_programs: list[str]

class VersionInfo(BaseModel):
    version: int
    user: str
    create_time: int
    comment: str
```

### 6.6 Erros e semântica de retry

Códigos estáveis propostos:

| Código | Causa típica | Retry? |
| --- | --- | --- |
| `server_unavailable` | RMI/ConnectException, servidor fora | Leitura: sim; escrita: verificar estado antes. |
| `server_incompatible` | versão cliente/servidor | Não. |
| `auth_failed` | credenciais/chave inválidas | Não (pode pedir nova credencial). |
| `permission_denied` | sem `WRITE`/`ADMIN` | Não. |
| `repository_not_found` | nome errado/repo removido | Não. |
| `repository_exists` | `createRepository` duplicado | Não; usar existente se autorizado. |
| `project_locked` | `.lock` de outro processo | Não; abrir outra cópia ou encerrar o outro processo. |
| `read_only` | repo/usuário/projeto read-only | Não. |
| `file_not_checked_out` | check-in/merge sem checkout | Não; fazer checkout. |
| `exclusive_checkout_unavailable` | outro usuário com checkout exclusivo | Não; usar shared checkout ou aguardar. |
| `file_in_use` | arquivo aberto/transação ativa | Não; salvar/fechar. |
| `file_modified` | alterações não salvas | Não; salvar. |
| `recovery_data_present` | crash anterior | Não; recuperar/descartar recovery. |
| `version_incompatible` | programa de versão antiga | Não; upgrade explícito. |
| `merge_conflict` | conflito no update | Não; resolução manual. |
| `merge_requires_gui` | check-out com alterações locais em modo headless | Não; update só é possível sem alterações locais; usar GUI, check-in ou undo. |
| `delete_blocked` | check-out/uso/permissão | Não. |
| `unknown_outcome` | timeout após escrita | Verificar estado; não repetir cegamente. |

Regras:

- **Leituras** (`status`, `list`, `versions`, `changes`) podem tentar reconectar e repetir com\
  backoff limitado.
- **Escritas** (`checkin`, `publish`, `delete`, `terminate_checkout`) não podem ser repetidas\
  cegamente. Após timeout, consultar `getVersionHistory`, `getLatestVersion`, `getCheckoutStatus`e o usuário atual para decidir.
- Check-in deve ser serializado por arquivo; duas chamadas simultâneas para o mesmo arquivo\
  retornam `file_in_use`/`operation_in_progress`, não duas versões.

### 6.7 Indexação e cache

- Índices (`chromadb`, strings, GZFs) continuam **locais** e não são versionados no repositório.
- Cada versão de um programa pode ter funções/strings diferentes. O índice precisa ser\
  invalidado/reindexado quando:
  - o arquivo é atualizado por `merge`;
  - o checkout muda de versão;
  - o arquivo é substituído por `checkout`/`undoCheckout`;
  - o usuário faz check-in e o conteúdo local muda.
- Recomendação: incluir `file_id`/`version`/hash no nome ou metadata da coleção e reconstruir\
  quando a versão mudar. Não reutilizar índice de versão anterior.
- Em projeto transiente, usar um diretório de cache estável (ex.: `--cache-path` ou\
  `~/.cache/pyghidra-mcp/<host>/<repo>`) para não perder índice a cada execução.
- Indexação deve ser lazy para arquivos não checados/read-only, ou explicitamente habilitada.

### 6.8 Modo GUI

- Se o projeto ativo já for compartilhado, detectar via `Project.getRepository()` /\
  `ProjectData.getSharedProjectURL()` e expor as mesmas ferramentas de status/checkout/checkin.
- Em GUI, as operações de arquivo devem rodar na Swing thread (`run_on_swing` já existe).
- O MCP não deve fechar o projeto nem os programas do GUI; o usuário controla o ciclo de vida.
- Ao abrir um projeto compartilhado no GUI, a autenticação é feita pelo Ghidra; não instalar\
  autenticador headless por cima.
- `--gui` continua exigindo HTTP; criação de projeto compartilhado pelo MCP não é necessária,\
  pois o wizard do Ghidra já existe.

## 7. Fluxos de referência

### 7.1 Criar repositório + projeto + importar/publicar

```text
1. Cliente instala autenticador (senha/SSH/PKI) e conecta ao servidor.
2. ClientUtil.getRepositoryServer(host, port) -> RepositoryServerAdapter.
3. server.createRepository("malware-lab") -> RepositoryAdapter (criador ADMIN).
4. repo.setUserList([...], anonymous=False) (opcional).
5. DefaultProjectManager.createProject(locator, repo, True) -> Project compartilhado.
6. ProgramLoader.import(...) -> arquivo privado no projeto.
7. análise e save.
8. df.addToVersionControl("import inicial", keepCheckedOut=True) -> versão 1.
9. status retorna version=1, checked_out=True.
```

### 7.2 Entrar em repo existente, fazer checkout, editar, check-in

```text
1. Autenticar e conectar.
2. server.getRepository("malware-lab") -> RepositoryAdapter.
3. createProject(locator_vazio, repo, True) -> projeto local vazio.
4. ProjectData.refresh(True) -> lista arquivos versionados.
5. df = projectData.getFile("/malware/a.bin").
6. df.checkout(exclusive=True, monitor) -> cópia privada; reabrir programa para escrita.
7. análise/edição; program.save(...).
8. df.checkin(handler(comment, keepCheckedOut=False), monitor) -> versão 2.
   (se o programa estiver aberto, keepCheckedOut será forçado a True)
9. outro cliente: sync_project -> detecta latest=2 > version=1; update_binary -> merge.
```

### 7.3 Sync/update

```text
sync_project(refresh=True, merge_checked_out=False):
  repo.connect() se necessário
  projectData.refresh(True)
  para cada arquivo:
    se checked_out e getLatestVersion() > getVersion():
      marcar needs_update=True
  retornar relatório (sem alterar conteúdo)

update_binary(file, ok_to_upgrade=False):
  validar canMerge
  # Em headless o Ghidra só faz update de check-out SEM alterações locais.
  # Com modifiedSinceCheckout()==true, merge lança
  # "Merge failed, merge is not supported in headless mode".
  df.merge(ok_to_upgrade, monitor)   # falha também com alterações não salvas/recovery
  fechar/reabrir programa; invalidar índice
  retornar versão anterior/nova e conflitos
```

### 7.4 Importação transiente (equivalente ao headless URL)

```text
1. URL = ghidra://host:port/repo[/folder]
2. GhidraURLQuery.queryRepositoryUrl(URL, readOnly=False, handler, NO_FOLLOW, monitor)
3. handler recebe DomainFolder e ProjectData transiente.
4. importar via ProgramLoader; analisar; salvar.
5. addToVersionControl/checkin para publicar.
6. fechar ProjectData transiente; o repositório mantém as versões.
```

## 8. Critérios de aceite

 1. **Criação**: com servidor vazio e usuário `alice`, `--create-repository --repository r --shared-project` cria o repositório, cria o `.gpr`/`.rep` local e o status mostra\
    `mode=shared-local`, `connected=true`, `repository=r`.
 2. **Join**: `bob` (com permissão `WRITE`) executa `--join-repository --repository r`, vê os\
    arquivos versionados e consegue abrir um arquivo em read-only sem checkout.
 3. **Publicação**: `alice` importa `a.bin` com `--publish-imports`; o status de `bob`, após sync,\
    mostra o arquivo na versão 1; `alice` não fica com check-out se `keep_checked_out=False` e o\
    programa foi fechado.
 4. **Checkout exclusivo**: `alice` faz checkout exclusivo; `bob` recebe\
    `exclusive_checkout_unavailable` com o nome de `alice`; `bob` pode usar checkout compartilhado\
    se solicitado.
 5. **Check-in**: após editar e salvar, `alice` faz check-in com comentário; `bob` faz sync e vê\
    `needs_update=true`; `update_binary` incorpora a versão 2; o índice de `bob` é invalidado.
 6. **Check-in sem salvar**: retorna `file_modified` e não cria versão.
 7. **Check-in com programa aberto**: retorna `keep_checked_out=true` efetivo e explica que o\
    Ghidra forçou o estado.
 8. **Undo checkout**: com `keep=true`, gera `.keep`; com `keep=false` e `confirm=true`, descarta\
    a cópia local; o arquivo volta ao estado versionado.
 9. **Read-only**: usuário `READ_ONLY` não consegue checkout/checkin/publicar; erros são\
    `permission_denied`/`read_only`.
10. **Offline**: com servidor parado, status mostra `connected=false`; leituras locais continuam;\
    escritas retornam `server_unavailable`; ao reconectar, `sync_project` volta a funcionar.
11. **Timeout de check-in**: simular queda após envio; a ferramenta retorna `unknown_outcome` e a\
    próxima consulta ao histórico mostra se a versão foi criada; nenhuma versão duplicada.
12. **Segurança**: em `stdio`, nenhuma ferramenta ou opção lê stdin para senha; logs não contêm a\
    senha; `ps` não mostra segredo.
13. **Destrutivo**: `delete_project_binary` em arquivo versionado exige `confirm=true`; sem isso,\
    retorna `delete_blocked`; com check-out de terceiro, recusa.
14. **Regressão**: todos os testes atuais de projeto local continuam passando sem novas flags.

## 9. Estratégia de testes

### 9.1 Unitários

- Parsing/validação de `ServerConfig`/`ProjectSpec`: URLs, portas, combinações inválidas,\
  precedência CLI &gt; env &gt; arquivo, redaction.
- Seleção de autenticador: senha vs SSH vs PKI vs anônimo; rejeição de prompt em stdio.
- Modelos de status e serialização.
- Mapeamento de exceções Java para códigos de erro.
- Regras de política: publish default, checkout default, confirmação de delete, `keep`.
- Invalidação de índice por versão.

### 9.2 Integração com Ghidra Server real

Criar um fixture que:

1. Gera um `server.conf` temporário com `ghidra.repositories.dir` em `tmp_path`.
2. Sobe `ghidraSvr console` em portas altas/livres (base, base+1, base+2), com `-a0 -e0` e o\
   repository path como **último** argumento; espera `Registered Ghidra Server` no log.
3. Só depois do startup, executa `svrAdmin -add` (senha inicial `changeme`) e, se necessário,\
   `-grant`.
4. Executa os cenários com `pyghidra` + `PasswordClientAuthenticator`; cada usuário precisa de\
   uma JVM própria com `-Duser.name=<usuario>`. `HeadlessClientAuthenticator` não serve para\
   senha sem prompt.
5. Encerra o servidor e remove o diretório temporário.

Cenários mínimos:

| Cenário | Atores | Resultado |
| --- | --- | --- |
| Criar repo + projeto + importar + publicar | alice | versão 1 visível para bob |
| Join + checkout read-only | bob | arquivo listado e aberto read-only |
| Checkout exclusivo concorrente | alice, bob | bob falha com código claro |
| Checkout compartilhado + update | alice, bob | versão 2 e update de check-out sem alterações funciona |
| Merge de check-out modificado | alice, bob | headless falha com `merge is not supported in headless mode`; estado preservado |
| Conflito de merge | alice, bob | merge retorna conflito e estado consistente |
| Undo checkout com keep | alice | `.keep` criado; versão preservada |
| Usuário read-only | bob | checkout/checkin negados |
| Servidor offline/reconexão | alice | status e retry corretos |
| Importação transiente | alice | versão no repo, sem `.gpr` local |
| Autenticação SSH | alice | conexão e operações funcionam |
| Autenticação anônima | guest | somente leitura |

### 9.3 Testes de segurança

- Garantir que `--server-password` não exista (ou que seja apenas `--password-env/file`).
- Verificar logs/erros sem segredo.
- Testar `stdio` com opção de prompt: deve falhar antes de iniciar o MCP.
- Testar certificado inválido/autoassinado e CA customizada.
- Testar nomes de repositório/arquivo com `..`, `/`, `\` e caracteres inválidos.

### 9.4 Testes de GUI

- Abrir projeto compartilhado no Ghidra GUI, verificar que o MCP detecta `repository` e expõe\
  status.
- Checkout/checkin pelo MCP na Swing thread, com o programa aberto, respeitando o estado do GUI.
- Não fechar o projeto do GUI no shutdown do MCP.

## 10. Plano de implementação sugerido

### Fase 0 — Spike/validação (1–2 dias)

- Protótipo com JPype: instalar `PasswordClientAuthenticator`, conectar, criar repo, criar\
  projeto via `PyGhidraProjectManager`, importar um binário pequeno, publicar, abrir segundo\
  projeto e fazer checkout.
- Validar abertura de `.gpr` compartilhado e modo transiente.
- Confirmar mensagens de erro e comportamento offline.
- Entregável: script de spike + notas; sem alterar o código de produção.

### Fase 1 — Configuração, autenticação e backend (3–5 dias)

- Criar `server_config.py`/`auth.py` com parsing, validação, redaction e instalação de\
  autenticador.
- Introduzir `ProjectBackend`/`ProjectHandle` desacoplando `PyGhidraContext` de `GhidraProject`.
- Implementar `local`, `shared-local` (criar/join/open/convert) e, se possível, `transient`.
- Ajustar `ProjectSpec` para `.gpr`, diretório e URL `ghidra://`.
- Atualizar `close`/`save` para respeitar read-only e não salvar o que não é gravável.
- Entregável: abertura/criação de projeto compartilhado + status básico + testes unitários.

### Fase 2 — Status, checkout e check-in (3–5 dias)

- Modelos `SharedFileStatus`, `SharedProjectStatus`, `CheckoutInfo`, `VersionInfo`.
- `shared_project_status`, `get_binary_shared_status`, `checkout_binary`, `checkin_binary`,\
  `undo_checkout_binary`.
- Regras de reabrir programa após checkout, salvar antes de check-in, serialização de escrita.
- Invalidar/reindexar após mudança de versão.
- Entregável: ciclo colaborativo básico funcionando + testes de integração.

### Fase 3 — Sync/update/histórico/admin (2–4 dias)

- `sync_project`, `update_binary`, `list_binary_versions`, `list_checkouts`,\
  `terminate_checkout`, `get_changes_since_checkout`.
- Política de delete versionado e operações destrutivas com `confirm`/`dry_run`.
- Auditoria local e logs estruturados.
- Entregável: colaboração completa e operações administrativas.

### Fase 4 — Transiente, CLI e GUI (2–4 dias)

- Modo `ghidra://` (se não entrou na Fase 1), com cache estável de índice.
- Comandos no `pyghidra-mcp-cli` (`server`, `repo`, `project`, `file`/`shared`).
- Paridade no modo GUI.
- Entregável: CLI e GUI com os mesmos fluxos.

### Fase 5 — Endurecimento e documentação (2–3 dias)

- Testes de segurança, offline, timeouts, conflitos e versões.
- README e exemplos.
- Matriz de compatibilidade Ghidra/cliente/servidor.
- Entregável: release candidate.

## 11. Riscos e decisões abertas

### Riscos

1. **RMI + JPype**: chamadas bloqueantes e exceções Java podem ser difíceis de cancelar; um\
   timeout de Python não interrompe a thread Java com segurança. Mitigação: timeouts por operação,\
   TaskMonitor, não matar thread em escrita, verificar estado após timeout.
2. **Autenticador global**: `ClientUtil.setClientAuthenticator` é global por JVM; múltiplos\
   servidores/identidades exigem processos separados. Mitigação: documentar e validar.
3. **Loop de arquivos**: abrir/listar todos os arquivos versionados pode ser caro e abrir\
   conexões desnecessárias. Mitigação: lazy loading e paginação.
4. **Índices obsoletos**: versões diferentes do mesmo path podem gerar respostas erradas.\
   Mitigação: versionar a chave do índice e invalidar.
5. **Check-in acidental**: publicar análise incompleta ou sobrescrever versão. Mitigação:\
   `publish` explícito, `expected_latest_version`, comentário obrigatório/recomendado e auditoria.
6. **Delete destrutivo**: `DomainFile.delete()` apaga todas as versões. Mitigação: confirmação,\
   `dry_run`, bloquear check-out de terceiros.
7. **Self-signed TLS**: usuários podem ser tentados a desabilitar verificação. Mitigação:\
   suportar CA customizada e documentar que, sem `cacerts`, o cliente Ghidra já opera em\
   trust-all (`OpenTrustManager`), o que não deve ser tratado como verificação forte.
8. **Servidor como dependência de teste**: subir Ghidra Server em CI aumenta tempo/flakiness.\
   Mitigação: testes de integração marcados, portas dinâmicas, fixture robusto; testes unitários\
   cobrem a maior parte.

### Decisões abertas

 1. **Criação de projeto via MCP ou somente CLI?** Recomendação: CLI/startup, porque o contexto\
    é único por processo. Se for requisito criar pelo MCP, será necessário um modo “admin sem\
    projeto” ou registry de múltiplos projetos.
 2. `publish` **padrão em importação para projeto shared**: recomendação `False` (seguro), com\
    `--publish-imports` para o fluxo de automação.
 3. `keep_checked_out` **padrão no check-in**: recomendação `True` quando o programa continua\
    aberto (evita a surpresa do Ghidra forçar), com `release_checkout` explícito para fechar e\
    liberar.
 4. **Checkout padrão**: recomendação `exclusive=True`, com `exclusive=False` explícito para\
    colaboração.
 5. **Sync padrão**: recomendação refresh-only; merge somente sob solicitação.
 6. **Suporte a** `.gar` **(project archive)**: fora do escopo inicial; pode ser uma fase futura.
 7. **Keyring**: usar `keyring` ou apenas env/arquivo? Dependência adicional versus segurança.
 8. **Múltiplos repositórios/servidores**: um por processo no V1; multi no futuro.
 9. **Modo read-only de projeto local**: implementar via `ProjectData` read-only ou exigir modo\
    transiente no V1.
10. **Merge assistido**: o Ghidra já faz merge de banco; conflitos finos de programa podem\
    exigir intervenção humana. Definir o que o MCP retorna em caso de conflito.

## 12. Referências consultadas

- `$GHIDRA_INSTALL_DIR/server/svrREADME.md` — instalação, autenticação, `svrAdmin`, backup,\
  checkouts obsoletos, PKI, SSH, opções de servidor.
- `$GHIDRA_INSTALL_DIR/support/analyzeHeadlessREADME.md` — modo URL `ghidra://`, `-commit`,\
  `-readOnly`, `-connect`, `-keystore`, importação em repositório.
- `$GHIDRA_INSTALL_DIR/docs/GhidraClass/Intermediate/Intermediate_Ghidra_Student_Guide.html` —\
  fluxo oficial de criação/join de shared project.
- Classes Java (Ghidra 12.1.2):
  - `ghidra.base.project.GhidraProject`
  - `ghidra.framework.project.DefaultProjectManager`
  - `ghidra.framework.data.DefaultProjectData`
  - `ghidra.framework.model.ProjectLocator`, `DomainFile`, `Project`, `ProjectData`
  - `ghidra.framework.client.ClientUtil`, `RepositoryServerAdapter`, `RepositoryAdapter`,\
    `HeadlessClientAuthenticator`, `PasswordClientAuthenticator`
  - `ghidra.framework.protocol.ghidra.GhidraURL`, `GhidraURLQuery`, `GhidraURLConnection`,\
    `TransientProjectManager`
  - `ghidra.app.util.headless.HeadlessAnalyzer`
  - `ghidra.server.remote.GhidraServer`, `ghidra.server.RepositoryManager`,\
    `ghidra.server.security.*`
- Código atual do repositório: `src/pyghidra_mcp/context.py`, `project_spec.py`, `server.py`,\
  `gui_context.py`, `context_protocol.py`, `mcp_tools.py`, `models.py` e o pacote `cli/`.