---
layout: post
title: "Golang: Implementando testes de unidade utilizando aws-sdk-v2"
date: 2023-04-15T00:00:00-03:00
author:
  name: Bruno Melo
  url: "https://github.com/brbarmex"
tags: [Go, Golang, Unit-Test,aws-sdk-v2]
---


Atire a primeira pedra quem nunca disse durante a daily: "tá pronto, só falta testar" e esse teste esta durante até o hoje?

Certamente você, assim como eu, sabe a importância dos testes no processo de engenharia de software e que essa atividade esta longe de ser simples se comparar a expressão da palavra com a própria ação. Bem, o proposito deste post é para orientar a como implementar testes de unidade utilizando a biblioteca [aws-sdk-go-v2]("https://github.com/aws/aws-sdk-go-v2") na linguagem GO.

![foto retirada da internet](./images/ta-pronto-falta-testar.png)

## AWS SDK

Em resumo, o AWS SDK é um pacote que permite interagir com os serviços da AWS (Amazon Web Services) usando uma linguagem de programação. Este SDK foi projetado para simplificar o desenvolvimento de aplicacões que utilizam serviços da AWS, fornecendo uma interface de programação fácil de usar.

O AWS SDK for Go tem duas versões principais: *v1* e *v2*. Basicamente na v2 teremos a evolucao de algumas deficiências da versão anterior (*v1*), ela foi construída em torno de uma abordagem mais moderna baseada em contextos e utilizando recursos mais recentes da linguagem Go, contém suporte para contextos do Go para cancelamento de solicitações e gerenciamento de tempo de vida além de recursos aprimorados para configuração e autenticação.

 *"Blz Brunão, mas o que tem haver a diferenca entre as versões com os testes de unidade ?"*

 *"Tudo meu caro... tudo ..."*

Bem, na versão 1 do sdk tinhamos acesso a **interface** de cada servico da aws o que facilitava a implementacao dos testes de unidade. Na versão atual essa interface já não existe, quando você inicializa algum servico o sdk te retornará uma referencia **concreta** da struct tornando então impossivel a possibilidade de criar **mocks** diretamente igual era na versão anterior.

*"Eu utilizei o paradigma OOP por muito tempo e não me enxergava sem interface... quando eu via alguma classe de servico sem interface eu dizia "como você vai testar?" e ganhava o argumento simplesmente pelo fato de que uma classe sem interface era uma classe sem teste (na maioria das vezes), mas isso mudou quando comecei ter contato com outras religiões (**NodeJs**, **Python**, **GO** e um pouco de **Rust**) e cara, hoje eu entendo que se você utiliza interface somente para facilitar os testes de unidade é sinal que teu design pode melhorar, mas isso eu explicarei num post futuro"*

## Show me the code!

Antes, vamos entender o problema que iremos resolver. O diagrama abaixo representa a jornada do nosso app de exemplo.
O nosso app validará se um metadata vindo do SQS é valido e persistirá ele no DynamoDB se ele for valido.

```
       ┌────┐                                 
       │Main│                                 
       └──┬─┘                                 
 ┌────────▽───────┐                           
 │Metadata Service│                           
 └────────┬───────┘                           
  ┌───────▽───────┐                           
  │Receive Message│                           
  │From SQS       │                           
  └───────┬───────┘                           
  ________▽________                           
 ╱                 ╲             ┌───┐        
╱ Metadata Is Valid ╲____________│Yes│        
╲                   ╱yes         └─┬─┘        
 ╲_________________╱    ┌──────────▽─────────┐
          │no           │Put Item in DynamoDB│
  ┌───────▽──────┐      └────────────────────┘
  │Printf message│                            
  └──────────────┘                            
```


### Caminho tradicional utilizando interface:





