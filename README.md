# MIRAI - LUA API DOCUMENTATION [STRUCTURES]

## Overview
Conventions that apply to the whole API. Read once.

#### Conventions
* `Scope` : A script runs in one slot. Single-bot scope pre-binds the global [`bot`](#bot); global scope (`SLOT_ID == "global"`) instead exposes `getBot`/`getBots`/`addBot`/`addDevice`/`removeBot`. Check `EXECUTION_SCOPE`.
* `Fire-and-forget` : Most [`Bot`](#bot) action methods only queue a command and return immediately (no value). Return does NOT mean the in-game action succeeded.
* `Replies via events` : "Request" methods (`get_friends`, `clan_data*`, `pwe_*`, `*_availability`, ...) only send a request packet. The answer arrives later as a `packet_received` event — listen for it.
* `Blocking reads` : [`Bot`](#bot) properties (`pos`, `gems`, `world`, ...) do a real round-trip and block the worker until they return live data. Cheap once, costly in tight loops; each read re-queries.
* `Coordinates` : Tile coords are integers. `*_at` methods take absolute tile x/y; relative methods (`hit`, `place`, `plant`, `hit_air`, ...) take an offset from the bot's current tile. World pixel positions are floats.
* `Cancellation` : A stop request aborts the script even mid-loop (instruction hook every 512 ops) and raises `lua stopped`.
* `Sandbox` : `os.execute`, `os.exit`, `os.getenv` and `debug` are removed.
* `nil` : A property/getter returns `nil` when the underlying data is unavailable (bot offline, no world loaded, item missing).

## Globals
Free functions and variables available in every script.

#### Properties
* `bot` ([`Bot`](#bot)) : The [`Bot`](#bot) instance for this slot. Single-bot scope only (nil in global scope).
* `SLOT_ID` (`string`) : The slot id (bot id, or `"global"`).
* `EXECUTION_SCOPE` (`string`) : `"global"` or `"single:<id>"`.
* `events` (`table`) : Table of event-name constants (see [`Events`](#events)).
* `InventoryType` (`table`) : Maps inventory-type name to byte. `Block`=0, `BlockBackground`=1, `Seed`=2, `BlockWater`=3, `WearableItem`=4, `Weapon`=5, `Throwable`=6, `Consumable`=7, `Shard`=8, `Blueprint`=9, `Familiar`=10, `FAMFood`=11, `BlockWiring`=12.

#### Methods
* `print(...)` : Prints args (tab-joined) to the script console. Output is capped to the last 200 lines.
* `log(...)` : Alias of `print`.
* `sleep(ms: number)` : Async-sleeps the current thread for `ms` milliseconds. Yields to other threads.
* `now_ms() -> number` : Wall-clock milliseconds since the Unix epoch.
* `dumpTable(t: table, depth: number)` : Recursively prints a table. Pass `0` as the starting depth. Aborts past depth 200.
* `toJson(value: any) -> string` : Alias of [`json`](#json)`.encode`.
* `parseJson(s: string) -> any` : Alias of [`json`](#json)`.decode`.
* `getBot(name: string) -> Bot` : Global scope only. Returns the [`Bot`](#bot) with that id, or nil.
* `getBots() -> Bot[]` : Global scope only. Returns all [`Bot`](#bot) instances.
* `addBot(email: string, password: string) -> Bot` : Global scope only. Creates an email/password [`Bot`](#bot) and returns it.
* `addDevice(device_id?: string) -> Bot` : Global scope only. Creates a device-id (guest) [`Bot`](#bot); generates an id when omitted.
* `removeBot(id: string)` : Global scope only. Removes a bot.

#### Example
```lua
if EXECUTION_SCOPE == "global" then
  for _, b in ipairs(getBots()) do
    print(b.id, b.name)
  end
else
  print("running for", bot.id)
end
```

## Bot
A structure representing a single bot. Action methods are async and fire-and-forget unless a return type is shown.

#### Properties
* `id` (`string`) : The bot's id.
* `name` (`string`) : The bot's username, or nil.
* `status` (`string`) : The bot state as a PascalCase string (e.g. `"InWorld"`, `"MenuIdle"`).
* `connected` (`boolean`) : True when the bot is `InWorld` or `MenuIdle`.
* `ping` (`number`) : Latest ping in milliseconds, or nil.
* `auto_reconnect` (`boolean`) : Readable and writable (writing dispatches a command).
* `gems` (`number`) : Gem count (account data).
* `level` (`number`) : Account level.
* `skin` (`number`) : Skin color index.
* `gender` (`number`) : Gender index.
* `clothes` (`number[]`) : Array of worn item ids.
* `pos` ([`Position`](#position)) : The bot's position, or nil if not in a world.
* `direction` (`number`) : Facing direction value.
* `animation` (`number`) : Current animation value.
* `account` ([`Account`](#account)) : The account data, or nil.
* `inventory` ([`Inventory`](#inventory)) : The inventory instance, or nil.
* `world` ([`World`](#world)) : The world instance, or nil. Player and collectable lists are a snapshot taken at access time.
* `world_name` (`string`) : Current world name, or nil.
* `seeds` ([`Seed[]`](#seed)) : Array of planted seeds in the current world, or nil.
* `pathfind_config` ([`PathfindConfig`](#pathfindconfig)) : The pathfinding-config instance.

#### Methods
* `is_in_world(world_name?: string) -> boolean` : True if `InWorld`; if a name is given, also requires the world to match (case-insensitive).
* `is_in_tile(x: number, y: number) -> boolean` : True if the bot stands on that tile.
* `connect()` : Connects to the game server.
* `disconnect()` : Disconnects.
* `respawn()` : Respawns the bot.
* `warp(world: string, special?: boolean)` : Joins a world (`special` defaults false).
* `join_world(world: string)` : Joins a world (non-special).
* `leave()` : Leaves the current world.
* `get_automation_conf() -> table` : Returns the bot's automation config table, or nil.
* `walk(dx: number, dy: number)` : Sends a movement step by the given delta.
* `find_path(x: number, y: number) -> boolean` : Pathfinds to the tile and waits until arrival (300s timeout). Returns success.
* `start_path(x: number, y: number) -> boolean` : Starts pathfinding to the tile and returns once accepted (does not wait for arrival).
* `hit(dx: number, dy: number)` : Hits the foreground block at an offset from the bot.
* `hit_at(x: number, y: number)` : Hits the foreground block at absolute tile coords.
* `hit_air(dx: number, dy: number)` : Sends a hit-air at an offset.
* `hit_water(dx: number, dy: number)` : Sends a hit-water at an offset.
* `hit_background(dx: number, dy: number)` : Sends a hit-background at an offset.
* `hit_action(dx: number, dy: number)` : Sends a hit-action at an offset.
* `place(dx: number, dy: number, item_id: number, inv_type?: number)` : Places an item at an offset. `inv_type` defaults from the item's block config. Only `inv_type` 0/1/2/3/12 (block/background/seed/water/wiring) are valid; others error.
* `place_at(x: number, y: number, item_id: number, inv_type?: number)` : Same as `place` but absolute coords.
* `plant(dx: number, dy: number, item_id: number)` : Plants a seed at an offset.
* `plant_at(x: number, y: number, item_id: number)` : Plants a seed at absolute coords.
* `lock_world(item_id: number, x?: number, y?: number)` : Places a lock; defaults to the bot's current tile when coords omitted.
* `select_block(item_id: number, inv_type?: number)` : Selects a block/item in hand.
* `safe_touch(x: number, y: number)` : Sends a safe-touch on a tile.
* `collect(collectable_id: number)` : Collects a specific collectable by id.
* `collect_at(x: number, y: number) -> number` : Collects every collectable on a tile; returns the count.
* `collect_all() -> number` : Collects every collectable on the bot's tile; returns the count.
* `set_auto_collect(enabled: boolean, interval_ms?: number)` : Toggles auto-collect (`interval_ms` defaults 250).
* `equip(item_id: number)` : Wears an item.
* `unequip(item_id: number)` : Unwears an item.
* `drop(item_id: number, amount: number)` : Drops items.
* `trash(item_id: number, amount: number, inv_type?: number)` : Trashes items.
* `wearing(item_id: number) -> boolean` : True if the item is currently worn.
* `has_item(item_id: number, min?: number, inv_type?: number) -> boolean` : True if the bot holds at least `min` (default 1) of the item.
* `say(message: string)` : Sends a world chat message.
* `say_global(message: string)` : Sends a global chat message.
* `say_private(player: string, message: string)` : Sends a private message; resolves a name to an id when possible.
* `say_clan(message: string)` : Sends a clan chat message.
* `get_friends()` : Requests the friend list (reply arrives as an event).
* `add_friend(user: string)` : Sends a friend request (name resolved to id).
* `accept_friend(user: string)` : Accepts a friend request.
* `decline_friend(user: string)` : Declines a friend request.
* `remove_friend(user: string)` : Removes a friend.
* `kick_player(user: string)` : Kicks a player (world owner).
* `world_ban(user_id: string)` : Bans a player from the world.
* `ask_trade(partner: string)` : Asks a player to trade (name resolved to id).
* `ask_world_trade(partner: string)` : Asks a player for a world trade.
* `accept_trade(partner: string)` : Accepts a trade.
* `update_trade_items(my_trade_bc: number, my_trade_items: string, amount: string, is_world_trade: boolean, i_am_key_owner: boolean)` : Updates offered trade items.
* `lock_trade()` : Locks the trade.
* `commit_trade()` : Commits the trade.
* `cancel_trade(partner: string)` : Cancels the trade.
* `create_clan(name: string, tag: string, faction: number, info: string)` : Creates a clan.
* `terminate_clan()` : Terminates the bot's clan.
* `clan_name_availability(name: string)` : Requests a clan-name availability check (reply via event).
* `clan_tag_availability(tag: string)` : Requests a clan-tag availability check (reply via event).
* `clan_invite(player: string)` : Invites a player to the clan.
* `clan_accept_invite(clan_id: number)` : Accepts a clan invite.
* `clan_decline_invite(clan_id: number)` : Declines a clan invite.
* `clan_leave()` : Leaves the clan.
* `clan_kick(player: string)` : Kicks a clan member.
* `clan_promote(player: string, rank: number)` : Promotes a member.
* `clan_demote(player: string, rank: number)` : Demotes a member.
* `clan_data(clan_id: number)` : Requests clan data (reply via event).
* `clan_member_list(clan_id: number)` : Requests the clan member list (reply via event).
* `clan_data_and_members(clan_id: number)` : Requests clan data + members (reply via event).
* `clan_edit_info(clan_id: number, info: string)` : Edits clan info text.
* `clan_donate(clan_id: number, gem_amount: number)` : Donates gems to the clan.
* `clan_level_up(clan_id: number)` : Requests a clan level-up.
* `clan_member_level_up(clan_id: number)` : Requests a clan member level-up.
* `clan_member_info(clan_id: number, target: string)` : Requests info on a clan member (reply via event).
* `claim_daily_clan_gem_bonus()` : Claims the daily clan gem bonus.
* `create_pet(block_type: number, item_data: table)` : Creates a pet from item data.
* `pet_feed(aipet_id: number)` : Feeds a pet.
* `pet_heal(aipet_id: number)` : Heals a pet.
* `pet_clean(aipet_id: number)` : Cleans a pet.
* `pet_pet(aipet_id: number)` : Pets a pet.
* `pet_train(aipet_id: number, training_type: number)` : Trains a pet.
* `pet_start_train(aipet_id: number)` : Starts pet training.
* `pet_end_train(aipet_id: number)` : Ends pet training.
* `pet_start_adventure(aipet_id: number)` : Sends a pet on an adventure.
* `pet_start_freeze(aipet_id: number, time_in_days: number)` : Freezes a pet for N days.
* `pet_end_freeze(aipet_id: number)` : Unfreezes a pet.
* `pet_level_up(aipet_id: number)` : Levels up a pet.
* `pet_claim_adventure_prize(aipet_id: number)` : Claims a pet adventure prize.
* `pet_command_chat(aipet_id: number, message: string)` : Sends a pet command via chat.
* `familiar_change(hot_spot_block_type: number)` : Changes the active familiar.
* `familiar_rename(item_data: table, rename_value: string)` : Renames a familiar.
* `familiar_feed(inventory_key_pair: string, item_data: table)` : Feeds a familiar.
* `play_familiar_animation(animation: number)` : Plays a familiar animation.
* `open_shop()` : Opens the in-game shop.
* `close_shop()` : Closes the shop (uses the current world).
* `buy_pack(pack_id: string)` : Buys an item pack.
* `bank_open(x: number, y: number)` : Opens a bank at a tile.
* `bank_deposit(x: number, y: number, item_id: number, amount: number)` : Deposits items into a bank.
* `bank_withdraw(x: number, y: number, index: number, amount: number)` : Withdraws items from a bank slot.
* `convert_items(inventory_key: number)` : Converts/crafts items by inventory key.
* `recycle_block(amount: number, x: number, y: number)` : Recycles blocks at a tile.
* `claim_recycle_prize(x: number, y: number)` : Claims a recycle prize.
* `craft_blueprint()` : Crafts the active blueprint.
* `craft_fishing_gear()` : Crafts fishing gear.
* `craft_fishing_rod_upgrade()` : Crafts a fishing-rod upgrade.
* `recycle_fish(amount: number)` : Recycles fish.
* `craft_flying_mount()` : Crafts a flying mount.
* `craft_mining_gear()` : Crafts mining gear.
* `craft_mining_pickaxe_upgrade()` : Crafts a pickaxe upgrade.
* `recycle_mining_gemstone(amount: number)` : Recycles mining gemstones.
* `buy_mining_roulette_tokens()` : Buys mining roulette tokens.
* `spin_mining_roulette()` : Spins the mining roulette.
* `pwe_get_item_listing(index: number, sort: number)` : Requests an auction item listing (reply via event).
* `pwe_get_main_category_listing(values: string, index: number)` : Requests a main-category listing.
* `pwe_get_category_listing(values: string, index: number)` : Requests a category listing.
* `pwe_get_custom_category_listing(values: string)` : Requests a custom-category listing.
* `pwe_get_player_item_listing()` : Requests the bot's own item listings.
* `pwe_get_order_item_listing(index: number, sort: number)` : Requests an order item listing.
* `pwe_get_order_main_category_listing(values: string, index: number)` : Requests an order main-category listing.
* `pwe_get_order_category_listing(values: string, index: number)` : Requests an order category listing.
* `pwe_get_order_custom_category_listing(values: string)` : Requests an order custom-category listing.
* `pwe_get_player_order_listing()` : Requests the bot's own orders.
* `pwe_query_pending_items()` : Queries pending auction items.
* `pwe_get_price_history()` : Requests price history.
* `pwe_get_price_history_pwe()` : Requests PWE price history.
* `pwe_get_price_history_pwe_order()` : Requests PWE order price history.
* `pwe_set_item_for_sale(amount: number, byte_coin_amount: number, time_in_minutes: number, x: number, y: number)` : Lists an item for sale.
* `pwe_cancel_item_sale(timestamp: number, x: number, y: number)` : Cancels a sale.
* `pwe_claim_sold_funds(timestamp: number, x: number, y: number)` : Claims funds from a sold item.
* `pwe_claim_expired_item(timestamp: number, x: number, y: number)` : Claims an expired listing's item.
* `pwe_buy_out_item(timestamp: number, x: number, y: number)` : Buys out a listing.
* `pwe_create_order(amount: number, byte_coin_amount: number, time_in_minutes: number, x: number, y: number)` : Creates a buy order.
* `pwe_fulfill_order(timestamp: number, x: number, y: number)` : Fulfills an order.
* `pwe_claim_order(timestamp: number, x: number, y: number)` : Claims a completed order.
* `pwe_claim_expired_order(timestamp: number, x: number, y: number)` : Claims an expired order.
* `pwe_cancel_order(timestamp: number, x: number, y: number)` : Cancels an order.
* `storage_interact(item_data: table, x: number, y: number)` : Interacts with a storage block.
* `mannequin_interact(item_data: table, x: number, y: number, swap: boolean)` : Interacts with a mannequin.
* `display_interact(item_data: table, x: number, y: number)` : Interacts with a display block.
* `enter()` : Walks through the portal under the bot (built-in portal-enter routine).
* `send_portal_enter(x: number, y: number)` : Sends a portal-enter packet for a tile.
* `send_portal_exit(x: number, y: number)` : Sends a portal-exit packet for a tile.
* `send_packets(packets: table[], buffer?: boolean)` : Sends raw packet documents (see [`Packet`](#packet)). `buffer` (default true) batches via the send gate; false sends immediately, bypassing the throttle.
* `send_packet(packet_id: string, fields: table, buffer?: boolean)` : Sends one raw packet built from `packet_id` + fields.
* `send_map_point(x: number, y: number)` : Queues a single map-point movement packet (binary blob built in Rust).
* `claim_xp_level()` : Claims an XP level reward.
* `start_tutorial()` : Starts the tutorial flow.
* `claim_quest(quest_id: string, tutorial?: boolean)` : Claims a quest. `tutorial` is accepted but unused.
* `claim_achievement_reward(achievement_reward_claimed: string)` : Claims an achievement reward.
* `open_treasure(remove_item?: boolean)` : Opens a treasure (`remove_item` default false).
* `set_status_icon(icon: number)` : Sets the bot's status icon.
* `play_audio(audio_type: number, block_type: number)` : Plays an audio cue.
* `buy_inventory_slots()` : Buys an inventory-slot expansion.
* `emit_event(event_name: string, password: string)` : Internal. Re-emits a bot event; requires the internal password and only supports `"tutorial_completed"`.

#### Example
```lua
bot:warp("MIRAI")
sleep(2000)
if bot:has_item(242, 1) then       -- 242 = dirt seed
  bot:plant(0, 0, 242)
end
bot:say("hello from " .. (bot.name or "?"))
```

## World
World state for a bot. Returned by [`Bot`](#bot)`.world`.

#### Properties
* `name` (`string`) : World name, or nil.
* `size` ([`Vector2i`](#vector2i)) : World dimensions (width, height).
* `entrance` ([`Vector2i`](#vector2i)) : Spawn/entrance tile, or nil.
* `players` ([`Player[]`](#player)) : Players in the world (snapshot at access time).
* `collectables` ([`Collectable[]`](#collectable)) : Collectables in the world (snapshot at access time).

#### Methods
* `player(user_id: string) -> Player` : Finds a [`Player`](#player) in the snapshot by id, or nil.
* `collectable(id: number) -> Collectable` : Finds a [`Collectable`](#collectable) in the snapshot by id, or nil.
* `find_portal_id(entrance_id: string) -> Vector2i` : Locates a portal tile by its entrance id (live lookup) as a [`Vector2i`](#vector2i), or nil.
* `get_tile(x: number, y: number) -> Tile` : Returns the [`Tile`](#tile) at coords (live), or nil.
* `get_tiles(matcher?: table) -> Tile[]` : Returns matching [`Tile`](#tile) entries. Matcher fields: `item_id`, `inventory_order_type`, `is_seed`, `is_foreground`, `is_background`, `is_water`, `is_wiring`. No matcher returns all tiles.
* `get_tile_item(x: number, y: number) -> table` : Returns the tile's extra item data, or nil.
* `find_tiles(block_type: number) -> Vector2i[]` : Returns the [`Vector2i`](#vector2i) position of every foreground tile of that block type.
* `is_walkable(x: number, y: number) -> boolean` : True if the foreground tile is passable.
* `find_player(name_or_id: string) -> string` : Resolves a name or id to a user id (live), or nil.

#### Example
```lua
local w = bot.world
if w then
  print(w.name, w.size.x .. "x" .. w.size.y)
  for _, door in ipairs(w:find_tiles(6)) do  -- 6 = main door
    print("door at", door.x, door.y)
  end
end
```

## Inventory
The bot's inventory. Returned by [`Bot`](#bot)`.inventory`.

#### Properties
* `slots` (`number`) : Total inventory slot count.
* `items` ([`InventoryItem[]`](#inventoryitem)) : All held items.

#### Methods
* `count(item_id: number, inv_type?: number) -> number` : Amount held of an item (0 if none).
* `item(item_id: number, inv_type?: number) -> InventoryItem` : The matching [`InventoryItem`](#inventoryitem), or nil.
* `has(item_id: number, inv_type?: number) -> boolean` : True if the item is present.

## PathfindConfig
Per-bot pathfinding tuning. Returned by [`Bot`](#bot)`.pathfind_config`.

#### Methods
* `get_step_delay_ms() -> number` : Delay between path steps, in milliseconds.
* `set_step_delay_ms(ms: number)` : Sets the step delay.
* `get_step_size() -> number` : Tiles moved per step.
* `set_step_size(n: number)` : Sets the step size.

## Position
A position, returned by [`Bot`](#bot)`.pos` and embedded in other structs.

#### Properties
* `x` (`number`) : World pixel X (float).
* `y` (`number`) : World pixel Y (float).
* `tile_x` (`number`) : Tile X (integer).
* `tile_y` (`number`) : Tile Y (integer).

## Player
Another player in the world.

#### Properties
* `id` (`string`) : User id.
* `name` (`string`) : Username.
* `pos` ([`Position`](#position)) : The player's position.

## Collectable
A dropped/spawned collectable in the world.

#### Properties
* `id` (`number`) : Collectable id.
* `block_type` (`number`) : Block type id.
* `inventory_type` (`number`) : Inventory type byte.
* `amount` (`number`) : Quantity.
* `is_gem` (`boolean`) : True if it is a gem.
* `gem_type` (`number`) : Gem type.
* `pos` ([`Position`](#position)) : The collectable's position.

## Tile
A single world tile. Returned by [`World`](#world)`:get_tile` / [`World`](#world)`:get_tiles`.

#### Properties
* `fg` (`number`) : Foreground block id (0 = none).
* `bg` (`number`) : Background block id (0 = none).
* `water` (`number`) : Water-layer block id (0 = none).
* `wiring` (`number`) : Wiring-layer block id (0 = none).
* `raw` (`table`) : Extra tile data (present only for tiles that carry it).
* `pos` ([`Position`](#position)) : The tile's position.

## Seed
A planted seed. Returned in the [`Bot`](#bot)`.seeds` array.

#### Properties
* `x` (`number`) : Tile X.
* `y` (`number`) : Tile Y.
* `block_type` (`number`) : Seed block type.
* `ready` (`boolean`) : True if growth is complete.
* `remaining_ms` (`number`) : Milliseconds left until ready (0 when ready).
* `harvest_seeds` (`number`) : Seeds yielded on harvest.
* `harvest_blocks` (`number`) : Blocks yielded on harvest.
* `harvest_gems` (`number`) : Gems yielded on harvest.

## Account
Account data. Returned by [`Bot`](#bot)`.account`.

#### Properties
* `xp` (`number`) : Total XP.
* `xp_to_next_level` (`number`) : XP needed for the next level.
* `hot_spots` (`number[]`) : Array of hot-spot block types.
* `belt` (`number`) : Belt value.
* `statistics` (`number[]`) : Array of statistic values.
* `face_animation` (`number`) : Face animation value.
* `country_code` (`number`) : Country code.
* `name_change_counter` (`number`) : Number of name changes used.

## InventoryItem
One inventory entry. Returned by [`Inventory`](#inventory)`.items` / [`Inventory`](#inventory)`:item`.

#### Properties
* `id` (`number`) : Item id.
* `amount` (`number`) : Quantity held.
* `flags` (`number`) : Item flag byte.
* `inventory_type` (`number`) : Inventory type byte.
* `inventory_type_name` (`string`) : Inventory type name (e.g. `"Seed"`).
* `name` (`string`) : Item name (present when known).

## Packet
The `packet` builder. Builds packet documents to feed [`Bot`](#bot)`:send_packets`. Each builder returns one packet table, or nil on a build failure. Passing a wrong argument type raises an error.

#### Properties
* `MOVE_ID` (`string`) : Player-position packet id.
* `HIT_BLOCK_ID` (`string`) : Hit-block packet id.
* `HIT_BACKGROUND_ID` (`string`) : Hit-background packet id.
* `HIT_WATER_ID` (`string`) : Hit-water packet id.
* `HIT_AIR_ID` (`string`) : Hit-air packet id.
* `SET_BLOCK_ID` (`string`) : Set-block packet id.
* `SET_BACKGROUND_ID` (`string`) : Set-background packet id.
* `SET_SEED_ID` (`string`) : Set-seed packet id.
* `COLLECT_ID` (`string`) : Collect-collectable packet id.
* `DROP_ITEM_ID` (`string`) : Drop-item packet id.
* `WORLD_CHAT_ID` (`string`) : World-chat packet id.
* `GLOBAL_CHAT_ID` (`string`) : Global-chat packet id.
* `WEAR_ID` (`string`) : Wear packet id.
* `UNWEAR_ID` (`string`) : Unwear packet id.
* `USE_CONSUMABLE_ID` (`string`) : Use-consumable packet id.
* `UPDATE_BELT_ID` (`string`) : Update-belt packet id.
* `JOIN_WORLD_ID` (`string`) : Join-world packet id.
* `LEAVE_WORLD_ID` (`string`) : Leave-world packet id.
* `RESPAWN_ID` (`string`) : Respawn packet id.
* `CHARACTER_CREATED_ID` (`string`) : Character-created packet id.
* `TUTORIAL_STATE_ID` (`string`) : Tutorial-state packet id.

#### Methods
* `build_move(x: number, y: number, animation?: number, direction?: number, teleport?: boolean) -> table` : Builds a position packet (`animation` default 1, `direction` 0, `teleport` false).
* `build_hit_block(x: number, y: number) -> table` : Builds a hit-block packet.
* `build_hit_background(x: number, y: number) -> table` : Builds a hit-background packet.
* `build_hit_water(x: number, y: number) -> table` : Builds a hit-water packet.
* `build_hit_air(x: number, y: number) -> table` : Builds a hit-air packet.
* `build_set_block(x: number, y: number, item_id: number) -> table` : Builds a set-block packet.
* `build_set_background(x: number, y: number, item_id: number) -> table` : Builds a set-background packet.
* `build_set_seed(x: number, y: number, item_id: number) -> table` : Builds a set-seed packet.
* `build_collect(collectable_id: number) -> table` : Builds a collect packet.
* `build_drop_item(x: number, y: number, item_id: number, inventory_type: number, amount: number) -> table` : Builds a drop-item packet.
* `build_world_chat(message: string) -> table` : Builds a world-chat packet.
* `build_global_chat(message: string) -> table` : Builds a global-chat packet.
* `build_wear(item_id: number) -> table` : Builds a wear packet.
* `build_unwear(item_id: number) -> table` : Builds an unwear packet.
* `build_use_consumable(item_id: number) -> table` : Builds a use-consumable packet.
* `build_update_belt(item_id: number) -> table` : Builds an update-belt packet.
* `build_join_world(world: string, special?: boolean) -> table` : Builds a join-world packet.
* `build_leave_world() -> table` : Builds a leave-world packet.
* `build_respawn(ticks?: number) -> table` : Builds a respawn packet (defaults to current .NET ticks).
* `build_character_created(gender: number, country: number, skin_color_index: number) -> table` : Builds a character-created packet.
* `build_tutorial_state(state: number) -> table` : Builds a tutorial-state packet.

#### Example
```lua
bot:send_packets({
  packet:build_move(120.0, 96.0),
  packet:build_world_chat("hi"),
})
```

## Events
The event system. Callbacks registered with `addEvent` only fire while `listenEvents` is running.

#### Properties
* `events.PACKET_RECEIVED` (`string`) : `"packet_received"`. Payload: [`PacketReceivedEvent`](#packetreceivedevent).
* `events.CHAT_MESSAGE` (`string`) : `"chat_message"`. Payload: [`ChatMessageEvent`](#chatmessageevent).
* `events.STATE_CHANGED` (`string`) : `"state_changed"`. Payload: [`StateChangedEvent`](#statechangedevent).
* `events.JOINED_WORLD` (`string`) : `"joined_world"`. Payload: [`JoinedWorldEvent`](#joinedworldevent).
* `events.ERROR` (`string`) : `"error"`. Payload: [`ErrorEvent`](#errorevent).
* `events.TUTORIAL_COMPLETED` (`string`) : `"tutorial_completed"`. Payload: [`TutorialCompletedEvent`](#tutorialcompletedevent).
* `events.PACK_PURCHASED` (`string`) : `"pack_purchased"`. Payload: [`PackPurchasedEvent`](#packpurchasedevent).
* `events.PRESEND` (`string`) : `"presend"`. Payload: [`PreSendEvent`](#presendevent).

#### Methods
* `addEvent(event: string, callback: function)` : Registers a callback for an event. Only the eight names above are accepted; others error. Multiple callbacks per event are allowed.
* `removeEvent(event: string)` : Removes all callbacks for one event.
* `removeEvents()` : Removes all callbacks for all events.
* `listenEvents(seconds: number)` : Subscribes and dispatches events to callbacks until the timeout elapses (or `unlistenEvents` is called). In global scope it listens to every bot; otherwise to this slot. Callbacks may be async.
* `unlistenEvents()` : Signals the running `listenEvents` loop to stop after the current callback.
* `waitForPacket(opts?: table, timeout_sec: number) -> boolean` : Blocks until a matching `packet_received` arrives or the timeout elapses; returns whether it matched. `opts.id` matches a packet id; `opts.match_cb(fields)` matches sub-document fields; both nil matches the first packet. Note: this clears all `packet_received` callbacks when it finishes.

#### Example
```lua
addEvent("chat_message", function(e)
  print(e.username .. ": " .. e.message)
  if e.message == "stop" then unlistenEvents() end
end)
listenEvents(30)
removeEvent("chat_message")
```

## PacketReceivedEvent
Payload passed to a `packet_received` callback.

#### Properties
* `type` (`string`) : `"packet_received"`.
* `bot_id` (`string`) : The bot id.
* `seq` (`number`) : Sequence number.
* `ids` (`string[]`) : Packet-id strings in the envelope.
* `packet` (`table`) : Table `{ seq, at, direction, ids, document }`. `document` holds the envelope sub-packets as `m0`, `m1`, ... plus the message count.
* `ping_ms` (`number`) : Ping in milliseconds, or nil.

## ChatMessageEvent
Payload for a `chat_message` callback.

#### Properties
* `type` (`string`) : `"chat_message"`.
* `bot_id` (`string`) : The bot id.
* `channel` (`string`) : Chat channel.
* `user_id` (`string`) : Sender id.
* `username` (`string`) : Sender name.
* `message` (`string`) : Message text.

## StateChangedEvent
Payload for a `state_changed` callback.

#### Properties
* `type` (`string`) : `"state_changed"`.
* `bot_id` (`string`) : The bot id.
* `state` (`string`) : The new bot state (PascalCase, e.g. `"InWorld"`).

## JoinedWorldEvent
Payload for a `joined_world` callback.

#### Properties
* `type` (`string`) : `"joined_world"`.
* `bot_id` (`string`) : The bot id.
* `world` (`string`) : World name joined.

## ErrorEvent
Payload for an `error` callback.

#### Properties
* `type` (`string`) : `"error"`.
* `bot_id` (`string`) : The bot id.
* `message` (`string`) : Error message.

## TutorialCompletedEvent
Payload for a `tutorial_completed` callback.

#### Properties
* `type` (`string`) : `"tutorial_completed"`.
* `bot_id` (`string`) : The bot id.

## PackPurchasedEvent
Payload for a `pack_purchased` callback.

#### Properties
* `type` (`string`) : `"pack_purchased"`.
* `bot_id` (`string`) : The bot id.
* `pack_id` (`string`) : Purchased pack id.
* `rewards` (`number[]`) : Reward item ids.
* `status` (`string`) : Result status.

## PreSendEvent
Payload for a `presend` callback.

#### Properties
* `type` (`string`) : `"presend"`.
* `bot_id` (`string`) : The bot id.

## json
JSON encoding helpers. Available as the `json` global.

#### Methods
* `encode(value: any) -> string` : Serializes a Lua value to JSON. Arrays are tables with contiguous integer keys `1..n`; an empty table encodes as `{}`. Functions/threads/userdata become null.
* `decode(s: string) -> any` : Parses a JSON string into a Lua value. Raises on invalid JSON.

## http
HTTP client. Available as the `http` global. All methods are async.

#### Methods
* `get(url: string, opts?: table) -> HttpResponse` : Performs a GET request, returning an [`HttpResponse`](#httpresponse).
* `post(url: string, opts?: table) -> HttpResponse` : Performs a POST request, returning an [`HttpResponse`](#httpresponse).
* `put(url: string, opts?: table) -> HttpResponse` : Performs a PUT request, returning an [`HttpResponse`](#httpresponse).
* `delete(url: string, opts?: table) -> HttpResponse` : Performs a DELETE request, returning an [`HttpResponse`](#httpresponse).
* `patch(url: string, opts?: table) -> HttpResponse` : Performs a PATCH request, returning an [`HttpResponse`](#httpresponse).

#### Options
* `headers` (`table`) : Request headers.
* `body` (`string`) : Raw request body.
* `json` (`any`) : A value serialized to a JSON body (also sets `Content-Type: application/json`). Ignored if `body` is set.
* `timeout` (`number`) : Request timeout in milliseconds (default 30000).

#### Notes
* Only `http://` and `https://` URLs are allowed. Requests to localhost, private ranges (10.x, 192.168.x, 172.16–31.x), `*.local`, and cloud metadata endpoints are blocked.

#### Example
```lua
local res = http.post("https://example.com/api", {
  json = { name = bot.name, gems = bot.gems },
  timeout = 5000,
})
if res.ok then print(res.status, res.body) end
```

## HttpResponse
The result of an [`http`](#http) call.

#### Properties
* `status` (`number`) : HTTP status code.
* `ok` (`boolean`) : True when status is 200–299.
* `body` (`string`) : Response body.
* `headers` (`table`) : Response headers.

## Storage
Persistent key/value store. `storage` is per-slot; `globalStorage` is shared across all scripts. Both expose the same methods.

#### Methods
* `get(key: string) -> any` : Returns the stored value (decoded), or nil.
* `set(key: string, value: any)` : Stores a JSON-encodable value.
* `delete(key: string) -> boolean` : Removes a key; returns whether it existed.
* `keys() -> string[]` : Returns all keys.
* `clear()` : Removes everything.

#### Notes
* Writes flush to disk at most every 2 seconds, plus a final flush on script end. A hard crash can lose writes made inside the last 2-second window.

#### Example
```lua
local runs = storage:get("runs") or 0
storage:set("runs", runs + 1)
print("run #" .. (runs + 1))
```

## datetime
Date/time helpers. Available as the `datetime` global.

#### Methods
* `unixSeconds() -> number` : Current Unix time in seconds (UTC).
* `unixMillis() -> number` : Current Unix time in milliseconds (UTC).
* `iso() -> string` : Current UTC time as an RFC 3339 string (millisecond precision).
* `format(fmt?: string) -> string` : Current UTC time formatted with a strftime pattern (default `%Y-%m-%dT%H:%M:%SZ`).
* `formatLocal(fmt?: string) -> string` : Current local time formatted (default `%Y-%m-%dT%H:%M:%S`).
* `monotonicMillis() -> number` : Milliseconds since the Unix epoch. Despite the name, this is wall-clock, not a monotonic clock.
* `dotnetTicks() -> number` : Current time as .NET ticks.

## random
Random helpers. Available as the `random` global.

#### Methods
* `integer(min: number, max: number) -> number` : Random integer in `[min, max]` (inclusive).
* `number(...) -> number` : Random float. No args → `[0,1)`; one arg `a` → `[0,a)`; two args `a,b` → `[a,b)`.
* `choice(t: table) -> any` : Random element from the array part of a table, or nil if empty.
* `shuffle(t: table) -> table` : Shuffles the array part in place (Fisher–Yates) and returns it.

## crypto
Hashing and encoding helpers. Available as the `crypto` global.

#### Methods
* `sha256(input: string) -> string` : SHA-256 hash as a hex string.
* `sha512(input: string) -> string` : SHA-512 hash as a hex string.
* `hexEncode(input: string) -> string` : Hex-encodes bytes.
* `hexDecode(input: string) -> string` : Decodes a hex string to bytes.
* `base64Encode(input: string) -> string` : Standard base64 encode.
* `base64Decode(input: string) -> string` : Standard base64 decode.
* `base64UrlEncode(input: string) -> string` : URL-safe base64 (no padding) encode.
* `base64UrlDecode(input: string) -> string` : URL-safe base64 (no padding) decode.

## regex
Regular expressions (Rust `regex` syntax — no lookaround or backreferences). Available as the `regex` global.

#### Methods
* `isMatch(pattern: string, haystack: string) -> boolean` : True if the pattern matches anywhere.
* `find(pattern: string, haystack: string) -> table` : First match as `{ match, start, end }` (1-based positions), or nil.
* `findAll(pattern: string, haystack: string) -> table[]` : All matches as `{ match, start, end }` entries.
* `replace(pattern: string, haystack: string, replacement: string) -> string` : Replaces all matches (`$1` etc. capture refs supported).
* `split(pattern: string, haystack: string) -> string[]` : Splits on the pattern.
* `escape(s: string) -> string` : Escapes regex metacharacters in a literal.

## uuid
UUID generator. Available as the `uuid` global.

#### Methods
* `new(fmt?: string) -> string` : Generates a v4 UUID. `fmt` may be `"simple"` (no hyphens) or `"urn"` (`urn:uuid:...`); default is hyphenated.

## Vector2
A 2D float vector. Construct with `Vector2.new(x, y)`. Supports `+`, `-`, `*` (vector × scalar only), `==`, and `tostring`.

#### Properties
* `x` (`number`) : X component (float).
* `y` (`number`) : Y component (float).

#### Methods
* `equals(other: Vector2) -> boolean` : True if both components of the other [`Vector2`](#vector2) match.
* `point() -> Vector2i` : Converts to a [`Vector2i`](#vector2i) by flooring each component.

#### Example
```lua
local a = Vector2.new(1.5, 2.0)
local b = a + Vector2.new(0.5, 0.0)
print(tostring(b), b:point().x)
```

## Vector2i
A 2D integer vector. Construct with `Vector2i.new(x, y)`. Supports `+`, `-`, `*` (vector × scalar only), `==`, and `tostring`.

#### Properties
* `x` (`number`) : X component (integer).
* `y` (`number`) : Y component (integer).

#### Methods
* `equals(other: Vector2i) -> boolean` : True if both components of the other [`Vector2i`](#vector2i) match.
* `position() -> Vector2` : Converts to a float [`Vector2`](#vector2).

## Threads
Cooperative background threads. They share the script's globals and are scheduled on the async runtime.

#### Methods
* `runThread(fn: function) -> number` : Starts `fn` as a background thread and returns its id. At most 5 background threads (plus the main thread) may run; exceeding the limit errors. An uncaught error in any thread stops the whole script.
* `removeThread(id: number)` : Cancels a thread by id and waits for it to finish.

#### Example
```lua
local id = runThread(function()
  while true do
    bot:collect_all()
    sleep(500)
  end
end)
sleep(10000)
removeThread(id)
```

## Config
Static game-data lookups. Available as globals.

#### Methods
* `getItemInfo(id: number) -> ItemInfo` : Returns an [`ItemInfo`](#iteminfo) by id, or nil.
* `getItemInfos() -> table` : All items as [`ItemInfo`](#iteminfo) entries, keyed by id.
* `getQuestInfo(id: string) -> QuestInfo` : Returns a [`QuestInfo`](#questinfo) by id, or nil.
* `getQuestInfos() -> table` : All quests as [`QuestInfo`](#questinfo) entries, keyed by id.
* `getItemPackInfo(id: string) -> ItemPack` : Returns an [`ItemPack`](#itempack) by id, or nil.
* `getItemPackInfos() -> table` : All item packs as [`ItemPack`](#itempack) entries, keyed by id.

## ItemInfo
Block/item definition. Returned by [`getItemInfo`](#config).

#### Properties
* `id` (`number`) : Item id.
* `name` (`string`) : Item name.
* `description` (`string`) : Item description.
* `is_seed` (`boolean`) : True if it is a seed.
* `is_tradeable` (`boolean`) : True if tradeable.
* `has_collider` (`boolean`) : True if it blocks movement.
* `is_platform` (`boolean`) : True if it is a platform.
* `health` (`number`) : Block health (hits to break).
* `growth_time` (`number`) : Seed growth time in seconds.
* `block_class` (`number`) : Block class id.
* `inventory_item_type` (`number`) : Inventory type byte.
* `level_requirement` (`number`) : Level needed to use.

## QuestInfo
Daily/tutorial quest definition. Returned by [`getQuestInfo`](#config).

#### Properties
* `id` (`string`) : Quest id.
* `display_name` (`string`) : Display name.
* `icon` (`string`) : Icon id.
* `title` (`string`) : Quest title.
* `description` (`string`) : Quest description.
* `dialogue` (`string`) : Quest dialogue text.
* `complete_message` (`string`) : Completion message.
* `quest_kind` (`string`) : Quest kind.
* `requirement_type` (`string`) : Requirement type.
* `requirement_item` (`string`) : Required item name.
* `requirement_is_seed` (`boolean`) : True if the requirement is a seed.
* `requirement_amount` (`number`) : Required amount.
* `vip_only` (`boolean`) : True if VIP-only.
* `is_tutorial` (`boolean`) : True if a tutorial quest.
* `prize_type` (`string`) : Prize type.
* `prize_is_seed` (`boolean`) : True if the prize is a seed.
* `prize_item_names` (`string[]`) : Prize item names.
* `prize_amounts` (`number[]`) : Prize amounts.
* `prize_weights` (`number[]`) : Prize weights.
* `weight` (`number`) : Selection weight.
* `xp_given` (`number`) : XP awarded.
* `min_level` (`number`) : Minimum level.
* `max_level` (`number`) : Maximum level.

## ItemPack
Shop / gacha pack definition. Returned by [`getItemPackInfo`](#config).

#### Properties
* `id` (`string`) : Pack id.
* `display_name` (`string`) : Display name.
* `category` (`string`) : Category name.
* `vip_exclusive` (`boolean`) : True if VIP-exclusive.
* `pack_type` (`string`) : Pack type (`None`, `ExtraItems`, `FreeWithAd`).
* `gacha_slot` (`number`) : Gacha slot index.
* `is_gacha` (`boolean`) : True if the pack is a gacha.
* `gem_price` (`number`) : Price in gems.
* `bytecoin_price` (`number`) : Price in bytecoins.
* `guaranteed_pool` ([`ItemPackEntry[]`](#itempackentry)) : Guaranteed reward entries.
* `gacha_pool_a` ([`ItemPackEntry[]`](#itempackentry)) : Pool-A reward entries.
* `gacha_pool_b` ([`ItemPackEntry[]`](#itempackentry)) : Pool-B reward entries.
* `availability_start` (`string`) : Availability start timestamp, or nil.
* `availability_end` (`string`) : Availability end timestamp, or nil.
* `is_active` (`boolean`) : True if currently active.
* `one_time_only` (`boolean`) : True if buyable once.
* `clan_type` (`string`) : Clan type (`Light`/`Dark`), or nil.
* `buy_limit` (`number`) : Purchase limit.

## ItemPackEntry
One reward entry inside an [`ItemPack`](#itempack) pool array.

#### Properties
* `item_id` (`number`) : Reward item id.
* `is_seed` (`boolean`) : True if the reward is a seed.
* `amount` (`number`) : Reward amount.
* `weight` (`number`) : Roll weight.
* `tier` (`number`) : Reward tier.
```
