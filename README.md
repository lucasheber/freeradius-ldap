# Servidor freeRADIUS com autenticação LDAP

## Introdução

Este relatório foi feito como trabalho prático da disciplina de **Gerencia e Serviços de Internet** do Curso **Superior de Tecnologia em Sistemas para Internet** .  
Nele será mostrado como configurar um servidor `RADIUS` com autenticação em uma base `LDAP`.

E para demonstrar o uso faremos o uso de um _hotspot._

### Protocolo RADIUS

O _Remote Authentication Dial In User Service_ \(RADIUS\) é um protocolo que é utilizado para gerenciar o acesso a diversos serviços de rede. O protocolo RADIUS define um padrão para ser utilizado na troca de informações entre um _Network Access Server_ \(NAS\) e um servidor de autenticação, autorização e informações de gerenciamento de contas \(auditoria\), ou também conhecido como servidor de _Authentication, Authorization e Accounting_ \(AAA\).

As principais vantagens na utilização do protocolo RADIUS, dentre uma série de funcionalidades que tornam este tipo de protocolo eficiente, são as seguintes:

* **Segurança**: As transferências de informações entre o cliente \(NAS\) e o servidor RADIUS são autenticadas através de um segredo compartilhado \(_shared secret_\). Este segredo é conhecido previamente, tanto pelo NAS quanto pelo servidor RADIUS e garante a autenticidade do usuário em uma determinada requisição.
* **Compatibilidade**: O servidor RADIUS pode utilizar um banco de dados de usuários de fontes externas para realizar a autenticação dos usuários, como por exemplo, banco de dados _Structured Query Language_ \(SQL\), Kerberos ou LDAP. 

## Instalando o freeRADIUS

Para instalar os servidor RADIUS digite os comandos abaixo:

```text
$ sudo install freeradius freeradius-ldap freeradius-utils
```

Em seguida iremos fazer uma configuração para testarmos se o servidor está funcionando.   
O servidor `RADIUS` oferece um arquivo de usuários, _plain text,_  por padrão a autenticação dos usuários é feita nele.  Vamos abri-lo:

```text
$ sudo vim /etc/raddb/users
```

Em seguida iremos descomentar as seguintes linhas:

```text
# The canonical testing user which is in most of the
# examples.
#
bob     Cleartext-Password := "hello"
        Reply-Message := "Hello, %{User-Name}"

```

A linha `bob Cleartext-Password := "hello"` representa a criação de um usuário `bob` com a senha em texto puro `hello` .  
Na linha `Reply-Message := "Hello, %{User-Name}"` indica que quando o usuário for autenticado irá mostrar uma mensagem `Hello` e `%{User-Name}` o nome do usuário.

Em seguida iremos definir a senha compartilhada para o cliente NAS se comunicar com o servidor `RADIUS`: 

```text
$ sudo vim /etc/raddb/clients.conf
```

Iremos editar a variável `secret` dentro do cliente `localhost` na linha `secret = testing123` podemos definir qualquer senha por exemplo: `secret = mySecretNAS`

Para testarmos a autenticação iremos inicializar o `freeRADIUS` em modo _debug_:

```text
$ sudo radiusd -X
```

{% hint style="info" %}
Faça isso em um terminal separado para vermos o resultado quando fizermos a autenticação logo a seguir, se tudo estiver certo irá aparacer uma mensagem no final`Ready to process requests`
{% endhint %}

```text
Listening on auth address * port 1812 bound to server default
Listening on acct address * port 1813 bound to server default
Listening on auth address :: port 1812 bound to server default
Listening on acct address :: port 1813 bound to server default
Listening on auth address 127.0.0.1 port 18120 bound to server inner-tunnel
Listening on proxy address * port 53202
Listening on proxy address :: port 47206
Ready to process requests

```

### Testando o freeRADIUS

Para testarmos basta executarmos o seguinte comando:

```text
$ sudo radtest bob hello 127.0.0.1 0 mySecretNAS
```

O comando `radtest` funciona basicamente da seguinte maneira:

```text
radtest <user> <password> <radius-server> <nas-port-number> <secret-shared>
```

O `user`e `password`definimos no arquivo `/etc/raddb/users`  , o `radius-server` é a nossa própria máquina, o `nas-port-number` seria a porta de autenticação do cliente, em nossa configuração não iremos mexer nela, por padrão colocamos `0`, e a `secret-shared` é a senha compartilhada entre o servidor e o cliente.

A saída do comando será a seguinte:

```text
[root@radius ~]# radtest bob hello 127.0.0.1 0 mySecretNAS
Sent Access-Request Id 42 from 0.0.0.0:58587 to 127.0.0.1:1812 length 73
        User-Name = "bob"
        User-Password = "hello"
        NAS-IP-Address = 192.168.0.101
        NAS-Port = 0
        Message-Authenticator = 0x00
        Cleartext-Password = "hello"
Received Access-Accept Id 42 from 127.0.0.1:1812 to 0.0.0.0:0 length 32
        Reply-Message = "Hello, bob"
[root@radius ~]#
```

No terminal que está rodando o `freeRADIUS` em modo debug podemos ver os logs:

```text
(0)   Auth-Type PAP {
(0) pap: Login attempt with password
(0) pap: Comparing with "known good" Cleartext-Password
(0) pap: User authenticated successfully
(0)     [pap] = ok
(0)   } # Auth-Type PAP = ok
[....]
(0)   Reply-Message = "Hello, bob"
```

{% hint style="danger" %}
Se por acaso a senha do usuário `bob` estiver incorreta ele iria mostrar a seguinte mensagem entre os logs:

`pap: ERROR: Cleartext password "helllo" does not match "known good" password`
{% endhint %}

## Configurando o modúlo LDAP

Primeiramente vamos adicinonar um link do módulo **LDAP** para o diretório com os módulos carregados pelo `freeRADIUS.`

```text
$ sudo cd /etc/raddb//mods-enabled/
$ sudo ln -s ../mods-available/ldap ./
```

Em seguida iremos iremos configurar o arquivo linkado para comunicar com a base `LDAP`

```text
$ sudo vim ldap
```

Iremos fazer as seguintes modificações:

```text
server = 'ldap://ldap1.labredes.info'
identity = 'cn=admin,dc=lucas,dc=labredes,dc=info'
password = <SUA_SENHA>
base_dn = 'ou=Usuarios,dc=lucas,dc=labredes,dc=info'
membership_filter = "(|(member=%{control:Ldap-UserDn})(memberUid=%{%{Stripped-User-Name}:-%{User-Name}}))"
```

A primeira configurção indica o IP ou DNS da máquina com a base LDAP.  
Em seguida passamos as configurações do de autenticação do `admin` e a `senha` de acesso.   
A linha `base_dn` indica o caminho que contém os usuários.  
E por último a regra de filtro, para fazer a busca, essa linha basta descomenta-la.

### Habilitando a autenticação LDAP

Com o arquivo [ldap](./#configurando-o-modulo-ldap) configurado, precisamos habilitar essa aunteticação. Iremos modificar os seguites arquivos:

```text
$ sudo vim /etc/raddb/sites-enabled/default
```

Dentro da sessão authorize iremos remover o `-` \(traço\) da linha `ldap`

```text
#
#  The ldap module reads passwords from the LDAP database.
ldap
```

Em seguida, na sessão `authenticate` iremos descomentar as linhas abaixo:

```text
Auth-Type LDAP {
     ldap
}
```

Faça o mesmo para o arquivo `/etc/raddb/sites-enabled/inner-tunnel` acrescentando a seguinte modificação: Comente a linha dentro da seção `authorize`

```text
#files
```

Pronto! Nossos arquivos de autenticação e confgiuração com a base `LDAP` estão configurados! Para testarmos, basta fazer os mesmo teste feito [anteriormente](./#testando-o-freeradius), mas neste caso passando o usuario e senha da base `LDAP`.

```text
$ sudo radtest marlon marlon 127.0.0.1 0 mySecretNAS
```

E o resultado será um `Access-Accept`

```text
Sent Access-Request Id 102 from 0.0.0.0:33244 to 192.168.11.12:1812 length 76
        User-Name = "marlon"
        User-Password = "marlon"
        NAS-IP-Address = 192.168.11.12
        NAS-Port = 0
        Message-Authenticator = 0x00
        Cleartext-Password = "marlon"
Received Access-Accept Id 102 from 192.168.11.12:1812 to 0.0.0.0:0 length 20
```

## Configuração para Hotspot

No roteador wifi, a aunteticação é feita via EAP-PEAP, para que o freeRADIUS consiga identificar a senha é preciso fazer uma configurção a mais. Abra o arquivo `/etc/raddb/mods-enabled/eap`, e na sessão `peap`altere a seguinte linha:

De: 

```text
default_eap_type = mschapv2
```

Para:

```text
default_eap_type = gtc
```

Feito isso basta testar a autenticação em dispositivo com `wifi.`

