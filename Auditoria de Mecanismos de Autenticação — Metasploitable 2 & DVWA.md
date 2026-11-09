

**Testes controlados de força bruta e enumeração de credenciais em ambiente de laboratório**

![Status](https://img.shields.io/badge/status-concluído-success)
![Ambiente](https://img.shields.io/badge/ambiente-laboratório%20isolado-blue)
![Licença](https://img.shields.io/badge/uso-apenas%20educacional-red)

---

## Aviso Legal

Este projeto foi executado **exclusivamente em ambiente de laboratório**, utilizando máquinas virtuais propositalmente vulneráveis (Metasploitable 2 e DVWA) conectadas por uma rede VirtualBox em modo Host-only. As credenciais, hosts e achados descritos neste documento são **exclusivos do laboratório** e não representam nenhum ambiente real ou de produção.

As técnicas aqui documentadas **não devem ser aplicadas contra sistemas, redes ou aplicações sem autorização explícita e por escrito do responsável pelo ambiente**. O objetivo deste material é exclusivamente educacional, no contexto do desafio de projeto da [Digital Innovation One (DIO)](https://www.dio.me/).

---

## Resumo Executivo

**Escopo:** auditoria de mecanismos de autenticação dos serviços FTP (portas 21 e 2121) e da aplicação web DVWA 1.0.7, hospedados no Metasploitable 2, a partir de uma estação Kali Linux na mesma rede isolada.

**Abordagem:** reconhecimento com Nmap, validação manual de credenciais, automação de testes de autenticação com Medusa, inspeção do código-fonte do mecanismo de proteção do DVWA e validação experimental do comportamento observado.

**Principais achados:**

- O serviço FTP (vsftpd 2.3.4, porta 21) permite autenticação anônima e aceitou a credencial fraca `msfadmin:msfadmin`.
- O formulário de autenticação do DVWA usa método GET e foi auditado com sucesso pelo módulo `web-form` do Medusa, mesmo no nível de segurança **High**.
- O único controle de proteção identificado no código do DVWA foi um atraso fixo de três segundos após falha de login — sem contador de tentativas, bloqueio de conta ou bloqueio por IP observados.
- O processamento de senha no código do DVWA utiliza MD5, algoritmo inadequado para esse fim segundo as práticas atuais.

**Conclusão geral:** o ambiente apresenta fragilidades típicas de credenciais fracas, ausência de controles de limitação de tentativas e uso de criptografia/hash obsoletos. As recomendações de mitigação estão descritas na seção correspondente e ainda não foram implementadas nem retestadas.

---

## Sumário

1. [Ambiente e Topologia](#ambiente-e-topologia)
2. [Ferramentas Utilizadas](#ferramentas-utilizadas)
3. [Metodologia](#metodologia)
4. [Fase 1 — Reconhecimento e Enumeração de Serviços](#fase-1--reconhecimento-e-enumeração-de-serviços)
5. [Fase 2 — Auditoria de Autenticação FTP](#fase-2--auditoria-de-autenticação-ftp)
6. [Fase 3 — Auditoria de Autenticação HTTP (DVWA)](#fase-3--auditoria-de-autenticação-http-dvwa)
7. [Fase 4 — Análise do Mecanismo de Proteção (Security High)](#fase-4--análise-do-mecanismo-de-proteção-security-high)
8. [Matriz de Riscos](#matriz-de-riscos)
9. [Plano de Mitigação](#plano-de-mitigação)
10. [Plano de Reteste](#plano-de-reteste)
11. [Conclusão Técnica](#conclusão-técnica)
12. [Tabela de Evidências](#tabela-de-evidências)
13. [Estrutura do Repositório](#estrutura-do-repositório)
14. [Referências](#referências)

---

## Ambiente e Topologia

| Papel | Sistema | Endereço IP | Função |
|---|---|---|---|
| Máquina de auditoria | Kali Linux | `192.168.211.129` | Execução dos testes |
| Máquina alvo | Metasploitable 2 | `192.168.211.131` | Alvo vulnerável (FTP, DVWA, etc.) |

- **Hipervisor:** VirtualBox
- **Modo de rede:** Host-only
- **Aplicação web auditada:** DVWA 1.0.7 (Apache 2.2.8 / PHP 5.2.4)

```
┌─────────────────────┐        Rede Host-Only        ┌──────────────────────────┐
│     Kali Linux       │  ───────────────────────▶   │     Metasploitable 2      │
│  192.168.211.129     │                              │    192.168.211.131        │
│  (máquina auditora)  │  ◀───────────────────────   │   FTP / HTTP / DVWA / SMB  │
└─────────────────────┘                               └──────────────────────────┘
```

> **Nota sobre isolamento:** o adaptador de rede das VMs foi configurado em modo Host-only. Essa configuração restringe o adaptador utilizado para o laboratório, mas não constitui, por si só, uma comprovação de que as máquinas não possuíam nenhuma outra interface ou rota de rede ativa. Nenhuma verificação adicional de rotas (`ip route`, interfaces adicionais) foi documentada neste projeto.

---

## Ferramentas Utilizadas

| Ferramenta | Versão | Finalidade |
|---|---|---|
| Nmap | 7.99 | Reconhecimento e enumeração de serviços |
| Medusa | 2.3 | Auditoria automatizada de autenticação (FTP e web-form) |
| cURL | — | Reprodução manual de requisições HTTP autenticadas |
| Cliente FTP de terminal | — | Validação manual de autenticação FTP |
| DVWA | 1.0.7 | Aplicação web vulnerável (módulo Brute Force) |

---

## Metodologia

```
Reconhecimento ──▶ Enumeração de Serviços ──▶ Validação Manual de Credenciais
      │                                                   │
      ▼                                                   ▼
Auditoria Automatizada (Medusa) ──▶ Análise de Código-Fonte ──▶ Validação Experimental
      │
      ▼
Matriz de Riscos ──▶ Plano de Mitigação ──▶ Plano de Reteste ──▶ Conclusão
```

Cada etapa foi documentada com objetivo, comando executado, resultado observado, análise técnica e evidência correspondente (ver [Tabela de Evidências](#tabela-de-evidências) e pasta [`screenshots/`](./screenshots)).

---

## Fase 1 — Reconhecimento e Enumeração de Serviços

### Objetivo
Mapear a superfície de ataque do alvo antes da execução dos testes de autenticação, identificando portas abertas, serviços e versões.


![[docs/1.png]]


### Comando

```bash
nmap -Pn -n -sS -sV --top-ports 1000 192.168.211.131
```

| Parâmetro | Função |
|---|---|
| `-Pn` | Ignora descoberta de host por ping (ICMP) |
| `-n` | Desabilita resolução DNS |
| `-sS` | Varredura TCP SYN |
| `-sV` | Detecção de versões de serviço |
| `--top-ports 1000` | Examina as 1000 portas TCP mais comuns |

### Resultados

- **23** portas TCP abertas / **977** fechadas
- Latência: `0.0020s` · Duração: `52.70s`
- Serviços identificados: FTP, SSH, Telnet, SMTP, DNS, HTTP, RPC, SMB, REXEC, RLOGIN, RSH, Java RMI, Bindshell, NFS, MySQL, PostgreSQL, VNC, X11, IRC, AJP13

**Achados relevantes:**

| # | Observação |
|---|---|
| 1 | Dois serviços FTP ativos — portas 21 e 2121 |
| 2 | SMB disponível nas portas 139 e 445 |
| 3 | Servidor HTTP na porta 80 |
| 4 | Serviço HTTP adicional na porta 8180 |
| 5 | Diversos serviços legados potencialmente vulneráveis |
| 6 | Serviço Bindshell identificado na porta 1524 |

### Análise

A superfície de ataque identificada é ampla, com múltiplos serviços de rede expostos. FTP, HTTP e SMB foram priorizados por estarem diretamente relacionados ao escopo definido para esta auditoria. A identificação de uma versão antiga, isoladamente, não comprova a existência de uma vulnerabilidade explorável — trata-se de um indicativo que direciona investigações posteriores.

### Limitações
A varredura utilizou `--top-ports 1000`; portanto, os resultados não cobrem necessariamente a totalidade das portas TCP do alvo.

---

## Fase 2 — Auditoria de Autenticação FTP

### Serviços Investigados

| Porta | Serviço | Versão |
|---|---|---|
| 21 | vsftpd | 2.3.4 |
| 2121 | ProFTPD | 1.3.1 |

### 2.1 Enumeração com Nmap (NSE)

```bash
nmap -Pn -n -p21,2121 -sV \
  --script ftp-anon,ftp-syst \
  -oA ~/password-security-assessment/evidence/nmap/ftp-enum \
  192.168.211.131
```

| Script | Finalidade |
|---|---|
| `ftp-anon` | Verifica a possibilidade de autenticação anônima |
| `ftp-syst` | Consulta informações de sistema disponibilizadas pelo servidor |

**Resultados:**

- **Porta 21 (vsftpd 2.3.4):** autenticação anônima permitida (código FTP `230`); canais de controle e dados sem criptografia; timeout de sessão de 300s.
- **Porta 2121 (ProFTPD 1.3.1):** autenticação anônima não confirmada na tentativa realizada. A ausência de confirmação não comprova que o acesso esteja desabilitado.

### 2.2 Validação Manual — Porta 21 (acesso anônimo)

Procedimento: conexão via cliente FTP de terminal → login como `anonymous` → `pwd` → `ls -la`.

| Verificação | Resultado |
|---|---|
| Autenticação | `230 Login successful` |
| Diretório remoto | `/` |
| Listagem | `226 Directory send OK` — apenas `.` e `..` |
| Permissões | `drwxr-xr-x` (sem escrita para outros) |

### 2.3 Validação Manual — Porta 2121

```bash
ftp 192.168.211.131 2121
```

Resultado: usuário `anonymous` reconhecido, porém autenticação recusada (`530 Login incorrect`). Comportamento distinto do vsftpd — o ProFTPD não validou o acesso anônimo nessa tentativa específica.

### 2.4 Teste de Permissão de Escrita (porta 21)

```bash
echo "FTP anonymous write permission test" > ~/ftp-write-test.txt
```

No cliente FTP:

```
put ftp-write-test.txt
```

**Resultado:** `553 Could not create file.` — criação de arquivo recusada no diretório raiz exposto à sessão anônima. Esse resultado é limitado ao diretório testado e não comprova que todos os diretórios do servidor estejam protegidos contra escrita.

### 2.5 Validação de Credencial de Referência

**Credenciais de laboratório** (uso exclusivo neste ambiente):

| Usuário | Senha | Resultado |
|---|---|---|
| `msfadmin` | `msfadmin` | `230 Login successful` → `/home/msfadmin` |

Essa credencial, validada manualmente, serviu de referência para o teste automatizado com Medusa.

### 2.6 Auditoria Automatizada com Medusa

```bash
medusa -h 192.168.211.131 \
  -u msfadmin \
  -P ~/password-security-assessment/evidence/medusa/ftp-passwords.txt \
  -M ftp \
  -n 21 \
  -t 1 \
  -f \
  -v 6 \
  -O ~/password-security-assessment/evidence/medusa/ftp-validation.log
```

Wordlist controlada (2 entradas): `senha_incorreta`, `msfadmin`.

**Resultado:**

```
ACCOUNT FOUND: [ftp] Host: 192.168.211.131 User: msfadmin Password: msfadmin [SUCCESS]
```

O Medusa identificou corretamente a credencial previamente validada de forma manual, confirmando o funcionamento do módulo FTP da ferramenta nesse cenário. O teste utilizou uma única conta, duas senhas e uma tarefa simultânea — não avalia a resistência do serviço a volumes maiores de tentativas.

### 2.7 Teste de Tentativas Inválidas Consecutivas

Wordlist: `senha_errada_01`, `senha_errada_02`, `senha_errada_03` (1 tarefa simultânea).

**Resultado:** as três tentativas foram processadas sem interrupção, sem mensagem de bloqueio e sem desconexão. Uma validação posterior com a credencial correta (`msfadmin:msfadmin`) confirmou que a conta permaneceu acessível.

> **Observação:** não foi observado bloqueio de conta após três tentativas inválidas consecutivas, nas condições específicas testadas. Esse resultado não comprova a ausência absoluta de mecanismos de bloqueio no serviço — apenas que, no cenário avaliado (três tentativas, uma conta, uma tarefa simultânea), nenhum bloqueio foi acionado.


![[4.png]]
![[5.png]]
![[8.png]]
![[11.png]]
![[12.png]]
![[13.png]]



---

## Fase 3 — Auditoria de Autenticação HTTP (DVWA)

### 3.1 Identificação do Formulário

Inspeção do HTML do módulo Brute Force do DVWA:

| Atributo | Valor |
|---|---|
| Método HTTP | `GET` |
| Destino | própria página (`action="#"`) |
| Campos | `username`, `password` |
| Token anti-CSRF | não identificado no formulário inspecionado |

### 3.2 Validação Manual de Autenticação

**Credenciais de laboratório** (uso exclusivo neste ambiente):

| Usuário | Senha | Resposta |
|---|---|---|
| `auditoria_teste` | `senha_incorreta` | `Username and/or password incorrect.` |
| `admin` | `password` | `Welcome to the password protected area admin` |

**Sessão identificada:** cookies `PHPSESSID` e `security=high`.

### 3.3 Reprodução via cURL

A requisição de autenticação foi reproduzida com sucesso via `curl`, utilizando método GET e os cookies de sessão, confirmando acesso à área protegida como usuário `admin` com nível de segurança High.

**Stack identificada via inspeção HTML:** DVWA 1.0.7 · Apache 2.2.8 · PHP 5.2.4.

### 3.4 Auditoria Automatizada com Medusa (web-form)

| Parâmetro | Valor |
|---|---|
| Alvo | `192.168.211.131:80` |
| Módulo | `web-form` |
| Método | `GET` |
| Usuário | `admin` |
| Sessão | `PHPSESSID`, `security=high` |
| Concorrência | 1 |

| Usuário | Senha | Resultado |
|---|---|---|
| admin | senha_incorreta | Recusada |
| admin | password | Aceita |

```
ACCOUNT FOUND: [web-form] Host: 192.168.211.131 User: admin Password: password [SUCCESS]
```

O Medusa reproduziu corretamente a autenticação HTTP, enviando os cookies necessários e diferenciando respostas de falha e sucesso — validando a compatibilidade do módulo `web-form` mesmo com o nível High habilitado. O teste demonstra que a credencial foi descoberta dentro da lista fornecida, não representando uma avaliação completa dos mecanismos de proteção contra força bruta.

### 3.5 Exposição de Credenciais via GET

O método GET transporta os parâmetros de autenticação diretamente na URL da requisição, o que pode expor essas informações em históricos de navegador, logs de servidores web e proxies, ferramentas intermediárias (caches, balanceadores) e outros registros ao longo do caminho da requisição. A utilização de HTTPS reduz a exposição em trânsito, mas não elimina os riscos de exposição em logs e históricos locais.

A ausência de um campo de token anti-CSRF identificado no formulário é uma observação estrutural; isoladamente, ela não constitui uma comprovação de vulnerabilidade CSRF explorável, que exigiria testes dedicados fora do escopo deste projeto.
![[14.png]]
![[15.png]]
![[16.png]]
![[17.png]]



---

## Fase 4 — Análise do Mecanismo de Proteção (Security High)

### 4.1 Revisão de Código-Fonte

Endpoint: `/dvwa/vulnerabilities/brute/` (GET). Trecho identificado no tratamento de falha de autenticação:

```php
else {
    // Login failed
    sleep(3);
    echo "<pre><br>Username and/or password incorrect.</pre>";
}
```

O código também evidenciou a aplicação da função `md5()` sobre a senha recebida, antes da consulta ao banco de dados.

**Mecanismos avaliados:**

| Mecanismo | Resultado da avaliação |
|---|---|
| Atraso fixo (`sleep(3)`) após falha | Identificado no código |
| Contador de tentativas | Não identificado no código analisado |
| Bloqueio de conta | Não observado nos testes realizados |
| Bloqueio por IP | Não identificado no código analisado |
| Processamento de senha com MD5 | Identificado no código |

> **Sobre o uso de MD5:** o código analisado aplica `md5()` à senha recebida durante o fluxo de autenticação. Não foi inspecionada diretamente a estrutura do banco de dados para confirmar o formato de armazenamento persistente. Ainda assim, MD5 é inadequado como função de hash de senha — caso seja de fato o algoritmo utilizado para armazenamento —, pois não oferece o custo computacional nem as proteções (salt adaptativo, resistência a ataques de força bruta em hardware dedicado) de funções modernas como Argon2id ou bcrypt.

### 4.2 Validação Experimental do Atraso

| Teste | HTTP | Tempo de resposta | Resultado |
|---|---|---|---|
| Credenciais inválidas | 200 OK | `3.020851s` | Rejeitada |
| Credenciais válidas | 200 OK | `0.021480s` | Aceita |

Os resultados confirmam experimentalmente o atraso de aproximadamente três segundos identificado no código-fonte, aplicado apenas em tentativas de autenticação falhas.

### 4.3 Tentativas Consecutivas e Bloqueio de Conta

Duas sequências de três tentativas inválidas foram testadas (usuário de teste e conta `admin`):

- Todas as seis tentativas retornaram `HTTP 200`, com tempo médio próximo de três segundos;
- Nenhum bloqueio foi observado em nenhum dos dois cenários testados;
- A autenticação com credencial correta (`admin:password`) foi aceita normalmente logo após as tentativas falhas.

**Limitação:** os resultados representam uma avaliação controlada com três falhas consecutivas por usuário. Eles não comprovam ausência absoluta de mecanismos de bloqueio em outros componentes da infraestrutura ou sob volumes de tentativas maiores.

### 4.4 Conclusão da Fase

O atraso fixo de três segundos aumenta o custo temporal de ataques automatizados, mas não impede sua execução nem substitui controles como limitação de tentativas, bloqueio progressivo ou monitoramento de padrões suspeitos de autenticação.

---

## Matriz de Riscos

| ID | Ativo | Achado | Severidade preliminar |
|---|---|---|---|
| R01 | FTP | Acesso anônimo permitido | Média |
| R02 | FTP | Credenciais `msfadmin:msfadmin` aceitas | Alta |
| R03 | FTP | Comunicação sem criptografia | Alta |
| R04 | DVWA | Processamento de senha com MD5 | Alta |
| R05 | DVWA | Ausência de bloqueio observado após tentativas inválidas | Média |
| R06 | DVWA | Atraso fixo (3s) como único controle anti-automação identificado | Média |
| R07 | Servidor Web | Apache e PHP obsoletos | Alta |

As severidades acima são preliminares e baseadas exclusivamente nos resultados observados em laboratório. Não representam uma classificação definitiva de risco em ambiente de produção, que deve considerar exposição real, probabilidade, impacto de negócio e controles compensatórios existentes.

**Resumo dos impactos potenciais:**

- **R01/R02:** acesso não autorizado a arquivos e recursos associados às contas comprometidas.
- **R03:** exposição de credenciais e dados em trânsito à interceptação na rede.
- **R04:** maior vulnerabilidade a ataques offline de quebra de hash, caso os hashes sejam obtidos.
- **R05/R06:** continuidade viável de ataques automatizados de força bruta.
- **R07:** exposição a vulnerabilidades já corrigidas em versões mais recentes dos componentes.

---

## Plano de Mitigação

| ID | Recomendação | Validação proposta |
|---|---|---|
| R01 | Desabilitar autenticação anônima; aplicar permissões mínimas; monitorar tentativas de acesso | Tentar login anônimo e confirmar rejeição |
| R02 | Remover credenciais padrão; exigir senhas fortes e únicas; política de rotação de credenciais comprometidas | Confirmar rejeição de credenciais antigas |
| R03 | Substituir FTP por SFTP ou FTPS; desabilitar o protocolo em texto claro após migração | Verificar disponibilidade do serviço seguro e bloqueio do protocolo inseguro |
| R04 | Substituir MD5 por Argon2id ou bcrypt, com salt individual e parâmetros de custo adequados | Inspecionar o algoritmo utilizado e testar o fluxo de autenticação |
| R05 | Implementar limitação de tentativas por conta e origem; bloqueio temporário; registro de falhas consecutivas | Executar tentativas controladas e validar a aplicação dos limites |
| R06 | Complementar o atraso fixo com rate limiting efetivo e controles anti-automação; monitorar volume e frequência de tentativas | Comparar o comportamento antes e depois da implementação |
| R07 | Atualizar Apache, PHP, sistema operacional e bibliotecas; remover serviços desnecessários | Reexecutar reconhecimento com Nmap após as atualizações |

**Priorização sugerida** (consistente com a Matriz de Riscos):

1. **Alta:** credenciais fracas e serviços sem criptografia (R02, R03)
2. **Alta:** atualização de componentes e substituição do MD5 (R04, R07)
3. **Média:** controles de limitação de tentativas e proteção contra automação (R05, R06)
4. **Média:** restrição de acesso anônimo e endurecimento de permissões (R01)

Nenhuma correção foi aplicada ou validada no ambiente original até o momento. O Metasploitable 2 deve permanecer como alvo vulnerável de laboratório — correções devem ser avaliadas em cópia separada ou em ambiente dedicado à validação defensiva.

---

## Plano de Reteste

| ID | Controle | Procedimento de Reteste | Resultado Esperado | Status |
|---|---|---|---|---|
| R01 | Acesso anônimo FTP | Tentar autenticação sem credenciais | Acesso anônimo rejeitado | Pendente |
| R02 | Credenciais padrão | Testar credenciais antigas; validar permissões da nova conta | Credenciais antigas rejeitadas; permissões restritas | Pendente |
| R03 | Criptografia em trânsito | Validar uso de SFTP/FTPS; testar FTP em texto claro | Transferência protegida; FTP em claro desabilitado ou bloqueado | Pendente |
| R04 | Armazenamento de senhas | Inspecionar o mecanismo e algoritmo de hash utilizado | Hash seguro, com salt e custo adequado (Argon2id ou bcrypt) | Pendente |
| R05 | Controle de tentativas | Executar tentativas inválidas controladas | Tentativas limitadas e eventos registrados | Pendente |
| R06 | Proteção contra automação | Repetir tentativas automatizadas autorizadas em baixa intensidade | Mecanismos identificam ou limitam a atividade | Pendente |
| R07 | Atualização e redução da superfície de ataque | Revarredura com Nmap | Componentes atualizados; serviços desnecessários desativados | Pendente |

Para cada reteste devem ser registrados: comando ou procedimento utilizado, data e horário da execução, resultado observado, captura de tela ou arquivo de saída, e comparação com o resultado obtido antes da mitigação. Uma recomendação é considerada validada apenas quando o procedimento demonstrar o resultado esperado com evidência técnica correspondente; caso contrário, o controle permanece pendente e exige nova análise.

---

## Conclusão Técnica

Este projeto aplicou, de forma prática e em ambiente controlado, um fluxo completo de auditoria de autenticação — reconhecimento, enumeração, validação manual, automação com Medusa, análise de código-fonte e validação experimental — contra os serviços FTP e a aplicação web DVWA do Metasploitable 2.

**Principais achados:**

- O serviço FTP (porta 21) permitiu autenticação anônima e aceitou a credencial fraca `msfadmin:msfadmin`, embora a tentativa de escrita no diretório raiz tenha sido recusada.
- O Medusa identificou com sucesso credenciais válidas tanto em FTP quanto no módulo web-form do DVWA, mesmo com o nível de segurança High habilitado.
- O formulário de login do DVWA utilizava método GET, expondo usuário e senha diretamente na URL da requisição.
- O único controle de proteção identificado no código do DVWA foi um atraso fixo de três segundos após falha — sem contador de tentativas, bloqueio de conta ou bloqueio por IP identificados.
- O código do DVWA aplica MD5 ao processar a senha durante a autenticação, prática inadequada caso esse seja também o algoritmo de armazenamento persistente.

**Lição central:** um atraso fixo após falhas de autenticação não substitui mecanismos efetivos de limitação de tentativas, monitoramento e proteção contra automação — ele eleva o custo temporal do ataque, sem impedi-lo.

### Limitações do estudo

- Testes restritos a um ambiente de laboratório isolado por configuração de rede Host-only, sem verificação adicional de rotas ou interfaces das VMs.
- Avaliação de bloqueio limitada a pequenas sequências de tentativas controladas (três tentativas por cenário).
- Não foi realizada inspeção direta da estrutura do banco de dados do DVWA para confirmar o formato de armazenamento persistente de senhas.
- Não foi realizada análise completa de todos os mecanismos internos de autenticação do ambiente.
- As recomendações de mitigação ainda não foram implementadas nem retestadas — este documento define os critérios de validação, não comprova a aplicação das correções.

---

## Tabela de Evidências

| Evidência | Fase | Descrição |
|---|---|---|
| `screenshots/01-reconhecimento-inicial.png` | 1 | Execução inicial do Nmap contra o alvo |
| `screenshots/02-reconhecimento-documentado.png` | 1 | Execução do Nmap com salvamento dos resultados (`-oA`) |
| `screenshots/03-ftp-enum.png` | 2 | Resultado da enumeração FTP com scripts NSE |
| `screenshots/04-ftp-anon-porta21.png` | 2 | Autenticação anônima e listagem de diretório — porta 21 |
| `screenshots/05-ftp-anon-porta2121.png` | 2 | Tentativa de autenticação anônima — porta 2121 |
| `screenshots/06-ftp-write-test.png` | 2 | Teste de escrita recusado na sessão anônima |
| `screenshots/07-ftp-msfadmin-manual.png` | 2 | Validação manual da credencial `msfadmin:msfadmin` |
| `screenshots/08-medusa-ftp.png` | 2 | Execução do Medusa e credencial FTP identificada |
| `screenshots/09-medusa-ftp-tentativas-invalidas.png` | 2 | Teste de tentativas inválidas consecutivas |
| `screenshots/10-ftp-validacao-pos-falhas.png` | 2 | Autenticação válida após falhas consecutivas |
| `screenshots/11-14-dvwa-formulario-curl-medusa.png` | 3 | Inspeção do formulário, reprodução via cURL e execução do Medusa (web-form) |
| `screenshots/15-codigo-fonte-security-high.png` | 4 | Código-fonte do mecanismo de proteção (Security High) |
| `screenshots/16-17-medicao-tempo-resposta.png` | 4 | Medição de tempo — tentativa inválida e válida |
| `screenshots/18-20-tentativas-consecutivas-bloqueio.png` | 4 | Tentativas consecutivas e validação pós-falhas |

> Os nomes de arquivo acima são sugestões de padronização. Ajuste conforme a nomenclatura final adotada na pasta `screenshots/` do repositório.

---

## Estrutura do Repositório

```
password-security-assessment/
├── README.md
├── .gitignore
├── docs/                          # Documentação complementar, se houver
├── screenshots/                   # Capturas de tela organizadas por fase
│   ├── 01-reconhecimento-inicial.png
│   ├── 02-reconhecimento-documentado.png
│   └── ...
├── evidence/
│   ├── nmap/                      # Saídas .nmap / .xml / .gnmap
│   └── medusa/                    # Wordlists e logs de execução (.txt / .log)
└── scripts/                       # Scripts auxiliares, se houver
```

**Antes do primeiro commit:**

- Revisar manualmente os arquivos em `evidence/` (incluindo logs não cobertos pelo `.gitignore`) para garantir que nenhum dado sensível além do já documentado seja publicado.
- Confirmar que o `.gitignore` cobre adequadamente arquivos de log sensíveis (ex.: `*-validation.log`), sem substituir a revisão manual do restante dos arquivos.
- Padronizar os nomes das capturas de tela conforme a [Tabela de Evidências](#tabela-de-evidências).

---

## Referências

- [Nmap — Manual Oficial](https://nmap.org/book/)
- [Medusa — Documentação](http://www.foofus.net/jmk/medusa/medusa.html)
- [DVWA — Damn Vulnerable Web Application](http://www.dvwa.co.uk/)
- [Kali Linux — Site Oficial](https://www.kali.org/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Credential Stuffing Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Credential_Stuffing_Prevention_Cheat_Sheet.html)

---

<sub>Projeto desenvolvido como parte do desafio de projeto da [Digital Innovation One (DIO)](https://www.dio.me/) — trilha de Segurança da Informação / Pentest. Ambiente de laboratório isolado, voltado exclusivamente para fins educacionais.</sub>
