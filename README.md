# apiops-meetup

Meetup de Containers e APIOps no Kong

# Pré-requisitos

- Instalar um engine Docker local (ex: Docker Desktop)
- Instalar o VKPR (docs.vkpr.net)
  
# Roteiro

## Instalação

Subir um cluster descartável local e instalar o Kong Gateway (operando em "Free Mode", o que inclui Kong OSS + Kong Manager):

```sh
vkpr infra up
vkpr kong install
```

As seguintes URLs estarão disponíveis:

* Kong Manager: http://manager.localhost:8000
* Kong Admin API: http://api.manager.localhost:8000
* Kong Proxy: http://kong.localhost:8000 (este é o endpoint do Gateway)

**IMPORTANTE**: talvez seja necessário incluir estas linhas em "/etc/hosts":

```
127.0.0.1 whoami.localhost api.manager.localhost manager.localhost admin.localhost kong.localhost
```

Testar o funcionamento deployando uma aplicação de teste:

```sh
# app de teste
vkpr whoami install
# pequeno fix, sorry
kubectl patch ing/whoami -n vkpr --type=json \
  -p='[{"op": "replace", "path": "/spec/ingressClassName", "value":"kong"}]'
# teste se retorna algo
curl whoami.localhost:8000
```

## Spec de API

Escolhemos o serviço online "ViaCEP" para servir como exemplo de API exposta pelo API Gateway. Imaginem que seria um serviço mantido por você e que rodaria dentro do cluster: o foco aqui é demonstrar o que faria um pipeline que entrega *também* o comportamento do API Gateway.

Contruímos uma spec OpenAPI para alguns endpoints deste serviço REST (que por acaso é público), ela se encontra no arquivo `openapi-spec.yaml`.

Esta spec contem anotações específicas do Kong que sinalizam o comportamento esperado para o API Gateway. Por exemplo:

```yml
paths:
  /{cep}/json:
    get:
      summary: Busca CEP
      description: Retorna endereço de um CEP
      operationId: buscaCEPJson
      tags:
        - cep
      parameters:
        - name: cep
          in: path
          description: CEP a consultar
          required: true
          schema:
            type: string
      responses:
        '200':
          description: OK
      x-kong-plugin-rate-limiting:
        enabled: true
        config:
          policy: local
          minute: 5
```

É possível gerar as configurações do Kong para esta spec automaticamente de duas formas:

1. Usando o Insomnia (Generate Config / Kong for Kubernetes)
2. Usando a CLI `inso`:
   
```sh
inso generate config -t kubernetes $PWD/openapi-spec.yaml
```

## Deployment

A deployment destes artefatos irá configurar automaticamente o Kong. Você verá um novo "service" e uma nova "route".

Isto possibilita que:

- A configuração do API Gateway seja delegada aos pipelines dos próprios serviços
- O Kong pode ter sua configuração mantida pelo cluster ("db-less" é mais escalável)

Nós separamos e simplificamos os componentes gerados pelo `inso` dentro da pasta "kic" deste repositório. Seu deployment pode ser feito com o comando:

```sh
kubectl apply -f kic/
```

Para testar a chamada a esta API basta rodar o comando abaixo:

```sh
curl kong.localhost:8000/cep/20020080/json
```

Rode este comando repetidas vezes para verificar o rate limiting atuando.

