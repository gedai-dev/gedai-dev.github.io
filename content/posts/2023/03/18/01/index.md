---
layout: post
title: "Passagem implícita de argumentos em NodeJS"
date: 2023-03-18T00:00:00-03:00
author:
  name: Getúlio Magela Silva
  url: "https://github.com/gm50x"
tags: [NodeJS, AsyncLocalStorage]
---

Você já se deparou com a necessidade de ficar propagando algumas propriedades e argumentos para dentro do core de suas aplicações que só são usados na infraestrutura?

## Um pouco de contexto do NodeJS

Em NodeJS é muito comum trabalhar com singletons para uma série de atividades realizadas pelas nossas aplicações. Por exemplo, podemos utilizar um logger central criado com winston, um único cliente HTTP com axios e uma única model do mongoose para realizar comunicação com o banco de dados. Esses recursos serão reutilizados a cada requisição que nossa aplicação receber durante seu ciclo de vida.

Nesse ponto você já deve estar pensando que eu estou me referindo exclusivamente à uma API, seja ela Rest, GraphQL ou qualquer outro sabor que queira adicionar, mas nem sempre. Podemos criar um cron job ou um consumidor de alguma fila e esses argumentos também seriam válidos, a aplicação ficaria de pé, esperando algum evento para executar uma tarefa e a cada tarefa ela pode reutilizar os recursos que já utilizou anteriormente.

## Problemas? Oba!

Essa forma Singleton de se trabalhar com o NodeJS gera alguns problemas de arquitetura, pois, não seria legal permitir que qualquer parte da nossa aplicação tenha acesso a qualquer coisa que estiver presente nela. Isso porque, quando estamos desenvolvendo software, temos a boa prática de separar as responsabilidades de cada parte da nossa aplicação, sair passando um parâmetro exclusivo de infraestrutura para todo canto, não é nada legal. E também, não queremos sair recriando todos os recursos da nossa aplicação a cada novo evento que ela receber. Por exemplo, se um request-scoped no NestJS receber muitas requisições em paralelo consome muito mais memória RAM do que se ele trabalhasse com singletons que atendem a quase todos os cenários.

Então como fazer para passar um parâmetro lá de infraestrutura para uma outra parte da infraestrutura que será acioanda por meio do core de nossas aplicações? No NodeJS existe um recurso no módulo de Async Hooks chamado Async Local Storage que nos permite compartilhar algumas informações com qualquer outra parte de nossa aplicação que estiver sendo executada num **mesmo contexto assíncrono**. É parecido com Thread Local Storage, mas sem as threads. Qualquer parte da aplicação dentro desse contexto terá acesso a esse armazenamento que é único para cada ramificação assíncrona que você estiver executando.

## Ah tá! Quero ver! Então bora!

Vamos primeiro criar um novo repositório NodeJS puro.

```bash
mkdir async-hooks-sample
cd async-hooks-sample
git init && \
  echo "console.log('start');" > main.js && \
  npm init -y && \
  git add . && \
  git commit -m "initial commit"
```

Agora vamos editar nosso main.js para criar nosso espaço assíncrono. E uma função assíncrona que utilizará este espaço.

```javascript
// main.js
// Criamos nosso espaço assíncrono
const { AsyncLocalStorage } = require("async_hooks");
const storage = new AsyncLocalStorage();

// Alguma atividade que será realizada na nossa aplicação
async function doSomethingAsynchronously() {
  // obtemos o estado de dentro do async local storage
  const data = storage.getStore();

  // e podemos utilizar esse espaço sem crise
  console.log(`Doing something asynchronously ${data}`);
}

// Nosso escopo principal
async function main() {
  for (let i = 0; i < 3; i++) {
    const someState = i + 1;
    console.log(`Running ${someState}`);
    // Executamos alguma função assíncrona passando o estado correspondente
    storage.run(someState, doSomethingAsynchronously);
  }
}
main();
```

A saida da nossa aplicação será:

```bash
Running 1
Doing something asynchronously 1
Running 2
Doing something asynchronously 2
Running 3
Doing something asynchronously 3
Running 4
Doing something asynchronously 4
Running 5
Doing something asynchronously 5
```

### Mas cuidado, nem tudo são flores!

Esta forma de passagem de argumentos entre camadas da aplicação pode ser difícil de compreender. Então é recomendável apenas para atividades periféricas e nunca como mecanismo central das aplicações. Por exemplo, você pode utilizar o AsyncLocalStorage num middleware do express para setar algum trace-id ou correlation-id na sua aplicação e aproveitar esse identificador no seu logger e no seu http client para propagar esse mesmo id para outros serviços. Um outro cenário em que esta técnica poderia ser bastante útil é para aplicações multi-tenant, em que você não quer ficar passando o tenant para todo lugar da aplicação, apenas para as partes de infra que precisam filtrar seus dados pelo tenant.

### Um exemplo com express por favor? Claro!

Vamos adicionar o express à nossa aplicação

```bash
npm install express
```

E agora vamos alterar nosso main para que se transforme numa API.
_OBS: Por simplicidade vamos deixar tudo num lugar só._

```javascript
// main.js
const { randomUUID } = require("crypto");
const { AsyncLocalStorage } = require("async_hooks");
const express = require("express");

// Criação do app
const app = express();

// Inclusão de middlewares
app.use(express.urlencoded({ extended: true }));
app.use(express.json());

// criação do storage
const storage = new AsyncLocalStorage();

// Logging utility
const log = (message, severity = "INFO") => {
  // Obtemos o requestId de qualquer lugar da nossa aplicação
  const requestId = storage.getStore() || "";
  console.log(severity.padStart(10, " "), requestId.padStart(36, " "), message);
};

// agora sempre que executarmos qualquer coisa com log que foi originado
// no request teremos nosso requestId disponível
// e podemos inclusive obter de volta para utilizar e passar um header a outros serviços
function doSomethingAsynchronously() {
  log("doing something asynchronously now");

  return "All Good!";
}

// inclusão de middleware para executar nossos handlers com o storage associado.
// e geração de requestId para cada request, ou utilizar um request id passado por headers.
app.use((req, _res, next) => {
  const requestId =
    req.requestId ?? req.headers["x-request-id"] ?? randomUUID();
  storage.run(requestId, next);
});

// montagem dos endpoints do app
app.get("", async (_req, res) => {
  log("got in");

  const data = await doSomethingAsynchronously();

  res.send(data);
});

// inicializa o app
app.listen(3000, () => console.log("running on localhost:3000"));
```

Agora é só chamar a aplicação e vòila! Temos um request id propagado por toda a nossa aplicação sem sujar o core com parâmetros exclusivos de infra!

```bash
curl localhost:3000
# ou com a passagem de um header
curl localhost:3000 --header="x-request-id: request123"
```
