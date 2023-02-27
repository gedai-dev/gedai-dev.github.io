---
layout: post
title:  "Golang - Escrevendo logs utilizando a biblioteca Zap da Uber"
categories: [Golang, bibliotecas]
keywords: [uber-go/zap]
pin: false
date: 2023-02-23T00:00:00-03:00
author: Bruno Melo
math: true
tags: []
---

# Implemente logs estruturados em sua apliçacão utilizando a biblioteza `zap` da Uber

>Se você tem dificuldade para entender o comportamento da sua aplicação em ambiente produtivo, e precisa depurar sua aplicação com frequência para tentar replicar um cenário que tenha ocorrido em produção, recomendo a leitura dos artigos abaixo para entender a importância dos pilares da observabilidade e como a adoção dessa prática irá te ajudar a obter um ambiente observável e monitorável.

[Em breve]()


# Escrevendo logs utilizando a biblioteca Zap da Uber

Registrar *logs* é legal, registra-lós bem é melhor ainda! para isso é preciso estruturar o nosso log e nada melhor que reaproveitar o esforço de alguém para economizar tempo, e nesse caso iremos reaproveitar os esforços da Uber e escrever logs igual a eles.

>Eu poderia adicionar varias linhas neste artigo para argumentar a motivação que me fez utilizar essa biblioteca, mas irei resumir em *desempenho* e *simplicidade*. Para saber mais navegue até a pagina oficial do [projeto](https://github.com/uber-go/zap).


Siga os passos abaixo para ter sucesso na sua implementação:

#### 1 - Importe o pacote zap

```sh
 
 go get go.uber.org/zap

```

#### 2 - Para estruturar o nosso log será preciso inicializar a struct `Config` do pacote zap, no exemplo abaixo eu irei alterar alguns campos para customizar o log e deixa-lo rico o suficiente para ter uma visão melhor do evento.

>Inicializar a struct `zap.Config` é importante somente em casos onde você precisa sobrescrever alguma propriedade que é impressa nos logs, o pacote zap disponibiliza duas formas para início rápido (quick start) desse pacote, são eles: `SugaredLogger` e `Logger` a maior diferença entre eles é o desempenho onde o `Logger` é mais rápido que `SugaredLogger` e aloca menos memoria, `SugaredLogger` é 4-10x mais rápido do que outros pacotes de log, então, quando o contexto pedir um bom desempenho, mas nada tão crítico, use o `SugaredLogger`, se o desempenho e a segurança de tipo forem críticos, use o `Logger`.

```go

// This constant values will be used in zap.Config bellow
const (
	timeKey          string = "Timestamp"
	nameKey          string = "Key"
	levelKey         string = "Severity"
	encoding         string = "json"
	CallerKey        string = "Caller"
	messageKey       string = "Body"
	outputPaths      string = "stdout"
	stacktraceKey    string = "Trace"
	errorOutputPaths string = "stderr"
)

zapConfig := &zap.Config{
	Development: false,
	Level:       zap.NewAtomicLevelAt(zap.InfoLevel),
	Encoding:    encoding,
	EncoderConfig: zapcore.EncoderConfig{
		MessageKey:     messageKey,
		LevelKey:       levelKey,
		TimeKey:        timeKey,
		NameKey:        nameKey,
		CallerKey:      CallerKey,
		FunctionKey:    zapcore.OmitKey,
		StacktraceKey:  stacktraceKey,
		SkipLineEnding: false,
		LineEnding:     zapcore.DefaultLineEnding,
		EncodeLevel:    zapcore.CapitalLevelEncoder,
		EncodeTime:     zapcore.RFC3339TimeEncoder,
		EncodeDuration: zapcore.MillisDurationEncoder,
		EncodeCaller:   zapcore.ShortCallerEncoder,
		EncodeName:     zapcore.FullNameEncoder,
	},
	OutputPaths:      []string{outputPaths},
	ErrorOutputPaths: []string{errorOutputPaths},
	InitialFields: map[string]interface{}{
		"Attributes": map[string]interface{}{
			"service.name":    "gedai",
			"service.version": "v1.0.0",
			"time":            time.Now().UTC(),
		},
		"Annotations": map[string]interface{}{
			"team":     "team-gedai",
			"contact":  "gedai-contact",
			"handbook": "http://handbook.io",
		},
	},
}

// This Variable will be used to write logs stdout
var logger *zap.SugaredLogger

_logger, err := zapConfig.Build(zap.AddCaller(), zap.AddCallerSkip(1))
if err != nil {
	panic(err)
}

logger = _logger.Sugar()

```

#### 3 - Pronto! vamos ver o resultado ao executar as funções de log:

```go


logger.Info("Hi there! I`m a log kind of INFO ...")

logger.Warn("Hey pay attention! I`m a log kind of WARN :o ...")

logger.Error("OH NO! I`m a log kind of ERR :o ...")

```

```sh
{"Severity":"INFO","Timestamp":"2023-02-26T22:06:29-03:00","Caller":"runtime/proc.go:250","Body":"Hi there! I`m a log kind of INFO ...","Annotations":{"contact":"gedai-contact","handbook":"http://handbook.io","team":"team-gedai"},"Attributes":{"service.name":"gedai","service.version":"v1.0.0","time":"2023-02-27T01:06:29.654943Z"}}

{"Severity":"WARN","Timestamp":"2023-02-26T22:06:29-03:00","Caller":"runtime/proc.go:250","Body":"Hey pay attention! I`m a log kind of WARN :o ...","Annotations":{"contact":"gedai-contact","handbook":"http://handbook.io","team":"team-gedai"},"Attributes":{"service.name":"gedai","service.version":"v1.0.0","time":"2023-02-27T01:06:29.654943Z"}}

{"Severity":"ERROR","Timestamp":"2023-02-26T22:06:29-03:00","Caller":"runtime/proc.go:250","Body":"OH NO! I`m a log kind of ERR :o ...","Annotations":{"contact":"gedai-contact","handbook":"http://handbook.io","team":"team-gedai"},"Attributes":{"service.name":"gedai","service.version":"v1.0.0","time":"2023-02-27T01:06:29.654943Z"}}

```

Para visualizar este exemplo completo clique aqui: [gedai-dev/go-uber-zap-sample](https://github.com/gedai-dev/go-uber-zap-sample)


### Observações do autor

Eu faço uso desse pacote há muito tempo, e por mais que assunto pareça ser simples — e é — saiba que estruturar seus logs e padroniza-lo em todos os seus serviços pode parecer uma tarefa sem valor ou até mesmo chata e isso faz com que a informação que você necessita para entender o comportamento do seu sistema seja fraca e pobre de informação e isso não te ajudará em nada quando o teu chefe te ligar de madrugada para entender a mensagem do tipo "deu erro aqui, entrar em contato com o Kioto", então trate essa peça do seu sistema com a importância da camada de persistência, pois são os logs que irão te dizer o que ocorreu, quando ocorreu e o rastro de um determinado evento.


