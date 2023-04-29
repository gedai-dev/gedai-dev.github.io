---
layout: post
title: "Publicação de libs npm local com Verdaccio"
date: 2023-04-29T00:00:00-03:00
author:
  name: Getúlio Magela Silva
  url: "https://github.com/gm50x"
tags: [publicação, npm, registry, local, verdaccio]
---

O Verdaccio é um Registry que pode ser utilizado para armazenar projetos NPM. Você pode utilizá-lo como um registry local ou como um registry privado na sua empresa. Eu utilizo bastante para realizar testes antes de entregar minhas libs internas, assim não preciso subir flags para testar estabilidade geral como alpha, beta, beta1, beta2...Consigo testar localmente e quando estiver tudo okay, subo a lib para o registry. Para fazer isso, vamos usar o Docker e o Docker Compose 😉.

## Bora ver?!

Primeiro vamos precisar configurar o verdaccio com um pequeno arquivo de configuração. Então vamos criar um repositório:

```bash
# Criar um diretório para nosso projeto
mkdir verdaccio-fun
# Acessar o diretório de nosso projeto
cd verdaccio-fun

# criar:
# - um diretório para nossa lib
# - um para nosso app
# - um para as configurações do verdaccio.
mkdir -p mylibz/src myappz/src verdaccio/conf

# abrir o VSCode no diretório do projeto
code .
```

## Primeiro, vamos configurar o Verdaccio como um registry local para pacotes com o prefixo **@gedai/\***:

> ----- Os próximos passos devem ser executados no diretório dedicado ao verdaccio: verdaccio -----

```bash
# Criar o arquivo de configuração do verdaccio
touch conf/config.yaml
# Criar usuário admin no verdaccio
# Embora não vamos utilizar, ambos usuário e senha são: verdaccio
echo "verdaccio:$apr1$boet6w2f$04kd3nnbp9s/BtesXZc7I0" > conf/htpasswd
```

E agora vamos adicionar o conteúdo do arquivo de configuração do verdaccio:

```yaml
storage: /verdaccio/storage

auth:
  htpasswd:
    file: ./htpasswd

uplinks:
  npmjs:
    url: https://registry.npmjs.org

packages:
  "@gedai/*":
    access: $all
    publish: $all
    unpublish: $all
  "@*/*":
    access: $all
    proxy: npmjs
  "**":
    access: $all
    proxy: npmjs

logs:
  - { type: stdout, format: pretty, level: http }
```

Nesse exemplo não vamos trabalhar com autenticação, então o acesso ao nosso registry está liberado com a flag **$all**. Se quiser, você pode ir lá na [doc do Verdaccio](https://verdaccio.org/pt-BR/docs/configuration) ver como faz para configurar um acesso mais restrito. Teremos apenas um usuário verdaccio com senha verdaccio, mas nem vamos utilizar. Se você quiser criar outros usuários também pode procurar na web qualquer gerador de htpasswd para criar seu próprio usuário e senha.

### Adicionando um usuário admin ao Verdaccio, você pode autenticar usando verdaccio como usuário e senha no browser e também no npm.

## Agora vamos criar nosso Compose para subir o ambiente:

> ----- Os próximos passos devem ser executados na raiz do projeto: verdaccio-fun -----

Vamos usar um docker-compose.yaml para montar nosso ambiente local, eu gosto de deixar o compose na raiz do projeto mesmo, assim caso eu precise consigo utilizar um mesmo compose para subir várias aplicações:

```bash
touch docker-compose.yaml
```

```yaml
version: "3"

networks:
  skynet:
    driver: bridge

services:
  verdaccio:
    image: verdaccio/verdaccio:5.24
    container_name: verdaccio
    environment:
      - VERDACCIO_PORT=4873
    ports:
      - 4873:4873
    networks:
      - skynet
    volumes:
      - "./verdaccio/conf:/verdaccio/conf"
```

## Agora vamos criar o código fonte de nossa lib lá no diretório mylibz:

> ----- Os próximos passos devem ser executados no diretório dedicado à lib: mylibz -----

Primeiro um package.json:

```bash
# Criar um package.json com escopo e nome
npm init -y --scope=@gedai --name=mylibz
```

E vamos fazer alguns ajustes para que o nosso package comporte a publicação local, as chaves que serão adicionadas são: **main**, **files**, **private**, e **publishConfig.registry**. Também aproveitamos para remover qualquer conteúdo de scripts. Nosso package.json ficará da seguinte maneira:

```json
{
  "name": "@gedai/mylibz",
  "version": "1.0.0",
  "description": "",
  "main": "src/index.js",
  "files": ["src/"],
  "private": false,
  "publishConfig": {
    "registry": "http://localhost:4873"
  },
  "scripts": {},
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

E adicionamos o código fonte de nossa lib lá em src/index.js

```javascript
function getGreeting(name) {
  return `Hello, ${name || "World"}!`;
}

module.exports = { getGreeting };
```

Agora para realizar a publicação é necessário primeiro apontar o npm para o registry correto com um .npmrc:

```bash
# cria .npmrc com escopo:
echo '@gedai:registry="http://localhost:4873"\n//localhost:4873/:_authToken="not-needed"' > .npmrc
```

E agora é só publicar a versão:

```bash
npm publish
```

Se quiser apagar uma versão basta:

```bash
npm unpublish -f
```

## Consumir a lib agora é uma tarefa tão fácil quanto publicar e republicar:

> ----- Os próximos passos devem ser executados no diretório dedicado ao app: myappz -----

Vamos primeiro criar o package da nossa aplicação e o entrypoint lá no src:

```bash
# Criar um package.json com escopo e nome
npm init -y --name=myappz

# app entrypoint
touch src/main.js
```

Agora vamos configurar o npm para usar o nosso registry local:

```bash
# cria .npmrc com escopo:
echo '@gedai:registry="http://localhost:4873"\n//localhost:4873/:_authToken="not-needed"' > .npmrc
```

Instlar a nossa lib:

```bash
npm install @gedai/mylibz
```

Vamos ajustar o package com o main de nosso app e o start script. O package.json ficara da seguinte maneira:

```json
{
  "name": "myappz",
  "version": "1.0.0",
  "description": "",
  "main": "src/main.js",
  "scripts": {
    "start": "node ."
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@gedai/mylibz": "^1.0.0"
  }
}
```

E agora é partir para o código fonte lá em src/main:

```javascript
const { getGreeting } = require("@gedai/mylibz");

function main() {
  const defaultGreeting = getGreeting();
  const userGreeting = getGreeting("Jack");

  console.log({ defaultGreeting, userGreeting });
  // { defaultGreeting: 'Hello, World!', userGreeting: 'Hello, Jack!' }
}

main();
```

E tudo certo! Você agora tem um servidor de testes para publicação com NPM que pode usar para testar versões instáveis nos seus projetos antes de entregar a versão estável em produção. Também pode aproveitar para garantir que os contratos estão corretos, índices de importação ou até mesmo começar a brincar com publicacão de libz com Typescript que precisam de processos de build!

O projeto completo está lá no github. [Verdaccio Fun](https://github.com/gm50x/verdaccio-fun)

## Limpeza do ambiente:

```bash
docker compose down -v
```
