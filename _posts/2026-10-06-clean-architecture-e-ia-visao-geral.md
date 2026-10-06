---
layout: post
title: "Clean Architecture & IA: visão geral da série"
description: "Por que restrições automatizadas valem mais que revisão linha por linha, e o que esta série vai cobrir."
serie: "Clean Architecture & IA"
ordem: 0
---
Esta é a abertura da série **Clean Architecture & IA: Engenharia de Restrições na Era dos LLMs**.

## A tese

Agentes geram código mais rápido do que qualquer pessoa consegue revisar linha por linha. A saída não é abrir mão da revisão, e sim **automatizar tudo o que é verificável** (estrutura, tipos, comportamento e qualidade dos testes) e reservar o olhar humano para o que a máquina não verifica: intenção, modelagem de domínio e trade-offs.

## Artigos

1. O fim da revisão linha por linha
2. Arquitetura de restrições no backend Java (Spring Boot e legado Java EE)
3. Fronteiras e contratos no frontend Angular
4. Testes estritos como quality gates
5. O manifesto do engenheiro orquestrador

Todos os exemplos evoluem em um mesmo repositório, com código que você pode rodar.

```java
// Exemplo de restrição: o domínio não conhece o Spring
@ArchTest
static final ArchRule dominio_sem_spring =
    noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAPackage("org.springframework..");
```
