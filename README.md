# Atividade 05 — Rodar um modelo local

## Objetivo

Rodar modelo aberto localmente (LM Studio) e pedir algo que exige abstração
(SVG de algo específico). Comparar 2 tamanhos de modelo no mesmo prompt.
Registrar custo, latência, dependência e soberania de dados.

## Minha máquina

- CPU: Intel i5-12400F (6c/12t)
- RAM: 16 GB
- GPU: AMD Radeon RX 7600 (8 GB VRAM)
- OS: Windows 11 Pro
- Ferramenta: LM Studio (Ollama no Windows não acelera GPU AMD via CUDA;
  LM Studio usa backend Vulkan, aproveita a RX 7600)

## Modelos testados

| # | Modelo | Params | Quantização | Tamanho arquivo | Coube? |
|---|--------|--------|-------------|------------------|--------|
| 1 | Llama 3.2 3B Instruct | 3B | Q4_K_M | 2.02 GB |  |
| 2 | Qwen2.5 7B Instruct | 7B | Q4_K_M | 4.68 GB |  |

(baixados via `lms get meta/llama-3.2-3b` e `lms get qwen/qwen2.5-7b`)

## Prompt usado (mesmo para os dois)

```
Gere um SVG de um pelicano andando de bicicleta.
```

## Resultados

Rodado via `lms chat <modelo> -p "<prompt>" --stats`.

### Modelo pequeno — Llama 3.2 3B Instruct

- Tempo até 1º token: 0.584s
- Tokens/s: 102.01
- Tokens gerados: 658 (prompt: 68, total: 726)
- Resposta: não entregou SVG puro — devolveu HTML+CSS+instruções de como
  salvar em arquivo, misturado com o SVG. O SVG em si não parece pelicano
  nem bicicleta: usa círculos genéricos, chama parte do desenho de
  "orelhas" (pelicano não tem orelha), pedais e rodas soltos sem estrutura
  de bike reconhecível. Não segue instrução "só me dê o SVG".
- SVG bruto extraído: `outputs/modelo-pequeno.svg`
- Screenshot renderizado: `outputs/modelo-pequeno.png` — só um círculo marrom
  (cabeça?) com dois semicírculos pretos embaixo (rodas?). Nenhuma bike,
  nenhum pelicano.

### Modelo grande — Qwen2.5 7B Instruct

- Tempo até 1º token: 0.148s
- Tokens/s: 49.80
- Tokens gerados: 387 (prompt: 46, total: 433)
- Resposta: SVG mais limpo e direto (bloco único, sem HTML/CSS extra),
  mas o próprio modelo admite na resposta que é "bastante simplificado" e
  sugere Illustrator/Inkscape pra algo realista. Formas ainda genéricas
  (retângulo = quadro da bike, dois círculos = rodas, dois semicírculos =
  corpo/cabeça do pelicano) — reconhecível como "algo com rodas e um bicho
  em cima", mas não como pelicano específico.
- SVG bruto extraído: `outputs/modelo-grande.svg`
- Screenshot renderizado: `outputs/modelo-grande.png` — traço preto (guidão?)
  flutuando sobre retângulo cinza (quadro) e blob marrom/amarelo (cabeça)
  com dois pontos pretos (olhos?). Também não reconhecível como pelicano
  de bicicleta.

### Comparação

| Métrica | Pequeno (3B) | Grande (7B) |
|---|---|---|
| Tokens/s | 102.01 | 49.80 |
| TTFT | 0.584s | 0.148s |
| Tokens gerados | 658 | 387 |
| Seguiu instrução (só SVG) | Não | Quase (SVG limpo, mas com nota textual) |
| SVG reconhecível como pedido | Não | Parcialmente |

Nota: 3B foi mais rápido em tokens/s (modelo menor = menos compute por
token), mas gerou resposta mais longa e menos obediente ao formato pedido
— gastou tokens em HTML/CSS/instruções que não foram pedidos. 7B teve TTFT
menor (carregamento/prompt processing mais eficiente nesse teste) mas
throughput por token mais baixo, esperado por ter mais que o dobro de
parâmetros.

## Segundo teste (prompt diferente)

Prompt: `Gere um SVG de um cachorro roxo de capacete.` — mesma régua, mas
testando ligação de atributo (cor) + objeto composto (animal + acessório).

### Modelo pequeno — Llama 3.2 3B Instruct

- Tempo até 1º token: 0.072s
- Tokens/s: 101.77
- Tokens gerados: 404 (prompt: 67, total: 471)
- Resultado: **ignorou completamente "roxo"** — corpo e cabeça saem
  `fill="#FFFFFF"` (branco), contorno vermelho. **Sem capacete nenhum** —
  só cabeça (círculo), olhos, nariz e dois retângulos (corpo/pata).
  Falhou tanto na cor quanto no acessório pedido.
- SVG: `outputs/modelo-pequeno-teste2.svg` | Print: `outputs/modelo-pequeno-teste2.png`

### Modelo grande — Qwen2.5 7B Instruct

- Tempo até 1º token: 0.147s
- Tokens/s: 49.77
- Tokens gerados: 336 (prompt: 45, total: 381)
- Resultado: **acertou a cor roxa** (`fill="#8713fe"` no corpo e na
  cabeça). Capacete não vira forma reconhecível — é só uma linha preta
  fina cruzando o rosto, o próprio modelo chama isso de "capacete" no
  texto mas visualmente não existe. Sem orelhas/focinho de cachorro.
- SVG: `outputs/modelo-grande-teste2.svg` | Print: `outputs/modelo-grande-teste2.png`

### Achado principal do 2º teste

Ligação de atributo (cor) é onde a diferença de tamanho apareceu mais
nítida: o 3B simplesmente descartou "roxo" da instrução, o 7B aplicou a
cor certa em todas as formas do corpo. Mas nenhum dos dois compôs
"capacete" como objeto geométrico reconhecível — sugere que atributo
simples (cor) escala melhor com tamanho do que composição estrutural
(objeto sobre objeto), que continua fraca mesmo no 7B.

## Terceiro teste (prompt diferente)

Prompt: `Gere um SVG de uma casa amarela com um telhado vermelho e a porta marrom.`
— 3 atributos de cor simultâneos + objeto com estrutura muito comum em
tutoriais SVG (parede+telhado+porta), pra ver se familiaridade do padrão
ajuda a composição.

### Modelo pequeno — Llama 3.2 3B Instruct

- Tempo até 1º token: 0.073s
- Tokens/s: 101.86
- Tokens gerados: 389 (prompt: 77, total: 466)
- Resultado: **acertou as 3 cores** (fundo amarelo `#FFFFCC`, telhado
  vermelho, porta/janela marrom `#964B00`) — melhor que no teste do
  cachorro. Mas não existe "parede" como forma — telhado é um
  retângulo+path estranho descolado, porta e "janela" são retângulos
  soltos flutuando no fundo amarelo. Sem estrutura de casa reconhecível.
- SVG: `outputs/modelo-pequeno-teste3.svg` | Print: `outputs/modelo-pequeno-teste3.png`

### Modelo grande — Qwen2.5 7B Instruct

- Tempo até 1º token: 0.156s
- Tokens/s: 49.77
- Tokens gerados: 209 (prompt: 55, total: 264 — resposta mais curta dos
  3 testes)
- Resultado: **acertou as 3 cores e as 3 formas certas** — retângulo
  amarelo (parede), triângulo vermelho (telhado), retângulo marrom fino
  (porta). Melhor decomposição estrutural do teste inteiro. MAS: o SVG
  bruto **não tem `xmlns`** (`<svg width="200" height="300">` sem
  namespace) — como arquivo `.svg` standalone, o Chrome recusa renderizar
  e mostra a árvore XML crua em vez da imagem. É um SVG tecnicamente
  incompleto/inválido fora do contexto de um `<svg>` inline em HTML.
  Corrigido manualmente (`outputs/modelo-grande-teste3-fixed.svg`) só pra
  poder ver a forma: as peças certas existem, mas o telhado não encosta
  no corpo — flutua separado por erro de coordenadas (`y=30-100` pro
  telhado vs `y=150-230` pro corpo, sem sobreposição).
- SVG bruto: `outputs/modelo-grande-teste3.svg` (não renderiza sozinho)
- SVG corrigido: `outputs/modelo-grande-teste3-fixed.svg` | Print: `outputs/modelo-grande-teste3-fixed.png`

### Achado principal do 3º teste

Esse foi o teste onde os modelos locais chegaram mais perto: "casa com
telhado e porta" é um padrão extremamente comum em exemplos de tutorial
SVG (provavelmente super-representado no treino), diferente de "pelicano
de bike" ou "cachorro com capacete", que são composições raras. O 7B
chegou a acertar forma+cor das 3 partes — só errou a montagem espacial
(coordenadas não se tocam) e ainda produziu markup incompleto (sem
xmlns). Isso sugere que o gargalo não é só "tamanho do modelo" nem só
"abstração geométrica" — é também **quão comum é aquele padrão
específico no treino**. Pedidos raros/nunca vistos degradam muito mais
que pedidos comuns, mesmo dentro do mesmo modelo.

## Comparação com modelo grande (nuvem — Claude, rodado aqui na conversa)

Teste real, não estimativa: pedi os 3 mesmos prompts pro Claude (modelo de
fronteira, nuvem) e renderizei os SVGs do mesmo jeito.

- `outputs/claude-pelicano-bike.svg` / `.png`
- `outputs/claude-cachorro-roxo-capacete.svg` / `.png`
- `outputs/claude-casa.svg` / `.png` — parede amarela, telhado vermelho
  triangular encostado corretamente no corpo, porta marrom com maçaneta,
  janelas com moldura, chaminé. As 3 cores certas nas 3 partes certas,
  peças coladas na posição certa — a casa que os modelos locais quase
  acertaram (mesmas cores e formas certas no 7B) mas erraram a montagem.

Resultado: bicicleta com rodas de raio real, quadro triangulado, guidão e
pedal — reconhecível como bicicleta de cara. Pelicano com pescoço curvo,
bico laranja longo com papo, corpo emplumado — reconhecível como ave numa
bike. Cachorro roxo com focinho, orelhas, cauda e capacete amarelo com
viseira — todos os atributos pedidos (cor + acessório + animal) presentes
e reconhecíveis numa única forma composta.

Diferença central: Claude decompôs cada substantivo em **partes
anatômicas/estruturais corretas** (bico ≠ círculo genérico; roda de bike ≠
círculo preto sólido; capacete ≠ linha solta) e montou essas partes em
proporção e posição coerentes. Llama 3.2 3B e Qwen2.5 7B pararam no nível
de "forma geométrica primitiva rotulada por comentário" — o código *diz*
"pelicano"/"capacete" no comentário, mas a geometria não corresponde ao
rótulo. Isso é o teto de abstração composicional que a atividade pedia pra
expor.

- O que degradou: a etapa de decompor conceito → partes → geometria certa
  → composição em cena. Local (3B/7B) acerta às vezes um atributo solto
  (cor no 7B), mas não estrutura multi-parte.
- O que a autonomia comprou: geração 100% offline, sem custo de API, sem
  dado saindo da máquina, resposta em segundos mesmo em GPU de
  consumidor (RX 7600) sem CUDA. Troca-se qualidade de abstração
  composicional por soberania e custo marginal zero.

## Conclusão

Rodar local em hardware modesto (16GB RAM, GPU AMD sem CUDA) é viável até
~7B em Q4 com boa velocidade (50-100 tok/s), mas a régua do SVG expõe bem
o teto de abstração composicional desses tamanhos: ambos os modelos
produzem SVGs que tecnicamente renderizam, mas nenhum captura a forma real
do que foi pedido — o gargalo não é a ferramenta (LM Studio + Vulkan
funcionou sem fricção), é a capacidade do modelo pequeno de compor
conceito visual a partir de texto.

Isso é a resposta prática pra discussão de soberania: rodar local elimina
dependência de API externa, custo por token e risco de dado sair da
máquina — mas o preço é qualidade de output em tarefas que exigem
raciocínio composicional forte. Pra uso que não exige esse nível de
abstração (resumo, extração, classificação, chat simples), 3-7B local já
resolve sem trade-off perceptível. Pra tarefas como a do SVG, a diferença
pro modelo grande de nuvem é gritante — e isso é o próprio dado: o quanto
se perde é proporcional a quão "criativa"/composicional a tarefa exige.
