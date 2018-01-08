[Voltar a home](../../readme.md)
___
<img src=".imgantifraude/logoblu365.png" alt="Logo BLU365" title="BLU365" align="right" height="60"/>

Automação Preventivo
======

## 1 Histórico do documento

Versão | Data | Comentário
---|---|---
1.0 | - | -

## 2 Sumário
1. [Histórico do documento](#1-histórico-do-documento)
2. [Sumário](#2-sumário)
3. [Visão geral](#3-visão-geral)
4. [Como funciona a automação do Preventivo](#4-como-funciona-a-automacao-do-preventivo)

## 3 Visão geral

#### Responsáveis pelo projeto:

- **Geração das querys e exportações**: Andre Petridis (Grego)
- **Criação do processo no Ni-Fi**: Andre Petridis (Grego)
- **Criaçao dos segmentos e réguas**: Roberto Keike Kogake

#### O que é?

A automação dos disparos de e-mail de Preventivo foi criada para melhorar a comunicação com os clientes finais (devedores) e o principal objetivo é diminuir o percentual de queba de acordo e o trabalho manual do squad de performance.

## 4 Como funciona a automação

### Iniciando a automação
O inicio da automação é através de uma query que busca todos os acordos gerados (primeira parcela e acordos gerados) e disponibiliza as informações em um SFTP onde a Dinamize/Mail2Easy Pro busca os arquivos em horários específicos.

### Importando os arquivos

A ferramenta busca os arquivos sempre na mesma pasta, com o nome padrão e com horários pré definidos.

Existe uma pasta no SFTP com os arquivos que serão importados:

 ###### Estrutura das pastas
 ![authorization](.img-auto-prev/pastas-sftp.png)

### Criando o authorizer na AWS

 Todas as lambdas que utilizarão o sistema anti-fraude, antes de serem invocadas pelo api gateway devem passar pelo gondul_authorizer, que vai validar a requisição e se estiver correta permite que o api gateway invoque a lambda normalmente.

 ###### Exemplo de onde ele é aplicado
 ![authorization](.img-auto-prev/authorization_midware_lambda.png)

 ### Lambda Gondul_authorizer

 Ela é responsável por descriptografar o token recebido, pegar as informações de IP, Email, fingerprint e começar a válidar.
 Ele segue uma ordem de validação.
  1. Checa se o **ip** consta na WhiteList, se constar pula todos os próximos passos e libera o acesso.
  2. Chega se o **domain** consta na WhiteList, se constar pula todos os próximos passos e libera o acesso.
  3. Checa se o **fingerprint** consta na BlackList, se não constar vai para a próxima validação.
  4. Checa se o **ip** consta na BlackList, se não constar vai para a próxima validação.
  5. Checa se o **email** consta na BlackList, se não constar vai para a próxima validação.

Vale lembrar que a WhiteList são os cadastros de ip e domain, efetuado pela equipe de ti, que podem fazer requisições sem restrição, como é exemplo dos testes da BLU365 e parceiros, já a BlackList são todos aqueles que não podem prosseguir no processo de geração de acordo.

A WhiteList é cadastrada pela equipe de TI, mas e como é gerado a BlackList?

### Blacklist

Tanto a BlackList ou a WhiteList ficam localizadas no Redis da AWS.

O processo de geração de blacklist começa desde a primeira chamada no **auth**, quando o usuário está validando o captcha desde a primeira requisição, se ele acertar o desafio, acertando ou errando este dado é computado no Kinesis somando niveis para os ip, fingerprint e email.

São níveis de 1 a 5, enquanto o usuário possuir nível 4 ou menos ele pode tentar acertar o captcha para acessar o sistema, mas no momento que ele chega ao nivel 5 a informações são incluidas automaticamente na blacklist do sistema.

### Kinesis Streamer

Quando o kinesis streamer **gondul_input** é invocado ele manda as informações para um kinesis analytics que que processa essas informações e manda para outro kinesis streamer chamado **gondul_output**. 

Este kineses de output está vinculado a uma lambda chamada **judgment**, que recebe este lote de informações as transforma de base64 para um json, filtra este json para retirar as repetições da lista e os que constam na WhiteList, tudo o que sobrar é salvo na BlackList e depois manda para o firehose.


