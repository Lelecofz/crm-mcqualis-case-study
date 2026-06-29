# CRM + Portal do Cliente — *Case Study*

**Português** · [English](README.en.md)

Um CRM interno e um portal do cliente para uma empresa de **intermediação de crédito**, desenvolvidos de raiz em **PHP/MySQL sem frameworks**. Este repositório é um *case study* do projeto — a narrativa, as decisões e os destaques técnicos. O código-fonte é privado (é uma ferramenta interna da empresa).

> 🎓 Projeto desenvolvido por mim durante a **Formação em Contexto de Trabalho (FCT)** na MCQualis. Trabalho *full-stack*, de ponta a ponta: modelo de dados, backend, frontend, UX e segurança.

---

## 📸 Demonstração

> _Capturas de ecrã (a adicionar em `assets/`):_
>
> - `assets/dashboard.png` — Dashboard com indicadores
> - `assets/kanban.png` — Funil de processos (Kanban)
> - `assets/cliente.png` — Ficha de cliente + linha do tempo
> - `assets/portal.png` — Portal do cliente (vista do processo)

---

## 🎯 O problema

Uma empresa de intermediação de crédito acompanha cada cliente num percurso longo: do primeiro contacto (*lead*), passando pela montagem do processo, decisão do banco e do cliente, até ao contrato. Esse acompanhamento estava disperso e não havia forma de o **cliente final** saber em que ponto estava o seu processo sem ligar ao consultor.

**Objetivo:** uma ferramenta única que (1) organize o trabalho da equipa e (2) dê ao cliente uma janela de transparência sobre o seu próprio processo.

---

## 🧩 A solução

Duas aplicações na mesma base de código, que partilham a base de dados mas são independentes:

- **CRM interno** (equipa) — leads, clientes, processos, tarefas, contactos, histórico, arquivo e gestão de utilizadores.
- **Portal do cliente** (área privada, só-leitura) — o cliente acompanha o estado do(s) seu(s) processo(s), sem ver notas internas nem comissões.

### Funcionalidades-chave
- **Funil lead → contrato** com conversão: uma lead torna-se cliente + processo, e a sua história inteira segue para a linha do tempo do processo.
- **Kanban de processos** com mudança de fase por arrastar (desktop) ou por seletor (telemóvel/tablet).
- **Linha do tempo** por cliente/processo: cada ação (criação, fase, comentário, tarefa, contacto) fica registada com autor e data.
- **Portal do cliente** por convite (token), com vista simplificada em *stepper*.
- **Exportação CSV** compatível com Excel pt-PT; **arquivo automático** de processos concluídos.
- **Papéis**: admin vê tudo; consultor só os seus registos.

---

## 🏗️ Arquitetura

```mermaid
flowchart TD
    subgraph Browser
        A[CRM - index.php shell] -->|fetch fragmentos + API| B
        P[Portal - index.php] -->|fetch API| B
    end
    B[API PHP - JSON { success, data, error }]
    B --> C[(MySQL / MariaDB)]
    A -. sessao PHPSESSID .-> S1[Auth equipa]
    P -. sessao PORTAL_CLIENTE .-> S2[Auth cliente]
    S1 --> C
    S2 --> C
```

**Decisões de arquitetura:**
- **SPA-lite sem framework:** o `index.php` é uma *shell* com a navegação; cada página é um fragmento HTML carregado via `fetch`, com o seu JS (re)injetado a cada visita. `api.js`/`comum.js` carregam uma vez. Resultado: navegação fluida sem o peso (nem o *build*) de uma framework.
- **Duas apps, uma BD:** CRM e portal têm autenticação e sessões **separadas** (`PHPSESSID` vs `PORTAL_CLIENTE`), o que permite expor só o portal à internet e manter o painel interno fechado.
- **API com envelope uniforme** `{ success, data, error }` — todo o frontend fala com o backend da mesma forma.

---

## 🔐 Destaques técnicos

Alguns pontos do projeto que mostram as decisões de engenharia (excertos ilustrativos):

**1. Contrato de API uniforme**, num único sítio — todos os endpoints respondem igual:
```php
function responder(bool $sucesso, $dados = null, ?string $erro = null, int $codigo = 200): void
{
    http_response_code($codigo);
    echo json_encode(['success' => $sucesso, 'data' => $dados, 'error' => $erro]);
    exit;
}
```

**2. CSRF num único ponto de controlo** — em vez de espalhar verificações, a proteção vive no `verificar_auth()`, por onde passam todas as mutações:
```php
if ($api && in_array($_SERVER['REQUEST_METHOD'], ['POST', 'PUT', 'DELETE'], true)) {
    $enviado = $_SERVER['HTTP_X_CSRF_TOKEN'] ?? '';
    if (!hash_equals($_SESSION['csrf'], $enviado)) {
        http_response_code(403); /* ... */ exit;
    }
}
```

**3. Rate limiting do login** — trava força-bruta contando falhas por IP numa janela curta:
```php
function login_bloqueado(PDO $pdo, string $ip): bool
{
    $stmt = $pdo->prepare(
        'SELECT COUNT(*) FROM login_tentativas
         WHERE ip = :ip AND sucesso = 0
           AND criado_em > (NOW() - INTERVAL 15 MINUTE)'
    );
    $stmt->execute([':ip' => $ip]);
    return (int) $stmt->fetchColumn() >= 5;
}
```

**4. XSS por omissão** — todo o output dinâmico passa por um *escape* central antes de ir para o DOM:
```js
const esc = (v) => String(v ?? '')
    .replace(/&/g, '&amp;').replace(/</g, '&lt;')
    .replace(/>/g, '&gt;').replace(/"/g, '&quot;');
```

Outras medidas: *prepared statements* em todo o lado, ordenação por lista-branca, bcrypt nas passwords, cabeçalhos de segurança (CSP/X-Frame-Options), cookies `HttpOnly`/`SameSite`/`Secure`, e um handler de erros global que nunca vaza detalhes internos.

---

## 🛠️ Stack
- **Backend:** PHP 8, PDO (*prepared statements*), arquitetura por endpoints.
- **Base de dados:** MySQL / MariaDB (11 tabelas com chaves estrangeiras e regras de integridade pensadas — `CASCADE`/`SET NULL` conforme a relação).
- **Frontend:** HTML, CSS e JavaScript *vanilla* — responsivo, sem dependências.
- **Sem** *build*, *bundlers* ou bibliotecas externas.

---

## 💡 O que aprendi
- Desenhar um **modelo de dados relacional** real (integridade referencial, o que apagar em cascata vs. anular).
- Construir uma **camada de API** consistente e um frontend dinâmico **sem** recorrer a uma framework.
- Pensar **segurança** de forma sistemática (do SQL/XSS ao CSRF, rate limiting e *hardening* de produção).
- Separar **produto** (a app da empresa) de **apresentação** (este case study) — incluindo a higiene de repositórios.

## 🚀 Próximos passos
Recuperação de password *self-service* e envio automático do convite por email (dependem de infraestrutura de email), logs de auditoria pesquisáveis e o deploy em produção (HTTPS, isolamento de rede, backups).

---

<sub>O código-fonte é privado por ser uma ferramenta interna da empresa. Posso mostrá-lo/explicá-lo numa conversa.</sub>
