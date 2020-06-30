# elastic_lab

Um repositório pra testes com o Elastic Stack.

## Objetivo

Nem tinha parado pra pensar nisso inicialmente, mas é uma boa, né? Objetivos nos ajudam a guiar os trabalhos. A ideia daqui realmente começou só como um "vamos ver o que eu consigo/dá pra fazer", mas melhorias sempre são bem vindas, e acho que objetivo é uma ótima melhoria. Então, eu vou considerar como meu objetivo nesse projeto explorar tudo o que eu conseguir do Elastic Stack, usando meu notebook com 16GB de RAM e algum espaço em disco que sobrou dos jogos ;)

Mas tem MUITA coisa no Elastic Stack, então eu vou olhar pros 4 principais e limitar algumas funcionalidades:
* Elasticsearch
  - ~~Instalando no docker~~ (feito)
  - ~~Cluster~~ (feito)
  - ~~Segurança~~ (feito, com persistência da keystore)
  - ~~Cross-cluster~~ (feito)
  - Gerenciamento de índices e ciclo de vida
  - Elasticsearch SQL
* Kibana
  - ~~Instalando no docker~~ (feito)
  - ~~Segurança~~ (feito, com persistência da keystore)
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
  - ~~Instalando no host (e em outras VMs?)~~
  - ~~Filebeat~~
  - Heartbeat
  - ~~Metricbeat~~
  - Packetbeat
  - Winlogbeat
* Logstash
  - Instalando no docker
  - Segurança
  - Ingestão de beats
  - Transformações
  - Múltiplos Pipelines

## Passo a passo

### Se você chegou agora...

Pode ser mais interessante pular lá pra baixo, em "[Reiniciando, vamos do zero.](#reiniciando-vamos-do-zero)"

Mas se quiser dar uma lida no que rolou desde o início, fique à vontade ;)

#### Elasticsearch no docker

Já mexi um pouco com o Elastic Stack, já li um pouco a respeito de docker... mas nunca usei um pra ter o outro. A ideia desse projetinho começou assim: vamos ver como é subir 3 nós do Elasticsearch pelo Docker. Primeiro passo: [documentação](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html). 

Parece fácil, mas tem coisa fora de ordem aí. 

Subi uma VMzinha com o Ubuntu Server 20.04 e durante as telas de instalação já puxei o docker. Depois disso, `sudo apt update && sudo apt install -y docker-compose`, tudo lindo. Criei o `docker-compose.yml` como a documentação manda, só copiar e colar pra dentro. Próximo passo, `docker-compose up`, certo? Errado, se fizer isso vai tomar erro na cara!
```
es02    | [1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```
Volta na documentação, mas [desce um pouco mais](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#_set_vm_max_map_count_to_at_least_262144): `sysctl -w vm.max_map_count=262144` pra resolver. Além de incluir `vm.max_map_count=262144` no `/etc/sysctl.conf` Agora sim, só subir, que maravilha! Deu até pra testar de outra máquina na rede! `curl -X GET "<ip_da_VM_ubuntu>:9200/_cat/nodes?v&pretty"` e temos:
```
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.19.0.3           27          93  10    1.14    0.69     0.54 dilmrt    -      es02
172.19.0.2           46          93  10    1.14    0.69     0.54 dilmrt    -      es01
172.19.0.4           27          93  10    1.14    0.69     0.54 dilmrt    *      es03
```

Tudo pronto com o Elastisearch, próximo passo é o Kibana!

#### Kibana também não foi de primeira...

Tá pensando em só seguir a [documentação](https://www.elastic.co/guide/en/kibana/current/docker.html) de novo, né? Não vai dar certo DE NOVO, porque o docker-compose da própria documentação do Elasticsearch já joga ele numa rede diferente da `default`, chamada `elastic`! Então, se você for tentar usar um `docker run --link es01:elasticsearch -p 5601:5601 docker.elastic.co/kibana/kibana:7.7.0
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

### Cross-cluster

Antes de começar a próxima fase eu pensei - vamos testar isso. Será que tá tudo certo? Dà pra considerar um passo-a-passo? Então eu dei um `system prune -a -f` no meu docker pra limpar tudo, e depois um `docker-compose up es01 es02 es03` e... nada funcionava. Óbvio, eu estava com a configuração já pronta pra ter os certificados, sem ter os certificados! E onde eu coloque isso na documentação? Nop, em lugar nenhum.

`docker-compose -f create-certs.yml run --rm create_certs` é o que precisa ser executado para criar os certificados para a comunicação TLS. Mas já que eu tenho que recriar, e o próximo passo é aprender como funciona cross-cluster, vamos mudar o `instances.yml`?

Alterei os nomes das 3 instâncias originais do Elasticsearch de `es0x` para `es1-0x` e adicionei três novas `es2-0x`. Agora sim, podemos rodar o `create-certs.yml`, certo? Tentei. Nada. Quando vou ver... os volumes anteriores ainda estavam lá! Mas o `docker system prune` não deveria ter removido tudo? EU TINHA ESQUECIDO DE PARAR O QUE ESTAVA RODANDO. Palmas para mim.

`docker-compose down`, `system prune` e `volume prune`. Tudo zerado. Beleza, vamos tentar de novo... não, pera. Vamos arrumar já o `docker-compose.yml` pra acertar tudo. Clusters ES1-xx e ES2-xx, com as chaves corretas nas configurações do xpack e nomes corretos dos volumes para dados persistentes (data1-xx e data2-xx). Não pode esquecer também de mudar os nomes dos clusters, na propriedade `cluster.name` dos arquivos de template! "Mas não é um arquivo só, `templates.yml`?" Era. Tem muita coisa que modifica pra dois clusters diferentes, então agora temos os arquivos `elastic_cluster01.yml` e `elastic_cluster02.yml` com os templates das definições comuns às 3 máquinas de cada cluster.

Ah sim, e como eles estão no mesmo lugar, precisamos de redes diferentes, e que os esX-01 sejam expostos em portas diferentes... ES1-01 ficou na porta 9200 como já estava, ES2-01 foi pra porta 9201. Mas isso apenas externamente, pois já que no Docker eles estarão em redes diferentes, elastic1 e elastic2, a porta nativa 9200 do container pode ser mantida. Mais um ponto: senhas diferentes para ambos, então temos uma nova variável no `.env` chamada `ELASTIC_PASSWORD2`, devidamente referenciada aonde precisa ser (no `elastic_cluster02.yml`).

Tudo pronto... dá pra subir os clusters. Vamos segurar o Kibana, porque ainda tem que configurar as senhas. E nisso, vai entrar a persistência da keystore, o que é OUTRA coisa. Então, foco: clusters, SUBAM!

Tudo rodando, maravilha. Hora de ver como funciona o cross cluster e... parece que já comecei errado. Não dá pra fazer containers em redes diferentes se falarem. Então, bora voltar todo mundo pra mesma rede.

Todo mundo verdinho. Agora bora ver essa configuração de "[External Cluster](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cross-cluster-search.html)". Bem simples, foi só usar o exemplo de Remote Cluster Setup e apontar o `cluster_one` do exemplo pro es1-01 e o `cluster_two ` para o es2-01, ambos na porta 9300. O `curl` abaixo, executado de um terminal da VM hospedeira (não de um dos containers) configura o Cross Cluster:

```
curl -X PUT --insecure --user elastic:<password> https://127.0.0.1:9200/_cluster/settings?pretty -H 'Content-Type: application/json' -d '{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_one": {
          "seeds": [
            "es1-01:9300"
          ],
          "transport.ping_schedule": "30s"
        },
        "cluster_two": {
          "seeds": [
            "es2-01:9300"
          ],
          "transport.compress": true,
          "skip_unavailable": true
        }
      }
    }
  }
}'
```

Parece ter funcionado, mas eu sei zero de fazer as queries do Elasticsearch por curl, então vou precisar do Kibana. Pensei que conseguiria subir ele desativando as opções de segurança/senha... e ele sobe, mesmo. Mas ainda pede login, e não aceita o usuário `elastic`. Vamos ter que ir atrás da persistência da keystore, mesmo...

### Monitorando com Filebeat

Aqui já começou com treta. A imagem do Elasticsearch, por padrão, joga os logs tudo pra stdout. E, um complicador pra mim, eu nunca mexi com log4j2. Então eu sai procurando pelo Google como eu ia fazer o Elasticsearch, de dentro de um container, criar essas desgraça de log em arquivo, pro Filebeat poder ler.

Depois de muito não encontrar nada de útil, uma alma abençoada no Slack da Elastic conseguiu me dar uma mão. As alterações de configuração do log4j2.properties são simples: ficam no arquivo /usr/share/elasticsearch/config/log4j2.properties do container! Então é só mapear da VM pra lá, como feito com o keystore.

Arquivo atualizado, mapeamento tudo certo, subi... e que coisa linda. Mas no meio do caminho tinha uma pedra. Eu fiz a VM com um HD virtual de só 10GB, e bateu. Como eu criei ele como `.vmdk`, fomos à caça e achamos rapidamente uma solução no [Pai dos Devs Burros](https://stackoverflow.com/questions/11659005/how-to-resize-a-virtualbox-vmdk-file):

```
VBoxManage clonemedium "source.vmdk" "cloned.vdi" --format vdi
VBoxManage modifymedium "cloned.vdi" --resize 51200
VBoxManage clonemedium "cloned.vdi" "resized.vmdk" --format vmdk
```

Achei prudente seguir a ideia de jogar ele pra um `.vdmk` novo ao invés de sobrescrever o antigo, afinal, Murphy me ama. Mas aparentemente deu tudo certo, e achei que valia a pena primeiro só subir de novo. Terminando de configurar isso, vai ser a hora de testar de novo isso daqui do zero! Ò.ó

Tudo certo, tudo lindo, all green. VMDK antigo apagado, porque tem 1TB mas 10GB faz diferença sim. E mano... eu achei que já tava fácil, levei um tempo e só consegui com uma ajuda MUITO foda, do @xeraa, com [esse artigo dele](https://xeraa.net/blog/2020_filebeat-modules-with-docker-kubernetes/) e mais [uma mão lá pelo Slack da comunidade](https://elasticstack.slack.com/archives/CNL174CQ7/p1590423259003200?thread_ts=1590254701.480300&cid=CNL174CQ7)!

Tá praticamente tudo no passo a passo dele, ficou de fora só a configuração de TLS e senha do filebeat, mas isso foi simples e em alguns minutos tava pronto! Um detalhe, que na documentação não fica claro (mais um) é que se você configurou TLS para o Elasticsearch e para o Kibana, você DEVE configurar os dois no `filebeat.yml`! A configuração do Elasticsearch precisa de hosts, certificados e usuário/senha, enquanto a do Kibana precisa de host e certificados.

Mais um lembrete aqui: o arquivo `filebeat.yml` [precisa ser do root ou do usuário que executa o filebeat](https://www.elastic.co/guide/en/beats/libbeat/current/config-file-permissions.html), com permissão 0644.

Outra coisa que eu peguei dele foi trocar o "7.7.0" por `$ELASTIC_VERSION` nas imagens!

Ah, vou desabilitar o segundo cluster, por enquanto. Se mais adiante aparecer algo legal pra usar e testar isso, a gente reativa.

### Reiniciando, vamos do zero.

`docker container prune -f; docker volume prune -f; docker network prune -f`, vamos deixar as imagens ali de boa pra não ter que baixar tudo de novo.

Primeira coisa, que o preguiçoso que não lê tudo não sabia - tem umas configurações do Docker pra fazer pra não precisar ficar no `sudo docker`, então [vai lá](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user) e faz.

#### Certificados e senhas

Abre o terminal e manda ver na criação dos certificados com o `docker-compose -f create-certs.yml run --rm create_certs`, pra começar bem a coisa!

Rolou umas cagadas no meio do caminho, mas a gente limpa pra deixar o importante: `docker run -ti -v elastic_lab_data1-01:/usr/share/elasticsearch/data --network=elastic_lab_elastic --env discovery.type=single-node --env-file .env --name=iniciar --rm "docker.elastic.co/elasticsearch/elasticsearch:${ELASTIC_VERSION}" /bin/bash` pra iniciar um container com a imagem que a gente quer usar. Nós vamos usar esse container pra criar as senhas de sistema. 

Primeiro a gente precisa ajeitar as configurações com `printf "discovery.type: single-node\nxpack.security.enabled: true" >> config/elasticsearch.yml`, e depois iniciar o Elasticsearch no fundo com `runuser -l elasticsearch -c '/usr/share/elasticsearch/bin/elasticsearch' &`. A hora que ele acalmar nos logs (vai ter uma linha sobre `Active license is now [BASIC]; Security is enabled`), a gente consegue gerar as senhas com `echo "y" | runuser -l elasticsearch -c 'bin/elasticsearch-setup-passwords auto' | grep "PASSWORD" | cut -d ' ' -f 2,4 > senhas.txt`.

Esse último comando vai fazer o Elasticsearch rodando ao fundo cuspir umas 3 linhas de log, e depois você vai ter 6 linhas com usuários e senhas, pra você salvar isso daí tudo num arquivo qualquer num canto. Duas senhas mais importantes nesse momento: kibana (vai lá pro arquivo `.env` no `KIBANA_PASSWORD`) e elastic, que é o superusuário da porra toda. Terminou? Ainda não, vamos aproveitar o container pra criar os papéis e usuários pros beats!

#### Preparando pros beats...

Primeiro, vamos gerar senhas pros usuários `beats_setup` e `beats_writer`, além de jogar tudo pra variáveis. Inclui uns prints porque elas serão necessárias fora desse container temporário! Salva elas num `.senhas` ou no próprio `.env`.

```bash
#gera senhas e exporta pra variáveis
printf "beats_setup $(cat /dev/urandom | base64 | head -c 42)\nbeats_writer $(cat /dev/urandom | base64 | head -c 42)\n" >> senhas.txt
cat senhas.txt | grep "elastic" && cat senhas.txt | grep "kibana" && cat senhas.txt | grep "beats_setup" && cat senhas.txt | grep "beats_writer"
export ELASTIC_PASSWORD="$(cat senhas.txt | grep "elastic" | cut -d ' ' -f 2)"
export KIBANA_PASSWORD="$(cat senhas.txt | grep "kibana" | cut -d ' ' -f 2)"
export BEATSSETUP_PASSWORD="$(cat senhas.txt | grep "beats_setup" | cut -d ' ' -f 2)"
export BEATSWRITER_PASSWORD="$(cat senhas.txt | grep "beats_writer" | cut -d ' ' -f 2)"

#cria os papéis de beats_setup e beats_writer
curl -X POST "localhost:9200/_security/role/beats_setup?pretty" -u elastic:${ELASTIC_PASSWORD} -k -H 'Content-Type: application/json' -d'{
    "cluster" : [
      "monitor",
      "manage_ilm",
      "manage_ml",
      "manage_index_templates",
      "manage_ingest_pipelines",
      "manage_pipeline"
    ],
    "indices" : [
      {
        "names" : [
          "filebeat-*",
          "auditbeat-*",
          "heartbeat-*",
          "metricbeat-*",
          "packetbeat-*",
          "winlogbeat-*",
          "metricbeat*"
        ],
        "privileges" : [
          "read"
        ],
        "allow_restricted_indices" : false
      },
      {
        "names" : [
          "*"
        ],
        "privileges" : [
          "manage"
        ],
        "field_security" : {
          "grant" : [
            "*"
          ]
        },
        "allow_restricted_indices" : false
      }
    ],
    "applications" : [ ],
    "run_as" : [ ],
    "metadata" : { },
    "transient_metadata" : {
      "enabled" : true
    }
  }'

curl -X POST "localhost:9200/_security/role/beats_writer?pretty" -u elastic:${ELASTIC_PASSWORD} -k -H 'Content-Type: application/json' -d'{
    "cluster" : [
      "monitor",
      "read_ilm",
      "cluster:admin/ingest/pipeline/get",
      "cluster:admin/ingest/pipeline/put",
      "cluster:admin/ilm/put"
    ],
    "indices" : [
      {
        "names" : [
          "auditbeat-*",
          "filebeat-*",
          "heartbeat-*",
          "metricbeat-*",
          "packetbeat-*",
          "winlogbeat-*"
        ],
        "privileges" : [
          "create_doc",
          "create_index",
          "view_index_metadata"
        ],
        "allow_restricted_indices" : false
      }
    ],
    "applications" : [ ],
    "run_as" : [ ],
    "metadata" : { },
    "transient_metadata" : {
      "enabled" : true
    }
  }'

#cria os usuários beats_setup e beats_writer
curl -X POST "localhost:9200/_security/user/beats_setup?pretty" -u elastic:${ELASTIC_PASSWORD} -k -H 'Content-Type: application/json' -d'{"password":"'$BEATSSETUP_PASSWORD'","roles":["beats_setup","kibana_admin","ingest_admin","beats_admin"],"full_name":"","email":"","metadata":{},"enabled":true}}'

curl -X POST "localhost:9200/_security/user/beats_writer?pretty" -u elastic:${ELASTIC_PASSWORD} -k -H 'Content-Type: application/json' -d'{"password":"'$BEATSWRITER_PASSWORD'","roles":["beats_writer"],"full_name":"","email":"","metadata":{},"enabled":true}}'
```

Agora sim, já podemos matar o container com \[CTRL+D\]. 

Os papéis, usuários e senhas são salvos no índice `.security-*` do Elasticsearch, então por isso a gente precisa iniciar esse container temporário mapeando o volume do data1-01. Se não, ao matar o container, elas se perdem. 

Hora de ver se tudo funcionou: `docker-compose up -d es1-01 es1-02 es1-03 kibana`. Em mais ou menos 1 minuto, os 3 nós do Elasticsearch já terminaram de subir, assim como o Kibana. 

Você consegue acessar a interface do Kibana pelo IP da VM que roda o Docker (apesar de que, aí depende de como você configurou a rede da sua VM), na porta 5601, e se autentica utilizando o usuário `elastic` e a senha que a gente gerou ali em cima com o `elasticsearch-setup-passwords`.

Não refiz a keystore com persistência porque, pelo menos por agora, mas eu acho que ela será necessária pra facilitar guardar a senha dos beats porque ele não expoem a configuração pro `docker-compose.yml` como o Elasticsearch e o Kibana fazem, e também não aceita variável do `.env` nos próprios arquivos de configuração `*beat.yml`. O que a gente precisa que fique persistente, no momento, são os índices do Elasticsearch. Também removi a parte que falava disso das instruções anteriores.

#### *beats!!

Então, para termos eles rodando direito, antes do agente coletor a gente precisa configurar o indice e os dashboards. As próprias ferramentas fazem isso, e dá pra fazer usando containers temporários, que nem fizemos com o Elasticsearch! Se você salvou as senhas expostas ali em cima no seu `.env`, é só usar o export abaixo e copiar e colar os comandos. Se não, ajusta aí.

```bash
## exporta senha do BEATSSETUP_PASSWORD 
export BEATSSETUP_PASSWORD="$(cat .env | grep "BEATSSETUP" | cut -d '=' -f 2)"

## executa o setup do metricbeat
docker run --rm \
--network=elastic_lab_elastic \
docker.elastic.co/beats/metricbeat:7.7.1 setup \
-E setup.kibana.host=https://kibana:5601 \
-E setup.kibana.ssl.verification_mode=none \
-E setup.ilm.overwrite=true \
-E output.elasticsearch.hosts=["https://es1-01:9200"] \
-E output.elasticsearch.ssl.verification_mode=none \
-E output.elasticsearch.username=beats_setup \
-E output.elasticsearch.password=$BEATSSETUP_PASSWORD

## executa o setup do filebeat 
docker run --rm \
--network=elastic_lab_elastic \
docker.elastic.co/beats/filebeat:7.7.1 setup \
-E setup.kibana.host=https://kibana:5601 \
-E setup.kibana.ssl.verification_mode=none \
-E setup.ilm.overwrite=true \
-E output.elasticsearch.hosts=["https://es1-01:9200"] \
-E output.elasticsearch.ssl.verification_mode=none \
-E output.elasticsearch.username=beats_setup \
-E output.elasticsearch.password=$BEATSSETUP_PASSWORD
```

Achou que agora era correr pro abraço? Não, você ainda tem que jogar a senha do beats_writer, criada ali nos cURLs, dentro dos arquivos `*beat.yml`. Feito isso, aí sim você pode correr pro abraço e mandar um `docker-compose up -d --no-deps metricbeat filebeat`, e ver o seu dashboard "[Metricbeat Docker] Overview ECS" ficar lindo, mostrando o status dos seus 6 containers!

Acho que o próximo passo pode, finalmente, ser a monitoração do endpoint com o Sysmon ;) e deixamos o Logstash pra depois

### Próximos passos
(não está em ordem, preferência ou prioridade)
  - ~~Monitoração do Kibana e dos Elasticsearch com Filebeat~~
  - ~~Monitoração do Kibana e dos Elasticsearch com Metricbeat~~
  - Logstash
  - Monitoração de endpoint com sysmon
  - Usar meu próprio script, do https://github.com/joaociocca/Graylog_Sysmon, ou um novo..?
  - ~~Cross-cluster, subir mais 3 instâncias de Elasticsearch com configuração de cluster diferente pra testar se funciona assim~~
  - Criar máquinas vulneráveis no Docker, pra monitorar pelo Elastic Stack
    * apenas Linux e Windows? Será possível incluir MacOS?
  - Transformar esse readme em uma wiki?
  - Usar o [DumpsterFire](https://github.com/TryCatchHCF/DumpsterFire) pra gerar eventos a serem visualizados..?
