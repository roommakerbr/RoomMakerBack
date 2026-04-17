# Revisão completa do projeto RoomMakerBack

Data da revisão: 2026-04-17
Escopo: revisão estática de arquitetura, segurança, qualidade, manutenção e testabilidade do backend.

## 1) Visão geral

O projeto está bem estruturado em camadas (controllers, domain/managers, services, repositórios) e cobre um caso real relevante: salas com categorias de jogos, chat e autenticação JWT.

Pontos positivos:
- Boa separação por domínio (usuário, sala, categorias).
- Uso de DTOs com validação Bean Validation nas entradas principais.
- Configuração de CORS externalizada via propriedade.
- Presença de documentação inicial no README.

## 2) Principais achados (priorizados)

## Crítico

1. Segurança do Spring Security está efetivamente desativada.
- Em `SecurityConfig`, todo request está com `permitAll()`.
- Isso força a proteção a ficar em interceptors manuais (HTTP/WS), aumentando risco de bypass e inconsistências.

2. Fluxo de build/teste quebrado por incompatibilidade de versão Java no ambiente.
- `./gradlew test` falhou com `Unsupported class file major version 69`.
- Isso indica mismatch entre bytecode/JDK em uso para o Gradle script e versão suportada pelo runtime local.

## Alto

3. Endpoint de auto-ping hardcoded para produção.
- `PingController` faz chamada periódica para `https://roommakerback.onrender.com/ping`.
- Em ambiente local/dev, esse comportamento pode gerar efeito colateral e mascarar healthcheck real.

4. Códigos de verificação/recuperação em memória local (`HashMap`) sem TTL/distribuição.
- Em caso de múltiplas instâncias, reinício da aplicação ou escala horizontal, o fluxo 2FA/recuperação pode falhar.

5. Cobertura de testes insuficiente.
- Apenas 1 arquivo de teste para um projeto relativamente grande com múltiplos domínios de regra.

## Médio

6. Arquivo de backup no repositório (`*.java~`).
- Encontrado artefato de editor em `chat/repository/ChatEntity.java~`.

7. Inconsistência de padrões de injeção.
- Mistura de `@Autowired` em campo com construtor.
- Padrão por construtor (imutável) facilitaria testes e manutenção.

8. Configuração de interceptor HTTP com exclusões frágeis.
- Exclusão de `/swagger-ui/` pode não cobrir todos os paths de swagger (`/swagger-ui/**`, `/v3/api-docs/**`).

## 3) Recomendações objetivas

### Segurança (curto prazo)
- Migrar autenticação para `SecurityFilterChain` com rotas públicas explícitas e rotas protegidas autenticadas.
- Deixar interceptor HTTP apenas para enriquecimento de contexto, não como barreira primária.
- No WebSocket, validar autenticação no `CONNECT` e usar `Principal` da sessão STOMP.

### Confiabilidade (curto/médio prazo)
- Mover códigos 2FA/recuperação para storage com expiração (Redis ou Mongo com TTL index).
- Tornar URL de auto-ping configurável por profile/env e desabilitar em ambiente local.

### Qualidade de engenharia (médio prazo)
- Padronizar injeção por construtor em toda base.
- Aumentar suíte de testes (managers + controllers + integração de segurança).
- Adicionar pipeline CI com validações mínimas (`test`, lint/checkstyle/spotbugs opcional).

### Higiene de repositório
- Remover artefatos temporários e ignorar backups de editor (`*~`).

## 4) Roteiro sugerido de execução

Fase 1 (1 sprint):
- Corrigir baseline de build/teste no CI e local.
- Limpar artefatos do repo.
- Endurecer segurança de rotas com Spring Security.

Fase 2 (1 sprint):
- Persistir e expirar códigos de autenticação fora da memória local.
- Cobrir fluxos críticos com testes automatizados.

Fase 3 (contínuo):
- Melhorias incrementais de arquitetura (injeção, observabilidade, documentação de operação).

## 5) Resultado desta revisão

Esta revisão foi principalmente estática (leitura de código + execução de teste inicial de build).

Alterações aplicadas no repositório nesta entrega:
1. Criação deste relatório de revisão completo.
2. Remoção de arquivo de backup indevido.
3. Atualização do `.gitignore` para prevenir novos backups `*~`.
