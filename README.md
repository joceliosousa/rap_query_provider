# Construindo APIs que acessam estruturas complexas em RAP
## Deep entity com Custom entity e RAP query provider

Já construi algumas APIs ou mesmo expus CDS custom entity para retornar algo, tipo um arquivo PDF, para o frontend. Mas e se você precisar retorar várias estruturas?
Como sabemos, criando custom entity e implementando Query provider é bem simples para retornar apenas uma estrutura.

E as chamadas Deep Entities ?

[!IMPORTANT]
Nesta abordagem utilizaremos os recursos do OData, com o GET. Neste cenário não é possível
passar um xml/json no body, mas podemos utilizar os tipos de parâmetros que o protocolo
oferece, como filters, expand, etc.

Utilizaremos CDS, tipo custom Entity, e a implementação da leitura dos parâmetros de entrada e
retorno dos parâmetros de saída será implementada via classe RAP query provider.

![image](https://github.com/user-attachments/assets/9261ea40-e047-4a9f-9a84-ae6121768e42)
