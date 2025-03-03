
Este repositorio foi criado para documentar os estudos realizados sobre AWS.


# Sumário

- [Definindo um orçamento](##definindo-um-orçamento)
- [Politicas IAM](#politicas-iam)
- [S3](#s3)
- [EC2](#ec2)
- [Redes privadas VPC](#redes-privadas-vpc)
- [Monitoramento de serviços na AWS](#monitoramento-de-serviços-na-aws)
- [Utilizando o Amazon Route 53](#utilizando-o-amazon-route-53)


## Definindo um orçamento

Acesse Billing and Cost Management e em Budgets crie um budget. A AWS disponibiliza templates de budgets. Para os nossos estudos foi criado um personalizado. É possível configurar um budget com
threshold de gastos definido por periodo (dia, mes etc) e serviço AWS (podendo definir um budget que se aplica a todos os serviços).

Tambem é possivel configura-lo para renovar periodicamente ou uma vez atingido esse budget ele se expire.

![Texto alternativo](assets/budget.png)

É recomendavel que além do budget voce também defina alertas que ao atingir (ou tambem estimar que atingirá) certo percentual do budget o alerta emite uma notificação por email.

Note que os alertas apenas notificam porem não executam ações. Para triggar uma ação é necessário configurar uma action. Para os estudos elas não serão criadas.

![Texto alternativo](assets/action.png)

## Politicas IAM

Ao criar um conta na AWS, o usuario criado é um usuario root. Por questões de segurança é aconsenhavel adicionar um fator de autenticação para acesso ao usuario root. Além disso,
para o uso rotineiro dos serviços da AWS voce deve criar um usuario que não tenha acesso irrestrito como o usuario root.

Ao criar um usuario voce pode definir policies especificas a este usuario ou tambem adicionar um usuario a um grupo de modo que o usuario herde as permissoes definidas nesse grupo.

Criaremos um usuario atribuindo uma policy especificamente a ele. Para restrigir as permissoes
deste usuario a apenas ações operacionais foi atribuído a policy PowerUserAccess. Todas as polcies podem ser definidas por arquivos json e possuem uma estrutura básica que consiste em um ou mais "statements". Abaixo é possível visualizar a definição da policy 
PowerUserAccess:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "NotAction": [
                "iam:*",
                "organizations:*",
                "account:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "account:GetAccountInformation",
                "account:GetPrimaryEmail",
                "account:ListRegions",
                "iam:CreateServiceLinkedRole",
                "iam:DeleteServiceLinkedRole",
                "iam:ListRoles",
                "organizations:DescribeOrganization"
            ],
            "Resource": "*"
        }
    ]
}
```



Cada statement é composto por:
- Effect:  Define o efeito da policy, que pode ser "Allow" ou "Deny".
- Action/NotAction: Especifica quais ações são permitidas (ou não são permitidas)
- Resource: Define os recursos aos quais a policy se aplica. O asterisco (*) indica que se aplica a todos os recursos.

No exemplo mostrado o primeiro statment permite aplicar os efeitos as ações não especificadas em "NotAction". Para o segundo statment é permitido todas as ações sobre as actions descritas.

Um "Deny" explicito sempre terá prioridade sobre um "Allow" devido à sua capacidade.

Após criar um usuário voce pode criar uma acess key.

![Texto alternativo](assets/access.png)


 Note que a forma recomendada pela AWS em muitos casos é criar uma "role", uma entidade que define um conjunto de permissões para realizar ações em recursos da AWS, sem associá-las diretamente a um usuário ou grupo específico. Ao contrário das IAM policies, que são regras que definem permissões, uma role é uma identidade da AWS que, quando assumida, concede temporariamente a quem a assume esses direitos.

As proximas ações descritas nessa documentação será utilizando o usuario criado.

## S3

O AWS S3 é um serviço de storage da AWS. Para utliza-ló é ncessario criar e configurar um bucket.

Ao criar um bucket voce pode configurar uma serie de atributos. Em geral, o comportamento desejado de um bucket é que seus arquivos armazenados não sejam 
públicos, porém para casos como a publicação de sites, ou disponibilização de imagens para acesso público é necessário desativar o Block Public.

![Texto alternativo](assets/acls.png)

Note que mesmo desativando o Block Public arquivos no bucket ainda permenecem privados. Para torna-los públicos você deve acessar Permissions e Bucket Policies.
Assim como as policies IAM, a policiy para um Bucket também pode ser configurada
via JSON.

![Texto alternativo](assets/bucket-policies.png)


```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid":"PublicRead",
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject",
                 "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::studies-s3/*"
    }
  ]
}
```

Após isso os arquivos podem ser acessados por um usário anônimo, como por exemplo a [imagem](https://studies-s3.s3.us-east-1.amazonaws.com/corinthians.jpeg). 

Um ponto importante que deve ser considerado é que ao tentar acessar o link https://studies-s3.s3.us-east-1.amazonaws.com/ diretamente o S# entende isso como uma operação de listagem, a qual não foi atribuida permissão. Para configurar sites estáticos por exemplo acesse "properties" -> "Static website hosting" e ative (enable)
o seu uso. É possível configurar para o para o link redirecionar para um objeto do bucket (como um arquivo html) ou até mesmo outro link ou bucket.

## EC2

O EC2 é o serviço de computação em nuvem da AWS. Uma instancia do EC2 provisiona recursos
em 5 aspectos: cpu, memória, disco, rede, FPGA/GPU. As intâncias da familia t disponibilzam um recurso que caso a demanda esteja abaixo de determinado patamar créditos são acumulados.

Para rodar uma instância a amazon disponibiliza imagens (como amazon linux, ubuntu etc). 
Você pode usar imagens de terceiros (não verificados pela AWS) ou até mesmo construir sua própria imagem.

Em relação ao disco, existem diferentes opções para diferentes demandas. No caso de optar por um disco EBS mesmo após o pause da instância o disco será preservado (porém o hardware não, ao despausar ele alocará o volume EBS em outa máquina diferente do reboot que não desaloca o hardware). 

![imagem](assets/ec2-images.png)

Entre outros configurações voce pode permitir o acesso a SHH, HTTP e HTTPS. Para recursos avançados vale mencionar EBS-optimized instance, que cria uma rede apenas
para comunição entre o EBS e a aplicação (obtendo melhor desempenho) e o User data 
que disponibiliza carregar um script para ser executado na primeira vez que a instância é configurada.

Caso opte por conectar por ssh utilize a chave privada gerada, o ec2-user é para o caso em que a imagem escolhida seja a Amazon Linux:
```
ssh -i ./name-ssh.pem ec2-user@public-ipv4-dns
```
Pode ser necessário reduzir as permissões da chave pem apenas para o usuário criador da chave, rode chmod 400.

Para criar uma imagem a partir da sua instância atual você pode selecionar Actions ->
Image and templates -> Create Image. A imagem não só mantém as configurações da instância como também mantém um snapshot com as os arquivos criados e instalados.

### Modo de seleção das instâncias

A AWS disponibiliza quatro modos de seleção das instâncias:

- On Demand: Modelo padrão, para imagens que utilizem Linux é cobrado por segundo utilizado.
-  Reserved Instance: Uma instancia é reservada por determinado tempo (de 1 a 3 anos)
implicando em menores taxas porém com pouco flexibilidade para troca de configurações da instância escolhida.
- Savings Plans: Ao invés de reservar uma instância especifica voce se compromete a gastar um valor especifico por hora, obtendo descontos em relação ao montante comprometido a ser gasto (maior flexibilidade que o Reserved Instance pois não necessariamente se compromete a reservar uma configuração especifica de instância).
- Instâncias Spot: Instâncias spot você negocia o uso de uma instância, se o valor proposto estiver acima do valor executado no momento (em geral bem abaixo do on demand e até mesmo em muitas situações abaixo do reserved instance)
a instância será atribuída a você, caso contrário a instância é interrompida (adequado para casos em que há redundância e não há problema em executar a instância em outro momento). 
- Host dedicado: Controle total sobre o host fisico onde é executado a instância (adequado para casos que necessitam compliance). Para obter esse controle as taxas executadas são maiores que as outras modalidades.

## Redes privadas VPC

Ao criar uma conta na AWS é criado uma VPC default, porém é possível configurar o seu próprio VPC, definindo sub-redes, se elas são privadas ou públicas, gateways, routes etc.

### Conceitos básicos de rede na AWS

A seguir será explicado alguns conceitos chaves para o entedimento e configuração de uma VPC.

Uma Virtual Private Cloud (VPC) é definido por região, dentro de uma região voce pode definir Availability Zones (AZs), que são  locais geograficamente separados dentro de uma região de um provedor de serviços da AWS. Cada AZ é composta por um ou mais data centers físicos, que são isolados uns dos outros para fornecer alta disponibilidade e resiliência contra falhas.

Em uma VPC voce pode definir sub-redes (publicas ou privads), cada sub-rede deve estar contida em uma única Availability Zone. Essa configuração permite que você controle a localização dos seus recursos. Por exemplo, você pode criar sub-redes em múltiplas AZs para garantir que seus aplicativos tenham alta disponibilidade e sejam resistentes a falhas.

Cada rota de comunicação entre as sub-redes e/ou internet gateway é definido pelo router, que redireciona a as rotas de comunicação. Por exemplo imagine que voce tem um frontend que consulta um banco de dados, por criterios de segurança e compliance
o banco de dados pode estar em um sub-rede privada que se comunica com o frontend.
O frontend por outro lado deve estar em uma sub-rede pública que se comnunica com o internet gateway. 

Para uma rede ter acesso a internet é necessário que ela tenha um IP público e uma rota pra um internet gateway. Também após o gateway pode ser configurado um Load Balancer que faz o gerencimaneto das requisições.

Para gerenciamneto de uma VPC existem dois aspectos:

- Security group: Definições sobre cada instancia EC2, é possível configurar aspectos como quais portas podem receber requisições etc. Nesse contexto tudo que deve ser permitido deve ser definido, caso não definido não é permitido.

- VACL's: Definições sobre a VPC, se aplica sobre a VPC. Pode ser definido o que é permitido e o que não é permitido.

### Criando uma VPC

Para criar uma VPC você deve escolher o número de AZs e o número de sub-redes públicas e privadas. Note na imagem abaixo que as duas AZs foram us-east-1a e 
us-east-1b.

![Texto alternativo](assets/AZ.png)

Na imagem abaixo apresenta o preview de como fica a VPC com 2 AZs e 4 sub-redes (2 públicas e 2 privadas).

![Texto alternativo](assets/main-vpc.png)

Dois aspectos que não serão abordados agora são NAT e VPC endpoints. Após criar a VPC todos os atributos são criados como abaixo:

![Texto alternativo](assets/result-vpc.png)

Nas imagens abaixo mostram a configuração de um security group e uma ACL:

![Texto alternativo](assets/security-groups.png)

![Texto alternativo](assets/ACLS.png)

Após isso  é possível criar uma nova instancia atribuindo a ela um par de AZ e se ela é publico ou privada. Note também a opção de seleção de instancias spot, como o preço de instancias spot é sempre limitado por on demand sempre opte por instancias spot.

![Texto alternativo](assets/spot.png)

## Monitoramento de serviçoes na AWS


Para o monitoramento das instancias ec2 (ou outros serviços) é possível utilizar
o cloudwatch. O cloudwatch disponibiliza o monitoramente a cada 5 minutos de métricas
como o uso de memória, uso de cpu etc. A partir de tais métricas é possível implementar triggers como 
disparar alarmes por email utilizando o Amazon SNS ou até mesmo executar ações como parar uma instância
caso esteja com pouco uso. 

É possivel ativar no cloudwatch o detalhamento de métricas a cada 1 minuto (um custo adicional).
Para ativar o detalhamento selecione Manage detailed monitoring -> Enable. 

Vamos configurar um monitoramento que emite um alerta no email, crie um tópico em Amazon SNS,
selecione o topico criado e crie uma subscription que o protocolo é via email, isto é, ao alarme ser disparado é enviado um email (outras ações podem ser configuradas como fazer uma requisição, sms, Amazon SQS e Lambda).

Após isso, voltando para a seção em EC2 em Actions -> Monitor and troubleshoot -> Manage Cloudwatch Alarms voce pode criar um alarme, inclusive selecionando uma ação (como pausar uma instancia). Ao criar o alarme selecione o topico com a subscription criada.

Note que  ao criar um alarme voce pode configurar em Alarm action ações como Stop, reboot etc.

## Utilizando o Amazon Route 53

Ao configurar uma instancia voce pode querer associar um dominio associado ao seu IP Public IPv4 address.
Para isso é necessário configurar algumas etapas:

- Alugar um dominio em Route 53
- Fixar um IP fixo a sua instancia. Toda vez que uma instancia é parada ao ser iniciada novamente por padrão será um IP Public diferente. Então na aba lateral de EC2 selecione Network & Security -> Elastic Ips -> Allocate Elastic IP adress. Após criar o IP selecione ele e em Actions -> Associate Elastic IP address, e então associe a instancia desejada. 
- Em Route 53 voce pode então associar o dominio ao IP criado criando um record, inserindo o IP.

##  Cloudshell

Por meio de terminal (AWS CLI) é possível interagir com os recursos da aws. No Cloudshell voce pode
verificar usuarios, listar buckets, verificar e executar instancias etc. Por exemplo voce pode listar os nomes
das regions para as instancias ec2:

```
>> aws ec2 describe-regions --query "Regions[].RegionName" --output text
```

Por padrão o output das respostas no AWS CLI é json.

Também é possivel utilizar o AWS CLI fora de ambientes da AWS, como no
seu computador. Para conectar ao seu usuario da AWS CLI, após sua instalação:

```
aws configure 
```
E insira a ACCESS_KEY e a SECRET_KEY. No  
[link](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html)
apresenta a documentação referente aos comandos.

## CloudTrail

Para rastrear todas as chamadas de API realizadas nos seus serviços a AWS disponibiliza
o CloudTrail, que monitora as chamadas e registra os logs em um bucket da AWS.

Para isso é necessário criar um trail (configurado por região). Os recursos disponiveis são:

- Gerenciamento dos recursos: Capturar as repostas dos eventos.
- Data events: Respostas das chamadas das API's
- Insights: Recursos que detectam eventos de segurança (como numero de chamadas atipicas etc).

## Route 53 com Health Check

Outro recurso de monitoramento que a AWS disponibiliza é o uso de  Health Check no Route 53.
Ao associar um dominio a um IP é possivel configurar o Health Check. Nele é possível verificar
os endpoints e emitir alertas em casos de falha ou caso não esteja disponivel.

## Amazon RDS

A aws fornece o o serviço Amazon RDS para o uso de banco de dados relacionais. O processo
de criação de uma Database (seja MySQL, Postgres, Oracle etc) é parecido com a criação de uma instancia
EC2. Além das configurações especificas para um database atente se para a criação de um security group
na vpc selecionada com permissão a comunicação na porta configurada para o Database (por exemplo no caso do
postgres 5432). Em geral, utilize o database em uma subrede privada.

Após isso a criação é possível utilizar o dabatase normalmente.

## Auto Scaling e Load Balancer

