Principal alteração a ser documentada aqui é a alteração no select, não só foi unificado todos os filtros para um só select e adicionado o filtro de data, como tambem a alteração dos CASES para a comparação de valores dos filtros.

``` sql
SELECT
    SUB_NFE_IDENTIFICACAO.*,
    Nvl(SUB_NFE_TOTAL.NFT_VL_TOTAL_NFE, 0) AS vlTotalNfe,
    COUNT(*) OVER() AS totalLinhas
FROM
    (
        SELECT
            nfe.*
        FROM
            NFE_IDENTIFICACAO nfe
            LEFT JOIN NFE_IDENTIFICA_DESTINATARIO dest on(
                nfe.EMI_CO_CNPJ = dest.EMI_CO_CNPJ
                and nfe.NID_CO_NFE = dest.NID_CO_NFE
                and nfe.UNF_CO_SERIE = dest.UNF_CO_SERIE
                and nfe.UNF_CO_Modelo = dest.UNF_CO_Modelo
            )
        WHERE
            nfe.EMI_CO_CNPJ = :emiCoCnpj
            AND NVL(dest.PTC_CO_CNPJ, -1) = (
                CASE
                    WHEN :ptcCoCnpj = -1 THEN NVL(dest.PTC_CO_CNPJ, -1)
                    else TO_NUMBER(:ptcCoCnpj)
                END
            )
            and nfe.NID_IN_AMBIENTE = :nidInAmbiente
            and nfe.NID_IN_STATUS = (
                CASE
                    WHEN :nidInStatus = -1 THEN nfe.NID_IN_STATUS
                    else :nidInStatus
                END
            )
            and nfe.NID_CO_NFE = (
                CASE
                    WHEN TO_NUMBER(:nidCoNfe) = -1 THEN nfe.NID_CO_NFE
                    else TO_NUMBER(:nidCoNfe)
                END
            )
            and nfe.ope_Co_Operacao = (
                CASE
                    WHEN TO_NUMBER(:opeCoOperacao) = -1 THEN nfe.ope_Co_Operacao
                    else TO_NUMBER(:opeCoOperacao)
                END
            )
            and nfe.NID_DT_EMISSAO BETWEEN TO_DATE(:dtPerInicio, 'DD/MM/YYYY')
            AND TO_DATE(:dtPerFinal, 'DD/MM/YYYY')
    ) SUB_NFE_IDENTIFICACAO
    LEFT JOIN (
        SELECT
            NFE_TOTAL.*
        FROM
            NFE_TOTAL
        WHERE
            EMI_CO_CNPJ = :emiCoCnpj
    ) SUB_NFE_TOTAL ON (
        SUB_NFE_IDENTIFICACAO.EMI_CO_CNPJ = SUB_NFE_TOTAL.EMI_CO_CNPJ
        AND SUB_NFE_IDENTIFICACAO.UNF_CO_MODELO = SUB_NFE_TOTAL.UNF_CO_MODELO
        AND SUB_NFE_IDENTIFICACAO.UNF_CO_SERIE = SUB_NFE_TOTAL.UNF_CO_SERIE
        AND SUB_NFE_IDENTIFICACAO.NID_CO_NFE = SUB_NFE_TOTAL.NID_CO_NFE
        AND SUB_NFE_IDENTIFICACAO.NID_IN_AMBIENTE = SUB_NFE_TOTAL.NID_IN_AMBIENTE
    )
ORDER BY
    SUB_NFE_IDENTIFICACAO.nid_Dt_Emissao DESC,
    SUB_NFE_IDENTIFICACAO.unf_Co_Modelo ASC,
    SUB_NFE_IDENTIFICACAO.unf_Co_Serie ASC,
    SUB_NFE_IDENTIFICACAO.nid_Co_Nfe DESC
```

A alteração reduziu o custo das operações em aproximadamente 30%, diminuindo de 46 para 32 na maioria dos casos.

---
Um caso especifico foi o filtro do Participante:
``` sql
AND NVL(dest.PTC_CO_CNPJ, -1) = (
	CASE
		WHEN :ptcCoCnpj = -1 THEN NVL(dest.PTC_CO_CNPJ, -1)
		else TO_NUMBER(:ptcCoCnpj)
	END
)
```

Como nem todas as notas possuem obrigatoriamente o Participante declarado no momento de sua criação, ele pode ser `null` em alguns casos, então foi adaptado essa condição usando NVL para trazer os registros com o Participante `null` caso o valor informado no filtro for -1.

---
Um problema em particular foi detectado, quando usado os parâmetros fixos como o CNPJ da loja ao invés de :emiCoCnpj, os valores de custo aumentavam consideravelmente (para 843). Porém durante as pesquisas, foi constatado que o valor apresentado pelo explain plan é generalizado e nem sempre reflete no resultado final, **fazendo as consultas o resultado de tempo era semelhante**.

Como o uso de **index** pode ser melhor aproveitado em **tabelas extensas com volumes de dados altos e tambem em caso de filtragens especificas**, foi considerado que seria melhor optar pelo index.

Para contornar o problema dos parâmetros fixos, foi criado o seguinte index para a tabela NFE_IDENTIFICACAO
``` sql
create index ind_nfe_filtro_data on NFE_IDENTIFICACAO(NID_DT_EMISSAO);
```

Oque **reduziu o custo da operação de 843 para 35** e o tempo de consulta usando o filtro do ano de 2012 até 2024 **reduziu de 3,33 segundos para cerca de 1,7 segundos**;

Foi incluso somente a data de emissão no index usando como base os conselhos [dessa pergunta](https://stackoverflow.com/a/688006) no Stack Overflow. Também foi optado por colocar apenas uma coluna pois o index por mais que seja eficiente para melhorar a velocidade de consultas, ele pode acabar atrasando a velocidade de UPDATE's, INSERT's E DELETE's

O Select usado para o teste:
```sql
SELECT
    SUB_NFE_IDENTIFICACAO.*,
    Nvl(SUB_NFE_TOTAL.NFT_VL_TOTAL_NFE, 0) AS vlTotalNfe,
    COUNT(*) OVER() AS totalLinhas
FROM
    (
        SELECT
            nfe.*
        FROM
            NFE_IDENTIFICACAO nfe
            LEFT JOIN NFE_IDENTIFICA_DESTINATARIO dest on(
                nfe.EMI_CO_CNPJ = dest.EMI_CO_CNPJ
                and nfe.NID_CO_NFE = dest.NID_CO_NFE
                and nfe.UNF_CO_SERIE = dest.UNF_CO_SERIE
                and nfe.UNF_CO_Modelo = dest.UNF_CO_Modelo
            )
        WHERE
            NVL(dest.PTC_CO_CNPJ, -1) = (
                CASE
                    WHEN -1 = -1 THEN NVL(dest.PTC_CO_CNPJ, -1)
                    else -1
                END
            )
            and nfe.EMI_CO_CNPJ = 2183783000111
            
            and nfe.NID_IN_AMBIENTE = 1
            and nfe.NID_IN_STATUS = (
                CASE
                    WHEN '-1' = -1 THEN nfe.NID_IN_STATUS
                    else '-1'
                END
            )
            and nfe.NID_CO_NFE = (
                CASE
                    WHEN -1 = -1 THEN nfe.NID_CO_NFE
                    else -1
                END
            )
            and nfe.ope_Co_Operacao = (
                CASE
                    WHEN -1 = -1 THEN nfe.ope_Co_Operacao
                    else -1
                END
            )
            and nfe.NID_DT_EMISSAO BETWEEN TO_DATE('01/08/2024', 'DD/MM/YYYY')
            AND TO_DATE('15/12/2024', 'DD/MM/YYYY')
    ) SUB_NFE_IDENTIFICACAO
    LEFT JOIN (
        SELECT
            NFE_TOTAL.*
        FROM
            NFE_TOTAL
        WHERE
            EMI_CO_CNPJ = 2183783000111
    ) SUB_NFE_TOTAL ON (
        SUB_NFE_IDENTIFICACAO.EMI_CO_CNPJ = SUB_NFE_TOTAL.EMI_CO_CNPJ
        AND SUB_NFE_IDENTIFICACAO.UNF_CO_MODELO = SUB_NFE_TOTAL.UNF_CO_MODELO
        AND SUB_NFE_IDENTIFICACAO.UNF_CO_SERIE = SUB_NFE_TOTAL.UNF_CO_SERIE
        AND SUB_NFE_IDENTIFICACAO.NID_CO_NFE = SUB_NFE_TOTAL.NID_CO_NFE
        AND SUB_NFE_IDENTIFICACAO.NID_IN_AMBIENTE = SUB_NFE_TOTAL.NID_IN_AMBIENTE
    )
ORDER BY
    SUB_NFE_IDENTIFICACAO.nid_Dt_Emissao DESC,
    SUB_NFE_IDENTIFICACAO.unf_Co_Modelo ASC,
    SUB_NFE_IDENTIFICACAO.unf_Co_Serie ASC,
    SUB_NFE_IDENTIFICACAO.nid_Co_Nfe DESC;
```
