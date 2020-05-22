# elastic_lab

Um repositório pra testes com o Elastic Stack.

## Objetivo

Nem tinha parado pra pensar nisso inicialmente, mas é uma boa, né? Objetivos nos ajudam a guiar os trabalhos. A ideia daqui realmente começou só como um "vamos ver o que eu consigo/dá pra fazer", mas melhorias sempre são bem vindas, e acho que objetivo é uma ótima melhoria. Então, eu vou considerar como meu objetivo nesse projeto explorar tudo o que eu conseguir do Elastic Stack, usando meu notebook com 16GB de RAM e algum espaço em disco que sobrou dos jogos ;)

Mas tem MUITA coisa no Elastic Stack, então eu vou olhar pros 4 principais e limitar algumas funcionalidades:
* Elasticsearch
  - ~Instalando no docker~ (feito)
    - ~~Cluster~~ (feito)
  - ~~Segurança~~ (feito)
  - Cross-cluster
  - Gerenciamento de índices e ciclo de vida
  - Elasticsearch SQL
* Kibana
  - ~~Instalando no docker~~ (feito)
  - ~~Segurança~~ (feito)
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

### Segurança achou que era PM, me deu uma surra...

Esse daqui além de dar me espancar dando muito trabalho fez eu me sentir muito burro. Passeei por uma montanha de documentação e páginas, entre elas (mas não apenas):

* [configurar TLS no Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/configuring-tls-docker.html)
* [configurar senha no Elasticsearch](https://discuss.elastic.co/t/how-to-set-passwords-for-built-in-users-in-batch-mode/119655/6)
* [configurar TLS no Kibana](https://www.elastic.co/guide/en/kibana/current/elasticsearch-mutual-tls.html)
* [configurando segurança no Kibana](https://www.elastic.co/guide/en/kibana/current/using-kibana-with-security.html)

Até que eu finalmente esbarrei, totalmente sem querer, em "[começando com docker](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html)". E eu senti que todo meu Google-fu deixou de existir e eu não manjo nada de pesquisas mais. Tá certo que eu tava na cara do gol, indo manualmente... mas aqui tá tudo PRONTO. TU-DO.

Mesmo assim, no meio do caminho teve um bug fantasma:

```
kibana              | {"type":"log","@timestamp":"2020-05-22T05:01:59Z","tags":["listening","info"],"pid":6,"message":"Server running at https://0.0.0.0:5601"}
kibana              | {"type":"log","@timestamp":"2020-05-22T05:02:00Z","tags":["info","http","server","Kibana"],"pid":6,"message":"http server running at https://0.0.0.0:5601"}
kibana              | {"type":"log","@timestamp":"2020-05-22T05:02:08Z","tags":["warning","plugins","monitoring","monitoring","kibana-monitoring"],"pid":6,"message":"Error: [parse_exception] no body content for monitoring bulk request\n    at respond (/usr/share/kibana/node_modules/elasticsearch/src/lib/transport.js:349:15)\n    at checkRespForFailure (/usr/share/kibana/node_modules/elasticsearch/src/lib/transport.js:306:7)\n    at HttpConnector.<anonymous> (/usr/share/kibana/node_modules/elasticsearch/src/lib/connectors/http.js:173:7)\n    at IncomingMessage.wrapper (/usr/share/kibana/node_modules/elasticsearch/node_modules/lodash/lodash.js:4929:19)\n    at IncomingMessage.emit (events.js:203:15)\n    at endReadableNT (_stream_readable.js:1145:12)\n    at process._tickCallback (internal/process/next_tick.js:63:19)"}
kibana              | {"type":"log","@timestamp":"2020-05-22T05:02:08Z","tags":["warning","plugins","monitoring","monitoring","kibana-monitoring"],"pid":6,"message":"Unable to bulk upload the stats payload to the local cluster"}
kibana              | {"type":"log","@timestamp":"2020-05-22T05:02:17Z","tags":["warning","plugins","monitoring","monitoring","kibana-monitoring"],"pid":6,"message":"Error: [parse_exception] no body content for monitoring bulk request\n    at respond (/usr/share/kibana/node_modules/elasticsearch/src/lib/transport.js:349:15)\n    at checkRespForFailure (/usr/share/kibana/node_modules/elasticsearch/src/lib/transport.js:306:7)\n    at HttpConnector.<anonymous> (/usr/share/kibana/node_modules/elasticsearch/src/lib/connectors/http.js:173:7)\n    at IncomingMessage.wrapper (/usr/share/kibana/node_modules/elasticsearch/node_modules/lodash/lodash.js:4929:19)\n    at IncomingMessage.emit (events.js:203:15)\n    at endReadableNT (_stream_readable.js:1145:12)\n    at process._tickCallback (internal/process/next_tick.js:63:19)"}
kibana              | {"type":"log","@timestamp":"2020-05-22T05:02:17Z","tags":["warning","plugins","monitoring","monitoring","kibana-monitoring"],"pid":6,"message":"Unable to bulk upload the stats payload to the local cluster"}
kibana              | {"type":"log","@timestamp":"2020-05-22T05:02:27Z","tags":["warning","plugins","monitoring","monitoring","kibana-monitoring"],"pid":6,"message":"Error: [parse_exception] no body content for monitoring bulk request\n    at respond (/usr/share/kibana/node_modules/elasticsearch/src/lib/transport.js:349:15)\n    at checkRespForFailure (/usr/share/kibana/node_modules/elasticsearch/src/lib/transport.js:306:7)\n    at HttpConnector.<anonymous> (/usr/share/kibana/node_modules/elasticsearch/src/lib/connectors/http.js:173:7)\n    at IncomingMessage.wrapper (/usr/share/kibana/node_modules/elasticsearch/node_modules/lodash/lodash.js:4929:19)\n    at IncomingMessage.emit (events.js:203:15)\n    at endReadableNT (_stream_readable.js:1145:12)\n    at process._tickCallback (internal/process/next_tick.js:63:19)"}
kibana              | {"type":"log","@timestamp":"2020-05-22T05:02:27Z","tags":["warning","plugins","monitoring","monitoring","kibana-monitoring"],"pid":6,"message":"Unable to bulk upload the stats payload to the local cluster"}
```

O principal que eu achei a respeito disso foi [um issue no próprio github da Elastic, mas relativo ao Kibana 6.4GA](https://github.com/elastic/kibana/issues/22842), e que foi fechado sem conclusão por não conseguirem reproduzir. Apareceu, pra mim, primeiro quando eu apenas subi o Kibana... mas ficou repetindo eternamente quando eu tentei um login inválido. Nisso, ao invés do Kibana só dar a mensagem de erro, ele resolveu me apresentar um 404 de uma linha só e não me deixar tentar logar novamente.

Derrubando tudo e subindo de novo, consegui fazer o login - e aí apareceu exatamente essa quantidade que vocês veem aí acima, ao invés de um loop infinito de bugs. Menos mal, seja o que tiver sido, já era.

Mas eu até que gosto do caminho que eu peguei. Além de ter aprendido um tanto, ainda acho que meu `docker-compose.yml` ficou mais bonito jogando aquele mundaréu de configuração igual que as instâncias do Elasticsearch tem pro `templates.yml`.

Além disso, passar as configurações do `kibana.yml` pras variáveis de ambiente no `docker-compose.yml` levou embora o `kibana.yml`, e considerando que apenas 6 são estáticas e o resto todo lida com variáveis, não vejo porque voltar.
