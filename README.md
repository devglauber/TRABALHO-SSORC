# Autenticação de Usuários no Squid Proxy usando LDAP

## Trabalho de SSORC
**Dupla:** Glauber De Melo Duarte e Eryka Rodrigues  
**Professor:** Dr. Juan Sebastian  

## Passo a Passo

### 1. Instalar o Squid
#### Debian/Ubuntu:
```bash
sudo apt update && sudo apt install squid -y
```

Após instalar, verifique se está rodando corretamente:
```bash
sudo systemctl status squid
```

---
### 2. Instalando e Configurando o OpenLDAP
Agora vamos instalar o OpenLDAP e configurá-lo:
```bash
sudo apt-get update
sudo apt-get install slapd ldap-utils -y
```
Durante a instalação, defina uma senha forte para o administrador do LDAP (usuário `admin` do diretório).

#### Após instalar aplique o comando para configurar o LDAP
```bash
sudo dpkg-reconfigure slapd
```
#### Configuração:
- **Omitir configuração do Banco de Dados LDAP?** → **Não**
- **Nome de Domínio Base:** `meu-dominio.local` (ou substitua pelo seu domínio real)
- **Nome da Organização:** `MinhaEmpresa`
- **Senha do administrador LDAP:** Mesma senha da instalação
- **Banco de dados:** **MDB**
- **Remover banco de dados ao remover o slapd?** → **Não**
- **Habilitar protocolo LDAPv2?** → **Não**

---
### 3. Criando a Base de Usuários no LDAP
Agora vamos adicionar usuários para autenticação no Squid.

Crie um arquivo chamado `base.ldif`:
```bash
sudo nano base.ldif
```

Adicione o seguinte conteúdo:
```ldif
dn: dc=meu-dominio,dc=local
objectClass: top
objectClass: dcObject
objectClass: organization
o: MinhaEmpresa
dc: meu-dominio

dn: ou=usuarios,dc=meu-dominio,dc=local
objectClass: organizationalUnit
ou: usuarios
```

Salve e execute:
```bash
sudo ldapadd -x -D "cn=admin,dc=meu-dominio,dc=local" -W -f base.ldif
```

#### 3.1 Criando Usuários para o Squid
Crie um arquivo `usuarios.ldif`:
```bash
sudo nano usuarios.ldif
```

Adicione o seguinte conteúdo:
```ldif
dn: uid=teste,ou=usuarios,dc=meu-dominio,dc=local
objectClass: top
objectClass: account
objectClass: posixAccount
cn: teste
uid: teste
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/teste
loginShell: /bin/bash
userPassword: GERAR PELO LDAP
```

Salve e aplique:
```bash
sudo ldapadd -x -D "cn=admin,dc=meu-dominio,dc=local" -W -f usuarios.ldif
```

---
### 4. Configurando o Squid para Autenticação LDAP
Agora que o LDAP está configurado, vamos integrá-lo com o Squid.

Abra o arquivo de configuração do Squid:
```bash
sudo nano /etc/squid/squid.conf
```

Adicione a seguinte configuração:
```bash
# Configuração do LDAP no Squid
auth_param basic program /usr/lib/squid/basic_ldap_auth -R -b "dc=meu-dominio,dc=local" \
-D "cn=admin,dc=meu-dominio,dc=local" -w "senha_do_admin" -f "uid=%s" -h IP_DO_SERVIDOR_SQUID

# Mensagem ao usuário ao solicitar credenciais
auth_param basic realm "Digite seu usuário e senha"

# Tempo de cache da autenticação
auth_param basic credentialsttl 2 hours

# Requer autenticação para navegação
acl usuarios_autenticados proxy_auth REQUIRED
http_access allow usuarios_autenticados

# Bloqueia qualquer outro acesso sem autenticação
http_access deny all
```

Salve o arquivo.

---
### 5. Reiniciando e Testando
Reinicie os serviços:
```bash
sudo systemctl restart slapd
sudo systemctl restart squid
```

Agora configure o navegador e teste a autenticação!

---
### 📌 Observações
- Certifique-se de substituir `IP_DO_SERVIDOR_SQUID` pelo IP correto do servidor.
- Caso precise adicionar mais usuários, basta criar novos registros `usuarios.ldif`.
- Para depurar erros, verifique os logs do Squid e do LDAP:
  ```bash
  sudo journalctl -u squid --no-pager | tail -n 50
  sudo journalctl -u slapd --no-pager | tail -n 50
  ```

