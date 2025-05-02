# Sistema de Mensageria com RabbitMQ

## Visão Geral

Este projeto implementa um sistema de mensageria utilizando RabbitMQ como message broker, demonstrando o fluxo completo de mensagens entre produtores (Senders) e consumidores (Receivers) com um serviço de validação intermediário.

O sistema foi desenvolvido usando Visual Studio 2022, .NET 6.0 e RabbitMQ 3.x executado via Docker.

## Arquitetura do Sistema

O sistema é composto por 5 componentes principais:

1. **Sender1.InfoFrutas**: Produtor que envia informações sobre frutas de época
2. **Sender2.InfoUsuario**: Produtor que envia informações sobre usuários
3. **Validation.Service**: Serviço que recebe, valida e encaminha mensagens
4. **Receiver1.ConsumidorFrutas**: Consumidor que recebe informações validadas sobre frutas
5. **Receiver2.ConsumidorUsuario**: Consumidor que recebe informações validadas sobre usuários

### Diagrama de Fluxo

Conforme o diagrama original:

```
   SENDER 1          RabbitMQ         VALIDATION          RabbitMQ           RECEIVE 1
  (Producer) ------> (Broker) ------> (Consumer) ------> (Broker) ------> (Consumer)
                                      (Producer)

   SENDER 2          RabbitMQ                             RabbitMQ           RECEIVE 2
  (Producer) ------> (Broker) -------------------------> (Broker) ------> (Consumer)
```

## Pré-requisitos

- Visual Studio 2022
- .NET 6.0 ou superior
- Docker Desktop

## Etapas de Desenvolvimento

### 1. Configuração do Ambiente RabbitMQ

O primeiro passo foi configurar o RabbitMQ usando Docker:

```bash
docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

Este comando:
- Cria um container Docker chamado "rabbitmq"
- Mapeia a porta 5672 para comunicação AMQP
- Mapeia a porta 15672 para acessar o painel de administração via HTTP
- Utiliza a imagem "rabbitmq:3-management" que inclui o plugin de gerenciamento web

Após a execução, o painel de administração do RabbitMQ fica disponível em `http://localhost:15672` com as credenciais padrão (usuário: guest, senha: guest).

### 2. Criação da Estrutura do Projeto

1. **Criar a solução no Visual Studio 2022**
   - Novo projeto do tipo "Aplicativo de Console"
   - Nome da solução: "RabbitMQFrutasEUsuarios"

2. **Adicionar os projetos de console**
   - Sender1.InfoFrutas
   - Sender2.InfoUsuario
   - Validation.Service
   - Receiver1.ConsumidorFrutas
   - Receiver2.ConsumidorUsuario

3. **Adicionar o projeto de biblioteca compartilhada**
   - SharedModels (para definir os modelos de dados comuns)

4. **Instalar o pacote NuGet RabbitMQ.Client**
   - Em todos os projetos, instalar a versão 7.1.2 do RabbitMQ.Client
   - Comando NuGet: `Install-Package RabbitMQ.Client -Version 7.1.2`

### 3. Implementação dos Modelos de Dados

Criação das classes de modelo no projeto SharedModels:

- **FrutaInfo**: Representa informações sobre frutas (Nome, Descrição, Data/Hora, Status de Validação)
- **UsuarioInfo**: Representa informações de usuários (Nome Completo, Endereço, RG, CPF, Data/Hora de Registro, Status de Validação)

### 4. Implementação dos Componentes

#### a. Produtores (Senders)

1. **Sender1.InfoFrutas**:
   - Cria uma conexão com o RabbitMQ
   - Declara a exchange "frutas_exchange" do tipo Direct
   - Permite ao usuário inserir informações sobre frutas
   - Serializa os dados em JSON
   - Publica a mensagem com routing key "frutas_info"

2. **Sender2.InfoUsuario**:
   - Cria uma conexão com o RabbitMQ
   - Declara a exchange "usuarios_exchange" do tipo Direct
   - Permite ao usuário inserir informações sobre usuários
   - Serializa os dados em JSON
   - Publica a mensagem com routing key "usuarios_info"

#### b. Serviço de Validação

**Validation.Service**:
   - Cria uma conexão com o RabbitMQ
   - Configura a estrutura para frutas:
     - Declara exchanges "frutas_exchange" e "frutas_validadas_exchange"
     - Cria a fila "frutas_validation_queue"
     - Vincula a fila à exchange com routing key "frutas_info"
   - Configura a estrutura para usuários:
     - Declara exchanges "usuarios_exchange" e "usuarios_validados_exchange"
     - Cria a fila "usuarios_validation_queue"
     - Vincula a fila à exchange com routing key "usuarios_info"
   - Consome mensagens das filas de validação
   - Executa a validação dos dados:
     - Para frutas: sempre valida (simulação)
     - Para usuários: valida se o CPF tem 11 dígitos
   - Publica mensagens validadas nas exchanges de saída

#### c. Consumidores (Receivers)

1. **Receiver1.ConsumidorFrutas**:
   - Cria uma conexão com o RabbitMQ
   - Declara a exchange "frutas_validadas_exchange"
   - Cria a fila "frutas_consumidor_queue"
   - Vincula a fila à exchange com routing key "frutas_validadas"
   - Consome mensagens e exibe as informações na tela

2. **Receiver2.ConsumidorUsuario**:
   - Cria uma conexão com o RabbitMQ
   - Declara a exchange "usuarios_validados_exchange"
   - Cria a fila "usuarios_consumidor_queue"
   - Vincula a fila à exchange com routing key "usuarios_validados"
   - Consome mensagens e exibe as informações na tela

### 5. Particularidades da Implementação

1. **Uso de Async/Await**
   - Todos os métodos de comunicação com o RabbitMQ foram implementados de forma assíncrona
   - Uso de `CreateConnectionAsync`, `CreateChannelAsync`, etc.
   - Consumidores utilizam `AsyncEventingBasicConsumer` para processamento assíncrono

2. **Tratamento de Mensagens**
   - Mensagens são serializadas/desserializadas com `System.Text.Json`
   - Utilização de `BasicAckAsync` para confirmar o processamento das mensagens
   - Utilização de `BasicNackAsync` para rejeitar mensagens inválidas

3. **Configuração de Durabilidade**
   - Exchanges e filas configuradas com `durable: true` para persistir mesmo após reinicialização do RabbitMQ
   - Mensagens marcadas como `Persistent` nas propriedades básicas

## Configuração dos Projetos de Inicialização

Para facilitar os testes, o Visual Studio foi configurado para iniciar todos os componentes simultaneamente:

1. Clicar com o botão direito na solução no Explorador de Soluções
2. Selecionar "Configurar Projetos de Inicialização..."
3. Escolher a opção "Múltiplos projetos de inicialização"
4. Definir a ação como "Iniciar" para todos os projetos
5. Ordenar os projetos na seguinte sequência:
   - Validation.Service
   - Receiver1.ConsumidorFrutas
   - Receiver2.ConsumidorUsuario
   - Sender1.InfoFrutas
   - Sender2.InfoUsuario

Esta ordem garante que os consumidores e o serviço de validação estejam prontos antes que os produtores comecem a enviar mensagens.

## Guia de Testes

### Teste 1: Fluxo de Informações de Frutas

**Objetivo**: Verificar o envio, validação e recebimento de informações sobre frutas.

**Passos**:
1. Iniciar a aplicação (F5)
2. No console do Sender1.InfoFrutas:
   - Digitar `1` para enviar informações sobre uma fruta
   - Inserir o nome da fruta (exemplo: "Maçã")
   - Inserir uma descrição (exemplo: "Fruta vermelha rica em antioxidantes")
3. Verificar no console do Validation.Service se a mensagem foi recebida e validada
4. Verificar no console do Receiver1.ConsumidorFrutas se a mensagem validada foi recebida

**Resultado esperado**:
- Em Sender1.InfoFrutas: "Mensagem enviada sobre a fruta: Maçã"
- Em Validation.Service: 
  - "Recebida informação sobre fruta: Maçã"
  - "Fruta validada: Maçã"
- Em Receiver1.ConsumidorFrutas:
  - "Nova mensagem recebida"
  - "Fruta: Maçã"
  - "Descrição: Fruta vermelha rica em antioxidantes"
  - "Validada: Sim"

### Teste 2: Fluxo de Informações de Usuários (Caso Válido)

**Objetivo**: Verificar o envio, validação e recebimento de informações sobre usuários válidos.

**Passos**:
1. No console do Sender2.InfoUsuario:
   - Digitar `1` para enviar informações sobre um usuário
   - Inserir o nome completo (exemplo: "João Silva")
   - Inserir o endereço (exemplo: "Rua das Flores, 123")
   - Inserir o RG (exemplo: "12.345.678-9")
   - Inserir o CPF (exemplo: "12345678901") - 11 dígitos
2. Verificar no console do Validation.Service se a mensagem foi recebida e validada
3. Verificar no console do Receiver2.ConsumidorUsuario se a mensagem validada foi recebida

**Resultado esperado**:
- Em Sender2.InfoUsuario: "Mensagem enviada sobre o usuário: João Silva"
- Em Validation.Service: 
  - "Recebida informação sobre usuário: João Silva"
  - "Usuário validado: João Silva"
- Em Receiver2.ConsumidorUsuario:
  - "Nova mensagem recebida"
  - "Nome: João Silva"
  - "CPF: 12345678901"
  - "Validada: Sim"

### Teste 3: Validação de CPF Inválido

**Objetivo**: Verificar o comportamento do sistema quando um CPF inválido é enviado.

**Passos**:
1. No console do Sender2.InfoUsuario:
   - Digitar `1` para enviar informações sobre um usuário
   - Inserir o nome completo (exemplo: "Maria Oliveira")
   - Inserir o endereço (exemplo: "Av. Principal, 456")
   - Inserir o RG (exemplo: "98.765.432-1")
   - Inserir o CPF (exemplo: "123456") - menos de 11 dígitos
2. Verificar no console do Validation.Service se a mensagem foi recebida e rejeitada
3. Verificar que nenhuma mensagem é recebida pelo Receiver2.ConsumidorUsuario

**Resultado esperado**:
- Em Sender2.InfoUsuario: "Mensagem enviada sobre o usuário: Maria Oliveira"
- Em Validation.Service: 
  - "Recebida informação sobre usuário: Maria Oliveira"
  - "CPF inválido: 123456"
- Em Receiver2.ConsumidorUsuario: nenhuma mensagem relacionada ao usuário Maria Oliveira

## Monitoramento pelo Painel de Gerenciamento do RabbitMQ

O painel de gerenciamento do RabbitMQ permite monitorar todo o fluxo de mensagens:

1. Acessar http://localhost:15672/ (usuário: guest, senha: guest)
2. Na aba "Exchanges", verificar:
   - frutas_exchange
   - frutas_validadas_exchange
   - usuarios_exchange
   - usuarios_validados_exchange
3. Na aba "Queues", verificar:
   - frutas_validation_queue
   - frutas_consumidor_queue
   - usuarios_validation_queue
   - usuarios_consumidor_queue
4. Na aba "Connections", visualizar as conexões estabelecidas pelos componentes
5. Na aba "Channels", visualizar os canais abertos para cada conexão

## Considerações Finais

Este projeto demonstra a implementação de um sistema de mensageria completo utilizando RabbitMQ, com fluxos de validação e processamento de mensagens. Os conceitos aplicados incluem:

- Exchanges do tipo Direct para roteamento de mensagens
- Filas duráveis para garantir a persistência das mensagens
- Confirmação de recebimento (ACK) para garantir o processamento correto
- Validação de mensagens com regras de negócio customizáveis
- Desacoplamento entre produtores e consumidores

