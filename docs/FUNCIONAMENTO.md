- Multi-tenant SaaS: cada conta pode ser 1 profissional autônoma hoje, ou 1 empresa com funcionários no futuro
- Comissão: percentual único por conta (ex: 50%), fixo mas editável em configurações — não varia por serviço
- Login obrigatório desde a v1
- Backend Django está limpo (sem apps, sem DRF, sem requirements.txt ainda) — projeto começando do zero

Arquitetura proposta

Separação entre tenant e usuário: "Conta" é quem é dona dos dados (hoje = 1 profissional, no futuro = 1 empresa). "User" é quem loga, e tem uma FK pra Conta. Hoje só existe 1 User por Conta; no futuro uma empresa pode ter vários Users (funcionários) apontando pra mesma Conta, todos compartilhando a mesma base de clientes/serviços.

Isso precisa ser decidido agora: trocar AUTH_USER_MODEL depois do primeiro migrate é muito custoso. Como o projeto ainda não tem nenhuma migração, é o momento certo para usar um User customizado.

Modelos (app Django, ex: comissao/)
Conta            → nome, comissao_percentual_padrao, criado_em
User (custom)    → conta(FK), cargo[dono/funcionario], campos padrão do AbstractUser
Cliente          → conta(FK), nome, telefone, observacoes
Servico          → conta(FK), nome, valor_padrao, ativo
ServicoRealizado → conta(FK), profissional(FK User, nullable), cliente(FK), servico(FK),
                    valor_cobrado, comissao_percentual, valor_comissao, data_realizacao
Agendamento      → conta(FK), profissional(FK User, nullable), cliente(FK), servico(FK, opcional),
                    data_hora_inicio, data_hora_fim,
                    status[agendado/atendido/cancelado], servico_realizado(FK, opcional)

Notas de design:
- comissao_percentual em ServicoRealizado é uma cópia do valor da Conta no momento do atendimento (snapshot), editável por atendimento — isso cobre "comissão personalizada" (uma ocorrência com comissão maior que o padrão), sem precisar de campo extra.
- valor_cobrado em ServicoRealizado é sempre editável por atendimento (permite valor personalizado por cliente/serviço).
- profissional em ServicoRealizado/Agendamento já existe desde já para não exigir migração quando houver múltiplos funcionários — hoje sempre aponta pro único User da conta.
- Quando marcar um agendamento como "atendido", ele já gera o ServicoRealizado correspondente (evita retrabalho de digitação).
- Agendamento usa data_hora_inicio + data_hora_fim (não um ponto único) porque a profissional decide livremente até que horas vai o atendimento. Validação no backend: dois agendamentos ativos (não cancelados) da mesma conta não podem ter intervalos sobrepostos (inicio_A < fim_B E inicio_B < fim_A → conflito, rejeitado). Cancelados não bloqueiam. Quando houver múltiplas profissionais por conta, o filtro de sobreposição passa a considerar também o profissional, sem mudar a estrutura.

Backend — peças que faltam instalar
- djangorestframework (API para o React consumir)
- djangorestframework-simplejwt (login por JWT, já que o frontend é um app separado)
- django-cors-headers (Vite em :5173 falando com Django em :8000)

Frontend — rotas (com react-router-dom)
/login
/                    → dashboard resumido
/clientes            → CRUD
/servicos             → CRUD
/servicos-realizados  → registrar atendimento + listagem
/relatorios           → filtro por período, total faturado, total comissão
/agenda               → agendamentos + check de atendido
