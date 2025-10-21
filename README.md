*Desafio: Análise de Força Bruta com Kali, Medusa e Ambientes Vulneráveis
Este repositório documenta a execução de simulações de ataques de força bruta em um ambiente de laboratório controlado. O objetivo é compreender as mecânicas de ataque a diferentes serviços (FTP, Web, SMB) e, principalmente, exercitar e documentar as medidas de mitigação e defesa.

*Ferramentas Utilizadas:

Kali Linux: Distribuição para testes de penetração.

Medusa: Ferramenta de linha de comando para força bruta paralela.

Metasploitable 2: VM intencionalmente vulnerável (nosso alvo).

DVWA (Damn Vulnerable Web Application): Aplicação web vulnerável, usada aqui no Metasploitable 2.

Disclaimer: Todas as atividades foram realizadas em uma rede privada e isolada (VirtualBox Host-Only). As técnicas aqui descritas destinam-se exclusivamente a fins educacionais e de auditoria em ambientes autorizados.

1. Configuração do Ambiente
O laboratório foi configurado no VirtualBox com os seguintes componentes:

VM Atacante: Kali Linux (ex: 192.168.56.101)

VM Alvo: Metasploitable 2 (ex: 192.168.56.102)

Ambas as VMs foram configuradas para usar a mesma rede "Host-Only" (vboxnet0), garantindo que o tráfego ficasse contido e isolado da rede principal.

Os endereços IP foram identificados usando ip a no Kali e ifconfig no Metasploitable.

2. Cenários de Simulação e Mitigação
Foram utilizados wordlists simples para os testes, conforme detalhado na Seção 3.

Cenário A: Força Bruta em FTP (vsftpd no Metasploitable 2)
O serviço FTP (File Transfer Protocol) é um vetor comum de ataque, pois muitas vezes é configurado com credenciais padrão ou fracas.

A. Reconhecimento: Primeiro, validamos que a porta 21 (FTP) estava aberta no alvo:

Bash

nmap -sV 192.168.56.102
Resultado: Porta 21/tcp aberta, serviço vsftpd 2.3.4.

B. Execução (Medusa): Utilizamos o Medusa para testar uma lista de senhas (pass.txt) contra um usuário conhecido (msfadmin):

Bash

medusa -h 192.168.56.102 -u msfadmin -P wordlists/pass.txt -M ftp
-h: Host/Alvo

-u: Usuário único

-P: Caminho para a wordlist de senhas

-M: Módulo a ser usado (neste caso, ftp)

C. Resultado: O Medusa rapidamente identificou a credencial correta:

[SUCCESS] - HOST: 192.168.56.102 (ftp) USER: msfadmin PASS: msfadmin
D. Recomendações de Mitigação:

Senhas Fortes: Nunca use credenciais padrão. Implementar uma política de senhas robusta que exija complexidade e rotação periódica.

Account Lockout: Implementar ferramentas como fail2ban no servidor para banir automaticamente endereços IP que falhem na autenticação múltiplas vezes.

Protocolos Seguros: Desabilitar o FTP. Substituí-lo por protocolos criptografados como SFTP (SSH File Transfer Protocol) ou FTPS (FTP over SSL/TLS).

Princípio do Menor Privilégio: Se o FTP for absolutamente necessário, garantir que o usuário de login tenha permissões restritas (ex: chroot jail) e não possa executar comandos no sistema.

Cenário B: Força Bruta em Formulário Web (DVWA)
Muitos ataques de força bruta visam formulários de login em aplicações web.

A. Reconhecimento: Acessamos o DVWA (http://192.168.56.102/dvwa/login.php) e definimos a segurança como "Low" para o teste. Inspecionando o código-fonte da página de login, identificamos os parâmetros do formulário:

URL da Ação (POST): login.php

Campo de Usuário: username

Campo de Senha: password

Botão de Envio: Login

Mensagem de Falha: "Login failed"

B. Execução (Medusa): O Medusa possui um módulo web-form que pode ser customizado:

Bash

medusa -h 192.168.56.102 -U wordlists/users.txt -P wordlists/pass.txt -M web-form \
-m FORM:"/dvwa/login.php" \
-m PARAMS:"username=^USER^&password=^PASS^&Login=Login" \
-m DENY-SIGNAL:"Login failed"
-U e -P: Listas de usuários e senhas.

-m FORM: A página que recebe o POST.

-m PARAMS: O payload do formulário. ^USER^ e ^PASS^ são substituídos pelo Medusa.

-m DENY-SIGNAL: A string que indica uma falha de login.

C. Resultado: O Medusa testou as combinações e encontrou a credencial válida:

[SUCCESS] - HOST: 192.168.56.102 (web-form) USER: admin PASS: password
D. Recomendações de Mitigação:

Bloqueio de Contas: Implementar uma política que bloqueie a conta do usuário (ou o IP do atacante) após um número baixo de tentativas falhas (ex: 5 tentativas).

CAPTCHA: Utilizar CAPTCHA ou reCAPTCHA no formulário de login para impedir a automação por bots.

Autenticação de Múltiplos Fatores (MFA): A medida mais eficaz. Mesmo que a senha seja descoberta, o atacante não terá o segundo fator (ex: token, código de app).

Monitoramento e Alertas: Gerar alertas para picos de falhas de login, permitindo uma resposta rápida da equipe de segurança.

Cenário C: Password Spraying em SMB
Diferente da força bruta (1 usuário, N senhas), o Password Spraying (N usuários, 1 senha) é uma técnica furtiva para evitar o bloqueio de contas.

A. Reconhecimento (Enumeração de Usuários): Serviços como SMB (Server Message Block) podem vazar nomes de usuários. Ferramentas como enum4linux podem ser usadas para extrair uma lista de usuários válidos do sistema.

Bash

enum4linux -U 192.168.56.102
Resultado: Obtivemos uma lista de usuários, incluindo msfadmin, user, service, etc., que salvamos em wordlists/smb-users.txt.

B. Execução (Medusa): Simulamos um spray com uma senha comum (ex: "password") contra a lista de usuários descoberta:

Bash

medusa -h 192.168.56.102 -U wordlists/smb-users.txt -p 'msfadmin' -M smbnt
-U: Lista de usuários que descobrimos.

-p: Senha única que estamos "borrifando".

-M: Módulo smbnt (para SMB no Windows/Samba).

C. Resultado: A ferramenta identificou que o usuário msfadmin possuía a senha msfadmin.

[SUCCESS] - HOST: 192.168.56.102 (smbnt) USER: msfadmin PASS: msfadmin
D. Recomendações de Mitigação:

Políticas de Senha Fortes: Proibir senhas óbvias, comuns (como "password", "123456") ou sazonais (como "Verao2024!").

Monitoramento de Padrões: A defesa contra password spraying é difícil. Requer monitoramento avançado que detecte falhas de login em múltiplas contas originadas do mesmo IP em um curto período.

Endurecimento (Hardening): Restringir o acesso ao SMB (Porta 445) apenas a IPs confiáveis através de regras de firewall. Desabilitar o SMBv1, que é notoriamente inseguro.

Não vazar nomes de usuários: Configurar serviços (como SMB e SMTP) para não enumerar ou confirmar nomes de usuários válidos.

3. Arquivos de Exemplo (Wordlists)
Como parte do desafio, aqui estão as wordlists simples utilizadas.

wordlists/users.txt
root
admin
msfadmin
user
test
guest
wordlists/pass.txt
admin
root
password
123456
msfadmin
1234
qwerty
wordlists/smb-users.txt
(Lista obtida via enumeração na Fase C)

msfadmin
user
postgres
service
backup
4. Conclusões e Aprendizados
Ameaça Real: A facilidade e velocidade com que o Medusa obteve acesso a todos os serviços em "Low Security" demonstra o risco crítico que senhas fracas e configurações padrão representam.

Defesa em Camadas: Nenhuma medida de mitigação é suficiente sozinha. A segurança eficaz combina senhas fortes, monitoramento de falhas (como fail2ban), uso de protocolos seguros (SFTP), firewalls e ferramentas anti-automação (CAPTCHA/MFA).

Valor do Laboratório: Este ambiente controlado foi essencial para entender a perspectiva do atacante de forma segura e ética, o que é fundamental para construir defesas mais robustas.
