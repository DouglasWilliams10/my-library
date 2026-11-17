# BUSINESS_RULES.md

## 1. Usuários e contas

1. Cada usuário deve possuir uma conta identificável por e-mail único.
2. O cadastro e o login do MVP usam e-mail e senha; login pelo Google não faz parte do MVP.
3. O cadastro exige os campos obrigatórios definidos na especificação da funcionalidade.
4. Senhas nunca podem ser armazenadas em texto puro.
5. Um usuário não autenticado não pode acessar sua biblioteca privada, arquivos ou operações de compartilhamento.
6. Cada usuário pode consultar e editar apenas os dados de seu próprio perfil, respeitando as permissões administrativas que venham a ser definidas.

## 2. Livros e propriedade

1. Todo livro cadastrado deve possuir um proprietário.
2. Um livro pode ser cadastrado manualmente ou associado a um arquivo importado.
3. O cadastro deve possuir, no mínimo, título e autor; os demais metadados são opcionais quando não conhecidos.
4. Cada livro pode ter no máximo um arquivo principal no MVP.
5. O proprietário pode editar os metadados, remover o livro de sua biblioteca e controlar seus compartilhamentos.
6. A remoção de um livro é lógica: o livro deixa de estar disponível na biblioteca e não pode mais ser acessado pelo proprietário nem por antigos destinatários.
7. Ao excluir logicamente um livro, todos os seus compartilhamentos ativos devem ser revogados.
8. Livros sem arquivo digital podem existir como registros de catálogo, mas não podem ser abertos no leitor nem baixados.
9. A política de remoção física e retenção dos arquivos de livros excluídos logicamente ainda precisa ser definida.

## 3. Formatos e arquivos

1. Na primeira versão, são aceitos PDF, EPUB e TXT.
2. A extensão do arquivo não é suficiente para validar o upload: o servidor deve verificar também o tipo de conteúdo.
3. Arquivos acima do limite configurado devem ser recusados com mensagem clara. O limite deve ser configurável, com valor inicial de 50 MB.
4. Arquivos TXT devem usar codificação UTF-8.
5. Arquivos não suportados, inválidos ou corrompidos não devem ser adicionados à biblioteca.
6. O nome original do arquivo deve ser preservado como metadado, mas o nome físico de armazenamento deve ser gerado pela aplicação.
7. Um usuário só pode baixar um arquivo se for seu proprietário ou possuir permissão de compartilhamento que inclua download.
8. A aplicação não deve disponibilizar publicamente arquivos privados por URLs permanentes sem autorização.
9. O armazenamento de desenvolvimento é local e deve ser acessado por uma abstração que permita futura migração para armazenamento de objetos em produção; o provedor de produção ainda não está definido.
10. O sistema não deve remover automaticamente arquivos de outros usuários ou arquivos ainda referenciados por livros válidos.

## 4. Leitura e progresso

1. Somente livros com arquivo válido e formato suportado pelo leitor podem ser abertos.
2. O leitor deve respeitar as permissões de acesso ao livro.
3. O progresso de leitura pertence ao usuário, não ao livro globalmente.
4. O progresso deve ser salvo por usuário e livro, permitindo que usuários diferentes mantenham posições distintas.
5. Se não houver progresso salvo, a leitura começa no início.
6. A posição salva deve respeitar as características do formato; a representação concreta por formato depende da solução de leitura escolhida e ainda precisa ser definida.
7. A aplicação deve informar quando um formato ainda não possui suporte de leitura, sem simular que o livro foi aberto.

## 5. Compartilhamento

1. O compartilhamento é feito pelo proprietário para um usuário cadastrado, identificado por e-mail.
2. O destinatário precisa possuir uma conta ativa para acessar o livro compartilhado.
3. O compartilhamento é privado: livros não aparecem em buscas públicas apenas por terem sido compartilhados.
4. Na primeira versão, o proprietário pode conceder permissão de leitura e, separadamente, permissão de download; download pode ser concedido sem leitura.
5. A permissão de download sem leitura autoriza baixar o arquivo, mas não abrir o livro no leitor. A visibilidade de metadados e do item na interface do destinatário nessa situação ainda precisa ser definida.
6. O destinatário não pode editar os metadados, excluir o livro original ou compartilhar novamente o arquivo.
7. O proprietário pode revogar o compartilhamento a qualquer momento.
8. Após a revogação, o destinatário perde o acesso pela aplicação.
9. A exclusão lógica do livro revoga todos os seus compartilhamentos ativos e impede o acesso dos destinatários.
10. Revogar o acesso não apaga cópias que o destinatário já tenha baixado; a interface deve deixar isso claro antes de conceder download.

## 6. Privacidade e segurança

1. Toda consulta, leitura, alteração ou download de arquivo deve verificar autenticação e autorização no servidor.
2. Não confiar apenas em controles de interface para proteger operações.
3. Mensagens de erro não devem revelar senhas, tokens, caminhos internos ou dados privados de outros usuários.
4. O sistema deve registrar informações mínimas necessárias para diagnosticar falhas de upload e operações relevantes, sem registrar o conteúdo dos livros.
5. A aplicação deve respeitar direitos autorais: o usuário é responsável por possuir autorização para armazenar e compartilhar os arquivos adicionados.

## 7. Resultado esperado

Uma operação só é considerada concluída quando as validações forem aprovadas, os dados persistidos e as permissões respeitadas. Em caso de falha, o sistema deve apresentar mensagem compreensível e não deixar registros incompletos.

## 8. Decisões ainda pendentes

- Campos obrigatórios adicionais no cadastro, além de e-mail e senha.
- Se será permitido substituir o arquivo principal de um livro e, em caso afirmativo, o efeito sobre o progresso de leitura existente.
- Política de remoção física e retenção de arquivos após exclusão lógica.
- Retenção ou remoção do progresso de leitura após exclusão lógica de um livro.
- Visibilidade de metadados e do item compartilhado quando o destinatário possui download, mas não leitura.
- Representação da posição de leitura específica para cada formato e solução de leitura.
- Unidade exata usada para configurar e aplicar o limite de 50 MB.
