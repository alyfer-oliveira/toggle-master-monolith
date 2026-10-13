# FIAP — Pós-Tech DevOps (DCLT)
## Tech Challenge — Fase 1

**Grupo:** ToggleMaster  
**Data da entrega:** [preencher]

### Integrantes

| Integrante | RM |
|---|---:|
| Alyfer Ricardo Sousa de Oliveira | 378829 |
| Diogo da Silva Frota Soares | 378178 |
| Flávio Cardoso de Oliveira Soares | 377532 |
| João Guilherme Schneider da Silva | 377419 |
| Luciano Luan da Silva Alves | 378735 |

### Links do projeto

- **Repositório Base:** https://github.com/dougls/toggle-master-monolith
- **Diagrama da arquitetura:** [inserir link]
- **Vídeo de demonstração (até 15 minutos):** [inserir link]
- **Aplicação implantada (IP ou DNS):** [inserir endereço, se aplicável]
- **Estimativa da AWS:** https://calculator.aws/#/estimate?id=7d0c874d49c46e28e7a05c8d1e658c78747d185c

---

## 1. Resumo executivo

O Tech Challenge da Fase 1 teve como objetivo disponibilizar na AWS o **ToggleMaster**, um MVP monolítico para gerenciamento de *feature flags*. A API permite criar, consultar, atualizar e excluir flags, possibilitando controlar a liberação de funcionalidades e desativá-las rapidamente em caso de problemas, sem exigir um novo deploy da aplicação.

O grupo analisou e executou a aplicação localmente e preparou sua implantação em uma arquitetura AWS com uma instância **Amazon EC2** para executar a aplicação e um banco **Amazon RDS** para persistir os dados. A solução utiliza uma **VPC** com sub-redes públicas e privadas, regras de acesso definidas por *Security Groups* e o **AWS Secrets Manager** para disponibilizar credenciais. A entrega documenta a arquitetura, a aplicação dos 12 fatores, os testes realizados e a estimativa de custos. Os resultados específicos da validação e os links finais devem ser preenchidos nas seções correspondentes.

## 2. Análise da aplicação e da arquitetura monolítica

O ToggleMaster é considerado monolítico porque suas funcionalidades de API são desenvolvidas e executadas como uma única aplicação. A própria aplicação recebe as requisições, processa as operações relacionadas às *feature flags* e acessa o banco de dados. Seus componentes são implantados em conjunto, em vez de serem distribuídos como serviços independentes.

### Vantagens para um MVP

- Estrutura mais simples de compreender e desenvolver;
- Implantação e testes iniciais mais diretos;
- Menor custo operacional no começo do projeto;
- Comunicação direta entre os componentes da aplicação.

### Desvantagens e pontos de atenção

- Uma alteração pode exigir a implantação da aplicação inteira;
- Reinicializações durante o deploy podem causar indisponibilidade se não houver estratégia de atualização contínua;
- A aplicação é escalada como um todo, mesmo que apenas uma funcionalidade exija mais recursos;
- Com o crescimento do código, a manutenção pode se tornar mais difícil se os componentes não estiverem bem organizados.

## 3. Análise dos 12 fatores (12-Factor App)

A avaliação abaixo considera a solução implementada pelo grupo durante o desafio. “Atende parcialmente” indica que existe uma implementação inicial, mas há melhorias necessárias para maior segurança, repetibilidade ou confiabilidade em produção. As evidências devem corresponder à configuração efetivamente usada e podem ser incluídas como capturas de tela, trechos de configuração, comandos ou referências ao repositório.

| Nº | Fator | Avaliação | Análise, evidência e evolução recomendada |
|---:|---|---|---|
| 1 | **Base de código** | **Atende** | O projeto está versionado em GitHub: https://github.com/dougls/toggle-master-monolith. O grupo utilizou a branch `main`, cuja versão de referência corresponde à Pull Request nº 2, mergeada pelo usuário `dougls`, conforme descrito no modelo do projeto. Para comprovar a versão implantada, registrar também o commit utilizado e demonstrar que eventuais alterações feitas pelo grupo foram versionadas. **Evidência:** [inserir captura/link do commit ou histórico de implantação]. |
| 2 | **Dependências** | **Atende** | As dependências Python são declaradas em `requirements.txt` e instaladas pelo `Dockerfile` com `pip install --no-cache-dir -r requirements.txt`. **Evidência:** `requirements.txt` e trecho correspondente do `Dockerfile`. Para maior reprodutibilidade, manter versões fixadas e atualizar dependências de forma controlada. |
| 3 | **Configurações** | **Atende parcialmente** | No ambiente AWS, as credenciais são obtidas do AWS Secrets Manager e fornecidas ao ambiente de execução por um script do grupo. O procedimento evita inserir segredos no código e reduz sua exposição em comandos do terminal. Segundo a implementação descrita, as variáveis temporárias do host são removidas após o deploy. **Evidência:** [anexar trecho do script sem valores secretos e captura das configurações, com segredos ocultos]. Para produção, preferir injetar segredos diretamente no processo ou serviço de execução, limitar permissões por IAM e evitar persistir credenciais em arquivos ou histórico de shell. |
| 4 | **Serviços de apoio** | **Atende** | O Amazon RDS é utilizado como banco de dados externo da aplicação, e o AWS Secrets Manager armazena/disponibiliza as credenciais necessárias. Esses recursos são configurados externamente à aplicação. **Evidência:** configuração da conexão ao endpoint do RDS e referência ao segredo, sempre ocultando usuário, senha e valores sensíveis. A VPC e o Elastic IP são componentes de infraestrutura/rede, não serviços de apoio consumidos pela lógica da aplicação. |
| 5 | **Build, release e execução** | **Atende parcialmente** | A imagem da aplicação é construída a partir do código e das dependências definidos no projeto e executada em um container na EC2. O processo de obtenção das configurações e subida do container é feito por script. **Evidência:** `Dockerfile`, comandos/script utilizados e identificação da imagem implantada. Para produção, separar claramente build, release e execução, identificar cada release por tag/commit e manter um procedimento de rollback. |
| 6 | **Processos** | **Atende parcialmente** | A API é executada como processo/container e usa o RDS para persistência dos dados. Isso evita que os dados principais dependam do armazenamento efêmero do processo da aplicação. **Evidência:** configuração de execução do container e demonstração de persistência no RDS. Confirmar que sessões, arquivos enviados ou outros estados não essenciais não ficam exclusivamente na memória ou no disco local da EC2. |
| 7 | **Vínculo de porta** | **Atende parcialmente** | A aplicação disponibiliza a API por uma porta e o acesso externo é controlado pela configuração de rede da EC2. **Evidência:** porta configurada no container/aplicação e regra correspondente no Security Group. Registrar a porta efetivamente usada e, para produção, considerar TLS por meio de proxy ou balanceador e não expor portas internas desnecessárias. |
| 8 | **Concorrência** | **Atende parcialmente** | A execução em container permite iniciar processos da aplicação, mas a implantação da Fase 1 utiliza uma única instância EC2 e não demonstra escalabilidade horizontal. **Evidência:** configuração atual da instância e do processo. Para produção, avaliar múltiplas instâncias/containers, balanceamento de carga, limites de conexão do RDS e escalabilidade conforme a demanda. |
| 9 | **Descartabilidade** | **Atende parcialmente** | O container pode ser reiniciado sem que os dados persistidos no RDS sejam perdidos. A recuperação após falhas e o comportamento de encerramento devem ser validados. **Evidência:** [inserir resultado de reinício do container e verificação dos dados]. Para produção, configurar política de reinício, verificações de saúde, tratamento adequado de sinais de encerramento e tempo de inicialização previsível. |
| 10 | **Dev/prod semelhantes** | **Atende parcialmente** | A aplicação foi executada localmente e implantada na AWS, mas os ambientes diferem em infraestrutura e serviços gerenciados. **Evidência:** registrar como cada ambiente configura variáveis, dependências e banco de dados. Para reduzir diferenças, padronizar versões, automatizar a configuração e usar serviços equivalentes nos testes, sem compartilhar credenciais ou dados de produção. |
| 11 | **Logs** | **Atende parcialmente** | A aplicação pode emitir logs durante sua execução no container. O local de coleta e retenção dos logs da implantação deve ser registrado. **Evidência:** [inserir captura de logs do container/EC2 ou configuração do CloudWatch, se utilizada]. Para produção, direcionar logs para uma saída centralizada, definir retenção e garantir que credenciais e dados sensíveis não sejam registrados. |
| 12 | **Processos administrativos** | **Precisa de evolução** | O projeto não documenta, no material disponível, processos administrativos pontuais separados da execução da API, como migrações ou rotinas operacionais. **Evidência:** [descrever se alguma tarefa administrativa foi executada e como]. Para produção, documentar e automatizar essas tarefas para que sejam executadas sob demanda, com a mesma versão e configurações da aplicação, sem depender de alterações manuais não rastreáveis na EC2. |

> **Nota:** Ajustem as avaliações se a implantação final do grupo diferir do descrito neste documento. Não incluam credenciais, chaves privadas, valores de segredos ou capturas que os revelem.

## 4. Arquitetura e tecnologias

### 4.1 Arquitetura proposta/implantada

A solução utiliza uma VPC com sub-redes públicas e privadas. A instância EC2 executa a aplicação e recebe o tráfego permitido pelas regras do seu Security Group. O Amazon RDS fica em sub-rede privada e aceita conexões na porta do banco somente do Security Group da EC2. O AWS Secrets Manager armazena as credenciais consumidas no processo de implantação/execução.

**Fluxo principal:**

1. O usuário envia uma requisição HTTP/HTTPS para o endereço público da aplicação;
2. A instância EC2 recebe a requisição conforme as regras do Security Group;
3. A API processa a operação solicitada sobre uma *feature flag*;
4. A aplicação conecta-se ao RDS por rede privada para consultar ou persistir os dados;
5. As credenciais são obtidas do AWS Secrets Manager conforme o procedimento configurado pelo grupo.

**Diagrama da arquitetura:** [inserir link para Miro, diagrams.net ou outro recurso]  
**Imagem do diagrama:** [inserir ou anexar figura]

### 4.2 Tecnologias e componentes

| Tecnologia/componente | Uso no projeto | Justificativa |
|---|---|---|
| ToggleMaster (Python) | API de gerenciamento de *feature flags* | Aplicação fornecida para o desafio e avaliada pelo grupo. |
| Docker | Empacotamento e execução da aplicação | Mantém aplicação e dependências em uma unidade implantável. |
| Amazon EC2 | Execução da aplicação na AWS | Hospeda o container da API. |
| Amazon RDS ([PostgreSQL/MySQL — confirmar]) | Persistência dos dados | Banco gerenciado acessado pela aplicação. |
| AWS Secrets Manager | Gestão das credenciais | Mantém as credenciais fora do código-fonte. |
| Amazon VPC, sub-redes e Security Groups | Segmentação e controle de tráfego | Define a comunicação entre EC2, RDS e usuários. |
| Elastic IP ([confirmar se utilizado]) | Endereço IPv4 público estável para a EC2, se aplicável | Facilita o acesso por endereço fixo. |

### 4.3 Segurança, privacidade e dados

- As credenciais do banco são armazenadas no AWS Secrets Manager e não devem ser incluídas no repositório ou em evidências sem mascaramento.
- O RDS deve permanecer em sub-redes privadas, sem acesso público, e aceitar conexões somente do Security Group da EC2.
- O Security Group da EC2 deve liberar HTTP/HTTPS conforme a necessidade da aplicação e restringir SSH ao endereço IP administrativo autorizado.
- A comunicação com o banco deve usar a porta e o protocolo esperados; quando configurado, validar o uso de TLS.
- A solução é um MVP acadêmico. Autenticação/autorização da API, TLS público, cópias de segurança, monitoramento, gestão de patches e alta disponibilidade devem ser avaliados conforme os requisitos de produção.
- **Dados tratados pela aplicação e classificação:** [descrever; não afirmar tratamento de dados pessoais sem verificar].

## 5. Desenvolvimento e implementação

### 5.1 Etapas executadas

1. Acesso ao repositório e análise da estrutura da aplicação;
2. Execução local para compreender o funcionamento da API;
3. Análise da arquitetura monolítica e dos 12 fatores;
4. Desenho da arquitetura AWS com VPC, sub-redes, EC2, RDS e Security Groups;
5. Provisionamento manual dos recursos pelo console da AWS;
6. Conexão à EC2 por SSH e instalação/configuração dos componentes necessários;
7. Configuração das credenciais e da conexão da aplicação ao RDS;
8. Inicialização da aplicação e validação de acesso público e persistência.

**Etapas efetivamente concluídas e diferenças em relação à lista:** [preencher com o que o grupo realmente realizou].

### 5.2 Estrutura do projeto

O código-fonte e seus componentes estão disponíveis no repositório: https://github.com/dougls/toggle-master-monolith.

- `requirements.txt`: dependências Python;
- `Dockerfile`: instruções para construir a imagem e instalar dependências;
- `[outros arquivos/diretórios principais]`: [descrever após conferir a estrutura usada].

### 5.3 Como executar e avaliar

> Os comandos abaixo são um roteiro. Ajustem nomes de arquivos, portas e variáveis aos valores reais do projeto. Não coloquem credenciais reais neste documento.

**Pré-requisitos:** Git, Docker e acesso ao ambiente AWS; Python/pip caso a execução local não seja feita via container.

**Execução local (preencher com o procedimento validado pelo grupo):**

```bash
# Exemplo: obter o projeto
 git clone https://github.com/dougls/toggle-master-monolith.git
 cd toggle-master-monolith

# Instalar dependências, se aplicável
 pip install -r requirements.txt

# Iniciar conforme o comando definido pela aplicação
 [COMANDO_DE_EXECUCAO]
```

**Execução na AWS:** [documentar os comandos ou script utilizados, removendo valores de credenciais e dados sensíveis].

**Endpoint público e porta:** [preencher]  
**Exemplos de requisições para CRUD:** [inserir endpoints/métodos e exemplos sem dados sensíveis]

## 6. Validação e resultados

A validação deve demonstrar que a API responde publicamente, consegue se conectar ao RDS e mantém os dados após a operação. Preencham os resultados com evidências reais da execução.

| Cenário | Resultado esperado | Resultado obtido/evidência | Status |
|---|---|---|---|
| Aplicação executada localmente | API inicia e responde às requisições | [descrever e anexar evidência] | [Aprovado/Pendente] |
| Acesso à aplicação na EC2 | Endpoint público responde pela porta configurada | [inserir URL/IP, horário e evidência] | [Aprovado/Pendente] |
| Criação de *feature flag* | Registro é criado pela API e persistido no RDS | [descrever e anexar evidência] | [Aprovado/Pendente] |
| Consulta de *feature flag* | API retorna o registro existente | [descrever e anexar evidência] | [Aprovado/Pendente] |
| Atualização de *feature flag* | Alteração é persistida e retornada | [descrever e anexar evidência] | [Aprovado/Pendente] |
| Exclusão de *feature flag* | Registro é removido conforme comportamento da API | [descrever e anexar evidência] | [Aprovado/Pendente] |
| Isolamento do RDS | Banco aceita conexão da EC2 e não fica aberto ao público | [inserir regra do Security Group e evidência] | [Aprovado/Pendente] |
| Gestão das credenciais | Credenciais vêm do Secrets Manager sem serem exibidas | [inserir evidência mascarada] | [Aprovado/Pendente] |

### 6.1 Estimativa de custos AWS

A estimativa deve ser produzida na Calculadora de Preços da AWS e corresponder à região, às dimensões e ao período considerados. Incluam também o print solicitado no enunciado.

- **Região AWS:** [preencher]
- **Premissas de uso:** [horas de execução, armazenamento, tráfego e demais premissas]
- **Custo mensal estimado:** [preencher moeda e valor]
- **Link da estimativa ou arquivo/captura:** [inserir]
- **Observações sobre itens não incluídos:** [por exemplo, tráfego variável, impostos ou serviços adicionais, se aplicável]

### 6.2 Limitações conhecidas

- A solução desta fase é um MVP monolítico executado em uma instância EC2;
- A alta disponibilidade e a escalabilidade automática não estão demonstradas, salvo se configuradas pelo grupo;
- [Registrar limitações de autenticação, observabilidade, backup, TLS, automação ou testes que realmente se aplicarem];
- [Registrar cenários não validados].

## 7. Gestão do trabalho em equipe

O grupo é composto por cinco integrantes. A equipe utilizou o repositório GitHub para compartilhar o código do projeto. Registrem também as ferramentas de comunicação e acompanhamento usadas durante a atividade.

- **Repositório e controle de versão:** GitHub — https://github.com/dougls/toggle-master-monolith
- **Comunicação:** [preencher]
- **Acompanhamento das tarefas:** [preencher]

### 7.1 Contribuições

| Integrante | Contribuições realizadas |
|---|---|
| Alyfer Ricardo Sousa de Oliveira | [preencher responsabilidades e entregas] |
| Diogo da Silva Frota Soares | [preencher responsabilidades e entregas] |
| Flávio Cardoso de Oliveira Soares | [preencher responsabilidades e entregas] |
| João Guilherme Schneider da Silva | [preencher responsabilidades e entregas] |
| Luciano Luan da Silva Alves | [preencher responsabilidades e entregas] |

### 7.2 Desafios e aprendizados

- **Desafio encontrado:** [descrever problema real]
- **Decisão ou solução adotada:** [descrever o que o grupo fez]
- **Aprendizado:** [descrever o resultado ou conhecimento adquirido]

## 8. Conclusão e próximos passos

A Fase 1 propõe validar o ToggleMaster como um MVP de gerenciamento de *feature flags* e exercitar a implantação de uma aplicação monolítica na AWS. A arquitetura definida separa a execução da aplicação, hospedada na EC2, do armazenamento de dados, fornecido pelo RDS, e utiliza recursos da AWS para segmentação de rede e gestão de credenciais.

A conclusão final deve refletir os resultados efetivamente demonstrados pelo grupo, indicando se a aplicação ficou acessível publicamente, se as operações CRUD foram validadas e se a conexão com o RDS foi comprovada. Como próximos passos, recomenda-se automatizar a infraestrutura e o deploy, aprimorar monitoramento e logs, implementar alta disponibilidade e escalabilidade quando necessário, fortalecer controles de acesso e completar os ajustes dos 12 fatores identificados como parciais ou pendentes.

## Referências

- **FIAP.** Tech Challenge — Fase 1, Pós-Tech DevOps (DCLT). Material fornecido para a atividade.
- **GitHub.** `toggle-master-monolith`. Disponível em: https://github.com/dougls/toggle-master-monolith. Acesso em: [preencher].
- **AWS.** AWS Well-Architected / documentação dos serviços utilizados: [inserir links oficiais consultados e data de acesso].
- **The Twelve-Factor App.** Disponível em: https://12factor.net/. Acesso em: [preencher].
- [Adicionar outras fontes realmente utilizadas.]

## Apêndices e evidências

Incluam, com credenciais e informações sensíveis ocultadas:

- diagrama da arquitetura AWS;
- capturas das sub-redes e dos Security Groups;
- evidência da instância EC2 e do RDS;
- evidência do acesso SSH (sem chave privada ou dados sensíveis);
- evidência das operações da API e da persistência no banco;
- trecho do script de deploy e configuração do Secrets Manager, sem valores secretos;
- print da estimativa de custos;
- link do vídeo de demonstração (até 15 minutos).
