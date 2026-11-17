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
│   ├── reader/              # Leitura e progresso
│   └── sharing/             # Compartilhamento e permissões
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
- **BookFile:** referência ao único arquivo principal do livro no MVP, tipo, tamanho e integridade.
- **Library:** organização dos livros de um usuário.
- **Share:** relação de compartilhamento entre proprietário, livro e destinatário.
- **ReadingProgress:** posição e progresso de leitura de um usuário em um livro.

## Armazenamento de arquivos

O acesso aos arquivos deve ocorrer por meio de uma interface de armazenamento, sem acoplar as regras de negócio ao provedor escolhido.

- Desenvolvimento: armazenamento local não versionado pelo Git.
- Produção: a abstração deve permitir migração futura para armazenamento de objetos. O provedor e a configuração de produção ainda não estão definidos.
- O banco guarda metadados e a referência do arquivo, não o conteúdo binário do livro.
- Cada livro pode ter no máximo um arquivo principal no MVP.

## Segurança e privacidade

- Senhas nunca são armazenadas em texto puro.
- O cadastro e o login do MVP usam e-mail e senha; login pelo Google não faz parte do MVP.
- Rotas e operações privadas exigem autenticação.
- O servidor verifica propriedade ou permissão de compartilhamento antes de visualizar ou baixar um arquivo.
- Arquivos enviados são validados por extensão, tipo de conteúdo e tamanho.
- Arquivos TXT devem usar codificação UTF-8.
- Segredos e credenciais ficam em variáveis de ambiente e não são enviados ao Git.

Livros excluídos logicamente e seus arquivos não podem ser acessados pelo proprietário nem por antigos destinatários. A exclusão lógica revoga os compartilhamentos ativos. A política de remoção física e retenção dos arquivos ainda precisa ser definida.

As permissões de leitura e download são independentes; portanto, um compartilhamento pode conceder download sem leitura. O progresso é individual por usuário e livro, e sua posição deve respeitar as características do formato. A representação concreta da posição ainda precisa ser definida junto à escolha das soluções de leitura.

## Estratégia de testes

- Testes unitários para regras de domínio.
- Testes de integração para persistência, autenticação, upload e permissões.
- Testes ponta a ponta para os principais fluxos do usuário.
