---

copyright:
  years: 2014, 2018
lastupdated: "2018-12-05"

---

{:new_window: target="_blank"}
{:shortdesc: .shortdesc}
{:screen: .screen}
{:pre: .pre}
{:table: .aria-labeledby="caption"}
{:codeblock: .codeblock}
{:tip: .tip}
{:note: .note}
{:important: .important}
{:deprecated: .deprecated}
{:download: .download}


# Configurando a conectividade de VPN
{: #vpn}

Com a conectividade de VPN, é possível conectar apps com segurança em um cluster do Kubernetes no {{site.data.keyword.containerlong}} a uma rede no local. Também é possível conectar apps que são externos ao seu cluster para um app que está em execução dentro de seu cluster.
{:shortdesc}

Para conectar seus nós do trabalhador e apps a um data center no local, é possível configurar uma das opções a seguir.

- **Serviço de VPN IPSec strongSwan**: é possível configurar um [Serviço de VPN IPSec strongSwan ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://www.strongswan.org/about.html) que conecte de forma segura seu cluster do Kubernetes a uma rede no local. O serviço de VPN do IPSec do strongSwan fornece um canal de comunicação seguro de ponta a ponta sobre a Internet que é baseado no conjunto de protocolos padrão de mercado da Internet Protocol Security (IPSec). Para configurar uma conexão segura entre seu cluster e uma rede no local, [configure e implemente o serviço VPN IPSec do strongSwan](#vpn-setup) diretamente em um pod no cluster.

- **Virtual Router Appliance (VRA) ou Fortigate Security Appliance (FSA)**: você pode escolher configurar um [VRA](/docs/infrastructure/virtual-router-appliance/about.html) ou [FSA](/docs/infrastructure/fortigate-10g/about.html) para configurar um terminal de VPN IPSec. Essa opção é útil quando você tem um cluster maior, deseja acessar múltiplos clusters por meio de uma única VPN ou precisa de uma VPN baseada em rota. Para configurar um VRA, veja [Configurando a conectividade VPN com o VRA](#vyatta).

## Usando o gráfico Helm do serviço de VPN IPSec strongSwan
{: #vpn-setup}

Use um gráfico Helm para configurar e implementar o serviço de VPN IPSec strongSwan dentro de um pod do Kubernetes.
{:shortdesc}

Como o strongSwan está integrado ao cluster, não é necessário um dispositivo de gateway externo. Quando a conectividade de VPN é estabelecida, as rotas são configuradas automaticamente em todos os nós do trabalhador no cluster. Essas rotas permitem a conectividade bidirecional por meio do túnel VPN entre pods em qualquer nó do trabalhador e o sistema remoto. Por exemplo, o diagrama a seguir mostra como um app no {{site.data.keyword.containerlong_notm}} pode se comunicar com um servidor local por meio de uma conexão VPN do strongSwan:

<img src="images/cs_vpn_strongswan.png" width="700" alt="Expor um app no {{site.data.keyword.containerlong_notm}} usando um balanceador de carga" style="width:700px; border-style: none"/>

1. Um app em seu cluster, `myapp`, recebe uma solicitação de um serviço Ingress ou LoadBalancer e precisa conectar-se com segurança a dados em sua rede local.

2. A solicitação para o data center local é encaminhada para o pod VPN do strongSwan do IPSec. O endereço IP de destino é usado para determinar quais pacotes de rede enviar para o pod VPN do strongSwan do IPSec.

3. A solicitação é criptografada e enviada pelo túnel VPN para o data center no local.

4. A solicitação recebida passa pelo firewall no local e é entregue ao terminal de túnel VPN (roteador) no qual ela é decriptografada.

5. O terminal de túnel VPN (roteador) encaminha a solicitação para o servidor ou mainframe no local, dependendo do endereço IP de destino que foi especificado na etapa 2. Os dados necessários são enviados de volta na conexão VPN para `myapp` por meio do mesmo processo.

## Considerações sobre o serviço de VPN strongSwan
{: strongswan_limitations}

Antes de usar o gráfico Helm do strongSwan, revise as considerações e limitações a seguir.
{: shortdesc}

* O gráfico Helm do strongSwan requer que a passagem NAT seja ativada pelo terminal de VPN remoto. A passagem NAT requer a porta UDP 4500, além da porta UDP IPSec padrão de 500. Ambas as portas UDP precisam ser permitidas por meio de qualquer firewall que esteja configurado.
* O gráfico Helm do strongSwan não suporta VPNs IPSec baseadas em rota.
* O gráfico Helm do strongSwan suporta VPNs IPSec que usam chaves pré-compartilhadas, mas não suportam VPNs IPSec que requerem certificados.
* O gráfico Helm do strongSwan não permite que múltiplos clusters e outros recursos IaaS compartilhem uma única conexão VPN.
* O gráfico Helm do strongSwan é executado como um pod do Kubernetes dentro do cluster. O desempenho da VPN é afetado pelo uso de memória e de rede do Kubernetes e outros pods que estiverem em execução no cluster. Se você tiver um ambiente de desempenho crítico, considere usar uma solução de VPN que seja executada fora do cluster no hardware dedicado.
* O gráfico Helm do strongSwan executa um único pod de VPN como o terminal de túnel IPSec. Se o pod falhar, o cluster reiniciará o pod. No entanto, você pode experienciar um curto tempo de inatividade enquanto o novo pod é iniciado e a conexão VPN é restabelecida. Se você precisar de uma recuperação de erro mais rápida ou de uma solução de alta disponibilidade mais elaborada, considere usar uma solução de VPN que seja executada fora do cluster no hardware dedicado.
* O gráfico Helm do strongSwan não fornece métricas ou monitoramento do tráfego de rede que flui por meio da conexão VPN. Para obter uma lista de ferramentas de monitoramento suportadas, consulte [Serviços de criação de log e monitoramento](cs_integrations.html#health_services).

## Configurando o gráfico Helm do strongSwan
{: #vpn_configure}

Antes de instalar o gráfico Helm do strongSwan, deve-se decidir sobre a configuração do strongSwan. 
{: shortdesc}

Antes de iniciar:
* [Instale um gateway de VPN IPSec em seu data center no local](/docs/infrastructure/iaas-vpn/set-up-ipsec-vpn.html#setting-up-an-ipsec-connection).
* [Efetue login em sua conta. Destine a região apropriada e, se aplicável, o grupo de recursos. Configure o contexto para seu cluster](cs_cli_install.html#cs_cli_configure).

### Etapa 1: obter o gráfico Helm do strongSwan
{: #strongswan_1}

Instale o Helm e obterá o gráfico Helm do strongSwan para visualizar as configurações possíveis. 
{: shortdesc}

1. [Instale o Helm para seu cluster e inclua o repositório do {{site.data.keyword.Bluemix_notm}} em sua instância do Helm](cs_integrations.html#helm).

2. Salve as definições de configuração padrão para o gráfico Helm do strongSwan em um arquivo YAML local.

    ```
    helm inspect values ibm/strongswan > config.yaml
    ```
    {: pre}

3. Abra o arquivo  ` config.yaml ` .

### Etapa 2: Configurar definições básicas de IPSec
{: #strongswan_2}

Para controlar o estabelecimento da conexão VPN, modifique as configurações básicas do IPSec a seguir.
{: shortdesc}

Para obter mais informações sobre cada configuração, leia a documentação fornecida no arquivo `config.yaml` para o gráfico Helm.
{: tip}

1. Se o terminal do túnel VPN no local não suportar `ikev2` como um protocolo para inicializar a conexão, mude o valor de `ipsec.keyexchange` para `ikev1`.
2. Configure `ipsec.esp` para uma lista de algoritmos de criptografia e de autenticação ESP que seu terminal de túnel VPN no local usa para a conexão.
    * Se `ipsec.keyexchange` estiver configurado como `ikev1`, esta configuração deverá ser especificada.
    * Se `ipsec.keyexchange` estiver configurado como `ikev2`, esta configuração será opcional.
    * Se você deixar essa configuração em branco, os algoritmos padrão do strongSwan `aes128-sha1,3des-sha1` serão usados para a conexão.
3. Configure `ipsec.ike` para uma lista de algoritmos de criptografia e autenticação IKE/ISAKMP SA que seu terminal de túnel VPN no local usa para a conexão. Os algoritmos devem ser específicos no formato `encryption-integrity[-prf]-dhgroup`.
    * Se `ipsec.keyexchange` estiver configurado como `ikev1`, esta configuração deverá ser especificada.
    * Se `ipsec.keyexchange` estiver configurado como `ikev2`, esta configuração será opcional.
    * Se você deixar essa configuração em branco, os algoritmos padrão do strongSwan `aes128-sha1-modp2048,3des-sha1-modp1536` serão usados para a conexão.
4. Mude o valor de `local.id` para qualquer sequência que você deseja usar para identificar o lado do cluster do Kubernetes local que seu terminal de túnel VPN usa. O padrão é  ` ibm-cloud `. Algumas implementações de VPN requerem que você use o endereço IP público para o endpoint local.
5. Mude o valor de `remote.id` para qualquer sequência que você deseja usar para identificar o lado no local remoto que seu terminal de túnel VPN usa. O padrão é  ` on-prem `. Algumas implementações de VPN requerem que você use o endereço IP público para o terminal remoto.
6. Mude o valor de `preshared.secret` para o segredo pré-compartilhado que seu gateway do terminal de túnel VPN no local usa para a conexão. Esse valor é armazenado em  ` ipsec.secrets `.
7. Opcional: configure `remote.privateIPtoPing` para qualquer endereço IP privado na sub-rede remota para executar ping como parte do teste de validação de conectividade do Helm.

### Etapa 3: selecionar a conexão VPN de entrada ou de saída
{: #strongswan_3}

Ao configurar uma conexão VPN strongSwan, você escolhe se a conexão VPN é de entrada para o cluster ou de saída do cluster.
{: shortdesc}

<dl>
<dt>Entrada</dt>
<dd>O terminal de VPN no local da rede remota inicia a conexão VPN e o cluster atende a conexão.</dd>
<dt>Saída</dt>
<dd>O cluster inicia a conexão VPN e o terminal de VPN no local da rede remota atende a conexão.</dd>
</dl>

Para estabelecer uma conexão VPN de entrada, modifique as configurações a seguir:
1. Verifique se `ipsec.auto` está configurado como `add`.
2. Opcional: configure `loadBalancerIP` para um endereço IP público móvel para o serviço de VPN strongSwan. Especificar um endereço IP é útil quando você precisa de um endereço IP estável, como quando se deve designar quais endereços IP são permitidos por meio de um firewall no local. O cluster deve ter pelo menos um endereço IP público disponível do balanceador de carga. [É possível verificar seus endereços IP públicos disponíveis](cs_subnets.html#review_ip) ou [liberar um endereço IP usado](cs_subnets.html#free).
    * Se você deixar essa configuração em branco, um dos endereços IP públicos móveis disponíveis será usado.
    * Deve-se também configurar o endereço IP público que você seleciona ou o endereço IP público que está designado ao terminal de VPN do cluster no terminal de VPN no local.

Para estabelecer uma conexão VPN de saída, modifique as configurações a seguir:
1. Altere  ` ipsec.auto `  para  ` start `.
2. Configure `remote.gateway` para o endereço IP público para o terminal de VPN no local na rede remota.
3. Escolha uma das opções a seguir para o endereço IP para o terminal de VPN do cluster:
    * **Endereço IP público do gateway privado do cluster**: se seus nós do trabalhador estiverem conectados a somente uma VLAN privada, a solicitação de VPN de saída será roteada por meio do gateway privado para atingir a Internet. O endereço IP público do gateway privado é usado para a conexão VPN.
    * **Endereço IP público do nó do trabalhador no qual o pod strongSwan está em execução**: se o nó do trabalhador no qual o pod strongSwan está em execução for conectado a uma VLAN pública, o endereço IP público do nó do trabalhador será usado para a conexão VPN.
        <br>
        * Se o pod strongSwan for excluído e reprogramado em um nó do trabalhador diferente no cluster, o endereço IP público da VPN mudará. O terminal de VPN no local da rede remota deve permitir que a conexão VPN seja estabelecida por meio do endereço IP público de qualquer um dos nós do trabalhador do cluster.
        * Se o terminal de VPN remoto não puder manipular conexões VPN de múltiplos endereços IP públicos, limite os nós em que o pod de VPN strongSwan é implementado. Configure `nodeSelector` para os endereços IP de nós do trabalhador específicos ou um rótulo do nó do trabalhador. Por exemplo, o valor `kubernetes.io/hostname: 10.232.xx.xx` permite que o pod de VPN seja implementado somente para esse nó do trabalhador. O valor `strongswan: vpn` restringe o pod de VPN à execução em quaisquer nós do trabalhador com esse rótulo. É possível usar qualquer rótulo de nó do trabalhador. Para permitir que diferentes nós do trabalhador sejam usados com diferentes implementações do gráfico Helm, use `strongswan: <release_name>`. Para alta disponibilidade, selecione pelo menos dois nós do trabalhador.
    * **Endereço IP público do serviço strongSwan**: para estabelecer uma conexão usando o endereço IP do serviço de VPN strongSwan, configure `connectUsingLoadBalancerIP` como `true`. O endereço IP do serviço strongSwan é um endereço IP público móvel que pode ser especificado na configuração `loadBalancerIP` ou um endereço IP público móvel disponível que é designado automaticamente ao serviço.
        <br>
        * Se você escolher selecionar um endereço IP usando a configuração `loadBalancerIP`, o cluster deverá ter pelo menos um endereço IP público disponível do Load Balancer. [É possível verificar seus endereços IP públicos disponíveis](cs_subnets.html#review_ip) ou [liberar um endereço IP usado](cs_subnets.html#free).
        * Todos os nós do trabalhador do cluster devem estar na mesma VLAN pública. Caso contrário, deve-se usar a configuração `nodeSelector` para assegurar que o pod de VPN seja implementado em um nó do trabalhador na mesma VLAN pública que a do `loadBalancerIP`.
        * Se `connectUsingLoadBalancerIP` está configurado como `true` e `ipsec.keyexchange` está configurado como `ikev1`, deve-se configurar `enableServiceSourceIP` como `true`.

### Etapa 4: acessar recursos de cluster por meio da conexão VPN
{: #strongswan_4}

Determine quais recursos do cluster devem ser acessíveis pela rede remota por meio da conexão VPN.
{: shortdesc}

1. Inclua os CIDRs de uma ou mais sub-redes do cluster para a configuração `local.subnet`. Deve-se configurar os CIDRs de sub-rede local no terminal de VPN no local. Essa lista pode incluir as seguintes sub-redes:  
    * O CIDR da sub-rede do pod do Kubernetes:  ` 172.30.0.0/ 16 `. A comunicação bidirecional é ativada entre todos os pods do cluster e qualquer um dos hosts nas sub-redes de rede remota que você listar na configuração `remote.subnet`. Caso seja necessário evitar o acesso de qualquer host `remote.subnet` a pods do cluster por razões de segurança, não inclua a sub-rede de pod do Kubernetes na configuração `local.subnet`.
    * O CIDR de sub-rede de serviço do Kubernetes:  ` 172.21.0.0/ 16 `. Os endereços IP de serviço fornecem uma maneira de expor múltiplos pods de app que são implementados em vários nós do trabalhador por trás de um único IP.
    * Se seus apps forem expostos por um serviço NodePort na rede privada ou um ALB do Ingresso privado, inclua o CIDR de sub-rede privada do nó do trabalhador. Recupere os três primeiros octetos do endereço IP privado de seu trabalhador executando `ibmcloud ks worker <cluster_name>`. Por exemplo, se for `10.176.48.xx`, anote `10.176.48`. Em seguida, obtenha o CIDR da sub-rede privada do trabalhador executando o comando a seguir, substituindo `<xxx.yyy.zz>` pelo octeto que você recuperou anteriormente: `ibmcloud sl subnet list | grep <xxx.yyy.zzz>`. **Nota**: se um nó do trabalhador for incluído em uma nova sub-rede privada, deve-se incluir o CIDR da nova sub-rede privada na configuração `local.subnet` e no terminal de VPN no local. Em seguida, a conexão VPN deve ser reiniciada.
    * Se você tiver apps que são expostos pelos serviços LoadBalancer na rede privada, inclua os CIDRs de sub-rede privada gerenciada pelo usuário do cluster. Para localizar esses valores, execute `ibmcloud ks cluster-get <cluster_name> --showResources`. Na seção **VLANS**, procure os CIDRs que possuem um valor **público** de `false`. **Nota**: se `ipsec.keyexchange` estiver configurado como `ikev1`, será possível especificar apenas uma sub-rede. No entanto, é possível usar a configuração `localSubnetNAT` para combinar múltiplas sub-redes de cluster em uma única sub-rede.

2. Opcional: remapeie as sub-redes de cluster usando a configuração `localSubnetNAT`. A Conversão de endereço de rede (NAT) para sub-redes fornece uma solução alternativa para conflitos de sub-rede entre a rede de cluster e a rede remota no local. É possível usar o NAT para remapear as sub-redes de IP local privado, a sub-rede do pod (172.30.0.0/16) ou a sub-rede do serviço de pod (172.21.0.0/16) para uma sub-rede privada diferente. O túnel VPN vê as sub-redes de IP remapeadas em vez das sub-redes originais. O remapeamento acontece antes de os pacotes serem enviados pelo túnel VPN, bem como após os pacotes chegarem do túnel VPN. É possível expor as sub-redes remapeada e não remapeada ao mesmo tempo pela VPN. Para ativar o NAT, é possível incluir uma sub-rede inteira ou endereços IP individuais.
    * Se você incluir uma sub-rede inteira no formato `10.171.42.0/24=10.10.10.0/24`, o remapeamento será 1 para 1: todos os endereços IP na sub-rede de rede interna são mapeados para sub-rede de rede externa e vice-versa.
    * Se você incluir endereços IP individuais no formato `10.171.42.17/32=10.10.10.2/32,10.171.42.29/32=10.10.10.3/32`, somente os endereços IP internos serão mapeados para os endereços IP externos especificados.

3. Opcional para gráficos Helm do strongSwan da versão 2.2.0 e mais recente: oculte todos os endereços IP do cluster atrás de um único endereço IP configurando `enableSingleSourceIP` como `true`. Essa opção fornece uma das configurações mais seguras para a conexão VPN porque nenhuma conexão da rede remota de volta para o cluster é permitida.
    <br>
    * Essa configuração requer que todo fluxo de dados sobre a conexão VPN seja de saída, independentemente se a conexão VPN é estabelecida por meio do cluster ou da rede remota.
    * O `local.subnet` deve ser configurado para somente uma sub-rede /32.

4. Opcional para gráficos Helm do strongSwan da versão 2.2.0 e mais recente: ative o serviço strongSwan para rotear as solicitações recebidas da rede remota para um serviço que exista fora do cluster usando a configuração `localNonClusterSubnet`.
    <br>
    * O serviço sem cluster deve existir na mesma rede privada ou em uma rede privada que seja atingível pelos nós do trabalhador.
    * O nó do trabalhador sem cluster não pode iniciar o tráfego para a rede remota por meio da conexão VPN, mas o nó sem cluster pode ser o destino de solicitações recebidas da rede remota.
    * Deve-se listar os CIDRs das sub-redes sem cluster na configuração `local.subnet`.

### Etapa 5: acessar recursos de rede remota por meio da conexão VPN
{: #strongswan_5}

Determine quais recursos de rede remota devem ser acessíveis pelo cluster por meio da conexão VPN.
{: shortdesc}

1. Inclua os CIDRs de uma ou mais sub-redes privadas no local na configuração `remote.subnet`. **Nota**: se `ipsec.keyexchange` estiver configurado como `ikev1`, será possível especificar apenas uma sub-rede.
2. Opcional para gráficos Helm do strongSwan Helm da versão 2.2.0 e mais recente: remapeie sub-redes de rede remota usando a configuração `remoteSubnetNAT`. A Conversão de endereço de rede (NAT) para sub-redes fornece uma solução alternativa para conflitos de sub-rede entre a rede de cluster e a rede remota no local. É possível usar NAT para remapear as sub-redes de IP da rede remota para uma sub-rede privada diferente. O remapeamento ocorre antes de os pacotes serem enviados por meio do túnel VPN. Os pods no cluster veem as sub-redes de IP remapeadas em vez das sub-redes originais. Antes que os pods enviem dados de volta por meio do túnel VPN, a sub-rede IP remapeada é comutada de volta para a sub-rede real que está sendo usada pela rede remota. É possível expor as sub-redes remapeada e não remapeada ao mesmo tempo pela VPN.

### Etapa 6 (opcional): ative o monitoramento com a integração do webhook do Slack
{: #strongswan_6}

Para monitorar o status da VPN do strongSwan, é possível configurar um webhook para postar mensagens de conectividade VPN automaticamente para um canal Slack.
{: shortdesc}

1. Conecte-se a sua área de trabalho do Slack.

2. Acesse a [página do app WebHooks recebidos ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://slack.com/apps/A0F7XDUAZ-incoming-webhooks).

3. Clique em **Solicitar para instalar**. Se esse app não estiver listado em sua configuração do Slack, entre em contato com o proprietário da área de trabalho do Slack.

4. Depois que sua solicitação para instalar for aprovada, clique em **Incluir configuração**.

5. Escolha um canal do Slack ou crie um novo canal para o envio das mensagens VPN.

6. Copie a URL do webhook que é gerada. O formato da URL é semelhante ao seguinte:
  ```
  https://hooks.slack.com/services/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
  ```
  {: screen}

7. Para verificar se o webhook do Slack está instalado, envie uma mensagem de teste para a URL do webhook executando o comando a seguir:
    ```
    curl -X POST -H 'Content-type: application/json' -d '{"text":"VPN test message"}' <webhook_URL>
    ```
    {: pre}

8. Acesse o canal Slack que você escolheu para verificar se a mensagem de teste foi bem-sucedida.

9. No arquivo `config.yaml` para o gráfico Helm, configure o webhook para monitorar a sua conexão VPN.
    1. Mude `monitoring.enable` para `true`.
    2. Inclua endereços IP privados ou terminais HTTP na sub-rede remota que você deseja que sejam acessíveis por meio da conexão VPN com `monitoring.privateIPs` ou `monitoring.httpEndpoints`. Por exemplo, é possível incluir o IP da configuração `remote.privateIPtoPing` em `monitoring.privateIPs`.
    3. Inclua a URL do webhook no `monitoring.slackWebhook`.
    4. Mude outras configurações opcionais de `monitoring` conforme necessário.

### Etapa 7: implementar o gráfico Helm
{: #strongswan_7}

Implemente o gráfico Helm do strongSwan em seu cluster com as configurações que você escolheu anteriormente. 
{: shortdesc}

1. Se você precisar definir configurações mais avançadas, siga a documentação fornecida para cada configuração no gráfico Helm.

3. Salve o arquivo `config.yaml` atualizado.

4. Instale o gráfico do Helm em seu cluster com o arquivo `config.yaml` atualizado.

    Se você tiver múltiplas implementações VPN em um único cluster, será possível evitar conflitos de nomenclatura e diferenciar entre suas implementações, escolhendo nomes de liberação mais descritivos do que `vpn`. Para evitar o truncamento do nome de liberação, limite o nome de liberação para 35 caracteres ou menos.
    {: tip}

    ```
    -f config.yaml helm install -- name=vpn ibm/strongswan
    ```
    {: pre}

5. Verifique o status de implementação do gráfico. Quando o gráfico estiver pronto, o campo **STATUS** perto da parte superior da saída terá um valor de `DEPLOYED`.

    ```
    helm status vpn
    ```
    {: pre}

6. Quando o gráfico for implementado, verifique se as configurações atualizadas no arquivo `config.yaml` foram usadas.

    ```
    helm get values vpn
    ```
    {: pre}

## Testando e verificando a conectividade VPN do strongSwan
{: #vpn_test}

Depois de implementar seu gráfico Helm, teste a conectividade VPN.
{:shortdesc}

1. Se a VPN no gateway local não estiver ativo, inicie a VPN.

2. Configure a variável de ambiente `STRONGSWAN_POD`.

    ```
    export STRONGSWAN_POD=$(kubectl get pod -l app=strongswan,release=vpn -o jsonpath='{ .items[0].metadata.name }')
    ```
    {: pre}

3. Verifique o status da VPN. Um status de `ESTABLISHED` significa que a conexão VPN foi bem-sucedida.

    ```
    Kubectl exec $STRONGSWAN_POD -- ipsec status
    ```
    {: pre}

    Saída de exemplo:

    ```
    Security Associations (1 up, 0 connecting):
    k8s-conn[1]: ESTABLISHED 17 minutes ago, 172.30.xxx.xxx[ibm-cloud]...192.xxx.xxx.xxx[on-premises]
    k8s-conn{2}: INSTALLED, TUNNEL, reqid 12, ESP in UDP SPIs: c78cb6b1_i c5d0d1c3_o
    k8s-conn{2}: 172.21.0.0/16 172.30.0.0/16 === 10.91.152.xxx/26
    ```
    {: screen}

    * Quando você tenta estabelecer a conectividade VPN com o gráfico Helm do strongSwan, é provável que o status da VPN não seja `ESTABLISHED` na primeira vez. Você pode precisar verificar as configurações do terminal de VPN e mudar o arquivo de configuração várias vezes antes que a conexão seja bem-sucedida:
        1. Execute `helm delete --purge <release_name>`
        2. Corrija os valores incorretos no arquivo de configuração.
        3. Execute `helm install -f config.yaml -- name=<release_name> ibm/strongswan`
      Também será possível executar mais verificações na próxima etapa.

    * Se o pod de VPN está em um estado `ERROR` ou continua travando e reiniciando, isso pode ser devido à validação do parâmetro das configurações do `ipsec.conf` no configmap do gráfico.
        1. Verifique se há erros de validação nos logs de pod do strongSwan executando `kubectl logs $STRONGSWAN_POD`.
        2. Se erros de validação existirem, execute `helm delete --purge <release_name>`
        3. Corrija os valores incorretos no arquivo de configuração.
        4. Execute `helm install -f config.yaml -- name=<release_name> ibm/strongswan`

4. É possível testar ainda mais a conectividade VPN executando os cinco testes do Helm que são incluídos na definição de gráfico do strongSwan.

    ```
    helm test vpn
    ```
    {: pre}

    * Se todos os testes passarem, sua conexão VPN do strongSwan foi configurada com êxito.
    * Se algum dos testes falhar, continue com a próxima etapa.

5. Visualize a saída de um teste com falha consultando os logs do pod de teste.

    ```
    kubectl logs <test_program>
    ```
    {: pre}

    Alguns dos testes têm requisitos que são configurações opcionais na configuração de VPN. Se alguns dos testes falharem, as falhas poderão ser aceitáveis, dependendo de se você especificou essas configurações opcionais. Consulte a tabela a seguir para obter informações sobre cada teste e por que ele pode falhar.
    {: note}

    {: #vpn_tests_table}
    <table>
    <caption>Entendendo os testes de conectividade VPN do Helm</caption>
    <thead>
    <th colspan=2><img src="images/idea.png" alt="Ícone de ideia"/> Entendendo os testes de conectividade VPN do Helm</th>
    </thead>
    <tbody>
    <tr>
    <td><code>vpn-strongswan-check-config</code></td>
    <td>Valida a sintaxe do arquivo <code>ipsec.conf</code> que é gerado por meio do arquivo <code>config.yaml</code>. Esse teste pode falhar devido a valores incorretos no arquivo <code>config.yaml</code>.</td>
    </tr>
    <tr>
    <td><code>vpn-strongswan-check-state</code></td>
    <td>Verifica se a conexão VPN tem um status de <code>ESTABLISHED</code>. Esse teste poderá falhar pelos motivos a seguir:<ul><li>Diferenças entre os valores no arquivo <code>config.yaml</code> e as configurações do terminal de VPN no local.</li><li>Se o cluster está em modo de "atendimento" (<code>ipsec.auto</code> está configurado para <code>add</code>), a conexão não é estabelecida no lado no local.</li></ul></td>
    </tr>
    <tr>
    <td><code>vpn-strongswan-ping-remote-gw</code></td>
    <td>Efetua ping do endereço IP público <code>remote.gateway</code> que você configurou no arquivo <code>config.yaml</code>. Se a conexão VPN tem o status <code>ESTABLISHED</code>, é possível ignorar o resultado desse teste. Se a conexão VPN não tiver o status <code>ESTABLISHED</code>, esse teste poderá falhar pelos motivos a seguir:<ul><li>Você não especificou um endereço IP do gateway de VPN no local. Se <code>ipsec.auto</code> está configurado para <code>start</code>, o endereço IP <code>remote.gateway</code> é necessário.</li><li>Os pacotes ICMP (ping) estão sendo bloqueados por um firewall.</li></ul></td>
    </tr>
    <tr>
    <td><code>vpn-strongswan-ping-remote-ip-1</code></td>
    <td>Efetua ping do endereço IP privado <code>remote.privateIPtoPing</code> do gateway de VPN no local por meio do pod de VPN no cluster. Esse teste poderá falhar pelos motivos a seguir:<ul><li>Você não especificou um endereço IP <code>remote.privateIPtoPing</code>. Se você intencionalmente não tiver especificado um endereço IP, essa falha será aceitável.</li><li>Você não especificou o CIDR de sub-rede de pod do cluster, <code>172.30.0.0/16</code>, na lista <code>local.subnet</code>.</li></ul></td>
    </tr>
    <tr>
    <td><code>vpn-strongswan-ping-remote-ip-2</code></td>
    <td>Efetua ping do endereço IP privado <code>remote.privateIPtoPing</code> do gateway de VPN no local por meio do nó do trabalhador no cluster. Esse teste poderá falhar pelos motivos a seguir:<ul><li>Você não especificou um endereço IP <code>remote.privateIPtoPing</code>. Se você intencionalmente não tiver especificado um endereço IP, essa falha será aceitável.</li><li>Você não especificou o CIDR de sub-rede privada do nó do trabalhador na lista <code>local.subnet</code>.</li></ul></td>
    </tr>
    </tbody></table>

6. Exclua o gráfico Helm atual.

    ```
    helm delete --purge vpn
    ```
    {: pre}

7. Abra o arquivo `config.yaml` e corrija os valores incorretos.

8. Salve o arquivo `config.yaml` atualizado.

9. Instale o gráfico do Helm em seu cluster com o arquivo `config.yaml` atualizado. As propriedades atualizadas são armazenadas em um configmap para seu gráfico.

    ```
    helm install -f config.yaml --name=<release_name> ibm/strongswan
    ```
    {: pre}

10. Verifique o status de implementação do gráfico. Quando o gráfico estiver pronto, o campo **STATUS** perto da parte superior da saída terá um valor de `DEPLOYED`.

    ```
    helm status vpn
    ```
    {: pre}

11. Quando o gráfico for implementado, verifique se as configurações atualizadas no arquivo `config.yaml` foram usadas.

    ```
    helm get values vpn
    ```
    {: pre}

12. Limpe os pods de teste atuais.

    ```
    Kubectl get pods -a -l app=strongswan-test
    ```
    {: pre}

    ```
    kubectl delete pods -l app=strongswan-test
    ```
    {: pre}

13. Execute os testes novamente.

    ```
    helm test vpn
    ```
    {: pre}

<br />


## Limitando o tráfego de VPN do strongSwan por namespace ou nó do trabalhador
{: #limit}

Se você tiver um cluster de locatário único ou se tiver um cluster de vários locatários em que os recursos de cluster são compartilhados entre os locatários, será possível [limitar o tráfego de VPN para cada implementação do strongSwan para os pods em determinados namespaces](#limit_namespace). Se você tiver um cluster de vários locatários em que os recursos de cluster são dedicados a locatários, será possível [limitar o tráfego de VPN para cada implementação do strongSwan para os nós do trabalhador dedicados a cada locatário](#limit_worker).
{: shortdesc}

### Limitando o tráfego VPN do strongSwan por namespace
{: #limit_namespace}

Quando você tem um cluster de locatário único ou de vários locatários, é possível limitar o tráfego de VPN para os pods em somente determinados namespaces.
{: shortdesc}

Por exemplo, diga que você deseja pods em somente um namespace específico, `my-secure-namespace`, para enviar e receber dados sobre a VPN. Você não deseja os pods em outros namespaces, como `kube-system`, `ibm-system` ou `default`, para acessar sua rede no local. Para limitar o tráfego de VPN somente a `my-secure-namespace`, é possível criar políticas de rede global do Calico.

Antes de usar essa solução, revise as considerações e limitações a seguir.
* Não é necessário implementar o gráfico Helm do strongSwan no namespace especificado. O pod VPN do strongSwan e as rotas daemonset podem ser implementados no `kube-system` ou em qualquer outro namespace. Se a VPN do strongSwan não for implementada no namespace especificado, então o teste do Helm `vpn-strongswan-ping-remote-ip-1` falhará. Essa falha é esperada e é aceitável. O teste executa ping do endereço IP privado `remote.privateIPtoPing` do gateway de VPN no local por meio do pod de VPN no cluster, que não está no namespace que tem acesso direto à sub-rede remota. No entanto, o pod VPN ainda é capaz de encaminhar o tráfego para os pods nos namespaces que possuem rotas para a sub-rede remota, e o tráfego ainda pode fluir corretamente. O estado VPN ainda é `ESTABLISHED` e os pods no namespace especificado podem se conectar por meio da VPN.
* As políticas de rede global do Calico nas etapas a seguir não evitam que os pods do Kubernetes que usam a rede do host enviem e recebam dados por meio da VPN. Quando um pod é configurado com a rede do host, o app em execução no pod pode atender nas interfaces de rede do nó do trabalhador no qual ele está. Esses pods de rede do host podem existir em qualquer namespace. Para determinar quais pods têm a rede de host, execute `kubectl get pods --all-namespaces -o wide` e procure por qualquer pod que não tenha um endereço IP de pod `172.30.0.0/16`. Se você desejar evitar que os pods de rede do host enviem e recebam dados por meio da VPN, será possível configurar as opções a seguir em seu arquivo de implementação `values.yaml`: `local.subnet: 172.30.0.0/16` e `enablePodSNAT: false`. Essas definições de configuração expõem todos os pods do Kubernetes por meio da conexão VPN com a rede remota. No entanto, somente os pods que estão localizados no namespace seguro especificado são acessíveis por meio da VPN.

Antes de iniciar:
* Crie ou use um cluster que execute o Kubernetes versão 1.10 ou mais recente.
* [Implemente o gráfico Helm do strongSwan](#vpn_configure) e [assegure-se de que a conectividade VPN esteja funcionando corretamente](#vpn_test).
* [Instale e configure a CLI do Calico](cs_network_policy.html#cli_install).

Para limitar o tráfego de VPN para um determinado namespace:

1. Crie uma política de rede global do Calico denominada `allow-non-vpn-outbound.yaml`. Essa política permite que todos os namespaces continuem a enviar o tráfego de saída para todos os destinos, exceto para a sub-rede remota que o strongSwan VPN acessa. Substitua `<remote.subnet>` com o `remote.subnet` especificado no arquivo de configuração Helm `values.yaml`. Para especificar múltiplas redes remotas, consulte a [documentação do Calico ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://docs.projectcalico.org/v3.3/reference/calicoctl/resources/globalnetworkpolicy).
    ```yaml
    apiVersion: projectcalico.org/v3
    kind: GlobalNetworkPolicy
    metadata:
      name: allow-non-vpn-outbound
    spec:
      selector: has(projectcalico.org/namespace)
      egress:
      - action: Allow
        destination:
          notNets:
          - <remote.subnet> order: 900 types:
      - Egress
    ```
    {: codeblock}

2. Aplique a política.

    ```
    calicoctl apply -f allow-non-vpn-outbound.yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

3. Crie outra política de rede global do Calico denominada `allow-vpn-from-namespace.yaml`. Essa política permite somente um namespace especificado para enviar tráfego de saída para a sub-rede remota que o strongSwan VPN acessa. Substitua `<namespace>` com o namespace que pode acessar a VPN e o `<remote.subnet>` com o `remote.subnet` especificado no arquivo de configuração Helm `values.yaml`. Para especificar múltiplos namespaces ou sub-redes remotas, consulte a [documentação do Calico ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://docs.projectcalico.org/v3.3/reference/calicoctl/resources/globalnetworkpolicy).
    ```yaml
    apiVersion: projectcalico.org/v3
    kind: GlobalNetworkPolicy
    metadata:
      name: allow-vpn-from-namespace
    spec:
      selector: projectcalico.org/namespace == "<namespace>"
      egress:
      - action: Allow
        destination:
          nets:
          - <remote.subnet> order: 900 types:
      - Egress
    ```
    {: codeblock}

4. Aplique a política.

    ```
    calicoctl apply -f allow-vpn-from-namespace.yaml --config=filepath/calicoctl.cfg
    ```
    {: pre}

5. Verifique se as políticas de rede global são criadas em seu cluster.
    ```
    calicoctl get GlobalNetworkPolicy -o wide -- config=filepath/calicoctl.cfg
    ```
    {: pre}

### Limitando o tráfego VPN do strongSwan por nó do trabalhador
{: #limit_worker}

Quando você tem múltiplas implementações de VPN do strongSwan em um cluster de vários locatários, é possível limitar o tráfego de VPN para cada implementação para nós do trabalhador específicos que são dedicados a cada locatário.
{: shortdesc}

Quando você implementa um gráfico Helm do strongSwan, uma implementação VPN do strongSwan é criada. Os pods VPN do strongSwan são implementados em qualquer nó do trabalhador não contaminado. Além disso, um daemonset do Kubernetes é criado. Esse daemonset configura automaticamente as rotas em todos os nós do trabalhador não contaminados no cluster para cada uma das sub-redes remotas. O pod de VPN do strongSwan usa as rotas nos nós do trabalhador para encaminhar solicitações para a sub-rede remota na rede no local.

As rotas não são configuradas em nós contaminados, a menos que você especifique a contaminação na configuração `tolerations` no arquivo `value.yaml`. Ao contaminar os nós do trabalhador, é possível evitar que quaisquer rotas da VPN sejam configuradas nesses trabalhadores. Em seguida, é possível especificar a contaminação na configuração de `tolerations` somente para a implementação de VPN que você deseja permitir nos trabalhadores contaminados. Dessa forma, os pods de VPN do strongSwan para a implementação do gráfico Helm do locatário usam somente as rotas nos nós do trabalhador do locatário para encaminhar o tráfego por meio da conexão VPN para a sub-rede remota.

Antes de usar essa solução, revise as considerações e limitações a seguir.
* Por padrão, o Kubernetes coloca os pods de app em quaisquer nós do trabalhador não contaminados que estejam disponíveis. Para se certificar de que essa solução funcione corretamente, cada locatário deve primeiro se assegurar de implementar seus pods de app somente para trabalhadores que estejam contaminados para o locatário correto. Além disso, cada nó do trabalhador contaminado também deve ter uma tolerância para permitir que os pods de app sejam colocados no nó. Para obter mais informações sobre contaminações e tolerações, veja a [documentação do Kubernetes ![Ícone de link externo](../icons/launch-glyph.svg "Ícone de link externo")](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/).
* Os recursos de cluster podem não ser usados de forma ideal porque nenhum locatário pode colocar os pods de app nos nós não contaminados compartilhados.

As etapas a seguir para limitar o tráfego de VPN do strongSwan pelo nó do trabalhador usam este cenário de exemplo: diga que você tem um cluster {{site.data.keyword.containerlong_notm}} com vários locatários com seis nós do trabalhador. O cluster suporta o locatário A e o locatário B. Você contamina os nós do trabalhador das maneiras a seguir:
* Dois nós do trabalhador são contaminados para que somente os pods do locatário A sejam planejados nos trabalhadores.
* Dois nós do trabalhador são contaminados para que somente os pods do locatário B estejam planejados nos trabalhadores.
* Dois nós do trabalhador não são contaminados porque pelo menos 2 nós do trabalhador são necessários para os pods de VPN do strongSwan e o IP do balanceador de carga para execução.

Para limitar o tráfego de VPN para nós contaminados para cada locatário:

1. Para limitar o tráfego de VPN para somente trabalhadores dedicados ao locatário A neste exemplo, você especifica a `tolerância` a seguir no arquivo `values.yaml` para o gráfico Helm do strongSwan do locatário A:
    ```
    tolerâncias:
     - key: dedicated
       operator: "Equal"
       value: "tenantA"
       effect: "NoSchedule"
    ```
    {: codeblock}
    Essa tolerância permite que o daemonset da rota seja executado nos dois nós do trabalhador que tenham a contaminação `dedicated="tenantA"` e nos dois nós do trabalhador não contaminados. Os pods VPN do strongSwan para essa implementação são executados nos dois nós do trabalhador não contaminados.

2. Para limitar o tráfego de VPN para somente os trabalhadores dedicados ao locatário B neste exemplo, você especifica a `tolerância` a seguir no arquivo `values.yaml` para o gráfico Helm do strongSwan do locatário B:
    ```
    tolerâncias:
     - key: dedicated
       operator: "Equal"
       value: "tenantB"
       effect: "NoSchedule"
    ```
    {: codeblock}
    Essa tolerância permite que o daemonset da rota seja executado nos dois nós do trabalhador que tenham a contaminação `dedicated="tenantB"` e nos dois nós do trabalhador não contaminados. Os pods VPN do strongSwan para essa implementação também são executados nos dois nós do trabalhador não contaminados.

<br />


## Fazendo upgrade do gráfico Helm do strongSwan
{: #vpn_upgrade}

Certifique-se de que o gráfico Helm do strongSwan esteja atualizado fazendo upgrade dele.
{:shortdesc}

Para fazer upgrade de seu gráfico Helm do strongSwan para a versão mais recente:

  ```
  Helm upgrade -f config.yaml < release_name> ibm/strongswan
  ```
  {: pre}

O gráfico Helm do strongSwan 2.0.0 não funciona com o Calico v3 ou o Kubernetes 1.10. Antes de [atualizar seu cluster para a 1.10](cs_versions.html#cs_v110), atualize primeiro o strongSwan para o gráfico de Helm 2.2.0 ou mais recente, que é compatível com versões anteriores com o Calico 2.6 e o Kubernetes 1.9. Em seguida, exclua o gráfico Helm do strongSwan. Após a atualização, será possível reinstalar o gráfico.
{:tip}

## Desativando o serviço de VPN IPSec strongSwan
{: vpn_disable}

É possível desativar a conexão VPN excluindo o gráfico Helm.
{:shortdesc}

  ```
  helm delete --purge <release_name>
  ```
  {: pre}

<br />


## Usando um Virtual Router Appliance
{: #vyatta}

O [Virtual Router Appliance (VRA)](/docs/infrastructure/virtual-router-appliance/about.html) fornece o sistema operacional Vyatta 5600 mais recente para servidores bare metal x86. É possível usar um VRA como um gateway de VPN para se conectar com segurança a uma rede no local.
{:shortdesc}

Todo o tráfego de rede pública e privada que entra ou sai das VLANs do cluster é roteado por meio de um VRA. É possível usar o VRA como um terminal de VPN para criar um túnel IPSec criptografado entre servidores na infraestrutura do IBM Cloud (SoftLayer) e recursos no local. Por exemplo, o diagrama a seguir mostra como um app em um nó do trabalhador somente privado no {{site.data.keyword.containerlong_notm}} pode se comunicar com um servidor no local por meio de uma conexão VPN do VRA:

<img src="images/cs_vpn_vyatta.png" width="725" alt="Expor um app no {{site.data.keyword.containerlong_notm}} usando um balanceador de carga" style="width:725px; border-style: none"/>

1. Um app em seu cluster, `myapp2`, recebe uma solicitação de um serviço Ingress ou LoadBalancer e precisa conectar-se com segurança a dados na rede local.

2. Como o `myapp2` está em um nó do trabalhador que está somente em uma VLAN privada, o VRA age como uma conexão segura entre os nós do trabalhador e a rede no local. O VRA usa o endereço IP de destino para determinar quais pacotes de rede enviar para a rede no local.

3. A solicitação é criptografada e enviada pelo túnel VPN para o data center no local.

4. A solicitação recebida passa pelo firewall no local e é entregue ao terminal de túnel VPN (roteador) no qual ela é decriptografada.

5. O terminal de túnel VPN (roteador) encaminha a solicitação para o servidor ou mainframe no local, dependendo do endereço IP de destino que foi especificado na etapa 2. Os dados necessários são enviados de volta na conexão VPN para `myapp2` por meio do mesmo processo.

Para configurar um Virtual Router Appliance:

1. [Pedir um VRA](/docs/infrastructure/virtual-router-appliance/getting-started.html).

2. [Configure a VLAN privada no VRA](/docs/infrastructure/virtual-router-appliance/manage-vlans.html).

3. Para ativar uma conexão VPN usando o VRA, [configure VRRP no VRA](/docs/infrastructure/virtual-router-appliance/vrrp.html#high-availability-vpn-with-vrrp).

Se você tiver um dispositivo de roteador existente e, em seguida, incluir um cluster, as novas sub-redes móveis que são ordenadas para o cluster não serão configuradas no dispositivo do roteador. Para usar os serviços de rede, deve-se ativar o roteamento entre as sub-redes na mesma VLAN [ativando o VLAN Spanning](cs_subnets.html#subnet-routing). Para verificar se o VLAN Spanning já está ativado, use o [comando](/docs/containers/cs_cli_reference.html#cs_vlan_spanning_get) `ibmcloud ks vlan-spanning-get`.
{: important}
