# elastic_lab

Um repositório pra testes com o Elastic Stack.

## Objetivo

Nem tinha parado pra pensar nisso inicialmente, mas é uma boa, né? Objetivos nos ajudam a guiar os trabalhos. A ideia daqui realmente começou só como um "vamos ver o que eu consigo/dá pra fazer", mas melhorias sempre são bem vindas, e acho que objetivo é uma ótima melhoria. Então, eu vou considerar como meu objetivo nesse projeto explorar tudo o que eu conseguir do Elastic Stack, usando meu notebook com 16GB de RAM e algum espaço em disco que sobrou dos jogos ;)

Mas tem MUITA coisa no Elastic Stack, então eu vou olhar pros 4 principais e limitar algumas funcionalidades:
* Elasticsearch
  - ~Instalando no docker~ (feito)
    - ~Cluster~ (feito)
  - *Segurança* (TLS feito, senhas eu não tenho certeza?)
  - Cross-cluster
  - Gerenciamento de índices e ciclo de vida
  - Elasticsearch SQL
* Kibana
  - ~Instalando no docker~ (feito)
  - *Segurança* (fazendo e me fudendo)
  - Discover
  - Visualize
  - Dashboard
  - Canvas
  - Maps
  - Metrics
  - Logs
  - Uptime
  - SIEM
* Beats
  - Heartbeat
  - Metricbeat
  - Packetbeat
  - Winlogbeat
* Logstash
  - Segurança
  - Ingestão de beats
  - Transformações
  - Múltiplos Pipelines

## Passo a passo

#### Elasticsearch no docker

Já mexi um pouco com o Elastic Stack, já li um pouco a respeito de docker... mas nunca usei um pra ter o outro. A ideia desse projetinho começou assim: vamos ver como é subir 3 nós do Elasticsearch pelo Docker. Primeiro passo: [documentação](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html). 

Parece fácil, mas tem coisa fora de ordem aí. 

Subi uma VMzinha com o Ubuntu Server 20.04 e durante as telas de instalação já puxei o docker. Depois disso, `sudo apt update && sudo apt install -y docker-compose`, tudo lindo. Criei o `docker-compose.yml` como a documentação manda, só copiar e colar pra dentro. Próximo passo, `sudo docker-compose up`, certo? Errado, se fizer isso vai tomar erro na cara!
```
es02    | [1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```
Volta na documentação, mas [desce um pouco mais](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_set_vm_max_map_count_to_at_least_262144): `sysctl -w vm.max_map_count=262144` pra resolver. Agora sim, só subir, que maravilha! Deu até pra testar de outra máquina na rede! `curl -X GET "<ip_da_VM_ubuntu>:9200/_cat/nodes?v&pretty"` e temos:
```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.19.0.3           27          93  10    1.14    0.69     0.54 dilmrt    -      es02
172.19.0.2           46          93  10    1.14    0.69     0.54 dilmrt    -      es01
172.19.0.4           27          93  10    1.14    0.69     0.54 dilmrt    *      es03
```

Tudo pronto com o Elastisearch, próximo passo é o Kibana!

#### Kibana também não foi de primeira...

Tá pensando em só seguir a [documentação](https://www.elastic.co/guide/en/kibana/current/docker.html) de novo, né? Não vai dar certo DE NOVO, porque o docker-compose da própria documentação do Elasticsearch já joga ele numa rede diferente da `default`, chamada `elastic`! Então, se você for tentar usar um `sudo docker run --link es01:elasticsearch -p 5601:5601 docker.elastic.co/kibana/kibana:7.7.0
`, você vai receber...
```
Unable to find image 'docker.elastic.co/kibana/kibana:7.7.0' locally
7.7.0: Pulling from kibana/kibana
86dbb57a3083: Already exists
428bffc1ae7f: Pull complete
d06182cbcb81: Pull complete
6288415a3c6b: Pull complete
6ca68992403a: Pull complete
71ae5cf0fadc: Pull complete
33a4ecc46c55: Pull complete
409e3523715e: Pull complete
9b9517adf619: Pull complete
3a4244116da1: Pull complete
0ce0ac546744: Pull complete
Digest: sha256:1682e44eb728e1de2027c2cc8787d206388d9f73391928bdbfbbd24d758dd927
Status: Downloaded newer image for docker.elastic.co/kibana/kibana:7.7.0
docker: Error response from daemon: Cannot link to /es01, as it does not belong to the default network.
ERRO[0105] error waiting for container: context canceled
```
A solução é ou fazer outro `docker-compose.yml` ou aproveitar o do Elasticsearch. Aqui já começou a tomar forma o `docker-compose.yml` que está no repositório. Definição de nome do container, da rede, exposição de porta. Mas tem uma coisa MUITO importante desse passo: definir o volume `./kibana.yml:/usr/share/kibana/config/kibana.yml` pra poder utilizar o `kibama.yml` externo dentro do container.

Pelo que eu li até aqui, também dá pra fazer isso com variável de ambiente no `docker-compose.yml`, mas eu achei que era mais interessante testar e aprender essa exposição, bem como lidar diretamente com o `kibana.yml` externamente por inteiro. 

E foi assim que chegamos no `kibana.yml` que está no repositório, com a configuração de `server.host` sem utilização de endereço "loopback" e definindo o `elasticsearch.hosts` para os três containers `["http://es01:9200","http://es01:9200","http://es01:9200"]`.

Uma coisa que faltou foi ativar a monitoração dos nós do Elasticsearch! Isso pode ser feito incluindo a linha `- xpack.monitoring.collection.enabled=true` no conjunto `enviroment` de cada um dos containers! Já vai ser o próximo commit daqui.

#### *Segurança* (em andamento)

[configurar TLS no Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/configuring-tls-docker.html)

[configurar senha no Elasticsearch](https://discuss.elastic.co/t/how-to-set-passwords-for-built-in-users-in-batch-mode/119655/6)

> Configurar senha, usar `docker-compose run <máquina> /bin/bash`
>
> Testar na mesma VM: `curl --insecure --user elastic:senha https://127.0.0.1:9200/_cluster/health/?pretty`


### Próximos passos
(não está em ordem, preferência ou prioridade)
  - Logstash
  - Monitoração de endpoint com sysmon + winlogbeat
  - Usar meu próprio script, do https://github.com/joaociocca/Graylog_Sysmon
