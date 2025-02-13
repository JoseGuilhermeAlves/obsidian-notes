quando precisar abrir a modal dentro de outra, para pode calibrar o z-index da modal e do backdrop (fundo cinza) use o seguinte trecho de código:

```html
<script>
	$jq('#idDaModal').on("shown.bs.modal", function (e) {
		$jq(this).last().css('z-index', 1070);
		$jq('.modal-backdrop').last().css('z-index', 1060);
	});
</script>
```

- substitua '#idDaModal' pelo id da Modal que voce está abrindo