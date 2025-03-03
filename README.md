# Redes de Computadores - DNS
Nesse trabalho, vamos configurar um servidor DNS no linux (WSL) usando o programa BIND9.
## Instalação do Windows Subsystem for Linux (WSL)
### Requisitos
- Windows 10 versão 2004 e superior (Build 19041 e superior) ou Windows 11.

### Passo a Passo
1. Ativar o WSL
    1. Abra o PowerShell como Administrador.
    2. Clique com o botão direito do mouse no ícone do menu Iniciar e selecione "Windows PowerShell (Admin)".
    3. Execute o comando abaixo para ativar o WSL:
        ```
        wsl --install
        ```
        _Esse comando instalará automaticamente o WSL 2 e a distribuição padrão do Ubuntu._

2. Reiniciar o Computador
    - Reinicie o computador para aplicar as alterações.


3. Configurar o WSL 2 como Padrão (Opcional)
    - Caso queira configurar o WSL 2 como a versão padrão, execute o seguinte comando no PowerShell:
        ```
        wsl --set-default-version 2
        ```
4. Instalar uma Distribuição Linux
    - Abra a Microsoft Store, procure por uma distribuição Linux (por exemplo, Ubuntu, Debian, Kali Linux) e clique em Instalar.
5. Configurar a Distribuição Linux
    - Após a instalação, abra a distribuição Linux (por exemplo, Ubuntu) a partir do menu Iniciar.
    - Siga as instruções na tela para configurar o seu usuário e senha.

## Configuração do BIND9
Antes de começar o passo, você vai precisar saber o seu endereço IP para configurar o DNS.
Para isso, rode:
```
ip a
```

No meu caso, o IP era 192.168.1.70/24, mas o seu vai ser diferente, então **siga o passo a passo de acordo com o seu**!

Agora sim, vamos configurar o BIND9 de fato.

### Passo a passo
1. Instalar o BIND9
    ```
    sudo apt-get install bind9
    ```
    _Para testar se foi instalado corretamente, rodar `sudo systemctl status bind9`_

2. Entrar no diretório de configuração
    ```
    cd /etc/bind
    ls
    ```
    Você verá o **named.conf** que é o arquivo de configurações do BIND.
    Ele chama os seguintes arquivos:
      - include "/etc/bind/named.conf.options";
      - include "/etc/bind/named.conf.local";
      - include "/etc/bind/named.conf.default-zones";

3. Configurar arquivo **named.conf.options**<br>
    Antes de alterar qualquer arquivo, fazer uma cópia de segurança
    ```
    sudo cp named.conf.options named.conf.options_copy
    ```

    Agora sim:
    ```
    sudo nano named.conf.options
    ```

    Adicionar as seguintes configurações no final do arquivo:
    ```
    listen-on-v6 { none; };
    listen-on port 53 { localhost; 192.168.1.0/24; };
    allow-query { localhost; 192.168.1.0/24; };
    forwarders { 8.8.8.8; };
    recursion yes;
    ```

    **Lembre-se de salvar as alterações!**

    Explicação:
    - `listen-on-v6 { none; };` Não vamos usar IPv6
    - `listen-on port 53 { localhost; 192.168.1.0/24; };` Vamos ouvir somente nas interfaces loopback e rede local
    - `allow-query { locahost; 192.168.1.0/24; };` Aceitar requisições somente das interfaces loopback e rede local
    - `forwarders { 8.8.8.8; };` Caso não saiba resolver, encaminhar para outro DNS (no caso, para o da Google)
    - `recursion yes;` permite requisições recursivas (requisitar a outro DNS atuando em nome do cliente)

4. Configurar arquivo **named.conf.local** <br>
    Entrar no arquivo
    ```
    sudo nano named.conf.local
    ```

    Adicionar as seguintes configurações no arquivo:
    ```
    zone "labredes.teste" {
        type master;
        file "/etc/bind/forward.labredes.teste";
    };
    zone "1.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/reverse.labredes.teste";
    };
    ```

5. Criar e configurar arquivo **forawrd.labredes.teste**
    Para facilitar, vamos fazer uma cópia do **db.local**
    ```
    sudo cp db.local forward.labredes.teste
    ```

    Entrar no arquivo
    ```
    sudo nano forward.labredes.teste
    ```

    Alterar o `localhost. root.localhost.` para `primario.labredes.teste x.labredes.teste.`.<br>
    E alterar o final do arquivo de
    ```
    @               IN      NS      localhost.
    @               IN      A       127.0.0.1
    @               IN      AAAA    ::1
    ```
    Para
    ```
    @               IN      NS      primario.labredes.teste.
    primario        IN      A       192.168.1.70
    labredes.teste. IN      MX      10 mail.labredes.teste.
    www             IN      A       192.168.1.70
    ```

6. Criar e configurar arquivo **reverse.labredes.teste**
    Para facilitar, vamos fazer uma cópia do **db.127**
    ```
    sudo cp db.127 reverse.labredes.teste
    ```

    Entrar no arquivo
    ```
    sudo nano reverse.labredes.teste
    ```

    Alterar o `localhost. root.localhost.` para `primario.labredes.teste x.labredes.teste.`.<br>
    E alterar o final do arquivo de
    ```
    @               IN      NS      localhost.
    @               IN      PTR     localhost.
    ```
    Para
    ```
    @        IN     NS      primario.labredes.teste.
    primario IN     A       192.168.1.70
    70       IN     PTR     primario.labredes.teste.
    70       IN     PTR     www.labredes.teste.
    ```

7. Reiniciar o servidor
    Restart do bind9
    ```
    sudo systemctl restart bind9
    ```

    Habilitar o bind9
    ```
    sudo systemctl enable bind9
    ```

8. Verificar erros
    Checar erros de sintaxe
    ```
    sudo named-checkconf /etc/bind/named.conf.local
    ```

    Checar a zona direta
    ```
    sudo named-checkzone labredes.teste /etc/bind/forward.labredes.teste
    ```

    Checar a zona reversa
    ```
    sudo named-checkzone labredes.teste /etc/bind/reverse.labredes.teste
    ```

9. Mudar o serviço de DNS
    Para definirmos qual DNS queremos usar, precisamos informar seu IP ao sistema operacional. <br>
    Por se tratar de um DNS configurado na própria máquina, podemos utilizar o 127.0.0.1. <br>

    Para isso, rode
    ```
    sudo nano /etc/resolv.conf
    ```
    Haverá um nameserver x.x.x.x no seu arquivo, adicione um # nessa linha e adicione uma nova linha
    ```
    nameserver 127.0.0.1
    ```
    Isso definirá o nosso DNS como serviço principal.

## Teste
Agora, podemos testar usando o comando **nslookup** por exemplo.

Para consultar informações do servidor
```
nslookup primario.labredes.teste
```

Você deveria ver algo assim:
```
Server:         127.0.0.1
Address:        127.0.0.1#53

Name:   primario.labredes.teste
Address: 192.168.1.70
```

Ou para consultas reversas
```
nslookup 192.168.1.70
```

Saída
```
70.1.168.192.in-addr.arpa       name = primario.labredes.teste.
70.1.168.192.in-addr.arpa       name = www.labredes.teste.
```