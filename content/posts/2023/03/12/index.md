---
layout: post
title: "Logs de aplicações NestJS"
date: 2023-02-23T00:00:00-03:00
author: gm50x
tags: [NestJS, Logging]
---

Trabalhar com logs é sempre uma tarefa bastante básica de qualquer aplicação, mas nem sempre a gente consegue o resultado que espera. Aqueles logs bonitos, coloridos e super bem explicados que tinhamos no nosso console fica difícil de entender quando fazemos o deploy para o ambiente de produção. Isso, sem contar na dificuldade de amarrar os logs de um fluxo completo numa arquitetura de micro-serviços.

## Problemas? Oba!

O primeiro problema é que, os logs padrões do nest logger são logs em formato de texto, que atendem muito bem a necessidade do desenvolvedor, mas fica um pouco difícil de amarrar quando utilizamos alguma ferramenta para agregar todos os logs num lugar só. Procurar por trechos de texto num Kibana, Grafana, DataDog e etc nem sempre é a tarefa mais simples, especialmente quando resolvemos colocar algumas partes da mensagem de forma customizada.
Um outro problema, é que hoje com microserviços um fluxo começa no serviço a, e percorre uma série de outros serviços até que seja concluído de verdade, e amarrar esses logs nem sempre é a tarefa mais óbvia do mundo. Isso porque geralmente estamos olhando para os logs no nosso console enquanto programamos a aplicação e sabemos exatamente em qual parte do fluxo estamos e qual recurso estamos acessando. Mas basta a gente jogar tudo num lugar só que a coisa começa a ficar difícil.

## Então como resolvemos esses problemas?

O primeiro passo é sempre escolher um logger de nível de produção, capaz de trabalhar com diversos formatos além de texto. Isso porque, nas ferramentas modernas temos vários recursos de filtro que funcionam muito bem com JSON. E logar no formato JSON nos habilita a possibilidade de filtrar por qualquer chave dentro do JSON, inclusive por propriedades aninhadas e listas.

Aqui, estamos utilizando o Winston devido a flexibilidade que ele possui e, para não perdermos a beleza dos logs em tempo de desenvolvimento, também incorporamos a lib nest-winston. Isso nos permite passar apenas um arquivo bem simples de configuração e voilà! Tudo está funcionando como esperamos em qualquer ambiente que a aplicação estiver rodando.

O setup fica bem simples, instalar as dependências necessárias, adicionar um arquivo de configuração do logger e injetar o logger do winston no lugar do logger padrão do nestjs.

## Bora ver o código então:

```bash
# Lembre-se de instalar as dependências antes de qualquer tentativa.
npm install winston nest-winston
```

```ts
// logger.config.ts
import {
  WinstonModuleOptions,
  utilities as nestWinstonUtils,
  WinstonModule,
} from "nest-winston";
import * as winston from "winston";

const { Console } = winston.transports;
const { combine, timestamp, json } = winston.format;
const { nestLike } = nestWinstonUtils.format;

// Adicionamos uma chave severity em todos os logs, para simplificar filtro configuração no Google Cloud.
const levels: object = {
  default: "DEFAULT",
  debug: "DEBUG",
  info: "INFO",
  warn: "WARNING",
  error: "ERROR",
};
const severity = winston.format((info) => {
  const { level } = info;
  return Object.assign({}, info, { severity: levels[level] });
});

// Um formato diferente para desenvolvimento vs produção
const remoteFormat = () => combine(severity(), json());
const localFormat = () => combine(timestamp(), severity(), nestLike());

export const createNestLogger = () => {
  // Determinamos se vamos usar o logger em formato local (coloridinho, com texto)
  // ou o logger em JSON puro adequado para filtragem nos serviços de log
  const useLocalFormat =
    process.env.NODE_ENV === "development" ||
    process.env.LOGGER_FORMAT === "local";

  // também verificamos se devemos silenciar os logs, para execução de pipelines de teste por exemplo
  const silent = Boolean(process.env.CI) || process.env.NODE_ENV === "test";

  // E pronto, já temos tudo que precisamos para criar o WinstonLogger.
  const loggerConfig: WinstonModuleOptions = {
    levels: winston.config.npm.levels,
    level: process.env.LOG_LEVEL || "info",
    format: useLocalFormat ? localFormat() : remoteFormat(),
    //usamos console porque nossa núvem faz a coleta dos logs para a gente, sem a necessidade de usar algum transport específico
    transports: [new Console()],
    silent,
  };

  return WinstonModule.createLogger(loggerConfig);
};
```

```ts
// main.ts
async function bootstrap() {
	// Basta instanciar o logger e passar para o NestFactory na hora de criar o app.
  const logger = createNestLogger();
  const app = await NestFactory.create(AppModule, { logger })
  await app.listen(3000);
  logger.log(`🐋 gedai is listening on :${port}`, 'main');
```

### Prontinho! Agora já temos um logger production grade que fica bonitinho na maquina do dev e adequado para as necessidades de produção da maioria das aplicações de agregação de logs. Num próximo post falamos sobre como correlacionar os logs entre diversos microserviços de uma forma que fique simples de amarrar todo um fluxo. Valeu e até mais!

