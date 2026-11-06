# 🔔 Alerta de Ofertas no WhatsApp (n8n + Evolution API + IA)

Você entra em vários grupos de promoção, arquiva todos e deixa este workflow ler as mensagens por você.
Quando aparecer **exatamente** o produto que você quer, ele te manda no privado com o produto, o preço e a oferta original.

```
Grupos WhatsApp ──► Evolution API ──┐
                                    ├──► Filtro por palavras ──► IA confirma ──► te avisa no WhatsApp
Canais Telegram (a cada 5 min) ─────┘
```

- **Filtro por palavras:** grátis, corta 95% do ruído.
- **IA (opcional):** confirma o que passou no filtro, separando "TV OLED" de "suporte para TV OLED", e permite definir preço máximo.
- **Sem repetição:** a mesma oferta postada em vários grupos chega só uma vez (janela de 12h).

---

## O que você precisa

| Item | Obrigatório? |
|---|---|
| n8n (cloud ou self-hosted) | ✅ |
| [Evolution API v2](https://doc.evolution-api.com/) com uma instância conectada ao seu WhatsApp | ✅ |
| Chave da OpenAI | Opcional (recomendado) |

> O seu número precisa estar **dentro** dos grupos. Pode arquivar os grupos à vontade: arquivar não impede a Evolution de receber as mensagens.

---

## Instalação (10 minutos)

### 1. Importar
No n8n: **Workflows → Import from File** → selecione `alerta-ofertas.n8n.json`.

### 2. Criar as credenciais
O arquivo **não traz credencial nenhuma**. Crie as suas:

| Nó | Tipo de credencial | Como preencher |
|---|---|---|
| **Envia no WhatsApp** | `Header Auth` | Name: `apikey` · Value: a API key da sua Evolution |
| **IA confirma oferta** | `OpenAI` | Sua API key da OpenAI (se não for usar IA, pule e coloque `usarIA: false` no CONFIG) |

### 3. Editar o nó `CONFIG`
É o **único nó que você precisa mexer**:

```js
evolutionUrl: 'https://sua-evolution.com.br',  // URL da sua Evolution, sem "/" no final
instancia: 'minha-instancia',                  // nome da instância na Evolution
meuNumero: '5511999999999',                    // para onde vão os alertas (DDI + DDD + número)
```

### 4. Ativar e ligar o webhook
1. Ative o workflow (toggle **Active**/**Publish**).
2. Abra o nó **WhatsApp (Evolution)** e copie a **Production URL**.
3. Na Evolution, configure o webhook da sua instância com essa URL e o evento **`MESSAGES_UPSERT`**.

> Se a sua instância já manda o webhook para outro workflow, adicione a saída desse outro workflow para chamar este (ou use um nó *Execute Workflow*). A Evolution só aceita uma URL de webhook por instância.

### 5. Descobrir o ID dos grupos
Entre no grupo de ofertas e **mande `!id`** (pelo seu próprio número). O workflow te responde no privado:

```
🆔 ID deste chat:
120363000000000001@g.us

Cole no nó CONFIG, em "grupos":
'120363000000000001@g.us': 'Nome do grupo',
```

Cole no CONFIG e apague o `!id` do grupo:

```js
grupos: {
  '120363000000000001@g.us': 'Ofertas de TVs',
  '120363000000000002@g.us': 'Promotec',
},
```

---

## Canais (WhatsApp Channels e Telegram)

**Canais do WhatsApp** (aqueles que você "segue") **não chegam pela Evolution**, só grupos chegam.
O jeito é usar o **espelho no Telegram**, que quase todo canal de oferta tem:

```js
canaisTelegram: [
  { usuario: 'achadosdotb', nome: 'Achados do Tecnoblog' },
  { usuario: 'ofertasadrenaline', nome: 'Ofertas Adrenaline' },
],
```

Teste antes: abra `https://t.me/s/<usuario>` no navegador. **Se aparecerem os posts, funciona.**
Se aparecer só o botão "View in Telegram", o canal desativou a prévia pública e não dá para ler. Isso acontece, por exemplo, com o canal do Canaltech (`ctofertas`).

O workflow confere os canais a cada 5 minutos. Na primeira execução ele só marca onde parou, sem te mandar o histórico.

---

## 🎯 Como filtrar outros produtos

Cada produto é um bloco em `alertas` dentro do CONFIG. **Para criar um novo, copie um bloco e ajuste.**

```js
{
  nome: 'Nome que aparece no alerta',
  ativo: true,
  precisaTer: [ [...], [...] ],  // pré-filtro: PELO MENOS 1 termo de CADA lista
  naoPodeTer: [ ... ],           // se tiver qualquer um, descarta
  criterio: '...',               // explicação em português para a IA
  precoMaximo: null,             // número em reais, ou null (precisa de IA)
  grupos: [],                    // [] = todas as fontes; ou só alguns nomes
},
```

### Regras do `precisaTer` (o mais importante)
- É uma **lista de listas**. Entre as listas vale **E**; dentro de cada lista vale **OU**.
  - `[['oled'], ['lg', 'samsung']]` → tem "oled" **E** ("lg" **OU** "samsung")
- Não diferencia maiúsculas nem acentos (`'polegadas'` casa com "Polegadas").
- Casa **palavra inteira**: `'55'` casa com `55"` mas **não** com `R$ 1.559`.
- Precisa de regex? Comece com `re:` → `'re:oled(55|65)'` casa com o modelo `OLED55C4`.

**Dica:** deixe o pré-filtro **largo** (é só pra economizar IA) e coloque o detalhe no `criterio`. A IA é quem decide de verdade.

### Exemplos prontos

**Air fryer grande até R$ 400**
```js
{
  nome: 'Air fryer 5L+ até R$ 400',
  ativo: true,
  precisaTer: [['air fryer', 'airfryer', 'fritadeira']],
  naoPodeTer: ['forma', 'papel', 'acessorio'],
  criterio: 'Oferta de fritadeira elétrica sem óleo (air fryer) com capacidade de 5 litros ou mais. Não vale acessórios.',
  precoMaximo: 400,
  grupos: [],
},
```

**Notebook com RTX 4060 ou superior**
```js
{
  nome: 'Notebook gamer RTX 4060+',
  ativo: true,
  precisaTer: [['notebook', 'laptop'], ['rtx', 're:\\b(4060|4070|4080|5060|5070)\\b']],
  naoPodeTer: ['mochila', 'cooler', 'suporte'],
  criterio: 'Oferta de notebook NOVO com placa de vídeo NVIDIA RTX 4060, 4070, 4080, 5060, 5070 ou superior.',
  precoMaximo: 7000,
  grupos: [],
},
```

**PS5 (qualquer versão), só de um grupo específico**
```js
{
  nome: 'PlayStation 5',
  ativo: true,
  precisaTer: [['ps5', 'playstation 5', 'playstation5']],
  naoPodeTer: ['controle', 'jogo', 'headset', 'capa', 'suporte', 'dualsense'],
  criterio: 'Oferta do console PlayStation 5 (Slim, Pro ou Digital). Não vale jogos, controles ou acessórios.',
  precoMaximo: null,
  grupos: ['Promotec'],
},
```

**Sem IA (só palavras)**: coloque `usarIA: false` no CONFIG. Aí capriche no `precisaTer` e no `naoPodeTer`, porque eles decidem sozinhos (e `precoMaximo` é ignorado).

---

## Extras

- **Mandar os alertas para um grupo** (você e sua família/amigos): em `meuNumero`, coloque o ID do grupo (`...@g.us`) em vez do número.
- **Modelo da IA:** `modeloIA` no CONFIG (padrão `gpt-5-mini`; qualquer modelo de chat da OpenAI com JSON mode serve).
- **A IA caiu ou acabou o crédito?** Você ainda recebe o alerta, marcado como _(não verificado pela IA)_. Melhor um alarme falso do que perder a oferta.

## Problemas comuns

| Sintoma | Causa provável |
|---|---|
| `!id` não responde | Webhook da Evolution não aponta pra *Production URL*, workflow inativo, ou evento `MESSAGES_UPSERT` desligado |
| Nada chega de um grupo | ID errado no CONFIG (use o `!id`), ou o grupo é na verdade um **canal** (veja a seção Canais) |
| Erro 401 no "Envia no WhatsApp" | Credencial Header Auth com Name diferente de `apikey` ou API key errada |
| Erro 404 no "Envia no WhatsApp" | `evolutionUrl` ou `instancia` errados no CONFIG |
| Canal do Telegram nunca manda nada | O canal não tem prévia pública (teste `https://t.me/s/<usuario>`) |
