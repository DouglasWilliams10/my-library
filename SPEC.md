# SPEC.md

# Feature: Biblioteca pessoal, importação de livros e compartilhamento controlado

## Objetivo

Entregar a primeira versão funcional do MyLibrary, permitindo que múltiplos usuários criem contas, mantenham bibliotecas privadas, importem livros digitais PDF, EPUB e TXT, leiam formatos suportados e compartilhem livros com outros usuários cadastrados mediante permissões.

Esta especificação descreve o escopo inicial. Funcionalidades futuras devem ser adicionadas em novas especificações ou em seções claramente versionadas, sem ampliar o escopo silenciosamente.

## Escopo da primeira versão

### Incluído

- Cadastro, login com e-mail e senha, logout e perfil básico. Login pelo Google não faz parte do MVP.
- Biblioteca privada por usuário.
- Cadastro manual de metadados de livros.
- Upload de PDF, EPUB e TXT.
- Um único arquivo principal por livro no MVP.
- Validação de formato, conteúdo e tamanho; arquivos TXT devem usar codificação UTF-8.
- Limite de upload configurável, com padrão inicial de 50 MB.
- Listagem, pesquisa e filtro da biblioteca.
- Leitura integrada de PDF, EPUB e TXT, conforme suporte da biblioteca escolhida.
- Download autorizado de arquivos.
- Compartilhamento privado com usuário cadastrado.
- Permissões separadas de leitura e download.
- Revogação de compartilhamento.
- Exclusão lógica de livros, revogando seus compartilhamentos ativos.
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

- Visualizar o livro compartilhado se tiver permissão de leitura.
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

### BookFile

- `id`: identificador único.
- `bookId`: livro relacionado.
- `originalFileName`: nome original.
- `storageKey`: referência interna ao arquivo.
- `format`: PDF, EPUB ou TXT.
- `mimeType`: tipo de conteúdo validado.
- `sizeBytes`: tamanho em bytes.
- `checksum`: hash para verificação de integridade, quando implementado.
- `createdAt`: data de importação.
- Cada livro pode ter no máximo um `BookFile` no MVP.

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

A representação de `position` deve respeitar as características de cada formato e da biblioteca de leitura escolhida; a representação concreta por formato ainda precisa ser definida. A retenção do progresso após exclusão lógica de um livro também precisa ser decidida.

## Contratos de API propostos

Os caminhos são propostas iniciais e podem ser ajustados antes da implementação, mantendo os comportamentos e as permissões descritos.

### Autenticação

- `POST /api/auth/register`: criar conta.
- Rotas de sessão e login fornecidas pela solução de autenticação escolhida.

### Livros

- `GET /api/books`: listar livros próprios e compartilhados com o usuário autenticado, identificando a origem e as permissões; a visibilidade dos compartilhamentos com download sem leitura ainda precisa ser definida.
- `POST /api/books`: cadastrar metadados de um livro.
- `GET /api/books/{id}`: consultar livro autorizado.
- `PATCH /api/books/{id}`: editar metadados, somente pelo proprietário.
- `DELETE /api/books/{id}`: excluir logicamente o livro, somente pelo proprietário, revogando seus compartilhamentos ativos e impedindo acessos subsequentes.

### Arquivos

- `POST /api/books/{id}/file`: importar arquivo para um livro próprio.
- `GET /api/books/{id}/file`: obter o arquivo para leitura, se houver permissão de leitura.
- `GET /api/books/{id}/download`: baixar o arquivo, se houver permissão de download.
- As permissões de leitura e download são independentes; a permissão de download sem leitura autoriza o download, mas não a abertura no leitor. A visibilidade de metadados e do item nessa situação ainda precisa ser definida.
- A implementação deve validar autorização no servidor em todas essas operações.

### Compartilhamento

- `POST /api/books/{id}/shares`: conceder acesso a um usuário cadastrado.
- `GET /api/books/{id}/shares`: listar compartilhamentos, somente pelo proprietário.
- `PATCH /api/books/{id}/shares/{shareId}`: alterar permissões, somente pelo proprietário.
- `DELETE /api/books/{id}/shares/{shareId}`: revogar acesso, somente pelo proprietário.

### Progresso de leitura

- `GET /api/books/{id}/progress`: consultar progresso do usuário autenticado.
- `PUT /api/books/{id}/progress`: salvar progresso do usuário autenticado.

## Exemplos de respostas e erros

- `200 OK`: consulta ou alteração concluída.
- `201 Created`: usuário, livro ou compartilhamento criado.
- `204 No Content`: remoção ou revogação concluída sem corpo de resposta.
- `400 Bad Request`: dados inválidos.
- `401 Unauthorized`: usuário não autenticado.
- `403 Forbidden`: usuário autenticado sem permissão.
- `404 Not Found`: recurso inexistente ou não acessível.
- `409 Conflict`: e-mail já cadastrado ou compartilhamento duplicado ativo.
- `413 Payload Too Large`: arquivo excede o limite.
- `415 Unsupported Media Type`: formato não suportado.
- `422 Unprocessable Entity`: arquivo inválido ou corrompido, quando aplicável.

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

- [ ] Criar modelos Book e BookFile.
- [ ] Implementar cadastro manual de livros.
- [ ] Implementar listagem, pesquisa e filtros.
- [ ] Implementar upload de PDF, EPUB e TXT.
- [ ] Validar extensão, conteúdo e tamanho no servidor; exigir UTF-8 para TXT e um único arquivo principal por livro.
- [ ] Aplicar limite de upload configurável, com padrão inicial de 50 MB.
- [ ] Configurar armazenamento local para desenvolvimento.
- [ ] Implementar leitura dos formatos suportados.
- [ ] Implementar download autorizado.
- [ ] Criar testes de upload, leitura, download e autorização.

### Compartilhamento e progresso

- [ ] Criar modelo BookShare.
- [ ] Implementar concessão de acesso por e-mail cadastrado.
- [ ] Implementar permissões independentes de leitura e download, incluindo download sem leitura.
- [ ] Implementar revogação de acesso.
- [ ] Implementar exclusão lógica de livros e revogação de seus compartilhamentos ativos.
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
4. Um usuário consegue importar um PDF, EPUB ou TXT válido dentro do limite configurado; TXT deve ser UTF-8 e cada livro aceita apenas um arquivo principal.
5. Um arquivo inválido, não suportado ou acima do limite é recusado sem deixar cadastro incompleto.
6. Um livro com arquivo compatível pode ser aberto no leitor.
7. O proprietário consegue baixar seu próprio arquivo.
8. O proprietário consegue compartilhar um livro com um usuário cadastrado, concedendo leitura e download de forma independente.
9. Um destinatário sem permissão de download não consegue baixar o arquivo, mesmo tentando acessar o endpoint diretamente.
10. Um destinatário com permissão de download, mas sem leitura, consegue baixar o arquivo, mas não abri-lo no leitor.
11. O proprietário consegue revogar o compartilhamento e o destinatário perde o acesso pela aplicação.
12. Ao excluir logicamente um livro, seus compartilhamentos ativos são revogados e antigos destinatários perdem o acesso pela aplicação.
13. O progresso de leitura de um usuário não altera o progresso de outro usuário, e a posição respeita as características do formato.
14. A aplicação funciona em telas de desktop e dispositivos móveis.

## Decisões ainda pendentes

- Campos obrigatórios adicionais no cadastro, além de e-mail e senha.
- Biblioteca de leitura EPUB e representação concreta da posição de progresso para cada formato.
- Visibilidade de metadados e do item na interface para destinatários com download, mas sem leitura.
- Se será permitido substituir o arquivo principal de um livro e, em caso afirmativo, o efeito sobre o progresso de leitura existente.
- Política de remoção física e retenção do arquivo após exclusão lógica.
- Retenção ou remoção do progresso de leitura após exclusão lógica.
- Unidade exata usada para configurar e aplicar o limite de 50 MB.
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
