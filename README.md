# ESP remoto (Drawing API)

ESP carregado do GitHub com `loadstring(game:HttpGet(url))()`. O arquivo
`ESP.luau` é um único chunk Luau que **retorna a tabela `api`**; nenhum
arquivo local é necessário no computador de quem executa.

> Ambiente: executor com `Drawing`, `loadstring`, `game:HttpGet` e (opcional)
> `getgenv`. Essas quatro funções **não** existem no Roblox padrão; todo o
> resto usado pelo ESP é API oficial do Roblox.

## Arquivos

| Arquivo | Para quê |
|---|---|
| `ESP.luau` | O ESP completo (Box, Corner, Fill, Nome, Distância, Vida, Tracer, Highlight, Bones R6/R15, entidades customizadas). É o arquivo publicado. |
| `loader/Main.luau` | Script principal: baixa, valida, configura e controla o ESP pela API (teclas F1/F2/F3, End). |
| `loader/Diagnostics.luau` | Verifica ambiente, download, compilação, API, configuração, instância única, R6/R15 e encerramento. |
| `types/executor.d.luau` | Tipos das APIs de executor (só para o editor / analisador). |
| `tests/` | Testes offline (mock do Roblox + VM Luau). Não precisam ser publicados. |

## Uso rápido

```lua
local ESP = loadstring(game:HttpGet("https://raw.githubusercontent.com/SEU_USUARIO/SEU_REPOSITORIO/main/ESP.luau"), "=ESP")()
ESP.SetConfigMany({ Box = true, Name = true, Bones = true })
```

Para uso real, prefira `loader/Main.luau`: ele trata erro de rede, URL
errada (404), arquivo corrompido, BOM, falha de inicialização e valida a API
antes de usá-la.

## Ciclo de vida

- **Instância única.** Ao ser executado, o ESP encerra a instância anterior
  registrada em `getgenv().__ESP_DRAWING_INSTANCE` (ou `_G`) antes de criar a
  nova. Executar duas vezes nunca duplica drawings, conexões ou o bind de
  render.
- **Falha de inicialização.** A instância se registra antes de conectar
  qualquer coisa; se algo falhar no meio, `Destroy()` roda e só então o erro é
  lançado (`[ESP] falha na inicialização: ...`). Nada fica ativo.
- **Erro de compilação** (download incompleto, página 404) não executa nada:
  a instância que já estava rodando continua intacta.
- **`Destroy()`** desvincula o render, desconecta todos os sinais, destrói
  os Highlights, remove todos os drawings (inclusive os do pool) e limpa o
  registro global. Pode ser chamado mais de uma vez.

## API pública

Todas as funções são chamadas com ponto (`ESP.SetConfig(...)`), não com `:`.

### Configuração

| Membro | Assinatura | Descrição |
|---|---|---|
| `SetConfig` | `(key: string, value) -> boolean` | Altera uma opção. Valida tipo, faixa e enum; valor inválido gera `warn`, retorna `false` e mantém o anterior. Números fora da faixa são limitados. |
| `SetConfigMany` | `(values: {[string]: any}) -> boolean` | Altera várias; retorna `false` se alguma for recusada (as válidas são aplicadas). |
| `GetConfig` | `(key: string?) -> any` | Valor de uma opção (`nil` se a chave não existe). **Sem argumento:** cópia de todas as opções (alterá-la não afeta o ESP). |
| `Config` | tabela | Proxy: `ESP.Config.Box = true` equivale a `SetConfig`; leitura e `for k, v in ESP.Config` funcionam. |
| `Defaults` | tabela | Cópia dos valores padrão. |
| `Refresh` | `() -> ()` | Força o redesenho de todas as entidades no próximo frame. |

### Ciclo de vida

| Membro | Assinatura | Descrição |
|---|---|---|
| `Version` | `string` | Versão do arquivo carregado (`"6.1.0"`). |
| `IsDestroyed` | `() -> boolean` | `true` depois de `Destroy()` ou de outra execução substituir esta instância. |
| `Destroy` | `() -> ()` | Encerra e libera tudo (idempotente). |

### Entidades e esqueleto

| Membro | Assinatura | Descrição |
|---|---|---|
| `GetEntityInfo` | `(player) -> EntityInfo?` | `status` (`none`, `loading`, `unsupported`, `dead`, `ready`), `reason`, `entity`, `source`, `adapter`, `anchor`, `partCount`, `health01`, `skeleton`, `boneCount`. |
| `DiagnoseSkeleton` | `(player) -> SkeletonDiagnosis?` | Por que um rig tem ou não esqueleto: `rig` (`R6`/`R15`/nil), `humanoidRigType`, `skeleton`, `segments`, `motors`, `animationConstraints`, `bones`, `incompleteJoints`, `profileJoints`/`profileExpected`, `missing`, `bonesEnabled`, `visibleLines`. |
| `SetEntitySources` | `(sources: {EntitySource}) -> boolean` | Quem o jogador controla, em ordem de prioridade. Padrão: `{ Sources.Character() }`. |
| `Sources` | tabela | Fontes prontas: `Character()`, `Vehicle()` (via `Humanoid.SeatPart`), `ObjectValue(nome)`. |
| `SetControlledEntity` | `(player, entity: Instance?) -> boolean` | Integração direta do seu sistema de controle; `nil` devolve às fontes. |
| `RegisterAdapter` | `(adapter) -> boolean` | Ensina a ler um tipo de entidade: `{ name, accepts, findAnchor?, collectParts?, readHealth?, watchHealth?, getSkeleton? }`. |
| `UnregisterAdapter` | `(name: string) -> boolean` | Remove um adaptador. |
| `Generic` | tabela | Implementações genéricas (`accepts`, `findAnchor`, `collectParts`, `readHealth`, `watchHealth`) para reaproveitar em adaptadores. |
| `RefreshEntity` | `(player?) -> ()` | Força nova resolução da entidade e releitura da estrutura (todos, se sem argumento). |

### Opções de configuração

Tabela gerada a partir de `DEFAULT_CONFIG` e `SCHEMA` em `ESP.luau`.
`*Trans` segue a convenção do Roblox: 0 = opaco, 1 = invisível.

| Opção | Padrão | Valores aceitos | Observação |
|---|---|---|---|
| `Box` | `true` | true / false |  |
| `Highlight` | `false` | true / false |  |
| `Name` | `false` | true / false |  |
| `Distance` | `false` | true / false |  |
| `HealthBar` | `false` | true / false |  |
| `Tracer` | `false` | true / false |  |
| `BoxFill` | `false` | true / false |  |
| `Bones` | `false` | true / false |  |
| `Shadows` | `true` | true / false |  |
| `DistanceFade` | `true` | true / false |  |
| `BoxMode` | `"Box"` | `Box` / `Corner` / `redraw` | "Box" / "Corner" |
| `BoxColor` | `RGB(236,239,244)` | Color3 |  |
| `BoxTrans` | `0` | 0 – 1 |  |
| `BoxThick` | `1` | 0.5 – 6 |  |
| `CornerRatio` | `0.25` | 0.05 – 0.5 |  |
| `ShadowColor` | `RGB(10,12,16)` | Color3 |  |
| `ShadowTrans` | `0.45` | 0 – 1 |  |
| `BoxFillMode` | `"Solid"` | `Solid` / `Gradient` / `redraw` | "Solid" / "Gradient" |
| `BoxFillColor` | `RGB(16,19,26)` | Color3 |  |
| `BoxFillTrans` | `0.8` | 0 – 1 |  |
| `BoxFillTopColor` | `RGB(110,150,255)` | Color3 |  |
| `BoxFillBotColor` | `RGB(16,19,26)` | Color3 |  |
| `BoxFillTopTrans` | `0.94` | 0 – 1 |  |
| `BoxFillBotTrans` | `0.7` | 0 – 1 |  |
| `TextColor` | `RGB(240,242,246)` | Color3 |  |
| `DistColor` | `RGB(148,156,170)` | Color3 |  |
| `TextSize` | `13` | 8 – 32 (inteiro) |  |
| `DistTextSize` | `11` | 8 – 32 (inteiro) |  |
| `TextFont` | `2` | 0 – 3 (inteiro) | Drawing.Fonts: 0 UI, 1 System, 2 Plex, 3 Monospace |
| `TextAutoScale` | `true` | true / false | escala com a altura da tela (ref. 1080p) |
| `TextGap` | `3` | 0 – 20 |  |
| `HpHigh` | `RGB(92,214,140)` | Color3 |  |
| `HpMid` | `RGB(240,192,84)` | Color3 |  |
| `HpLow` | `RGB(232,84,84)` | Color3 |  |
| `HealthBarWidth` | `2` | 1 – 8 |  |
| `HealthBarGap` | `3` | 0 – 20 |  |
| `TracerOrigin` | `"Bottom"` | `Bottom` / `Top` / `Center` / `tracer` | "Bottom" / "Top" / "Center" |
| `TracerColor` | `RGB(110,150,255)` | Color3 |  |
| `TracerThick` | `1` | 0.5 – 6 |  |
| `TracerTrans` | `0.45` | 0 – 1 |  |
| `HlFillColor` | `RGB(110,150,255)` | Color3 |  |
| `HlOutlineColor` | `RGB(110,150,255)` | Color3 |  |
| `HlFillTrans` | `0.85` | 0 – 1 |  |
| `HlOutlineTrans` | `0.2` | 0 – 1 |  |
| `MaxDistance` | `600` | 0 – ∞ |  |
| `DistNear` | `80` | 0 – ∞ |  |
| `DistMid` | `250` | 0 – ∞ |  |
| `FadeFar` | `0.65` | 0.1 – 1 | opacidade relativa no nível "longe" |
| `CompactBelow` | `12` | 0 – 200 | px: abaixo disso, oculta distância e barra de vida |
| `BonesColor` | `RGB(236,239,244)` | Color3 |  |
| `BonesThick` | `1` | 0.5 – 6 |  |
| `BonesTrans` | `0.15` | 0 – 1 |  |
| `BonesMaxDistance` | `250` | 0 – ∞ | esqueleto é detalhe: some antes da box |
| `HealthAttribute` | `"Health"` | texto (1–100) |  |
| `MaxHealthAttribute` | `"MaxHealth"` | texto (1–100) |  |
| `PointSize` | `2` | 0.1 – 50 | studs: entidade só com Attachment |

## Publicar no GitHub

1. Crie um repositório (público, para que o Raw funcione sem autenticação).
2. Envie `ESP.luau` para a raiz (pelo site: *Add file → Upload files*; ou pelo git):
   ```sh
   git init
   git add ESP.luau loader/ README.md
   git commit -m "ESP 6.1.0"
   git branch -M main
   git remote add origin https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git
   git push -u origin main
   ```
3. Abra o arquivo no GitHub e clique em **Raw**. A URL tem o formato
   `https://raw.githubusercontent.com/SEU_USUARIO/SEU_REPOSITORIO/main/ESP.luau`.
   Cole essa URL em `ESP_URL` no `loader/Main.luau` e no `loader/Diagnostics.luau`.
4. Rode `loader/Diagnostics.luau` no executor e confira `0 falha(s)`.

Cuidados com o arquivo:
- Salve como **UTF-8 sem BOM** e não edite pelo editor web de forma que
  quebre o arquivo. O Main remove BOM, mas `loadstring` puro não aceita.
- Não use a URL da página (`github.com/.../blob/...`): ela devolve HTML.
  O Main e o Diagnostics detectam isso e avisam.

## Atualizar

1. Edite `ESP.luau`, altere `Version` na tabela `api` (fim do arquivo) e envie
   (`git commit` + `git push`, ou *Edit* no site).
2. Quem usa a URL com `main` recebe a nova versão. O `raw.githubusercontent.com`
   usa cache de alguns minutos: se a versão antiga continuar aparecendo, espere
   ou use a URL fixada abaixo.
3. Para **fixar** uma versão (recomendado para não receber mudanças sem
   querer), use o hash do commit no lugar de `main`:
   `https://raw.githubusercontent.com/SEU_USUARIO/SEU_REPOSITORIO/<hash-do-commit>/ESP.luau`.
   Uma *tag* (`v6.1.0`) também funciona no lugar do hash.
4. Rodar o Main de novo troca a instância em execução pela nova versão.

## Testes offline

Sem Roblox: um mock das APIs usadas e uma VM Luau real. Ver `tests/README.md`.

```sh
tests/tools/runner/target/release/luaurun tests/run_tests.luau tests/mock_roblox.luau ESP.luau tests/baseline/esp_original.lua
tests/tools/runner/target/release/luaurun tests/remote_tests.luau tests/mock_roblox.luau ESP.luau loader/Main.luau loader/Diagnostics.luau
```
