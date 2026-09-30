# DIO – Análise de Dados

Notebook originalmente criado no Google Colab:
<https://colab.research.google.com/drive/1cBcxul2CE72JocaJtTZsOQvmuFZC0Rqg>

## Objetivo

Automatizar um fluxo de ponta a ponta (ETL + IA generativa):

1. **Extract** – ler uma lista de `UserID` de um arquivo CSV.
2. **Transform** – buscar os dados de cada usuário em uma API pública.
3. **Load / Uso** – gerar, com o Gemini, uma mensagem de marketing personalizada sobre investimentos para cada usuário.

## Requisitos

- Python 3
- Ambiente Google Colab (o código usa `google.colab.userdata`)
- Bibliotecas: `pandas`, `requests`, `google-genai` / `google-generativeai`
- Arquivo `SDW2023.csv` com a coluna `UserID`
- Chave da API do Gemini salva no **Secrets** do Colab com o nome `GEMINI_API_KEY`

## Estrutura do código

### 1. Leitura do CSV

```python
import pandas as pd

df = pd.read_csv('SDW2023.csv')
user_ids = df['UserID'].tolist()
print(user_ids)
```

Lê o arquivo `SDW2023.csv` e converte a coluna `UserID` em uma lista Python.

### 2. Consulta à API de usuários

```python
import requests
import json

users_api = 'https://dummyjson.com/'

def get_user(id):
  response = requests.get(f'{users_api}/users/{id}')
  return response.json() if response.status_code == 200 else None

users = [user for id in user_ids if (user := get_user(id)) is not None]
print(json.dumps(users, indent=2))
```

- `get_user(id)` faz um `GET` em `/users/{id}` da API [DummyJSON](https://dummyjson.com/) e retorna o JSON, ou `None` se o status não for 200.
- A *list comprehension* com o operador walrus (`:=`) chama `get_user` para cada ID e mantém apenas os usuários encontrados.
- `json.dumps(..., indent=2)` imprime o resultado formatado.

### 3. Teste do Gemini (SDK `google-genai`)

```python
!pip install -U google-genai

from google import genai
from google.colab import userdata

client = genai.Client(api_key=userdata.get('GEMINI_API_KEY'))

interaction = client.interactions.create(
    model="gemini-3.6-flash",
    input="Explique a IA em 2 paragrafo"
)
print(interaction.output_text)
```

Instala o SDK, recupera a chave de forma segura no Secrets Manager do Colab e faz uma chamada de teste pedindo uma explicação sobre IA em dois parágrafos.

### 4. Mensagens personalizadas (SDK `google-generativeai`)

```python
import google.generativeai as genai
from google.colab import userdata

genai.configure(api_key=userdata.get('GEMINI_API_KEY'))
model = genai.GenerativeModel('gemini-1.5-flash')

def generate_investment_message(user_first_name):
  prompt_parts = [
    "Você é um especialista em marketing bancário. Crie uma mensagem para ",
    f"{user_first_name} sobre a importância dos investimentos (máximo de 100 caracteres)"
  ]
  try:
    response = model.generate_content(prompt_parts)
    return response.text.strip()
  except Exception as e:
    return f"Erro ao gerar mensagem para {user_first_name}: {e}"

print("Gerando mensagens personalizadas para cada usuário:")
for user in users:
  message = generate_investment_message(user['firstName'])
  print(f"Mensagem para {user['firstName']}: {message}")
```

- Configura o modelo `gemini-1.5-flash`.
- `generate_investment_message` monta um prompt pedindo uma mensagem de até 100 caracteres sobre investimentos, usando o primeiro nome do usuário. Em caso de erro, retorna a mensagem de erro em vez de interromper o programa.
- O loop final percorre a lista `users` e imprime a mensagem gerada para cada um.

## Fluxo resumido

```
SDW2023.csv ──► user_ids ──► API DummyJSON ──► users ──► Gemini ──► mensagens personalizadas
```

## Pontos de atenção

- **URL duplicada com barra:** `users_api` termina com `/` e o f-string adiciona outra (`https://dummyjson.com//users/1`). Costuma funcionar, mas o ideal é remover uma das barras.
- **Dois SDKs diferentes:** as seções 3 e 4 importam `genai` de pacotes distintos (`google-genai` e `google-generativeai`). O segundo import sobrescreve o nome `genai`, então o `client` da seção 3 deixa de ser usado depois disso. Vale padronizar em um único SDK.
- **Linhas `#` soltas:** os `#` isolados após algumas células são resíduos da exportação do Colab e podem ser removidos.
- **Comando mágico:** `!pip install` só funciona em notebooks (Colab/Jupyter), não em um script `.py` comum.
- **Nomes de modelos:** confira se os modelos usados (`gemini-3.6-flash` e `gemini-1.5-flash`) estão disponíveis na sua conta, pois os modelos mudam com o tempo.
- **Segurança:** nunca deixe a chave da API escrita no código; mantenha-a no Secrets, como já é feito.
