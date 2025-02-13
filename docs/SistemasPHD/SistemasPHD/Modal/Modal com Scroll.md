Para adicionar o scroll no interior da modal e não na direita da pagina, use os seguintes passos:

- Na div de conteudo da modal, adicione a seguinte propriedade style: 'style="height: auto !important; max-height: calc(100vh - 26vh) !important;'
- E na tag form adicione a classe 'overflow-auto'

Dessa forma seu código deve ficar próximo a isso

```html 
<div class="modal-dialog ui-modal" role="document">
	<div class="modal-content" style="height: auto !important; max-height: calc(100vh - 26vh) !important;">
		<div class="modal-header">
			CONTEUDO DO HEADER
		</div>
		<div class="modal-body">
			<h:form id="frm#{modalPanelId}" styleClass="overflow-auto" enctype="multipart/form-data">
				CONTEUDO DO FORM
			</h:form>
		 </div>
	 </div>
</div>
                 
```