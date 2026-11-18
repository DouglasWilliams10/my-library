# ARCHITECTURE.md

## Visão geral

O MyLibrary será uma aplicação web organizada como um monólito modular, com separação entre interface, aplicação, domínio e infraestrutura. Essa abordagem mantém o projeto simples para desenvolvimento individual, mas permite evoluir os módulos sem misturar responsabilidades.

## Estrutura inicial

```text
src/
├── app/                    # Páginas e rotas da aplicação
│   ├── (auth)/             # Cadastro e login
│   ├── (library)/          # Biblioteca e leitura
│   └── api/                # Endpoints HTTP
├── components/             # Componentes reutilizáveis de interface
├── modules/
│   ├── users/              # Cadastro com e-mail e senha, autenticação e perfil
│   ├── books/              # Catálogo e metadados dos livros
│   ├── files/              # Upload, validação e download
│   ├── reader/             # Leitura e progresso
│   └── sharing/            # Compartilhamento e permissões
├── lib/                    # Configurações e utilitários compartilhados
└── generated/              # Código gerado por ferramentas, se aplicável

prisma/
├── schema.prisma
└── migrations/

storage/                    # Arquivos locais apenas em desenvolvimento
docs/                       # Apresentação do produto
tests/                      # Testes de integração e ponta a ponta
```

A estrutura poderá ser ajustada durante a implementação, desde que as responsabilidades e regras de dependência sejam preservadas.

## Camadas e responsabilidades

- **Apresentação (`app` e `components`):** páginas, formulários, componentes e interação com o usuário.
- **Aplicação (`modules`):** casos de uso, orquestração de operações e autorização de ações.
- **Domínio (`modules/*`):** entidades, regras de negócio e validações independentes de framework.
- **Infraestrutura (`prisma`, `lib` e adaptadores):** banco de dados, autenticação, armazenamento de arquivos e bibliotecas externas.

## Regras de dependência

1. A apresentação chama os casos de uso da aplicação.
2. Os casos de uso aplicam as regras do domínio.
3. A infraestrutura implementa os recursos necessários, como persistência e armazenamento.
4. O domínio não deve depender de Next.js, React, Prisma ou bibliotecas de interface.
5. Componentes visuais não devem acessar o banco de dados diretamente.
6. Toda operação sobre livros ou arquivos deve verificar a identidade e a permissão do usuário no servidor.

## Módulos do domínio

- **User:** identidade, credenciais e perfil do usuário.
- **Book:** metadados, proprietário e situação do livro, incluindo exclusão lógica.
- **BookFile:** metadados e ciclo de vida das versões de arquivos do livro; várias versões históricas podem existir, mas no máximo uma pode estar atual.
- **Library:** organização dos livros de um usuário.
- **Share:** relação de compartilhamento entre proprietário, livro e destinatário.
- **ReadingProgress:** posição e progresso de leitura de um usuário em um livro.

## Armazenamento de arquivos

O acesso aos arquivos deve ocorrer por meio de uma interface de armazenamento, sem acoplar as regras de negócio ao provedor escolhido.

- Desenvolvimento: armazenamento local não versionado pelo Git.
- Produção: a abstração deve permitir migração futura para armazenamento de objetos. O provedor e a configuração de produção ainda não estão definidos.
- O banco guarda metadados e referências das versões, não o conteúdo binário dos livros.
- Cada importação inicial ou substituição validada cria um novo registro de versão. A substituição torna a nova versão atual e a anterior histórica, sem manter mais de uma versão atual por livro. Essa invariável deve ser garantida pela aplicação e também pelo banco de dados, inclusive diante de operações concorrentes.
- A aplicação e o banco de dados devem coordenar operações concorrentes que importem, substituam ou removam versões, preservando a invariável de no máximo uma versão atual por livro. A forma de implementar essa proteção com Prisma e PostgreSQL deve ser avaliada antes de ser escolhida.
- Somente a versão atual pode ser lida ou baixada pela aplicação. Versões históricas ficam imediatamente inacessíveis, mesmo enquanto a remoção física estiver pendente.
- A abstração de armazenamento deve permitir remover fisicamente uma versão substituída e todas as versões de um livro excluído, sem acoplar o domínio ao provedor.
- O registro e os metadados da versão são preservados, inclusive enquanto houver uma remoção pendente. Somente após a confirmação da remoção física sua referência de armazenamento (`storageKey`) é limpa e a data da remoção (`deletedAt`) é registrada.
- Falhas de remoção devem ser registradas para recuperação posterior. A camada de aplicação coordena o bloqueio de acesso e a pendência; não pode registrar a remoção como concluída antes da confirmação do armazenamento.
- Se a remoção da versão antiga falhar após uma substituição efetivada, a nova versão continua atual e funcional; a falha não desfaz a substituição. A versão antiga permanece histórica, inacessível e com remoção pendente para novas tentativas.

## Segurança e privacidade

- Senhas nunca são armazenadas em texto puro.
- O cadastro e o login do MVP usam e-mail e senha; login pelo Google não faz parte do MVP.
- Rotas e operações privadas exigem autenticação.
- O servidor verifica propriedade ou permissão de compartilhamento antes de visualizar ou baixar um arquivo.
- A autorização de leitura e download deve restringir o acesso à versão atual; identificadores ou referências de versões históricas não podem permitir acesso.
- Arquivos enviados são validados por extensão, tipo de conteúdo e tamanho.
- Arquivos TXT devem usar codificação UTF-8.
- Segredos e credenciais ficam em variáveis de ambiente e não são enviados ao Git.

O início da exclusão lógica torna o livro e todas as suas versões imediatamente inacessíveis e revoga seus compartilhamentos ativos. A exclusão só é concluída depois que todos os arquivos físicos das versões forem removidos com confirmação. Os metadados do livro, os registros de progresso e os registros históricos das versões permanecem no banco de dados.

Se houver falha no armazenamento durante a remoção de uma versão substituída ou de um livro excluído, a versão e, no caso de exclusão, o livro devem permanecer inacessíveis para todos, inclusive o proprietário, mesmo que um arquivo ainda exista fisicamente. A falha não pode restaurar permissões nem permitir acesso por caminhos alternativos. Na substituição, a nova versão permanece funcional e a operação não é desfeita. Na exclusão, o estado permanece pendente e não pode ser apresentado como concluído. A pendência deve permitir novas tentativas; somente após confirmação da remoção o registro da versão pode ter `storageKey` limpo e `deletedAt` preenchido.

As permissões de leitura e download são independentes; portanto, um compartilhamento pode conceder download sem leitura. O progresso é individual por usuário e livro, e sua posição deve respeitar as características do formato. A representação concreta da posição ainda precisa ser definida junto à escolha das soluções de leitura.

## Estratégia de testes

- Testes unitários para regras de domínio.
- Testes de integração para persistência, autenticação, upload e permissões.
- Testes ponta a ponta para os principais fluxos do usuário.
