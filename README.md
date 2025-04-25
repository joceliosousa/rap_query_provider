# Construindo APIs que acessam estruturas complexas em RAP
## Deep entity com Custom entity e RAP query provider

Já construi algumas APIs ou mesmo expus CDS custom entity para retornar algo, tipo um arquivo PDF, para o frontend. Mas e se você precisar retorar várias estruturas?
Como sabemos, criando custom entity e implementando Query provider é bem simples para retornar apenas uma estrutura.

E as chamadas Deep Entities ?

!IMPORTANT
Nesta abordagem utilizaremos os recursos do OData, com o GET. Neste cenário não é possível
passar um xml/json no body, mas podemos utilizar os tipos de parâmetros que o protocolo
oferece, como filters, expand, etc.

Utilizaremos CDS, tipo custom Entity, e a implementação da leitura dos parâmetros de entrada e
retorno dos parâmetros de saída será implementada via classe RAP query provider.

![image](https://github.com/user-attachments/assets/9261ea40-e047-4a9f-9a84-ae6121768e42)

Exemplo de utilização dos parâmetros em uma chamada GET a um serviço OData standard.
Para utilização desta arquitetura precisaremos criar uma CDS Root tipo Custom Entity com uma
ou mais filhas. Essa estrutura é um padrão OData que deve ter um header e um ou mais estrutura
de itens, tipo tabela (array).

Página | 2
Nissan Internal
Nosso cenário: Utilizaremos duas tabelas de retorno, com dois tipos diferentes, e utilizaremos
filtros.
1. Crie a CDS Root, tipo Custom Entity, incluindo os campos que serão utilizados como
filtros (parâmetros de entrada) e as filhas _Item e _Msg (tabelas que serão retornadas).
Utilizaremos um parâmetro simples (não é range), chamado p_param1
Cenários simples com campos não complexos podem ser passados no with
parameters, ou pode mesclar parâmetros simples com filtros.

```
@ObjectModel.query.implementedBy: &#39;ABAP:ZMHL_CL_CUSTOM_ENTITY&#39;
@EndUserText.label: &#39;Header&#39;
define root custom entity ZMHL_i_Custom_Header
with parameters
p_param1 : char1
{
key HeaderID : abap.char( 5 );
HeaderType : abap.char( 2 );
status : abap.char( 1 );
_Item : composition [1..*] of ZMHL_I_Custom_Item;
_Msg : composition [1..*] of ZMHL_I_CUSTOM_MSG;
}
```

Deve existir uma composition, 1 para vários, com o tipo de estrutura (CDS) que será
retornada.

2. Salve a CDS criada, mas ainda não as ative. Crie as outras CDSs que serão retornadas
como uma tabela com várias linhas.
Para o meu cenário ZMHL_I_Custom_Item e ZMHL_I_CUSTOM_MSG.

```
@ObjectModel.query.implementedBy: &#39;ABAP:ZMHL_CL_CUSTOM_ENTITY&#39;
@EndUserText.label: &#39;Item&#39;
define custom entity ZMHL_I_Custom_Item
{
key HeaderID : abap.char( 5 );
key Item : abap.numc( 2 );
Description : abap.char( 40 );
_Header : association to parent ZMHL_I_Custom_Header on _Header.HeaderID =
$projection.HeaderID;
}


@ObjectModel.query.implementedBy: &#39;ABAP:ZMHL_CL_CUSTOM_ENTITY&#39;
@EndUserText.label: &#39;Item&#39;
define custom entity ZMHL_I_CUSTOM_MSG
{
key HeaderID : abap.char( 5 );
key Item : abap.numc( 2 );
Message : abap.char( 40 );
_Header : association to parent ZMHL_i_Custom_Header on _Header.HeaderID =
$projection.HeaderID;
}
```
