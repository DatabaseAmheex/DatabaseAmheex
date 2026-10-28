# AMHEEX Database API — Documentação Técnica

> Banco de dados JSON em árvore no estilo Firebase, servido pela **AMHEEX Data API** e persistido no **AMHEEX SERVER**.

| Item | Valor |
|---|---|
| Engine | Firebase-Like AMHEEX Data API |
| Repositório | `DatabaseAmheex/STORAGE` |
| Branch principal | `main` |
| Diretório raiz | `X/` |
| Base URL | `https://database.amheex.workers.dev` |

---

## Sumário

1. [Arquitetura e funcionamento](#1-arquitetura-e-funcionamento)
2. [Endpoints e caminhos](#2-endpoints-e-caminhos)
3. [Salvar dados (POST / PUT)](#3-salvar-dados-post--put)
4. [Ler dados (GET)](#4-ler-dados-get)
5. [Excluir dados ou pastas (DELETE)](#5-excluir-dados-ou-pastas-delete)
6. [Upload por partes (chunks de 25 MiB)](#6-upload-por-partes-chunks-de-25-mib)
7. [Erros e códigos de status](#7-erros-e-códigos-de-status)
8. [Recomendações de integração](#8-recomendações-de-integração)
9. [Exemplos por linguagem](#9-exemplos-por-linguagem)

---

## 1. Arquitetura e funcionamento

| Componente | Funcionamento |
|---|---|
| Persistência | Infraestrutura do **AMHEEX SERVER** via **AMHEEX Data API**. |
| Modelo atomizado | Objetos JSON são desmembrados em nós de diretórios dentro de `X/`. Cada valor primitivo é armazenado como um arquivo (`part_1.txt`). |
| Dados massivos | Arquivos e buffers maiores são divididos em partes (chunks) de até **25 MiB**. |
| Exclusão estrutural | Reindexação forçada via `sha => null` na árvore do AMHEEX, garantindo a remoção física dos blobs. |
| Tempo real | O ponteiro da branch principal é atualizado com `force => true` para contornar latências de propagação. |

### Exemplo de árvore (ilustrativo)

```
X/usuarios/maria/
├── nome/part_1.txt           → Maria
├── cargo/part_1.txt          → Desenvolvedora
├── ativo/part_1.txt          → true
└── configuracoes/
    └── tema/part_1.txt       → escuro
```

---

## 2. Endpoints e caminhos

```
GET / POST / PUT / DELETE  ->  https://database.amheex.workers.dev/{caminho_do_no}
```

Exemplos de caminho: `usuarios/maria`, `produtos/eletronicos/item_102`.

| Método | Ação |
|---|---|
| `GET` | Lê um nó ou reconstrói toda a sub-árvore como JSON |
| `POST` / `PUT` | Salva dados no nó; JSON válido é desmembrado em sub-nós |
| `DELETE` | Remove o nó e todos os sub-nós |

---

## 3. Salvar dados (POST / PUT)

Submete informações para o nó especificado. Se o corpo for um JSON válido, ele é desmembrado recursivamente em sub-nós.

- **Headers:** `Content-Type: application/json`

```json
{
  "nome": "Maria",
  "cargo": "Desenvolvedora",
  "ativo": true,
  "configuracoes": { "tema": "escuro" }
}
```

**Resposta de sucesso (HTTP 200/201):**

```json
{ "success": true, "path": "usuarios/maria", "desmembrado": true }
```

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"nome":"Maria","cargo":"Desenvolvedora","ativo":true,"configuracoes":{"tema":"escuro"}}' \
  https://database.amheex.workers.dev/usuarios/maria
```

---

## 4. Ler dados (GET)

Recupera o conteúdo de um nó individual ou reconstrói toda a árvore de sub-nós como um objeto JSON.

```bash
curl "https://database.amheex.workers.dev/usuarios/maria?_t=$(date +%s)"
```

**Resposta (HTTP 200):**

```json
{
  "nome": "Maria",
  "cargo": "Desenvolvedora",
  "ativo": true,
  "configuracoes": { "tema": "escuro" }
}
```

**Se o nó não existir (HTTP 200):** `null`

---

## 5. Excluir dados ou pastas (DELETE)

Remove **permanentemente** o nó, arquivo ou diretório informado e todos os seus sub-nós.

```bash
curl -X DELETE "https://database.amheex.workers.dev/usuarios/maria?_t=$(date +%s)"
```

**Sucesso (HTTP 200):**

```json
{ "success": true, "deleted_path": "usuarios/maria" }
```

**Não encontrado (HTTP 404):**

```json
{ "success": false, "message": "Nó ou diretório não encontrado para exclusão" }
```

---

## 6. Upload por partes (chunks de 25 MiB)

| Cabeçalho | Descrição |
|---|---|
| `X-File-Size` | Tamanho total do arquivo/payload em bytes |
| `X-Part-Index` | Número da parte enviada (começando em 1) |
| `Content-Type` | `application/json` ou `application/octet-stream` |

Cada parte tem no máximo **26.214.400 bytes** (25 MiB).

```javascript
const API = "https://database.amheex.workers.dev";
const CHUNK = 25 * 1024 * 1024;

async function enviarEmPartes(caminho, file) {
  const total = Math.ceil(file.size / CHUNK);
  for (let i = 0; i < total; i++) {
    const res = await fetch(`${API}/${caminho}`, {
      method: "POST",
      headers: {
        "Content-Type": "application/octet-stream",
        "X-File-Size": String(file.size),
        "X-Part-Index": String(i + 1),
      },
      body: file.slice(i * CHUNK, Math.min((i + 1) * CHUNK, file.size)),
    });
    if (!res.ok) throw new Error(`Parte ${i + 1} falhou: HTTP ${res.status}`);
  }
}
```

---

## 7. Erros e códigos de status

| Status | Significado |
|---|---|
| `200 OK` | Operação concluída com sucesso |
| `400 Bad Request` | Dados ou parâmetros ausentes/inválidos |
| `404 Not Found` | Caminho para exclusão ou leitura não localizado |
| `405 Method Not Allowed` | Método HTTP não suportado pela rota |
| `500 Internal Server Error` | Conflito no AMHEEX SERVER ou erro de processamento PHP |

```json
{
  "erro_servidor": "Descrição do erro",
  "arquivo": "nome_do_arquivo.php",
  "linha": 123
}
```

---

## 8. Recomendações de integração

1. **Prevenção de cache:** em `GET` e `DELETE`, acrescente um timestamp na query string (ex.: `?_t=1690000000`) para evitar cache do navegador ou do Cloudflare.
2. **Formatação de caminho:** use apenas caracteres alfanuméricos, `/`, `-` e `_`.

---

## 9. Exemplos por linguagem

### JavaScript (mini-cliente estilo Firebase)

```javascript
const API = "https://database.amheex.workers.dev";

function ref(caminho) {
  const req = async (method, body) => {
    const bust = method === "GET" || method === "DELETE" ? `?_t=${Date.now()}` : "";
    const res = await fetch(`${API}/${caminho}${bust}`, {
      method,
      headers: body !== undefined ? { "Content-Type": "application/json" } : {},
      body: body !== undefined ? JSON.stringify(body) : undefined,
    });
    const data = await res.json().catch(() => null);
    if (!res.ok && res.status !== 404) throw new Error(data?.erro_servidor || `HTTP ${res.status}`);
    return data;
  };
  return {
    get: () => req("GET"),
    set: (d) => req("POST", d),
    update: (d) => req("PUT", d),
    remove: () => req("DELETE"),
    child: (sub) => ref(`${caminho}/${sub}`),
  };
}

const maria = ref("usuarios/maria");
await maria.set({ nome: "Maria", ativo: true });
console.log(await maria.get());
```

### Python

```python
import time, requests
API = "https://database.amheex.workers.dev"

requests.post(f"{API}/usuarios/maria", json={"nome": "Maria", "ativo": True}).raise_for_status()
print(requests.get(f"{API}/usuarios/maria", params={"_t": int(time.time())}).json())
requests.delete(f"{API}/usuarios/maria", params={"_t": int(time.time())})
```

---

<sub>AMHEEX Database API · Firebase-Like AMHEEX Data API · Processado pelo AMHEEX SERVER · https://database.amheex.workers.dev</sub>
