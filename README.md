# FordCare API

API REST para gerenciamento de clientes, veículos e manutenções de uma oficina/concessionária, com autenticação via JWT e controle de acesso por perfil.

## Link do github
https://github.com/PP950/Fordcare-api.git

## Integrantes

- Paulo Poças - RM556080
- André Luiz Fernandes de Queiroz - RM554503
- Rafael Bocchi - RM557603
- Rafael Federici de Oliveira - RM554736
- Marcos Vinícius da Silva Costa - RM555490

## Tecnologias

- Java 21
- Spring Boot 4
- Spring Data JPA / Hibernate
- Spring Security
- JWT (`com.auth0:java-jwt`)
- Flyway (migrations de banco)
- MySQL
- Springdoc OpenAPI (Swagger)
- JUnit 5 / MockMvc / Mockito

## Pré-requisitos

- JDK 21+
- Maven 3.9+
- MySQL 8 rodando localmente (ou acessível pela rede)

## Configuração

Antes de rodar o projeto, ajuste `src/main/resources/application.properties` com as credenciais do seu banco:

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/fordcare?createDatabaseIfNotExist=true&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=SUA_SENHA_AQUI

api.security.token.secret=SUA_CHAVE_SECRETA_AQUI
```

- `createDatabaseIfNotExist=true` faz o schema `fordcare` ser criado automaticamente na primeira conexão, caso não exista.
- `api.security.token.secret` é a chave usada para assinar e validar os tokens JWT. Em produção, isso deveria vir de uma variável de ambiente, não ficar hardcoded no arquivo.

As tabelas são criadas e versionadas via **Flyway**, a partir dos scripts em `src/main/resources/db/migration/`. Não é necessário criar as tabelas manualmente — elas são geradas automaticamente na primeira vez que a aplicação sobe.

## Rodando a aplicação

```bash
mvn clean install
mvn spring-boot:run
```

A API sobe em `http://localhost:8080`.

## Criando o primeiro usuário (admin)

Como todo cadastro de recurso exige autenticação, é necessário inserir o primeiro usuário administrador diretamente no banco. A senha deve estar criptografada com **BCrypt** (não é possível salvar a senha em texto puro).

Exemplo de insert, usando o hash BCrypt correspondente à senha `admin123`:

```sql
INSERT INTO usuarios (login, senha, perfil)
VALUES ('admin', '$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy', 'ADMIN');
```

Para gerar o hash de outra senha, use qualquer gerador de hash BCrypt (ex.: bibliotecas online ou o próprio `PasswordEncoder` do Spring em um teste rápido).

## Autenticação

A API usa **JWT**. O fluxo é:

1. Faça login em `POST /login`:
   ```json
   {
     "login": "admin",
     "senha": "admin123"
   }
   ```
2. A resposta traz o token:
   ```json
   {
     "token": "eyJhbGciOiJIUzI1NiJ9..."
   }
   ```
3. Envie esse token no header `Authorization` das próximas requisições:
   ```
   Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
   ```

O token expira em 2 horas.

## Perfis de acesso

| Perfil    | Permissões |
|-----------|------------|
| `ADMIN`   | Acesso total: criar, listar, atualizar e excluir clientes, veículos e manutenções |
| `CLIENTE` | Apenas leitura (`GET`) dos recursos |

## Endpoints principais

| Método | Rota                          | Acesso           | Descrição                          |
|--------|-------------------------------|-------------------|-------------------------------------|
| POST   | `/login`                      | Público           | Autenticação e geração do token JWT |
| GET    | `/clientes`                   | ADMIN, CLIENTE    | Lista clientes                      |
| POST   | `/clientes`                   | ADMIN             | Cadastra cliente                    |
| PUT    | `/clientes/{id}`               | ADMIN             | Atualiza cliente                    |
| DELETE | `/clientes/{id}`               | ADMIN             | Remove cliente                      |
| GET    | `/veiculos`                   | ADMIN, CLIENTE    | Lista veículos                      |
| POST   | `/veiculos`                   | ADMIN             | Cadastra veículo                    |
| PUT    | `/veiculos/{id}`               | ADMIN             | Atualiza veículo                    |
| DELETE | `/veiculos/{id}`               | ADMIN             | Remove veículo                      |
| GET    | `/manutencoes`                | ADMIN, CLIENTE    | Lista manutenções                   |
| POST   | `/manutencoes`                | ADMIN             | Cadastra manutenção                 |
| PUT    | `/manutencoes/{id}`             | ADMIN             | Atualiza manutenção                 |
| DELETE | `/manutencoes/{id}`             | ADMIN             | Remove manutenção                   |
| GET    | `/manutencoes/previsao/{veiculoId}` | ADMIN, CLIENTE | Previsão de próxima manutenção      |

Todas as respostas de erro seguem um formato padronizado (`status`, `mensagem`, `timestamp`), tratado centralmente por um `@RestControllerAdvice`.

## Documentação (Swagger)

Com a aplicação rodando, acesse:

```
http://localhost:8080/swagger-ui.html
```

Para testar endpoints protegidos direto pelo Swagger:
1. Faça `POST /login` e copie o token retornado
2. Clique no botão **Authorize** (ícone de cadeado, no topo da página)
3. Cole o token (sem o prefixo `Bearer`) e confirme
4. As próximas requisições no Swagger já enviarão o token automaticamente

## Testes automatizados

```bash
mvn test
```

Os testes cobrem:
- Cenário de sucesso (cadastro de cliente por um usuário ADMIN)
- Cenário de erro (dados inválidos retornando `400`)
- Cenário de acesso não autorizado (usuário sem permissão tentando escrever, retornando `403`)

## Estrutura do projeto

```
src/main/java/br/com/ford/fordcare/
├── domain/
│   ├── cliente/       # Entidade, DTOs, repositório, service e controller de Cliente
│   ├── veiculo/       # idem, para Veículo
│   ├── manutencao/    # idem, para Manutenção
│   └── usuario/       # Entidade Usuario, Role, DTOs e repositório
├── infra/
│   ├── security/      # TokenService, SecurityFilter, SecurityConfig, AuthController
│   ├── documentation/ # Configuração do Swagger/OpenAPI
│   └── exception/     # Tratamento global de erros
└── controller/        # Controllers ainda não migrados para o padrão de domínio
```
