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

_Nosso cenário: Utilizaremos duas tabelas de retorno, com dois tipos diferentes, e utilizaremos
filtros._

### 1. Crie a CDS Root, tipo Custom Entity, incluindo os campos que serão utilizados como
filtros (parâmetros de entrada) e as filhas _Item e _Msg (tabelas que serão retornadas).

_Utilizaremos um parâmetro simples (não é range), chamado p_param1
Cenários simples com campos não complexos podem ser passados no with
parameters, ou pode mesclar parâmetros simples com filtros._

```
@ObjectModel.query.implementedBy: 'ABAP:ZMHL_CL_CUSTOM_ENTITY'
@EndUserText.label: 'Header'
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

_Deve existir uma composition, 1 para vários, com o tipo de estrutura (CDS) que será
retornada._

###2. Salve a CDS criada, mas ainda não as ative. Crie as outras CDSs que serão retornadas
como uma tabela com várias linhas.
Para o meu cenário ZMHL_I_Custom_Item e ZMHL_I_CUSTOM_MSG.

```
@ObjectModel.query.implementedBy: 'ABAP:ZMHL_CL_CUSTOM_ENTITY'
@EndUserText.label: 'Item'
define custom entity ZMHL_I_Custom_Item
{
key HeaderID : abap.char( 5 );
key Item : abap.numc( 2 );
Description : abap.char( 40 );
_Header : association to parent ZMHL_I_Custom_Header on _Header.HeaderID =
$projection.HeaderID;
}


@ObjectModel.query.implementedBy: 'ABAP:ZMHL_CL_CUSTOM_ENTITY'
@EndUserText.label: 'Item'
define custom entity ZMHL_I_CUSTOM_MSG
{
key HeaderID : abap.char( 5 );
key Item : abap.numc( 2 );
Message : abap.char( 40 );
_Header : association to parent ZMHL_i_Custom_Header on _Header.HeaderID =
$projection.HeaderID;
}
```

Salve todas as CDS criadas.

3. Criada as três CDSs, perceba que o campo chave é o mesmo para as três e existe
associação entre as filhas e a CDS root. Perceba, também, que existe uma
implementação comum entre elas, 'ABAP:ZMHL_CL_CUSTOM_ENTITY'.

4. Agora cria a Classe de implementação, onde vai ser feita a chamada da RFC com os
filtros do oData e retornada as tabelas item e msg.

![image](https://github.com/user-attachments/assets/c330e2cd-9b5f-4c1c-b310-3dac1d7cbfe8)
A classe de implementação será criada com a interface if_rap_query_provider e o método
select ficará aberto para implementar a chamada da RFC.

![image](https://github.com/user-attachments/assets/e684ec15-97d9-48ed-aca0-32eb3ccd3831)

5. Ative todos os objetos criados até agora e implemente a classe para ler filtros e
retornar a tabela correta para cada filha criada.

```
METHOD if_rap_query_provider~select.
DATA: lt_header TYPE STANDARD TABLE OF zmhl_i_custom_header,
lt_item TYPE STANDARD TABLE OF zmhl_i_custom_item,
lt_msg TYPE STANDARD TABLE OF zmhl_i_custom_msg,
lv_order_by TYPE string.
DATA: lt_filter_ranges_status TYPE RANGE OF ZMHL_I_Custom_Header-status,
ls_filter_ranges_status LIKE LINE OF lt_filter_ranges_status.
&quot;Get Filters
DATA(lt_filter_cond) = io_request-&gt;get_filter( )-&gt;get_as_ranges( ).
&quot;-get filter for STATUS
READ TABLE lt_filter_cond WITH KEY name = 'STATUS' INTO
DATA(ls_status_cond).
IF sy-subrc EQ 0.
LOOP AT ls_status_cond-range INTO DATA(ls_sel_opt_status).
MOVE-CORRESPONDING ls_sel_opt_status TO ls_filter_ranges_status.
INSERT ls_filter_ranges_status INTO TABLE lt_filter_ranges_status.
ENDLOOP.
ENDIF.

***
*** &quot;Get paging properties
*** DATA(lv_offset) = io_request-&gt;get_paging( )-&gt;get_offset( ).
&quot;Get positive page size, to avoid -1
DATA(lv_page_size) = abs( io_request-&gt;get_paging( )-&gt;get_page_size( ) ).
&quot;Get Parmeters
DATA(lt_parameters) = io_request-&gt;get_parameters( ).
&quot;Get parameters for RFC
DATA(lv_status) = VALUE #( lt_parameters[ parameter_name = 'P_STATUS' ]-
value OPTIONAL ).
IF gt_header IS INITIAL.
&quot; CALL RFC
*** CALL FUNCTION 'RFC...'
*** EXPORTING
*** im_status = lv_status
*** TABLES
*** t_item = lt_item
*** t_msg = lt_msg.
&quot; RETURNING PAYLOAD
lt_header = VALUE #( ( headerid = 'RFC01' ) ).
gt_header = lt_header.
&quot; Items for Return
lt_item = VALUE #(
( headerid = 'RFC01' item = '01' description = 'Item 1.1' )
( headerid = 'RFC01' item = '02' description = 'Item 1.2' )
( headerid = 'RFC01' item = '03' description = 'Item 1.3' )
).
gt_item = lt_item.
&quot; Msgs for Return
lt_msg = VALUE #(
( headerid = 'RFC01' item = '01' message = 'Success 1.1' )
( headerid = 'RFC01' item = '02' message = 'Success 1.2' )
( headerid = 'RFC01' item = '03' message = 'Error 1.3' ) ).
gt_msg = lt_msg.
ELSE.
lt_header = gt_header.
lt_item = gt_item.
lt_msg = gt_msg.
ENDIF.
CASE io_request-&gt;get_entity_id( ).

WHEN 'ZMHL_I_CUSTOM_HEADER'.
&quot;set total count of results
IF io_request-&gt;is_total_numb_of_rec_requested( ).
io_response-&gt;set_total_number_of_records( lines( lt_header ) ).
ENDIF.
&quot;set data
io_response-&gt;set_data( lt_header ).
WHEN 'ZMHL_I_CUSTOM_ITEM'.
&quot;set total count of results
IF io_request-&gt;is_total_numb_of_rec_requested( ).
io_response-&gt;set_total_number_of_records( lines( lt_item ) ).
ENDIF.
&quot;set data
io_response-&gt;set_data( lt_item ).
WHEN 'ZMHL_I_CUSTOM_MSG'.
&quot;set total count of results
IF io_request-&gt;is_total_numb_of_rec_requested( ).
io_response-&gt;set_total_number_of_records( lines( lt_msg ) ).
ENDIF.
&quot;set data
io_response-&gt;set_data( lt_msg ).
ENDCASE.
ENDMETHOD.

Veja que na implementação você terá que retornar o campo chave em todas as
estruturas, na Header, Item e Msg. Pode ser um valor fixo no código, não precisa vir pelo
GET.

6. Cria o Service Definition e dê nomes (as Header) a cada CDS que está sendo exposta
para facilitar o acesso as filhas pelo serviço.

@EndUserText.label: &#39;Custom Entity Service Definition&#39;
define service ZMHL_Custom_Entity {
expose ZMHL_i_Custom_Header as Header;
expose ZMHL_I_Custom_Item as Item;
expose ZMHL_I_CUSTOM_MSG as Msg;
}
```

7. Crie o Service Binding para serviço odata V2 (tipo webapi) e clique no botão publicar.

   ![image](https://github.com/user-attachments/assets/7ddd8902-83b7-4e00-ac03-6edd69be9c98)

* *Veja que a navegação entre Header e filhas é criada e fica fácil de passar o que precisa ser
retornado na chamada do serviço, via parâmetro $expand do Odata.

> [!NOTE]
> O serviço publicado fica local, para publicar o serviço que será transportado, utilize:
/IWFND/MAINT_SERVICE.

![image](https://github.com/user-attachments/assets/2f83fea1-3bd7-4fc3-a1ee-d26e006c2547)

Teste Nosso cenário: Utilizaremos a ferramenta POSTMAN duas tabelas de retorno, com dois
tipos diferentes, e utilizaremos filtros e parâmetros. Os parâmetros são passados na URL entre ( ),
no exemplo abaixo: (p_param1=’A’).

Passe a navegação to_Item e to_Msg através do $expand. Isso irá fazer com que ele consiga ler
a implementação, dentro da classe, para cada entidade que você criou como filha. Não passar a
navegação fará com que ele passe apenas uma vez na implementação e não será possível
retornar o tipo de dados das filhas da estrutura complexa criada.

![image](https://github.com/user-attachments/assets/0bc68380-460d-4ac5-be51-deb5564b2a18)


