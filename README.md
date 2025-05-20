---
# 🔄 API Gateway HTTP Policy Transform Extension  
### Extensão Customizada Mule para Manipulação de Atributos HTTP em Políticas

<p align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=0:0C2340,100:00BFFF&height=220&section=header&text=Transforma%C3%A7%C3%A3o%20HTTP%20em%20Policies&fontSize=38&fontColor=ffffff&animation=fadeIn" alt="Extension Banner" />
</p>

<p align="center">
  <a href="https://docs.mulesoft.com/mule-sdk/latest/"><img src="https://img.shields.io/badge/Mule%20SDK-M%C3%B3dulo%20de%20Extens%C3%A3o-003B71?logo=mulesoft" /></a>
  <a href="https://openjdk.org/projects/jdk/8/"><img src="https://img.shields.io/badge/Java-8%20%2F%2011-FF6A00?logo=java" /></a>
  <a href="https://maven.apache.org/"><img src="https://img.shields.io/badge/Maven-3.x-C71A36?logo=apache-maven" /></a>
</p>

---

## 📑 Sumário

* [Visão Geral](#visão-geral)  
* [Motivação](#motivação)  
* [Funcionalidades](#funcionalidades)  
* [Arquitetura](#arquitetura)  
* [Estrutura do Projeto](#estrutura-do-projeto)  
* [Diagrama Detalhado de Fábricas e Handlers](#diagrama-fábricas-handlers)  
* [Operações Disponíveis](#operações-disponíveis)  
* [Exemplos de Uso](#exemplos)  
* [Instalação & Build](#instalação)  
* [Publicação no Exchange](#publicação)  
* [FAQ](#faq)  
* [Roadmap](#roadmap)  
* [Troubleshooting](#troubleshooting)  
* [Contribuindo](#contribuindo)  
* [Licença](#licença)  
* [Contato](#contato)  

---

## 📖 Visão Geral <a id="visão-geral"></a>

A **HTTP Policy Transform Extension** é um módulo Mule 4 capaz de **ler, alterar, remover ou acrescentar** cabeçalhos HTTP e outros metadados **dentro de políticas customizadas** do API Gateway.  
Diferente de transformações em flows, aplicar a lógica na camada de policy garante governança centralizada, reutilização e zero impacto no código das APIs.

---

## 💡 Motivação <a id="motivação"></a>

| Cenário | Descrição |
|---------|-----------|
| Multi-IdP / Zero-Trust | Políticas podem exigir cabeçalhos especiais para escolher o JWKS ou IdP corretos antes de validar o JWT. |
| Observabilidade | Adicionar _trace headers_ (`X-Correlation-Id`) em todas as requisições/respostas na borda do gateway. |
| Segurança | Remover `Server`, `X-Powered-By` ou outros cabeçalhos de fingerprinting antes da resposta deixar a nuvem. |
| LGPD / PCI | Mascarar dados sensíveis em cabeçalhos antes de armazenar em logs. |
| Roteamento Dinâmico | Redirecionar chamadas internas de acordo com path, método ou query param capturados. |

---

## 🚀 Funcionalidades <a id="funcionalidades"></a>

* **Operação 1 – `get-request-attributes`**  
  Devolve um wrapper rico com headers, queryParams, pathParams e campos técnicos.
* **Operação 2 – `transform-request`**  
  Permite *add/replace/remove* cabeçalhos **antes** da API.
* **Operação 3 – `transform-response`**  
  Permite *add/replace/remove* cabeçalhos **antes** de retornar ao consumidor.
* **Handler Selector Automático**  
  Reconhece se está executando em Listener, Requester ou Policy e escolhe o handler ideal.
* **Limite de Headers**  
  `LimitedHttpHeadersMultimapFactory` evita overload de memória se a requisição trouxer milhares de cabeçalhos.
* **Enum de Erros Customizados** (invalid header, missing value, limit exceeded…)
* **Metadata Resolver** para o Design Center exibir documentação e autocomplete.

---

## 🏗️ Arquitetura High-Level <a id="arquitetura"></a>

```text
┌──── Client ────┐
│  (HTTP)        │
└──────┬─────────┘
       │
       ▼
┌────────────── API Gateway ──────────────┐
│                                         │
│ 1. http-attrs:get-request-attributes     │
│ 2. http-transform:transform-request      │
│ 3. (outras políticas)                    │
│ 4. API Implementation (Flow)             │
│ 5. http-transform:transform-response     │
└──────────────────────────────────────────┘
````

---

## 📂 Estrutura do Projeto <a id="estrutura-do-projeto"></a>

| Arquivo / Pasta                                         | Função                                                         |         |             |
| ------------------------------------------------------- | -------------------------------------------------------------- | ------- | ----------- |
| `pom.xml`                                               | Build Maven (packaging `mule-extension`)                       |         |             |
| **Extension Core**                                      |                                                                |         |             |
| ├─ `HttpPolicyTransformExtension.java`                  | Classe anotada `@Extension` (name = "http-policy-transform")   |         |             |
| ├─ `HttpTransformOperations.java`                       | Registra as três operações (request/response/attrs)            |         |             |
| ├─ `Header.java`                                        | Estrutura \`\<header key="" value="" action="ADD               | REPLACE | REMOVE"/>\` |
| **Wrapper**                                             |                                                                |         |             |
| ├─ `HttpRequestAttributesWrapper.java`                  | POJO com `headers`, `queryParams`, etc.                        |         |             |
| **Handler Contracts & Selector**                        |                                                                |         |             |
| ├─ `HttpAttributesHandler.java`                         | Interface pai (`populateWrapper()`)                            |         |             |
| ├─ `HttpAttributesHandlerSelector.java`                 | Decide implementações corretas em runtime                      |         |             |
| **Request Handlers**                                    |                                                                |         |             |
| ├─ `HttpRequestAttributesHandler.java`                  | Listener (entradas externas)                                   |         |             |
| ├─ `HttpRequesterRequestAttributesHandler.java`         | Requester (chamadas internas)                                  |         |             |
| ├─ `HttpPolicyRequestAttributesHandler.java`            | Policy (fase request)                                          |         |             |
| **Response Handlers**                                   |                                                                |         |             |
| ├─ `HttpResponseAttributesHandler.java`                 | Listener (respostas externas)                                  |         |             |
| ├─ `HttpPolicyResponseAttributesHandler.java`           | Policy (fase response)                                         |         |             |
| **Factories (Request)**                                 |                                                                |         |             |
| ├─ `HttpPolicyRequestAttributesFactory.java`            | Interface genérica de fábrica                                  |         |             |
| ├─ `HttpPolicyRequestAttributesDefaultFactory.java`     | Implementa via API Gateway SDK                                 |         |             |
| ├─ `HttpPolicyRequestAttributesReflectiveFactory.java`  | Fallback usando reflection                                     |         |             |
| **Factories (Response)**                                |                                                                |         |             |
| ├─ `HttpPolicyResponseAttributesFactory.java`           | Interface genérica                                             |         |             |
| ├─ `HttpPolicyResponseAttributesDefaultFactory.java`    | Implementação oficial                                          |         |             |
| ├─ `HttpPolicyResponseAttributesReflectiveFactory.java` | Fallback refletivo                                             |         |             |
| ├─ `HttpPolicyResponseAttributesHandlerFactory.java`    | Cria handlers de resposta prontos                              |         |             |
| **Factories (Requester)**                               |                                                                |         |             |
| ├─ `HttpRequesterRequestAttributesHandlerFactory.java`  | Instancia handler p/ HTTP Requester                            |         |             |
| **Factories (Genéricas)**                               |                                                                |         |             |
| ├─ `HttpResponseAttributesFactory.java`                 | Factory raiz para respostas                                    |         |             |
| ├─ `HttpResponseAttributesDefaultFactory.java`          | Listener padrão                                                |         |             |
| ├─ `HttpResponseAttributesReflectiveFactory.java`       | Listener fallback (reflection)                                 |         |             |
| **Headers Multimap**                                    |                                                                |         |             |
| └─ `LimitedHttpHeadersMultimapFactory.java`             | Produz `Multimap<String,String>` com limite configurável       |         |             |
| **Reflection Utils**                                    |                                                                |         |             |
| ├─ `FieldInspector.java`                                | Descobre campos e tipos via Reflection                         |         |             |
| ├─ `FieldCopier.java`                                   | Copia valores field-to-field                                   |         |             |
| └─ `CrossClassFieldCopier.java`                         | Copia entre classes diferentes                                 |         |             |
| **Error & Metadata**                                    |                                                                |         |             |
| ├─ `HttpPolicyTransformErrorTypes.java`                 | Enum: `INVALID_HEADER`, `MISSING_VALUE`, `LIMIT_EXCEEDED`, ... |         |             |
| ├─ `HttpPolicyTransformErrorTypeProvider.java`          | Provider registrado no SDK                                     |         |             |
| └─ `HttpPolicyTransformMetadataResolver.java`           | Envia metadados ao Design Center                               |         |             |

---

## 🔍 Diagrama Detalhado de Fábricas e Handlers <a id="diagrama-fábricas-handlers"></a>

```text
                                    ┌────────────────────────────────────────┐
                                    │  HttpAttributesHandlerSelector         │
                                    │  • contextType = LISTENER | REQUESTER  │
                                    │  • execution = POLICY | FLOW           │
                                    └──────────────┬─────────────────────────┘
                                                   │ decide()
                 ┌─────────────────────────────────┼─────────────────────────────────┐
                 ▼                                 ▼                                 ▼
      ┌────────────────────┐            ┌────────────────────────┐      ┌──────────────────────────┐
      │ Request Handlers   │            │ Response Handlers      │      │ Requester Handler        │
      │ (Policy/Listener)  │            │ (Policy/Listener)      │      │ (HTTP Requester)         │
      └──┬─────────────────┘            └───────────────┬────────┘      └─────────┬────────────────┘
         │ factory()                                     │ factory()               │ factory()
         ▼                                               ▼                        ▼
┌───────────────────────┐                     ┌────────────────────┐   ┌────────────────────────────────┐
│  *DefaultFactory      │                     │ *ResponseFactory    │   │ HttpRequesterRequestAttributes │
│  *ReflectiveFactory   │                     │ (default/reflect)   │   │ Handler                        │
└───────────────────────┘                     └────────────────────┘   └────────────────────────────────┘
```

*Factories* criam *handlers* que, por sua vez, **popularão o wrapper** usando `FieldInspector` ➜ `FieldCopier`.
O **Multimap Factory** é usado dentro dos handlers para representar headers com limite de tamanho.

---

## ⚙️ Operações Disponíveis <a id="operações-disponíveis"></a>

### 1 ▪ `get-request-attributes`

Retorna em `target` o objeto `HttpRequestAttributesWrapper`.

### 2 ▪ `transform-request`

Aceita lista `<http-transform:header .../>` com atributos:

| Atributo | Obrig. | Descrição                           | Notas                               |
| -------- | ------ | ----------------------------------- | ----------------------------------- |
| `key`    | ✔      | Nome do header                      | Case-insensitive                    |
| `value`  | ✖      | Novo valor                          | Ignorado se `action="REMOVE"`       |
| `action` | ✖      | `ADD`(default)\|`REPLACE`\|`REMOVE` | `REPLACE` troca primeira ocorrência |

### 3 ▪ `transform-response`

Mesmos atributos, mas aplica‐se à resposta.

---

## 🧪 Exemplos de Uso <a id="exemplos"></a>

### Exemplo Completo de Policy

```xml
<configuration>
    <http-attrs:get-request-attributes target="reqAttrs"/>
    <!-- Add Forwarded Tenant Header -->
    <http-transform:transform-request>
        <http-transform:headers>
            <http-transform:header key="X-Tenant" value="#[vars.reqAttrs.headers['tenant']]"/>
            <http-transform:header key="X-Env" value="HML"/>
        </http-transform:headers>
    </http-transform:transform-request>

    <!-- …chama API… -->

    <http-transform:transform-response>
        <http-transform:headers>
            <http-transform:header key="Server" action="REMOVE"/>
            <http-transform:header key="X-Correlation-Id"
                                    value="#[vars.reqAttrs.headers.correlation-id default uuid()]"/>
        </http-transform:headers>
    </http-transform:transform-response>
</configuration>
```

---

## 🏗️ Instalação & Build <a id="instalação"></a>

```bash
git clone https://github.com/seu-org/mule-http-policy-transform-extension.git
cd mule-http-policy-transform-extension
mvn clean install -DskipTests
```

**Resultado:**
`target/mule-http-policy-transform-extension-<version>-mule-plugin.jar`

---

## ☁️ Publicação no Exchange <a id="publicação"></a>

```bash
# settings.xml com Connected App
mvn clean deploy -s .maven/settings.xml
```

A asset aparecerá em **Exchange → Private Assets** pronta para aplicar em políticas.

---

## ❓ FAQ <a id="faq"></a>

| Pergunta                                               | Resposta                                                                          |
| ------------------------------------------------------ | --------------------------------------------------------------------------------- |
| Posso aplicar múltiplas operações `transform-request`? | Sim, elas serão avaliadas na ordem em que aparecem.                               |
| O limite de headers pode ser alterado?                 | Injete a propriedade `http.headers.limit` no `mule-artifact.json`.                |
| `action="REMOVE"` sem value gera erro?                 | Somente se a policy marcar header como obrigatório; caso contrário apenas remove. |

---

## 🗺️ Roadmap <a id="roadmap"></a>

* **1.1.0** → Suporte a cookies e query‐params transform
* **1.2.0** → Unit tests MUnit + Jacoco cobertura
* **2.0.0** → Compatibilidade Mule 5 (quando GA)

---

## 🐞 Troubleshooting <a id="troubleshooting"></a>

| Sintoma                               | Diagnóstico                                 | Solução                                                  |
| ------------------------------------- | ------------------------------------------- | -------------------------------------------------------- |
| Policy falha com `LIMIT_EXCEEDED`     | Request possui cabeçalhos > limite          | Aumente propriedade `http.headers.limit` ou filtre antes |
| Headers não aparecem na resposta      | `transform-response` aplicado antes de erro | Use `error-handler` para repetir header em falhas        |
| Metadata não carrega no Design Center | Falta do resolver no classpath              | Rebuild e cheque `HttpPolicyTransformMetadataResolver`   |

---

## 🤝 Contribuindo <a id="contribuindo"></a>

1. Fork → `git checkout -b feature/NOME`
2. `mvn clean test` (MUnit)
3. `git commit -m "feat: descrição"`
4. Pull Request

Padrão **Conventional Commits**. Issues e melhorias são bem-vindas!

---

## 📄 Licença <a id="licença"></a>

**MIT** – veja `LICENSE`.

---

## 👨‍💼 Contato <a id="contato"></a>

**Leonel Dorneles Porto**
[leoneldornelesporto@outlook.com.br](mailto:leoneldornelesporto@outlook.com.br)
Accenture / Telefônica Vivo

---

<p align="center">
  <img src="https://readme-typing-svg.demolab.com?font=Fira+Code&size=22&pause=1000&color=00BFFF&center=true&vCenter=true&width=1000&lines=Transforma%C3%A7%C3%A3o+HTTP+pronta+para+qualquer+policy+%F0%9F%9A%80" />
</p>

---
