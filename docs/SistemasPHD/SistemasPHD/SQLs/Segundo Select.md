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
            nfe.EMI_CO_CNPJ = :emiCoCnpj
            and (dest.PTC_CO_CNPJ = :ptcCoCnpj or (:ptcCoCnpj = -1))
            and nfe.NID_IN_AMBIENTE = :nidInAmbiente
            and (nfe.NID_IN_Status = :nidInStatus or (:nidInStatus = -1))
            and (nfe.NID_CO_NFE = :nidCoNfe OR (:nidCoNfe = -1))
            and (nfe.ope_Co_Operacao = :opeCoOperacao or (:opeCoOperacao = -1))
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