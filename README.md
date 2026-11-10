# Demo: Lakehouse do Microsoft Fabric → Google Sheets

Um notebook Python do Fabric lê uma tabela Delta do lakehouse e escreve o resultado numa planilha do
Google Sheets. A autenticação usa uma **service account do Google** cuja chave JSON fica no **Azure Key Vault**.

```
Lakehouse (lh_demo_google)          Azure Key Vault                 Google Cloud
  Tables/sales_summary ──┐         secret: gcp-sa-sheets-json      Service account
                         ▼                 │                       (Editor da planilha)
         Notebook nb_export_google_sheets ◄┘  notebookutils.credentials.getSecret
                         │  gspread + google-auth
                         ▼
             Google Sheets → aba "sales_summary"
```

**Comportamento entre execuções:** a planilha é criada uma vez (manualmente) e sempre reutilizada.
A aba `sales_summary` é criada só na primeira execução. Depois disso, cada execução **limpa e reescreve**
a mesma aba, sem criar planilhas novas e sem acumular linhas.

> ⚠️ Use apenas dados sintéticos. Nunca rode esta demo com dados de cliente e nunca cole o JSON da
> service account em chats, no repositório ou em prints de tela.

## Conteúdo

| Arquivo | Função |
| :--- | :--- |
| [notebooks/nb_setup_sample_data.ipynb](notebooks/nb_setup_sample_data.ipynb) | Gera a tabela Delta `sales_summary` (200 linhas sintéticas, seed fixa) |
| [notebooks/nb_export_google_sheets.ipynb](notebooks/nb_export_google_sheets.ipynb) | A demo: Key Vault → leitura Delta → escrita no Sheets |

## Pré-requisitos

### 1. Google Cloud

1. Crie (ou reutilize) um projeto de demo, ex.: `fabric-demo`.
2. **APIs & Services → Enable APIs** → habilite a **Google Sheets API**.
3. **IAM & Admin → Service Accounts** → crie `fabric-sheets-writer` → **Keys → Add key → JSON**.
   - Se a criação de chave estiver bloqueada pela org policy `iam.disableServiceAccountKeyCreation`, use um projeto sandbox.
4. Crie a planilha `Fabric Demo - Sales` no Google Sheets e **compartilhe como Editor** com o e-mail da
   service account (`fabric-sheets-writer@<project>.iam.gserviceaccount.com`).
5. Anote o **spreadsheet key**, que é o trecho da URL entre `/d/` e `/edit`.

### 2. Azure Key Vault

1. No Key Vault de demo, crie o secret `gcp-sa-sheets-json` e cole **o conteúdo inteiro** do JSON baixado.
2. Apague o arquivo JSON local.
3. Dê a role **Key Vault Secrets User** a quem vai executar o notebook. O `getSecret` roda com a identidade de quem executa o notebook.

### 3. Microsoft Fabric

1. No workspace de demo, crie o lakehouse `lh_demo_google`.
2. **Import → Notebook → From this computer** e importe os dois `.ipynb`.
3. Em cada notebook:
   - confira que a linguagem é **Python**, não PySpark (seletor na barra de ferramentas);
   - anexe `lh_demo_google` como **lakehouse padrão**.
4. Rode `nb_setup_sample_data` uma vez.
5. Em `nb_export_google_sheets`, preencha `KEY_VAULT_URI` e `SPREADSHEET_KEY` na célula de parâmetros.

## Roteiro da apresentação (~10 min)

1. **Origem:** mostre a tabela `sales_summary` no lakehouse (preview ou SQL analytics endpoint).
2. **Segurança:** mostre o secret no Key Vault (só o nome) e a planilha vazia, compartilhada com a service account.
3. **Notebook:** execute `nb_export_google_sheets` célula a célula:
   1. `%pip install` → bibliotecas do Google;
   2. parâmetros → nada sensível no código;
   3. autenticação → `getSecret` + `Credentials.from_service_account_info` (imprime só o e-mail da conta);
   4. leitura Delta com `deltalake`;
   5. conversão para o formato da API (datas, decimais e nulos);
   6. escrita com `clear` → `resize` → `update` → `freeze`.
4. **Resultado:** abra o Google Sheets e mostre os dados.
5. **Idempotência:** rode de novo e mostre que a aba continua com o mesmo número de linhas.
6. **Próximos passos:** agendar o notebook ou chamá-lo de um pipeline (a célula de parâmetros já permite),
   carga incremental, e BigQuery quando o volume passar do limite do Sheets (10 milhões de células).

## Troubleshooting

| Sintoma | Causa provável |
| :--- | :--- |
| Erro 403 / `Forbidden` no `getSecret` | Falta a role **Key Vault Secrets User** para quem executa |
| `PermissionError` / 403 no `open_by_key` | Planilha não compartilhada com o e-mail da service account |
| `SpreadsheetNotFound` | `SPREADSHEET_KEY` errado |
| 403 `Google Sheets API has not been used in project` | API não habilitada no projeto GCP |
| `TableNotFoundError` / caminho inexistente | Lakehouse padrão não anexado, ou `nb_setup_sample_data` não executado |
| `OSError: Unable to rename file` ao gravar Delta | O caminho `/lakehouse/default` (FUSE) não permite renomear arquivos. Grave via `abfss://` com token, como o `nb_setup_sample_data` já faz; esse caminho só serve para leitura |
| `KeyError: 'defaultLakehouseId'` | Nenhum lakehouse padrão anexado ao notebook |

## Antes de gravar ou apresentar

- Limpe os outputs dos notebooks (**Edit → Clear all outputs**) e confira que nenhuma célula mostra conteúdo do secret.
- Confira que `KEY_VAULT_URI` e `SPREADSHEET_KEY` não aparecem em prints publicados.
