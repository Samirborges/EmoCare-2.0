# EmoCare 2.0

Sistema web para gerenciamento e agendamento de consultas de terapia emocional, desenvolvido como projeto-vitrine para praticar engenharia de software moderna com **Spring Boot, Angular e PostgreSQL**.

O EmoCare 2.0 moderniza uma solução acadêmica anterior e busca demonstrar decisões de arquitetura explícitas, segurança, testes automatizados, entrega contínua, observabilidade e documentação técnica. O projeto é educacional e **não deve ser considerado pronto para uso clínico ou produção**.

> **Dados e privacidade:** durante desenvolvimento, demonstrações e testes devem ser usados exclusivamente dados fictícios. Não inserir dados reais de pacientes, informações reais de saúde ou credenciais pessoais.

## Objetivos

- Separar responsabilidades de negócio em serviços bem delimitados.
- Aplicar arquitetura hexagonal onde ela trouxer clareza e testabilidade.
- Documentar a API com OpenAPI.
- Gerenciar a evolução do banco por migrações Flyway.
- Desenvolver testes unitários, de integração e de carga.
- Praticar Docker, Kubernetes local com `kind` e GitHub Actions.
- Explorar mensageria assíncrona com Kafka, padrão Outbox e consumidores idempotentes.
- Implementar observabilidade com Spring Boot Actuator, Prometheus, Grafana e logs com identificador de rastreamento.
- Documentar decisões técnicas por meio de ADRs (Architecture Decision Records).

## Stack planejada

| Área | Tecnologia | Responsabilidade |
|---|---|---|
| Backend | Java + Spring Boot | Regras de negócio e API |
| Frontend | Angular | Interface web |
| Banco relacional | PostgreSQL | Persistência dos dados |
| Migrações | Flyway | Evolução versionada do esquema |
| Contrato da API | OpenAPI | Documentação e contrato HTTP |
| Autenticação | Spring Security | Autenticação e autorização |
| Hash de senha | BCrypt | Armazenamento não reversível de senhas |
| Login social | Google OAuth 2.0 / OpenID Connect | Autenticação social de pacientes |
| Mensageria | Apache Kafka | Eventos assíncronos entre serviços |
| Containerização | Docker | Imagens reproduzíveis |
| Orquestração local | Kubernetes + kind | Exercício de implantação |
| CI/CD | GitHub Actions | Testes, build e publicação de imagem |
| Observabilidade | Actuator, Prometheus e Grafana | Saúde, métricas e visualização |
| Teste de carga | k6 | Medição de desempenho antes/depois de otimização |
| Nuvem | AWS (escopo mínimo a definir) | Hospedagem de ao menos um componente |

A lista representa a stack planejada. Tecnologias, versões, topologia dos serviços e detalhes de implantação serão definidos e registrados conforme o projeto evoluir.

## Escopo funcional inicial

### Perfis

O sistema terá três perfis de acesso:

- **Paciente (`PATIENT`)**: cria a própria conta e acessa as funcionalidades destinadas a pacientes.
- **Terapeuta (`THERAPIST`)**: tem conta criada pelo administrador e acessa as funcionalidades profissionais.
- **Administrador (`ADMIN`)**: administra contas de terapeutas.

O frontend poderá direcionar cada perfil para um dashboard diferente, mas **redirecionamento de tela não é controle de segurança**. O backend deverá validar autenticação e autorização em cada operação protegida.

### Cadastro de paciente

O cadastro público solicitará:

- Nome completo.
- E-mail.
- Senha.
- Confirmação de senha.

Regras definidas:

1. Todos os campos são obrigatórios.
2. A confirmação da senha será validada no Angular e também no backend, conforme a implementação escolhida.
3. A confirmação não será armazenada no banco de dados.
4. O nome será tratado removendo espaços desnecessários no início e no fim; aceitará acentos, hífens e nomes compostos.
5. O nome terá limite máximo de caracteres, cujo valor será definido na implementação.
6. A senha terá comprimento mínimo de 8 caracteres. Regras adicionais e limites técnicos serão documentados antes da implementação final.
7. O e-mail será normalizado para minúsculas (`lowercase`) e espaços externos serão removidos antes da validação.
8. O e-mail será único, sem distinção entre maiúsculas e minúsculas. A aplicação e o PostgreSQL deverão proteger essa regra, inclusive contra cadastros concorrentes.
9. Após cadastro bem-sucedido, o paciente será direcionado à tela de login.
10. A senha nunca será armazenada em texto puro: será armazenado um hash produzido por BCrypt.

### Login e persistência da sessão

O login tradicional usará e-mail e senha. O mecanismo de autenticação e a estratégia de persistência de sessão serão implementados com Spring Security.

**OAuth 2.0 não é um algoritmo para criptografar senhas.** OAuth 2.0 é um framework de autorização; para login com Google, normalmente será usado OpenID Connect sobre OAuth 2.0. BCrypt será utilizado para gerar hashes das senhas locais.

O usuário deseja continuar autenticado após fechar e reabrir o navegador. A solução concreta — por exemplo, sessão persistente com cookie seguro ou access token de curta duração com mecanismo de renovação — ainda precisa ser escolhida e registrada em ADR. A implementação deverá considerar armazenamento seguro, expiração, revogação, proteção contra CSRF quando aplicável e prevenção de exposição de credenciais. Não se deve persistir tokens sensíveis em `localStorage` sem uma análise explícita dos riscos.

### Login social e associação de contas

- O provedor social inicial será o Google.
- O login social estará disponível apenas para pacientes.
- Se uma conta local já existir com o mesmo e-mail, o sistema não associará automaticamente a identidade Google apenas pela coincidência do endereço. Será exigida uma etapa adicional de verificação e vinculação segura.
- O fluxo deverá tratar cancelamento, falha de autenticação e contas ainda não vinculadas.

### Recuperação de senha

A redefinição segura de senha deverá:

- Permitir que o titular solicite a redefinição.
- Evitar revelar desnecessariamente se um e-mail está cadastrado.
- Usar token ou código de recuperação aleatório, temporário e de uso único.
- Validar a nova senha pelas regras vigentes.
- Impedir reutilização de tokens expirados ou consumidos.
- Aplicar uma política de invalidação de sessões após a redefinição.

### Administração de terapeutas

A interface administrativa deverá permitir:

- Listar terapeutas cadastrados.
- Criar contas de terapeutas.
- Desativar terapeutas.

A criação solicitará nome completo, e-mail e senha inicial. Neste escopo, o administrador fornecerá as credenciais ao terapeuta. Essa decisão é mantida para fins educacionais, embora um fluxo de convite no qual o terapeuta defina a própria senha seja preferível em um sistema real, pois evita que o administrador conheça a senha definitiva.

Terapeutas criados pelo administrador estarão autorizados a atender imediatamente, sem uma etapa adicional de aprovação profissional dentro do escopo atual.

A criação de administradores não terá tela pública no frontend. A conta inicial será provisionada por procedimento controlado, como uma operação administrativa de bootstrap ou inserção direta no banco. **A senha nunca deverá ser inserida como texto puro no banco.** Deve-se gerar o hash BCrypt usando ferramenta ou código de bootstrap seguro, sem registrar a senha ou o hash em logs. A criação de administradores adicionais deverá ser restrita a administradores já autenticados ou a um procedimento operacional controlado. Qualquer endpoint administrativo deverá exigir autorização explícita `ADMIN`; a documentação OpenAPI não substitui autenticação e autorização.

### Desativação de terapeuta e consultas

A remoção administrativa de um terapeuta será uma **desativação lógica**, e não uma exclusão física imediata dos registros.

Regras definidas:

1. O terapeuta desativado perde o acesso imediatamente.
2. Consultas futuras/agendadas associadas ao terapeuta serão canceladas automaticamente.
3. Pacientes afetados serão notificados sobre os cancelamentos.
4. Registros históricos de consultas e atendimentos serão preservados conforme a política de retenção definida para o projeto.
5. A operação deverá manter a integridade referencial e ser segura diante de tentativas repetidas.

Ainda será necessário definir precisamente quais estados de consulta contam como “agendada/futura”, como tratar consultas em andamento e como coordenar a desativação, os cancelamentos e as notificações caso um dos serviços esteja indisponível.

## Regras transversais de segurança

- A autorização será aplicada no backend para cada operação protegida.
- Os perfis iniciais serão `PATIENT`, `THERAPIST` e `ADMIN`.
- O cadastro público não poderá escolher nem alterar seu próprio perfil.
- Senhas locais serão armazenadas como hashes BCrypt, nunca em texto puro.
- A normalização do e-mail será consistente entre aplicação e banco.
- Endpoints de cadastro, login e recuperação terão proteção contra abuso automatizado, com controles como limitação de taxa, limites de requisições e respostas seguras. Os limites exatos serão configurados e testados.
- Segredos, senhas, tokens e dados pessoais não deverão aparecer em logs.
- Dados fictícios serão utilizados em desenvolvimento, testes, demonstrações e testes de carga.
- A documentação deverá conter uma seção sobre LGPD, minimização de dados, controle de acesso, retenção e tratamento de incidentes. O projeto não alegará conformidade legal apenas por conter essa documentação.

## Arquitetura e integração entre serviços

O desenho inicial prevê separar pelo menos:

- **Serviço de agendamento**: regras e ciclo de vida das consultas.
- **Serviço de notificações**: entrega de notificações relacionadas a eventos de negócio.

Eventos previstos incluem, entre outros, consulta marcada e consulta cancelada. A desativação de um terapeuta que provoque cancelamentos deverá respeitar essas mesmas regras.

A comunicação assíncrona utilizará Kafka. Para reduzir o risco de atualizar o banco sem publicar o evento correspondente, será avaliado e implementado o padrão **Transactional Outbox**: a alteração de negócio e o registro do evento serão persistidos na mesma transação local, e um processo separado publicará o evento no Kafka.

Os consumidores deverão ser idempotentes: receber o mesmo evento mais de uma vez não poderá causar efeitos duplicados, como enviar repetidamente a mesma notificação sem uma política explícita. A entrega de mensagens e notificações pode ser repetida; não se deve presumir entrega exatamente uma vez de ponta a ponta.

A arquitetura hexagonal será aplicada para separar domínio e casos de uso de detalhes de infraestrutura, como banco, mensageria e transporte HTTP. A divisão definitiva dos serviços, seus limites de dados e a estratégia de autenticação entre serviços serão registrados em ADRs. A adoção de microsserviços aumenta o custo operacional; a separação deve ser mantida porque atende ao objetivo didático do projeto e será documentada como tal.

## Banco de dados e migrações

- PostgreSQL será o banco relacional planejado.
- Flyway controlará as migrações versionadas.
- Alterações no esquema serão feitas por novas migrações, evitando mudanças manuais não rastreadas nos ambientes compartilhados.
- Restrições importantes, como unicidade de e-mail, deverão ser protegidas também no banco.
- Relações e políticas de exclusão deverão preservar a integridade de consultas e registros históricos.
- Dados de teste deverão ser fictícios e identificáveis como tal.

## Testes e qualidade

A estratégia de testes incluirá:

- **Testes unitários** para regras de domínio e casos de uso.
- **Testes de integração** para API, persistência, migrações e integrações relevantes.
- **Testes de autorização** para comprovar que cada perfil só executa operações permitidas.
- **Testes de idempotência** para consumidores de eventos.
- **Testes de fluxo** para cadastro, login, recuperação de senha, criação/desativação de terapeuta e cancelamento/notificação.
- **Testes de carga com k6**, com resultados documentados antes e depois de uma otimização justificada, como índice de banco ou cache.

Os testes deverão ser executáveis de forma reproduzível e integrados ao pipeline de CI.

## DevOps, implantação e observabilidade

### Docker, Kubernetes e CI/CD

- Criar Dockerfiles multi-stage para os componentes aplicáveis.
- Criar manifests Kubernetes adequados para execução local com `kind`.
- Configurar GitHub Actions para executar testes, construir artefatos/imagens e publicar imagens conforme a estratégia de branches e releases.
- Não armazenar credenciais de nuvem ou segredos nos arquivos versionados.
- Se não houver Docker disponível localmente, utilizar GitHub Codespaces como ambiente de desenvolvimento alternativo.

### Nuvem

Ao menos um componente será publicado na AWS. A alternativa inicial a avaliar é hospedar o frontend Angular em S3 com distribuição via CloudFront. A arquitetura final de hospedagem, DNS, TLS e permissões será documentada antes da implantação.

Um alerta de orçamento da AWS deverá ser configurado. Alertas de orçamento não garantem bloqueio automático de gastos; os custos e recursos provisionados precisarão ser acompanhados.

### Observabilidade

A solução deverá avaliar e documentar:

- Spring Boot Actuator para saúde e métricas da aplicação.
- Prometheus para coleta de métricas.
- Grafana para visualização de métricas.
- Logs estruturados com identificador de correlação/trace.
- Propagação de contexto de rastreamento entre requisições e eventos assíncronos, conforme a instrumentação escolhida.
- Ausência de senhas, tokens e conteúdo sensível de atendimentos nos logs.

## LGPD e dados fictícios

O sistema tem um domínio relacionado a atendimentos terapêuticos, que pode envolver dados pessoais sensíveis. Embora seja um projeto-vitrine, a documentação abordará minimização de dados, finalidade, controle de acesso, retenção, segurança, exclusão e resposta a incidentes.

Não serão usados dados reais de pacientes ou registros reais de atendimento. A existência de controles técnicos ou de uma seção sobre LGPD não significa, por si só, que o projeto esteja legalmente adequado para produção.

## Métricas de desempenho

Um cenário de carga será criado com k6. A medição deverá registrar condições reproduzíveis, incluindo ambiente, volume de requisições, concorrência, duração e dados utilizados.

A comparação antes/depois deverá informar, pelo menos:

- Latência (p50, p95 e, quando útil, p99).
- Taxa de requisições concluídas e de erros.
- Throughput observado.
- Mudança aplicada, como índice ou cache.
- Limitações do experimento e condições que possam afetar a comparação.

Nenhuma melhoria de desempenho será declarada antes de medir os resultados.

## Documentação e decisões arquiteturais

O repositório deverá manter:

- Este `README.md`, atualizado conforme o projeto evoluir.
- Diagrama de arquitetura e fluxo principal.
- Contrato OpenAPI.
- Instruções para configuração, execução local e testes.
- Exemplos de configuração sem segredos.
- ADRs para decisões significativas.

Sugestão de estrutura para ADRs:

```text
docs/
  architecture/
    diagrams/
    adr/
      0001-stack-principal.md
      0002-arquitetura-hexagonal.md
      0003-limites-dos-servicos.md
      0004-autenticacao-e-sessao.md
      0005-outbox-e-consumidores-idempotentes.md
```

Cada ADR deverá conter contexto, decisão, alternativas consideradas, consequências e estado (proposto, aceito ou substituído). Os nomes acima são sugestões; os ADRs ainda precisam ser escritos e aprovados.

## Questões em aberto

As decisões abaixo ainda precisam ser fechadas antes ou durante o desenho técnico:

- Limite máximo de caracteres do nome e da senha.
- Estratégia exata de sessão persistente após fechar o navegador.
- Política de expiração, renovação e revogação de sessão.
- Provedor de e-mail e mecanismo de envio de recuperação de senha e notificações.
- Estados de consulta e regra exata para determinar quais consultas serão canceladas ao desativar um terapeuta.
- Comportamento para consultas em andamento e falhas parciais durante cancelamentos/notificações.
- Política de vinculação segura entre conta local e Google.
- Estratégia de provisionamento seguro da primeira conta administrativa.
- Limites e mecanismo de proteção contra cadastro/login automatizado.
- Limites definitivos dos serviços e modelo de autenticação entre eles.
- Ambiente AWS final, custo estimado e alertas complementares.
- Valores de referência e cenário de carga do k6.

Decisões relevantes serão registradas em ADRs, evitando que suposições temporárias sejam confundidas com requisitos aprovados.

## Princípios do projeto

1. **Segurança no backend:** a interface não é uma fronteira de autorização.
2. **Integridade dos dados:** regras críticas devem ser protegidas pela aplicação e pelo banco quando apropriado.
3. **Falhas explícitas:** eventos, notificações e operações distribuídas precisam ter política de repetição e recuperação.
4. **Evidência em vez de alegações:** qualidade e desempenho devem ser demonstrados por testes e métricas.
5. **Aprendizado documentado:** decisões, limitações e trade-offs devem ser explicados, não apenas implementados.
6. **Escopo educacional transparente:** o projeto é uma vitrine de engenharia e não está autorizado para uso com dados clínicos reais.

## Status

**Fase atual:** levantamento e refinamento de requisitos.

Nenhuma funcionalidade deve ser considerada implementada apenas por estar descrita neste documento. O README será atualizado à medida que requisitos forem confirmados, decisões arquiteturais forem registradas e componentes forem efetivamente construídos.
