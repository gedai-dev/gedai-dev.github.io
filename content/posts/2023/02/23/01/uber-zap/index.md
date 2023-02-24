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
draft: true
---

# Implemente logs estruturados em sua apliçacão utilizando a biblioteza `zap` da Uber

>Se você tem dificuldade para entender o comportamento da sua aplicacão em ambiente produtivo, e precisa depurar sua aplicacão *localmente* com frequencia para tentar replicar um cenário que tenha ocorrido em producão por exemplo, recomendo a leitura dos posts abaixo para entender a importancia dos pilares da obervabilidade e como a adocão dessa pratica irá te ajudar a obter um ambiente observavel e monitorável.

    -   [Observabilidade](#)


# Escrevendo logs utilizando a biblioteca Zap da Uber

Registrar *logs* é legal, registra-lós bem é melhor ainda! e para isso é preciso estruturar o nosso log e nada melhor que reaproveitar o esforco de alguem para economizar tempo, e nesse caso iremos reaproveitar os esforcos da Uber galera, vamos escrever logs igual a eles hehe.

Eu poderia adicionar varias linhas neste post para argumentar a motivacao que me fez utilizar essa biblioteca mas irei resumir em *performance*, e *simplicidade*. Para saber mais navegue até a pagina oficial do [projeto](https://github.com/uber-go/zap)./



