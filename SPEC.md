# SPEC.md

# Funcionalidade: Biblioteca pessoal, importação de livros e compartilhamento controlado

## Objetivo

Entregar a primeira versão funcional do MyLibrary, permitindo que múltiplos usuários criem contas, mantenham bibliotecas privadas, importem livros digitais PDF, EPUB e TXT, leiam formatos suportados e compartilhem livros com outros usuários cadastrados mediante permissões.

Esta especificação descreve o escopo inicial. Funcionalidades futuras devem ser adicionadas em novas especificações ou em seções claramente versionadas, sem ampliar o escopo silenciosamente.

## Escopo da primeira versão

### Incluído

- Cadastro, login com e-mail e senha, logout e perfil básico. Login pelo Google não faz parte do MVP.
- Biblioteca privada por usuário.
- Cadastro manual de metadados de livros.
- Upload de PDF, EPUB e TXT.
- Histórico de versões de arquivos: cada livro pode ter várias versões históricas, mas somente uma versão atual.
- Validação de formato, conteúdo e tamanho; arquivos TXT devem usar codificação UTF-8.
- Limite de upload de 50 MiB (52.428.800 bytes), configurável por variável de ambiente ou configuração centralizada, validado no servidor e sem duplicação do valor pelo código.
- Listagem, pesquisa e filtro da biblioteca.
- Leitura integrada de PDF, EPUB e TXT, conforme suporte da biblioteca escolhida.
- Download autorizado de arquivos.
- Compartilhamento privado com usuário cadastrado.
- Permissões separadas de leitura e download.
- Revogação de compartilhamento.
- Substituição do arquivo atual pelo proprietário, criando uma nova versão, preservando os metadados das versões, removendo fisicamente a versão anterior e reiniciando o progresso de todos os usuários sem alterar os metadados do livro ou as permissões de compartilhamento.
- Exclusão lógica de livros, revogando seus compartilhamentos ativos, removendo fisicamente os arquivos de todas as versões e preservando metadados do livro, das versões e registros de progresso no banco.
- Registro do progresso de leitura por usuário e livro, com posição adequada ao formato.
- Interface responsiva com identidade clássica e aconchegante.

### Fora do escopo inicial

- Aplicativo nativo para celular.
- Leitura de MOBI, AZW/AZW3, DOCX e CBZ.
- OCR de livros digitalizados.
- Conversão entre formatos.
- Marketplace ou catálogo público de livros.
- Compartilhamento anônimo ou links públicos.
- Sincronização offline entre dispositivos.
- DRM ou proteção contra cópia de arquivos já baixados.
- Integração com lojas ou serviços externos de e-books.

## Perfis e permissões

### Usuário autenticado

- Criar e editar seu perfil.
- Cadastrar livros em sua biblioteca.
- Importar e baixar seus próprios arquivos.
- Ler seus próprios livros.
- Compartilhar seus livros com usuários cadastrados.
- Consultar livros compartilhados com ele, conforme permissões concedidas.

### Proprietário do livro

- Editar metadados do próprio livro.
- Remover o livro da própria biblioteca.
- Conceder ou revogar acesso de leitura.
- Conceder ou revogar permissão de download.

### Destinatário do compartilhamento

- Visualizar os detalhes do livro se tiver permissão de leitura.
- Se tiver `canDownload = true` e `canRead = false`, visualizar somente título e capa na biblioteca de compartilhamentos recebidos; não acessar os demais metadados nem o leitor.
- Baixar o arquivo somente se tiver permissão de download.
- Não pode editar, excluir ou compartilhar novamente o livro original.

## Modelo de dados inicial

Os nomes abaixo são conceituais; os nomes finais devem ser consistentes no código e no schema do Prisma.

### User

- `id`: identificador único.
- `name`: nome de exibição.
- `email`: e-mail único.
- `passwordHash`: hash da senha usada no cadastro e login do MVP.
- `createdAt`: data de criação.
- `updatedAt`: data da última atualização.

### Book

- `id`: identificador único.
- `ownerId`: usuário proprietário.
- `title`: título.
- `author`: autor.
- `description`: descrição opcional.
- `publisher`: editora opcional.
- `category`: categoria opcional.
- `coverPath`: referência opcional da capa.
- `createdAt`: data de cadastro.
- `updatedAt`: data de atualização.
- A exclusão é lógica; o campo ou mecanismo que a representa será definido no schema.
- Ao iniciar a exclusão lógica, o livro e todas as suas versões ficam imediatamente inacessíveis. A exclusão só é concluída após a remoção física confirmada de todas as versões; livros excluídos não podem ser acessados, lidos ou baixados por nenhum usuário.
- A exclusão lógica preserva os metadados do livro e os registros de progresso. O estado pendente ou concluído da exclusão deve poder ser consultado pelo proprietário sem reabrir o acesso ao livro ou às versões.
- O estado do processo de exclusão deve ser mantido em um registro persistente, independentemente da existência dos arquivos físicos ou da limpeza de `storageKey`.

### BookDeletion

- Entidade conceitual que registra persistentemente o processo de exclusão lógica de um livro.
- `id`: identificador único.
- `bookId`: livro relacionado.
- `status`: estado do processo; somente `pending` ou `completed` no MVP.
- `startedAt`: data de início da exclusão lógica.
- `completedAt`: data em que todas as remoções físicas aplicáveis foram confirmadas e o processo passou a `completed`.
- Um livro pode possuir somente um processo ativo de exclusão. Reprocessar uma exclusão já iniciada não cria um novo processo nem reabre o livro.
- `pending` significa que a exclusão lógica já foi efetivada, o livro e suas versões permanecem inacessíveis e uma ou mais tarefas físicas ainda não foram confirmadas.
- `completed` significa que todas as tarefas físicas aplicáveis foram concluídas e o processo de exclusão pode ser considerado finalizado.
- Falhas da remoção física não alteram o status do livro ou de `BookDeletion` para um estado de sucesso. Elas permanecem registradas em `FileRemovalTask`.

### BookFile

- `id`: identificador único.
- `bookId`: livro relacionado.
- `originalFileName`: nome original.
- `storageKey`: referência interna ao arquivo, nula após confirmação da remoção física.
- `format`: PDF, EPUB ou TXT.
- `mimeType`: tipo de conteúdo validado.
- `sizeBytes`: tamanho em bytes.
- `checksum`: hash para verificação de integridade, quando implementado.
- `createdAt`: data de importação.
- `isCurrent`: indica se esta versão é a atual; cada livro pode ter no máximo uma versão atual.
- `deletedAt`: data da remoção física confirmada, nula até essa confirmação.
- `deletionPending`: visão conceitual derivada da existência de uma tarefa `FileRemovalTask` pendente, com falha ou em execução; não substitui nem duplica o registro persistente da tarefa.
- Cada importação inicial e cada substituição efetivada cria um novo registro `BookFile`; um livro pode ter várias versões, mas no máximo uma pode estar atual.
- Os registros históricos de `BookFile` preservam metadados para auditoria e rastreabilidade, mas não preservam o conteúdo binário das versões removidas nem permitem recuperá-las pela aplicação.
- A aplicação e o banco de dados devem proteger a invariável de no máximo uma versão atual por livro, inclusive contra operações concorrentes. Restrições de integridade e transações devem ser avaliadas para a solução PostgreSQL/Prisma; esta especificação não determina uma implementação específica.
- O novo arquivo deve ser completamente validado antes de efetivar a substituição. Após a validação, a nova versão torna-se atual e a anterior histórica e imediatamente inacessível. A substituição não altera metadados do livro nem permissões de compartilhamento.
- A efetivação da nova versão não depende da remoção física da versão anterior. Se essa remoção falhar, a substituição não é desfeita: a nova versão permanece atual e funcional, e a anterior permanece histórica, inacessível e com remoção pendente registrada para recuperação.
- Após a confirmação da remoção física, o registro histórico da versão é preservado, `storageKey` é limpo e `deletedAt` é preenchido. Enquanto a remoção estiver pendente, o registro e seus metadados são preservados, `storageKey` não é limpo e `deletedAt` não indica uma remoção concluída.
- Após a substituição, o progresso de todos os usuários do livro é reiniciado. Versões históricas nunca podem ser lidas ou baixadas pela aplicação.
- Ao iniciar a exclusão lógica do livro, o livro e todas as versões ficam imediatamente inacessíveis e seus compartilhamentos ativos são revogados. A exclusão só é concluída quando todos os arquivos físicos forem removidos com confirmação.
- Na exclusão lógica, os registros e metadados de todas as versões são preservados. Cada remoção confirmada limpa `storageKey` e preenche `deletedAt`; falhas mantêm as versões afetadas inacessíveis e pendentes de recuperação. A recuperação pode repetir a remoção sem restaurar o acesso.

### FileRemovalTask

- Entidade conceitual que registra uma tentativa persistente de remover fisicamente um arquivo armazenado localmente pelo MyLibrary.
- `id`: identificador único.
- `targetType`: tipo do arquivo afetado, como `book_file` ou `local_cover`.
- `targetId`: identificador do registro lógico afetado, por exemplo o `BookFile` ou a capa de um `Book`.
- `storageKey`: referência local persistida até a confirmação da remoção física.
- `reason`: motivo da tarefa, como `replacement` ou `book_deletion`.
- `status`: estado da tarefa; estados claros no MVP:
  - `pending`: aguarda uma tentativa.
  - `in_progress`: possui uma tentativa em execução pela aplicação.
  - `failed`: uma tentativa terminou com falha e a tarefa pode ser tentada novamente.
  - `completed`: o armazenamento confirmou que o arquivo foi removido ou que já não existia no destino.
- `attemptCount`: número de tentativas realizadas.
- `lastAttemptAt`: data da última tentativa.
- `lastError`: informação técnica mínima para diagnóstico, sem caminhos internos desnecessários ou conteúdo do arquivo.
- `completedAt`: data da conclusão confirmada.
- `createdAt` e `updatedAt`: datas de criação e atualização da tarefa.
- Uma tarefa representa uma única referência de armazenamento local e deve ser idempotente: repetir a remoção do mesmo alvo não deve causar dano ou alterar a efetivação lógica já realizada.
- Capas externas referenciadas por URL não são alvo de `FileRemovalTask`; o MyLibrary não solicita ou realiza remoção no servidor externo.
- A capa local de um livro também entra no mecanismo quando sua capa anterior é substituída ou quando o livro é excluído.
- Tarefas encerradas como `completed` são históricas e não devem ser reexecutadas automaticamente.

## Persistência e recuperação de remoções

- Criar `BookDeletion` e `FileRemovalTask` em persistência são pré-requisitos para considerar efetivada uma substituição ou exclusão que envolva remoção física.
- O estado lógico deve ser confirmado no banco antes de tentar remover arquivos. A tentativa física não pode desfazer ou atrasar a efetivação lógica já aprovada.
- Ao iniciar ou reiniciar a aplicação, o processo de recuperação deve localizar tarefas persistidas em `pending`, `failed` ou em `in_progress` e permitir novas tentativas.
- Uma tarefa encontrada como `in_progress` após reinicialização pode ser retomada, pois reiniciar a aplicação não prova que a remoção falhou ou teve sucesso.
- Novas tentativas devem ocorrer pelo menos no processo de recuperação iniciado pela aplicação. O MVP não exige uma agenda externa ou uma ferramenta específica de fila.
- Uma tentativa começa alterando `status` para `in_progress`, executa a remoção por meio da abstração de armazenamento e termina em `completed` após confirmação ou em `failed` após falha.
- A ausência do arquivo no armazenamento durante uma operação idempotente é considerada conclusão da remoção, pois o resultado esperado já foi alcançado.
- Após `completed`, a referência de armazenamento da entidade afetada deve ser limpa e `deletedAt` deve ser preenchido no `BookFile`; em uma capa local removida, a referência local pode ser limpa apenas no contexto já definido como histórico ou excluído.
- `storageKey` não pode ser limpo por uma falha nem por uma simples reinicialização.
- A recuperação não pode tornar o livro excluído acessível, tornar uma versão substituída atual novamente ou reativar compartilhamentos.
- Nenhuma tarefa depende apenas de memória do processo. Depois de reiniciar a aplicação, o estado consultável e as pendências persistidas continuam disponíveis.

### BookShare

- `id`: identificador único.
- `bookId`: livro compartilhado.
- `ownerId`: proprietário que concedeu acesso.
- `recipientId`: usuário destinatário.
- `canRead`: permite leitura.
- `canDownload`: permite download.
- `createdAt`: data de concessão.
- `revokedAt`: data de revogação, nula enquanto ativo.

### ReadingProgress

- `id`: identificador único.
- `userId`: usuário leitor.
- `bookId`: livro lido.
- `position`: posição de leitura em formato definido pelo leitor utilizado.
- `progressPercent`: percentual de progresso, quando calculável.
- `updatedAt`: data da última atualização.

Restrições de integridade, índices e relacionamentos devem ser definidos no schema do Prisma. O progresso deve ser único por combinação de usuário e livro.

A representação de `position` deve respeitar as características de cada formato e da biblioteca de leitura escolhida; a representação concreta por formato ainda precisa ser definida. Na substituição do arquivo, o progresso de todos os usuários do livro deve voltar ao início. Na exclusão lógica, os registros de progresso permanecem preservados no banco.

## Contratos de API propostos

Os caminhos são propostas iniciais e podem ser ajustados antes da implementação, mantendo os comportamentos e as permissões descritos.

### Autenticação

- `POST /api/auth/register`: criar conta.
- Rotas de sessão e login fornecidas pela solução de autenticação escolhida.

### Livros

- `GET /api/books`: listar livros próprios e compartilhados com o usuário autenticado, identificando a origem e as permissões. Para compartilhamento com `canDownload = true` e `canRead = false`, incluir o item na biblioteca recebida e disponibilizar somente título e capa.
- `POST /api/books`: cadastrar metadados de um livro.
- `GET /api/books/{id}`: consultar livro autorizado. Para compartilhamento com `canDownload = true` e `canRead = false`, retornar somente título e capa.
- `PATCH /api/books/{id}`: editar metadados, somente pelo proprietário.
- `DELETE /api/books/{id}`: iniciar a exclusão lógica, somente pelo proprietário. No início, revogar os compartilhamentos ativos e bloquear imediatamente o acesso ao livro e a todas as versões; preservar os metadados do livro e os registros de progresso.
  - Retornar `204 No Content`, sem corpo de resposta, somente quando todos os arquivos físicos das versões tiverem sido removidos com confirmação e o registro persistente do processo de exclusão também estiver marcado como `completed`.
  - Se a remoção de qualquer versão falhar, retornar `202 Accepted` com corpo contendo `deletionStatus: "pending"` e `statusUrl: "/api/books/{id}/deletion-status"`. A resposta deve permitir ao cliente consultar posteriormente o estado. Não apresentar a exclusão como concluída; manter livro e versões inacessíveis e registrar as remoções pendentes para recuperação.
- `GET /api/books/{id}/deletion-status`: consultar, somente pelo proprietário, o registro persistente do processo de exclusão, sem disponibilizar metadados do livro ou acesso a arquivos. A consulta independe da existência dos arquivos físicos e de `storageKey` estar preenchido ou limpo. Retornar `200 OK` com `deletionStatus: "pending"` enquanto houver remoção não confirmada e `deletionStatus: "completed"` somente após a remoção física confirmada de todas as versões. Novas tentativas de remoção não podem restaurar o acesso.

O estado persistido não precisa expor tarefas internas individuais por um contrato obrigatório do MVP. Se o contrato de diagnóstico for ampliado, ele deve continuar protegido pelo proprietário e não deve revelar caminhos internos, referências privadas ou dados de usuários.

### Arquivos

- `POST /api/books/{id}/file`: importar uma primeira versão para um livro próprio, criando um registro `BookFile` atual.
- `PUT /api/books/{id}/file`: substituir a versão atual, somente pelo proprietário. Validar completamente o novo arquivo antes de efetivar a operação; após essa validação, criar o novo registro atual e marcar a versão anterior como histórica e imediatamente inacessível; preservar os metadados do livro e os compartilhamentos; reiniciar o progresso de todos os usuários; tentar remover fisicamente o arquivo antigo.
  - Retornar `200 OK` com `replacementStatus: "effective"` quando a substituição estiver efetivada. Informar `previousVersionCleanup: "completed"` quando a remoção física da versão anterior tiver sido confirmada ou `"pending"` quando a remoção falhar e precisar de recuperação.
  - A limpeza física pendente não desfaz nem invalida a substituição: a nova versão permanece atual, acessível conforme as permissões e funcional. A versão anterior permanece histórica e inacessível; não registrar sua remoção como concluída antes da confirmação.
- `GET /api/books/{id}/file`: obter o arquivo para leitura, se houver permissão de leitura.
- `GET /api/books/{id}/download`: baixar o arquivo, se houver permissão de download.
- Os contratos de leitura e download operam somente sobre a versão atual; nenhuma versão histórica pode ser acessada pela aplicação.
- As permissões de leitura e download são independentes. Com `canDownload = true` e `canRead = false`, o destinatário pode baixar o arquivo, não pode abrir o leitor e só pode consultar título e capa. O servidor deve validar autorização e limitar os dados retornados em todas as operações.
- A implementação deve validar autorização no servidor em todas essas operações.
- O limite máximo do upload é configurável por variável de ambiente ou configuração centralizada; o padrão é 50 MiB (52.428.800 bytes). A validação é obrigatória no servidor e o limite não deve ser duplicado pelo código.

### Compartilhamento

- `POST /api/books/{id}/shares`: conceder acesso a um usuário cadastrado.
- `GET /api/books/{id}/shares`: listar compartilhamentos, somente pelo proprietário.
- `PATCH /api/books/{id}/shares/{shareId}`: alterar permissões, somente pelo proprietário.
- `DELETE /api/books/{id}/shares/{shareId}`: revogar acesso, somente pelo proprietário.

### Progresso de leitura

- `GET /api/books/{id}/progress`: consultar progresso do usuário autenticado.
- `PUT /api/books/{id}/progress`: salvar progresso do usuário autenticado.
- O acesso ao progresso de um livro compartilhado exige permissão de leitura; permissão de download, por si só, não autoriza consultar ou alterar progresso.

## Exemplos de respostas e erros

- `200 OK`: consulta ou alteração concluída.
- `201 Created`: usuário, livro ou compartilhamento criado.
- `204 No Content`: exclusão concluída ou revogação concluída, sem corpo de resposta. Para exclusão de livro, só retornar após a confirmação da remoção física de todas as versões e após marcar como `completed` o registro persistente do processo de exclusão.
- `400 Bad Request`: dados inválidos.
- `401 Unauthorized`: usuário não autenticado.
- `403 Forbidden`: usuário autenticado sem permissão.
- `404 Not Found`: recurso inexistente ou não acessível.
- `409 Conflict`: e-mail já cadastrado ou compartilhamento duplicado ativo.
- `413 Payload Too Large`: arquivo excede o limite.
- `415 Unsupported Media Type`: formato não suportado.
- `422 Unprocessable Entity`: arquivo inválido ou corrompido, quando aplicável.
- `202 Accepted`: exclusão lógica iniciada, mas remoção física de uma ou mais versões ainda pendente; o corpo deve conter `deletionStatus: "pending"` e `statusUrl` para consultar o estado posteriormente.
- Uma substituição efetivada retorna `200 OK` com `replacementStatus: "effective"` mesmo que a remoção física da versão anterior esteja pendente, identificando essa condição em `previousVersionCleanup`. A nova versão continua funcionando.
- Falha de remoção física não pode ser apresentada como conclusão de exclusão; o estado e a recuperação devem ser identificados e consultáveis sem reabilitar acesso. A limpeza de cada versão só é concluída após confirmação física.

As respostas não devem revelar a existência de recursos privados de outros usuários.

## Interface esperada

- Página de apresentação e acesso.
- Cadastro e login.
- Dashboard da biblioteca.
- Listagem em grade e/ou lista.
- Formulário de cadastro e importação.
- Detalhes do livro.
- Leitor digital.
- Área de livros compartilhados.
- Gerenciamento de compartilhamentos.
- Perfil do usuário.

## Identidade visual

- Estilo clássico, acolhedor e confortável para leitura.
- Paleta sugerida: creme, papel, marrom, madeira e tons terrosos.
- Tipografia legível, com hierarquia clara.
- Capas dos livros em destaque.
- Leitor com controles discretos e boa largura de texto.
- Interface responsiva para desktop, tablet e celular.
- Usar tokens semânticos de tema; evitar cores fixas espalhadas pelos componentes.

## Checklist de implementação

### Fundação e autenticação

- [ ] Criar aplicação e configurar TypeScript.
- [ ] Configurar banco de dados e Prisma.
- [ ] Criar modelo de usuário.
- [ ] Implementar cadastro, login por e-mail e senha e logout (sem login pelo Google no MVP).
- [ ] Proteger páginas e endpoints privados.
- [ ] Criar testes para autenticação e isolamento entre usuários.

### Biblioteca e arquivos

- [ ] Criar modelos Book e BookFile com histórico de versões e garantir, também no banco de dados, no máximo uma versão atual por livro, inclusive em operações concorrentes; avaliar as limitações do Prisma antes de escolher a implementação.
- [ ] Implementar cadastro manual de livros.
- [ ] Implementar listagem, pesquisa e filtros.
- [ ] Implementar upload de PDF, EPUB e TXT.
- [ ] Validar extensão, conteúdo e tamanho no servidor; exigir UTF-8 para TXT e garantir somente uma versão atual por livro.
- [ ] Aplicar limite de upload configurável por variável de ambiente ou configuração centralizada, padrão de 50 MiB (52.428.800 bytes), validado no servidor e sem valor duplicado no código.
- [ ] Permitir que o proprietário substitua o arquivo atual somente após validar completamente o novo arquivo; efetivar a nova versão como atual e a anterior como histórica e imediatamente inacessível; preservar metadados e compartilhamentos e reiniciar o progresso de todos os usuários.
- [ ] Fazer com que a falha na remoção física da versão anterior deixe a substituição efetivada, mantenha a nova versão funcional e registre a limpeza pendente para recuperação.
- [ ] Registrar e recuperar falhas de remoção física de versões sem permitir leitura ou download da versão pendente; limpar `storageKey` e preencher `deletedAt` somente após confirmação.
- [ ] Configurar armazenamento local para desenvolvimento.
- [ ] Implementar leitura dos formatos suportados.
- [ ] Implementar download autorizado.
- [ ] Criar testes de upload, leitura, download e autorização.

### Compartilhamento e progresso

- [ ] Criar modelo BookShare.
- [ ] Implementar concessão de acesso por e-mail cadastrado.
- [ ] Implementar permissões independentes de leitura e download, incluindo download sem leitura.
- [ ] Implementar revogação de acesso.
- [ ] Implementar exclusão lógica, revogação dos compartilhamentos ativos e bloqueio imediato do livro e de todas as versões; concluir a exclusão somente após remoção física confirmada de todas as versões, preservando metadados e progresso.
- [ ] Manter um registro persistente do processo de exclusão, independente da existência física dos arquivos ou do valor de `storageKey`.
- [ ] Expor o estado pendente da exclusão ao cliente com informação suficiente para consulta posterior; permitir novas tentativas de remoção sem restaurar o acesso.
- [ ] Criar modelo ReadingProgress.
- [ ] Salvar e recuperar progresso por usuário e livro.
- [ ] Criar testes para permissões e progresso.

### Finalização

- [ ] Aplicar identidade visual clássica e aconchegante.
- [ ] Validar responsividade.
- [ ] Executar lint, verificação de tipos, testes e build.
- [ ] Documentar instalação e execução no README.
- [ ] Criar apresentação do produto em `/docs`.

## Critérios de aceite da primeira versão

1. Um visitante consegue criar uma conta e entrar na aplicação usando e-mail e senha; login pelo Google não é oferecido.
2. Um usuário não consegue consultar a biblioteca privada de outro usuário.
3. Um usuário consegue cadastrar um livro sem arquivo digital.
4. Um usuário consegue importar um PDF, EPUB ou TXT válido de até o limite configurado no servidor (padrão de 50 MiB, 52.428.800 bytes), com TXT em UTF-8; a importação cria uma versão atual e um livro nunca tem mais de uma versão atual.
5. Um arquivo inválido, não suportado ou maior que o limite configurado é recusado no servidor sem deixar cadastro incompleto.
6. Um livro com arquivo compatível pode ser aberto no leitor.
7. O proprietário consegue baixar seu próprio arquivo.
8. O proprietário consegue compartilhar um livro com um usuário cadastrado, concedendo leitura e download de forma independente.
9. Um destinatário sem permissão de download não consegue baixar o arquivo, mesmo tentando acessar o endpoint diretamente.
10. Um destinatário com `canDownload = true` e `canRead = false` vê o livro na biblioteca recebida, pode consultar apenas título e capa, consegue baixar o arquivo e não consegue abrir o leitor; acessos diretos também respeitam essas restrições no servidor.
11. O proprietário consegue revogar o compartilhamento e o destinatário perde o acesso pela aplicação.
12. O proprietário consegue substituir o arquivo atual após validação completa; é criado novo registro atual e a versão anterior torna-se histórica e imediatamente inacessível; metadados e permissões permanecem iguais e o progresso de todos os usuários reinicia. Se a validação falhar, a versão atual e o progresso permanecem inalterados.
13. Uma versão histórica nunca pode ser lida ou baixada pela aplicação, inclusive enquanto sua remoção física estiver pendente.
14. Após a remoção física confirmada de uma versão, seu registro e metadados históricos permanecem, `storageKey` é limpo e `deletedAt` é registrado.
15. Se a remoção física da versão anterior falhar durante a substituição, a API ainda informa a substituição como efetivada, a nova versão permanece atual e funcional, e a antiga fica histórica e inacessível com remoção pendente registrada.
16. A restrição de no máximo uma versão atual por livro é protegida também no banco de dados contra inconsistências e operações concorrentes.
17. Ao iniciar a exclusão lógica, o livro e todas as versões ficam imediatamente inacessíveis e os compartilhamentos ativos são revogados; metadados do livro, metadados das versões e registros de progresso permanecem no banco.
18. Os registros históricos de `BookFile` preservam os metadados para auditoria e rastreabilidade, mas não armazenam o conteúdo binário das versões removidas nem permitem recuperá-las pela aplicação.
19. `DELETE /api/books/{id}` retorna `204 No Content`, sem corpo, somente após a remoção física confirmada de todas as versões e após o registro persistente do processo de exclusão estar marcado como `completed`.
20. Se qualquer remoção estiver pendente, `DELETE /api/books/{id}` retorna `202 Accepted` com `deletionStatus: "pending"` e `statusUrl`; o cliente consegue consultar o estado posteriormente.
21. `GET /api/books/{id}/deletion-status` consulta o registro persistente do processo de exclusão mesmo quando os arquivos físicos já não existem ou `storageKey` já foi limpo; informa `pending` até todas as remoções serem confirmadas e `completed` depois disso.
22. O estado `pending` continua consultável após reiniciar a aplicação, comprovando que o processo de exclusão não depende apenas da memória do processo.
23. `FileRemovalTask` continua persistida em `pending`, `failed` ou `in_progress` após reiniciar a aplicação e permite que o processo de recuperação reexecute a remoção sem reabrir o acesso.
24. Quando uma tentativa confirma sucesso, `FileRemovalTask` passa para `completed`, a referência local é limpa e `deletedAt` é preenchido no `BookFile` afetado.
25. Quando uma tentativa falha, `FileRemovalTask` passa para `failed` e permite novas tentativas; `storageKey` e `deletedAt` permanecem consistentes com uma remoção ainda não confirmada.
26. A substituição efetiva cria ou mantém uma tarefa persistente para a versão anterior se a remoção física falhar; a nova versão continua atual e funcional.
27. Na exclusão lógica, uma tarefa persistente é criada ou mantida para cada arquivo local aplicável de todas as versões e para a capa local do livro, quando houver.
28. Capas externas por URL não geram tarefas de remoção física e não são apagadas no servidor externo.
29. A recuperação considera uma ausência já confirmada no armazenamento como remoção concluída e não exige que o arquivo exista para uma segunda tentativa.
30. Novas tentativas de remoção durante a recuperação não restauram o acesso ao livro ou a qualquer versão; após a confirmação de cada remoção, `storageKey` é limpo e `deletedAt` é registrado.
31. O progresso de leitura de um usuário não altera o progresso de outro usuário, e a posição respeita as características do formato.
32. A aplicação funciona em telas de desktop e dispositivos móveis.

## Decisões ainda pendentes

- Campos obrigatórios adicionais no cadastro, além de e-mail e senha.
- Biblioteca de leitura EPUB e representação concreta da posição de progresso para cada formato.
- Detalhes de implementação para proteger a unicidade da versão atual no banco com Prisma/PostgreSQL, após avaliação das limitações do Prisma.
- Provedor e configuração de armazenamento de objetos para produção.
- Versões efetivas das dependências e frameworks de teste, conforme a stack instalada e as decisões de configuração.

## Orientação ao GitHub Copilot

Antes de implementar qualquer funcionalidade:

1. Leia `ARCHITECTURE.md`, `STACK.md`, `BUSINESS_RULES.md` e este `SPEC.md`.
2. Informe resumidamente quais regras e partes da arquitetura serão afetadas.
3. Implemente somente o escopo solicitado, sem criar funcionalidades não especificadas.
4. Não invente dependências ou versões; confira o `package.json`.
5. Implemente validações e autorização no servidor, não apenas na interface.
6. Crie ou atualize testes para cada regra implementada.
7. Ao concluir, informe arquivos alterados, decisões tomadas, testes executados e pendências.
8. Se houver conflito entre os arquivos, interrompa e solicite a atualização da especificação antes de assumir uma regra.
