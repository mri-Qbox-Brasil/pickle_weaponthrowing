# pickle_weaponthrowing — Manual

Permite arremessar a arma equipada: a arma sai do inventário, vira um prop físico no mundo e pode ser recolhida por qualquer jogador que chegue perto.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Comandos e teclas](#comandos-e-teclas)
5. [Como funciona](#como-funciona)
6. [Integrações](#integrações)
7. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
8. [Localização](#localização)
9. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `ox_lib` | Sim | `showTextUI` do prompt de coleta |
| `qb-core` ou `es_extended` | Sim, na prática | Sem um deles, a bridge `standalone` só imprime um aviso no console e as funções de inventário não existem |
| `ox_inventory` | Não | Bridge de inventário preferida. Preserva metadata e slot da arma |
| `qb-inventory` | Não | Bridge alternativa |
| `qs-inventory` | Não | Bridge alternativa |
| `core_inventory` | Não | Bridge alternativa |

Se nenhum inventário dedicado estiver iniciado, o recurso cai no inventário nativo do framework (`AddItem`/`RemoveItem` do `qb-core` ou do `es_extended`). Nesse caso a arma volta sem metadata (munição, acessórios e durabilidade são perdidos).

---

## Instalação

1. Copie a pasta `pickle_weaponthrowing` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure ox_lib
   ensure pickle_weaponthrowing
   ```
3. Não há SQL nem itens novos — o recurso só move armas que já existem no inventário.
4. **Standalone não é suportado de fato.** O `bridge/standalone/server.lua` apenas avisa que é preciso editar os arquivos de bridge.

---

## Configuração

Arquivo: `config.lua`.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Config.Debug` | bool | Não | Presente no config, mas não é lido em nenhum lugar do código |
| `Config.Language` | string | Sim | Idioma ativo. Deve existir em `locales/translations/` |
| `Config.DeathDropsWeapon` | bool | Sim | `true` faz o jogador soltar a arma equipada ao morrer |
| `Config.ThrowKeybind` | string | Sim | Tecla padrão do arremesso, registrada com `RegisterKeyMapping`. Padrão: `MOUSE_MIDDLE` |
| `Config.Weapons` | array de strings | Sim | Lista de armas arremessáveis. Uma arma fora dessa lista não é arremessada nem cai na morte |

O `Config.Weapons` que vem no repositório inclui praticamente todo o arsenal do jogo (corpo a corpo, pistolas, SMGs, escopetas, rifles, snipers e lançadores). Remova entradas para restringir.

---

## Comandos e teclas

| Comando | Permissão | Descrição |
|---|---|---|
| `/throwGun` | Todos | Arremessa a arma equipada, se ela estiver em `Config.Weapons` |

O comando é ligado a uma tecla via `RegisterKeyMapping` com o padrão de `Config.ThrowKeybind` (botão do meio do mouse). O jogador pode remapear em Configurações → Teclas → FiveM.

Para recolher uma arma no chão, chegue a menos de 1,25 m e pressione **E**. Um prompt aparece automaticamente a partir de 5 m.

---

## Como funciona

1. Ao arremessar, o client toca uma animação de takedown, remove a arma do ped, cria um prop com o modelo da arma e aplica velocidade na direção da câmera (potência fixa de 25).
2. O client informa o servidor com o nome da arma e o `net_id` do prop. O servidor confirma que o jogador realmente possui a arma, guarda os dados (incluindo metadata e slot, quando o inventário suporta) e remove o item do inventário.
3. Todos os clients passam a enxergar o prop como coletável.
4. Ao coletar, o servidor deleta o prop e devolve o item ao inventário de quem coletou — não necessariamente ao dono original.
5. Ao parar o recurso, todos os props de armas arremessadas são deletados. **As armas que estavam no chão são perdidas.**

O arremesso não funciona dentro de veículo, e o loop de coleta é ignorado para jogadores mortos ou dentro de veículo.

---

## Integrações

### ox_inventory

Usa `GetCurrentWeapon` para descobrir a arma equipada e `AddItem`/`RemoveItem` com metadata e slot. É a única bridge em que a arma volta com munição e acessórios preservados sem configuração extra.

### qb-inventory

O client escuta `inventory:client:UseWeapon` e informa ao servidor qual é a arma equipada. Preserva `slot` e `info`.

### qs-inventory

Usa `exports['qs-inventory']:GetCurrentWeapon` e as funções `AddItem`/`RemoveItem` do recurso.

### core_inventory

O client escuta `core_inventory:client:handleWeapon` e informa a arma e o inventário atual ao servidor. As notificações de adicionar/remover item do `core_inventory` são disparadas.

---

## Entrypoints para outros recursos

O recurso não expõe exports. Os eventos abaixo são internos, mas podem ser úteis para integração.

```lua
-- Client -> servidor: registra o arremesso. O servidor valida a posse da arma antes de aceitar.
TriggerServerEvent('pickle_weaponthrowing:throwWeapon', { weapon = 'WEAPON_PISTOL', net_id = ObjToNet(prop) })

-- Client -> servidor: coleta uma arma arremessada pelo seu ID interno.
TriggerServerEvent('pickle_weaponthrowing:pickupWeapon', weaponID)

-- Servidor -> todos os clients: sincroniza a lista de armas no chão. data = nil remove a entrada.
TriggerClientEvent('pickle_weaponthrowing:setWeaponData', -1, weaponID, data)

-- Client -> servidor: usado pelas bridges do qb-inventory e do core_inventory
-- para informar a arma equipada.
TriggerServerEvent('pickle_weaponthrowing:SetCurrentWeapon', weaponData)
```

---

## Localização

As strings ficam em `locales/translations/`, uma tabela `Language["<codigo>"]` por arquivo. Idiomas incluídos:

- `de.lua` — alemão
- `en.lua` — inglês
- `fr.lua` — francês

O idioma ativo é escolhido pelo `Config.Language` (não pela convar `ox:locale`). **Não há tradução para português** — para adicionar, crie `locales/translations/pt-br.lua` seguindo a estrutura dos existentes e aponte o `Config.Language` para ele.

---

## Estrutura de arquivos

```
pickle_weaponthrowing/
├── config.lua                    — armas arremessáveis, tecla, idioma, drop na morte
├── core/
│   └── client.lua                — helpers CreateProp, PlayAnim, ShowInteractText
├── modules/
│   └── weapon/
│       ├── client.lua            — arremesso, física do prop, drop na morte, loop de coleta, /throwGun
│       └── server.lua            — validação de posse, registro das armas no chão, coleta
├── bridge/
│   ├── qb/                       — qb-core: GetWeapon, AddWeapon, RemoveWeapon
│   ├── esx/                      — es_extended: GetWeapon, AddWeapon, RemoveWeapon
│   ├── inventory/                — ox_inventory, qb-inventory, qs-inventory e core_inventory
│   └── standalone/               — apenas avisa que nenhum framework suportado foi encontrado
├── locales/
│   ├── locale.lua                — função _L
│   └── translations/
│       ├── de.lua
│       ├── en.lua
│       └── fr.lua
└── fxmanifest.lua
```
