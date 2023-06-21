# Implementando a Stack com Traefik e jBPM

A stack de serviços que você quer implementar inclui o Traefik como um proxy reverso e um balanceador de carga, e o jBPM, um sistema de gestão de processos de negócio. Aqui está uma análise detalhada de cada parte do arquivo Docker Compose que você forneceu.

## Traefik

O Traefik é um proxy reverso e balanceador de carga moderno e rápido que trabalha com microserviços. Ele suporta vários back-ends (Docker, Kubernetes, Mesos, etc.) e pode ser facilmente estendido para mais.

Nesta configuração, você está usando a imagem v2.4 do Traefik. As opções de comando configuram o Traefik para funcionar com o Docker em modo Swarm, ou seja, para orquestração de serviços em um cluster de Docker.

Aqui estão os detalhes dos comandos:

- `--providers.docker=true`: Habilita o Docker como um provedor de configuração.
- `--providers.docker.swarmMode=true`: Habilita o Docker Swarm Mode. Isso permite que o Traefik se conecte ao Docker Swarm API e descubra os serviços declarados no Swarm.
- `--providers.docker.exposedbydefault=false`: Evita que o Traefik exponha automaticamente todos os serviços Docker. Isso é útil para garantir que apenas os serviços que você deseja sejam expostos pelo Traefik.
- `--entrypoints.web.address=:80`: Define o ponto de entrada "web" para ouvir na porta 80.
- `--entrypoints.websecure.address=:443`: Define o ponto de entrada "websecure" para ouvir na porta 443.
- `--providers.docker.network=traefik_public`: Informa ao Traefik para se conectar à rede "traefik_public".
- `--api.insecure=true`: Habilita a API do Traefik e permite o acesso sem restrições.
- `--ping=true`: Habilita a rota de ping do Traefik.
- `--serversTransport.insecureSkipVerify=true`: Desativa a verificação do certificado SSL dos back-ends. Isso pode ser útil em ambientes de teste, mas não é recomendado em produção.

As opções de mapeamento de portas (ports) expõem as portas 80, 443 e 8082 do contêiner Traefik para o host. A opção de volume monta o soquete Docker do host dentro do contêiner Traefik, permitindo que ele interaja com o daemon Docker.

A seção de deploy especifica que o serviço Traefik deve ser implantado globalmente (em todos os nós do Swarm) e apenas nos nós que têm o papel de "manager".

## jBPM

O jBPM é um sistema de gestão de processos de negócio (BPM) flexível. Ele suporta a modelagem de processos baseada em BPMN 2.0. A imagem Docker fornecida é do servidor jBPM completo.

A opção de ambiente configura os limites de memóriada JVM para o jBPM.

As etiquetas (labels) na seção de deploy servem para configurar como o Traefik deve manipular o serviço jBPM:

- `traefik.enable=true`: Habilita o Traefik para este serviço.
- `traefik.http.services.jbpm.loadbalancer.server.port=8080`: Define a porta do servidor jBPM como 8080.
- `traefik.http.services.jbpm.loadbalancer.sticky=true`: Habilita a funcionalidade "sticky sessions". Isso significa que, uma vez que um cliente se conecta a um servidor específico através do balanceador de carga, todas as futuras solicitações desse cliente serão enviadas para o mesmo servidor, desde que esteja disponível. Isso é útil para aplicações que armazenam informações de estado no servidor.
- `traefik.http.services.grecco.loadbalancer.sticky.cookie.name=jbpm`: Define o nome do cookie de sessão "sticky" para "jbpm".
- `traefik.http.routers.jbpm.rule=Host(`jbpm.example.org`)`: Define uma regra para que o Traefik direcione as solicitações para o serviço jBPM se o host da solicitação corresponder a "jbpm.example.org".
- `traefik.http.routers.jbpm.entrypoints=web`: Define o ponto de entrada para o serviço jBPM como "web".

A política de reinicialização está definida para "on-failure", o que significa que o Docker tentará reiniciar o serviço se ele falhar.

## Redes

O arquivo Docker Compose define uma rede chamada "traefik_public" que é usada por ambos os serviços. Esta rede deve ser criada antecipadamente porque está marcada como externa. Como estamos trabalhando em um ambiente Docker Swarm, a rede deve ser do tipo overlay para permitir a comunicação entre os serviços em diferentes nós. Você pode criar esta rede com o seguinte comando Docker:

```bash
docker network create --driver=overlay traefik_public
```

Depois de criar a rede, você pode implantar a stack com o comando `docker stack deploy`.
```bash
docker stack deploy -c jbpm.yaml jbpm
```
> Lembre de trocar `traefik.http.routers.jbpm.rule=Host(`jbpm.example.org`)` jbpm.example.org pelo seu dominio.

Veja abaixo 3 replicas do jBPM funcionando com o traefik.
![](https://user-images.githubusercontent.com/125938946/247745739-267525fe-8d9c-4503-9a07-ad2a98199d90.png)
