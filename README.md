# 🔍 Guía rápida: Open Code Review (`ocr`) — revisión de código con IA, de Alibaba (ES · PT)

Una sola página para poner a trabajar **Open Code Review**: instalación, los modos que de verdad se usan, el mínimo para conectarlo a CI y cómo cambiar de modelo sin tocar el resto.

**Autor:** Guangyi Zhao · Octubre 2026

Proyecto original: https://github.com/alibaba/open-code-review (47.332 estrellas, consultado el 15 de octubre de 2026) · Documentación: https://open-codereview.ai/docs

> **Nota de autoría:** esta guía es una síntesis original, redactada por el autor a partir de la documentación pública del proyecto y de su propio uso. No es una traducción ni una copia de la documentación oficial, y no es un proyecto oficial de Alibaba. Para el detalle completo, ve al repositorio enlazado.

---

## 🇪🇸 Español

### Qué es, sin humo

`ocr` es un CLI de revisión de código con IA que nació como el asistente interno de revisión de Alibaba. Su diseño es híbrido: la parte que no puede fallar (qué archivos se revisan, cómo se agrupan, dónde va cada comentario) la decide ingeniería determinista, y el modelo de lenguaje hace el juicio sobre el código. Lo verificable que promete:

- Comentarios **a nivel de línea**, con módulos externos de posicionamiento y reflexión para que el comentario caiga donde toca.
- **Reglas integradas** por lenguaje: nulos (NPE), seguridad de hilos, XSS, inyección SQL, entre otras; las reglas se ajustan a cada archivo y se pueden personalizar por ruta.
- Selección de archivos precisa y agrupación de archivos relacionados en unidades de revisión, para que en cambios grandes no se «salte» archivos.
- Compatible con endpoints tipo **OpenAI y Anthropic**; tú eliges proveedor y modelo.

> **Sobre la cifra de «1/9 de tokens»:** sale del *benchmark* publicado por el propio proyecto (AACR-Bench: 50 repos, 200 PR reales, 1.505 problemas anotados), donde `ocr` logra mayor precisión y F1 que un agente generalista usando el mismo modelo, a cambio de menor *recall*. Es un resultado de los autores, no una medición independiente: tómalo como referencia, no como hecho.

### Instalación y primer uso

Requisito: **Git ≥ 2.41**.

```bash
npm install -g @alibaba-group/open-code-review

# Configura el modelo (interactivo: proveedor, clave, modelo, y prueba la conexión)
ocr config provider
ocr config model

cd tu-proyecto
ocr review
```

### Los modos, distinguidos

| Modo | Comando | Cuándo |
|---|---|---|
| Espacio de trabajo | `ocr review` | Revisa todo lo que tengas sin commitear: staged, unstaged y archivos nuevos |
| Rango de ramas | `ocr review --from main --to feature` | Revisar una rama contra su base (modo merge-base): el caso típico de un PR |
| Un commit | `ocr review --commit abc123` | Auditar un cambio concreto |
| Escaneo completo | `ocr scan` / `ocr scan --path internal/agent` | Revisar archivos enteros sin diff: auditar un repo que no conoces |
| Delegación | `ocr delegate preview` / `ocr delegate rule <archivos>` | Tu propio agente (Claude Code, Codex…) hace la revisión con su modelo; `ocr` solo selecciona archivos y resuelve reglas. **No necesita configurar LLM** |
| Reanudar | `ocr session list` + `--resume <id>` | Una revisión larga se cortó: sigue donde iba |
| Salida para máquinas | `ocr review --format json --output result.json` | Cuando otro programa o agente va a consumir el resultado |

### CI mínimo (GitHub Actions)

El proyecto trae una Action oficial reutilizable. Esto es todo lo que necesita tu repo, en `.github/workflows/ocr-review.yml`:

```yaml
name: OpenCodeReview PR Review
on:
  pull_request_target:
    types: [opened, synchronize, reopened]
permissions:
  contents: read
  pull-requests: write
jobs:
  code-review:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: alibaba/open-code-review@main
        with:
          llm_url: ${{ secrets.OCR_LLM_URL }}
          llm_auth_token: ${{ secrets.OCR_LLM_AUTH_TOKEN }}
          llm_model: ${{ vars.OCR_LLM_MODEL }}
          llm_use_anthropic: ${{ vars.OCR_LLM_USE_ANTHROPIC }}
```

Tres datos van como *secrets/vars* del repo: la URL del endpoint, el token y el nombre del modelo (`OCR_LLM_USE_ANTHROPIC` vale `'true'` si el endpoint es Anthropic y `'false'` si es compatible con OpenAI). Se usa `pull_request_target` para que los secrets estén disponibles también en PRs de forks; la Action solo lee el diff, no ejecuta código del PR. También hay ejemplos oficiales para GitLab CI, GitFlic CI y Gerrit en el repo.

**Para que sea reproducible:** fijar `@main` no basta, porque la Action instala el CLI desde npm en cada ejecución. Fija las dos cosas: la Action por SHA completo y la versión del CLI con el input `ocr_version`.

### Cambiar de modelo: las tres vías

| Vía | Cómo | Notas |
|---|---|---|
| Interactiva | `ocr config provider` → `ocr config model` | La normal en local; prueba la conexión al terminar |
| Variables de entorno (CI y scripts) | `OCR_LLM_URL`, `OCR_LLM_TOKEN`, `OCR_LLM_MODEL`, `OCR_LLM_USE_ANTHROPIC` | Sin tocar archivos de configuración |
| Sin modelo propio | Modo *delegate* | El modelo lo pone tu agente de código; `ocr` aporta selección y reglas |

### ⚠️ Errores típicos

- **Lanzar `ocr review` sin configurar modelo.** Salvo en modo *delegate*, no hay revisión sin endpoint: configura primero proveedor y modelo.
- **Esperar que lo encuentre todo.** La herramienta está afinada para precisión (pocas falsas alarmas) a costa de *recall*: úsala como un revisor más, no como el único.
- **Dejar las reglas por defecto en un repo raro.** Las reglas integradas cubren lo común; para tu proyecto, añade reglas personalizadas con filtros por ruta para que el modelo mire lo que a tu equipo le importa.

---

## 🇵🇹 Português

### O que é, sem fumaça

O `ocr` é um CLI de revisão de código com IA que nasceu como o assistente interno de revisão do Alibaba. O desenho é híbrido: a parte que não pode falhar (quais arquivos são revisados, como são agrupados, onde vai cada comentário) é decidida por engenharia determinística, e o modelo de linguagem faz o julgamento sobre o código. O que ele promete de verificável:

- Comentários **no nível da linha**, com módulos externos de posicionamento e reflexão para o comentário cair no lugar certo.
- **Regras integradas** por linguagem: nulos (NPE), segurança de threads, XSS, injeção de SQL, entre outras; as regras se ajustam a cada arquivo e podem ser personalizadas por caminho.
- Seleção de arquivos precisa e agrupamento de arquivos relacionados em unidades de revisão, para não «pular» arquivos em mudanças grandes.
- Compatível com endpoints tipo **OpenAI e Anthropic**; você escolhe provedor e modelo.

> **Sobre o número de «1/9 dos tokens»:** ele vem do *benchmark* publicado pelo próprio projeto (AACR-Bench: 50 repos, 200 PRs reais, 1.505 problemas anotados), em que o `ocr` alcança maior precisão e F1 do que um agente generalista usando o mesmo modelo, em troca de menor *recall*. É um resultado dos autores, não uma medição independente: trate como referência, não como fato.

### Instalação e primeiro uso

Requisito: **Git ≥ 2.41**.

```bash
npm install -g @alibaba-group/open-code-review

# Configure o modelo (interativo: provedor, chave, modelo, e testa a conexão)
ocr config provider
ocr config model

cd seu-projeto
ocr review
```

### Os modos, distinguidos

| Modo | Comando | Quando |
|---|---|---|
| Área de trabalho | `ocr review` | Revisa tudo o que está sem commit: staged, unstaged e arquivos novos |
| Intervalo de branches | `ocr review --from main --to feature` | Revisar uma branch contra a base (modo merge-base): o caso típico de um PR |
| Um commit | `ocr review --commit abc123` | Auditar uma mudança específica |
| Varredura completa | `ocr scan` / `ocr scan --path internal/agent` | Revisar arquivos inteiros sem diff: auditar um repo que você não conhece |
| Delegação | `ocr delegate preview` / `ocr delegate rule <arquivos>` | Seu próprio agente (Claude Code, Codex…) faz a revisão com o modelo dele; o `ocr` só seleciona arquivos e resolve regras. **Não precisa configurar LLM** |
| Retomar | `ocr session list` + `--resume <id>` | Uma revisão longa foi interrompida: continua de onde parou |
| Saída para máquinas | `ocr review --format json --output result.json` | Quando outro programa ou agente vai consumir o resultado |

### CI mínimo (GitHub Actions)

O projeto traz uma Action oficial reutilizável. Isto é tudo de que o seu repo precisa, em `.github/workflows/ocr-review.yml`:

```yaml
name: OpenCodeReview PR Review
on:
  pull_request_target:
    types: [opened, synchronize, reopened]
permissions:
  contents: read
  pull-requests: write
jobs:
  code-review:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: alibaba/open-code-review@main
        with:
          llm_url: ${{ secrets.OCR_LLM_URL }}
          llm_auth_token: ${{ secrets.OCR_LLM_AUTH_TOKEN }}
          llm_model: ${{ vars.OCR_LLM_MODEL }}
          llm_use_anthropic: ${{ vars.OCR_LLM_USE_ANTHROPIC }}
```

Três dados vão como *secrets/vars* do repo: a URL do endpoint, o token e o nome do modelo (`OCR_LLM_USE_ANTHROPIC` vale `'true'` se o endpoint for Anthropic e `'false'` se for compatível com OpenAI). Usa-se `pull_request_target` para os secrets ficarem disponíveis também em PRs de forks; a Action só lê o diff, não executa código do PR. Também há exemplos oficiais para GitLab CI, GitFlic CI e Gerrit no repo.

**Para ser reproduzível:** fixar `@main` não basta, porque a Action instala o CLI do npm a cada execução. Fixe as duas coisas: a Action pelo SHA completo e a versão do CLI com o input `ocr_version`.

### Trocar de modelo: as três vias

| Via | Como | Notas |
|---|---|---|
| Interativa | `ocr config provider` → `ocr config model` | A normal no local; testa a conexão ao terminar |
| Variáveis de ambiente (CI e scripts) | `OCR_LLM_URL`, `OCR_LLM_TOKEN`, `OCR_LLM_MODEL`, `OCR_LLM_USE_ANTHROPIC` | Sem tocar em arquivos de configuração |
| Sem modelo próprio | Modo *delegate* | O modelo é o do seu agente de código; o `ocr` entra com seleção e regras |

### ⚠️ Erros típicos

- **Rodar `ocr review` sem configurar modelo.** Fora do modo *delegate*, não há revisão sem endpoint: configure provedor e modelo primeiro.
- **Esperar que ele ache tudo.** A ferramenta é afinada para precisão (poucos alarmes falsos) ao custo de *recall*: use como mais um revisor, não como o único.
- **Deixar as regras padrão num repo fora do comum.** As regras integradas cobrem o básico; para o seu projeto, adicione regras personalizadas com filtros por caminho para o modelo olhar o que importa para o seu time.

---

## 🇬🇧 English summary

One-page cheat sheet (Spanish + Portuguese) for Alibaba's **Open Code Review** (`ocr`): install (`npm i -g @alibaba-group/open-code-review`, Git ≥ 2.41), model setup, the review modes that matter (workspace diff, branch range, single commit, full-file `scan`, delegation mode, resume), a minimal GitHub Actions setup using the official reusable action, and the three ways to switch models. The widely quoted "~1/9 tokens" figure is flagged as the project's own benchmark (AACR-Bench), not an independent measurement. Original synthesis by the author from the project's public documentation; not an official Alibaba document and not a translation.

**Author:** Guangyi Zhao · October 2026 · MIT License
