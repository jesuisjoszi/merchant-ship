# Hooking your mod into MerchantShip

MerchantShip runs harbours: abandoned ports a player can claim, merchant ships that dock and trade, a notice
board with offers, and a quest board where the harbour hands out work. Three of those things are open to other
mods — where the ports are, what happens to them, and what work the harbour posts.

Everything here is **static**, takes **plain Java types** and **never throws**. That is on purpose: you can call
it by reflection and keep MerchantShip an optional dependency, so your mod still loads on a server without it.
A call that cannot be answered returns `false`, `null`, `0` or an empty array — never an exception.

MerchantShip also drops a plain-text copy of this reference into its own data folder on every start —
`Saves/<world>/mods/joszza_MerchantShip/MerchantShip-API.txt`, right next to `port.json` and `ship.json`.

| What you want | Class | Since |
|---|---|---|
| Find ports, read them, listen to what happens to them | `MerchantShipApi` | VERSION 1–2 |
| Put your own jobs on the harbour's quest board | `MerchantShipQuests` | VERSION 1 |

---

## 1. Ports: find them, read them, react to them

### Where is the nearest harbour?

```java
double[] out = new double[4];
if (MerchantShipApi.nearestPort(world.getName(), x, z, 256, out)) {
    // out = { middleX, middleY, middleZ, radiusInBlocks }
}
```

BetterHytale uses exactly this to make dolphins gather around harbours.

### List every port of a world

```java
for (String line : MerchantShipApi.ports(worldName)) {
    // "portId|role|title|ownerUuid|x|y|z|radius"
    //  role is ABANDONED or CLAIMED; ownerUuid is empty for an abandoned one
    String[] f = line.split("\\|", -1);
}
```

`MerchantShipApi.portAt(worldName, x, y, z)` gives you the same line for the port whose protected area contains
a block, or `null` when there is none. Good for "is this inside somebody's harbour?" checks.

### Does this player belong to that port?

```java
if (MerchantShipApi.isPortMember(worldName, portId, playerUuid)) {
    // owner or member - treat them as a local
}
```

Useful if your own mod protects or rewards things inside a harbour and you want to respect the port's own
member list instead of inventing a second one.

### Put your own content at a port

MerchantShip tells you *where*; the thing you place is yours, with your own lifecycle. Nothing of yours is ever
despawned or managed by MerchantShip.

```java
double[] at = new double[3];
if (MerchantShipApi.portSpot(worldName, portId, "charter", at)) {
    spawnMyHarbourMaster(at[0], at[1], at[2]);   // your NPC, your rules
}
```

| spot | where it is |
|---|---|
| `charter` | the charter block of a claimed port — its heart, where the owner manages everything |
| `center` | the middle of the built port, at its top |
| `origin` | the corner the port prefab was pasted at |

### React to what happens

```java
MerchantShipApi.onPortEvent("mymod", (event, payload) -> {
    // payload is "portId|worldName|extra"
    switch (event) {
        case "claimed"      -> welcome(payload);        // extra = the new owner's uuid
        case "abandoned"    -> cleanUp(payload);        // extra = why (given up, inactive, removed)
        case "shipDocked"   -> ringTheBell(payload);    // extra = berth index
        case "shipDeparted" -> waveGoodbye(payload);
    }
});
```

One listener per mod, called on the world thread right after the thing happened. Anything your listener throws
is caught and logged — a bug of yours cannot break a claim or a docking. Call
`MerchantShipApi.forgetPortEvents("mymod")` when your mod shuts down.

A natural pair: listen for `claimed`, ask for a `portSpot`, and place your own NPC or decoration there; listen
for `abandoned` and take it away again.

---

## 2. The harbour contract board

A port's notice board carries contracts: *bring 64 oak trunks to the ship, get paid on delivery*. A player takes
one, gathers the goods, hands them in at the board, and the harbour pays in coins.

Your mod can put its own jobs on that board. MerchantShip does the boring half — posting, expiry, the hand-in
transaction, the payout — and calls you when a job is done, so you can add whatever reward fits your mod:
reputation, an item, the next step of your own quest line.

### Register a kind of job

```java
MerchantShipQuests.registerDelivery(
    "mymod",            // your mod id, lower case, no colon
    "driftwood",        // unique inside your mod
    "Wood_Oak_Trunk",   // the item to bring, as the server knows it
    32, 64,             // roll range: a posting asks for 32..64 of them
    4,                  // coins per item (the server's own percentage still applies)
    3,                  // weight: how often this one is picked next to the others
    null, null);        // your lang keys for the title and the second line, or null for ours
```

Call it again with the same ids to change it. It returns `false` when an argument is unusable or the server does
not know that item — no exception, no surprise.

### Get told when one is filled

```java
MerchantShipQuests.onComplete("mymod", (player, contract) -> {
    // contract is "mymod:driftwood"; the coins are already paid
    giveMyOwnReward(player);
});
```

### Put one specific job on a board

```java
MerchantShipQuests.post(worldName, portId, "mymod", "story1",
                        "Ore_Iron", 24, 400, 120); // 24 iron ore, 400 coins, 120 minutes
```

Good for story beats or a reward for something your own mod tracks. Pass `null` as the port and it lands on the
first live port of that world. The posting shows up within a minute.

### Ask what a player is carrying

```java
for (String line : MerchantShipQuests.active(playerId)) {
    // "mymod:driftwood|Wood_Oak_Trunk|12|64|256"
    //  contract        item            done count coins
}
```

### Clean up

```java
MerchantShipQuests.unregister("mymod");   // on shutdown; postings already up run out by themselves
MerchantShipQuests.contractCount();       // cheap "is the board alive" probe
```

---

## 3. Calling it without a compile-time dependency

```java
try {
    Class<?> api = Class.forName("com.joszza.merchantship.api.MerchantShipQuests");
    api.getMethod("registerDelivery", String.class, String.class, String.class,
                  int.class, int.class, int.class, int.class, String.class, String.class)
       .invoke(null, "mymod", "driftwood", "Wood_Oak_Trunk", 32, 64, 4, 3, null, null);
} catch (ClassNotFoundException e) {
    // MerchantShip is not installed - carry on without the board
}
```

If you would rather compile against it, add the jar as `compileOnly` and declare MerchantShip under
`OptionalDependencies` in your manifest. Wrap the first call in a `try/catch (NoClassDefFoundError)` and you are
safe either way.

Listeners are `java.util.function.BiConsumer`, so they work through reflection too — nothing in the signatures
comes from MerchantShip.

---

## 4. Two things worth knowing

**Versions.** Both classes carry a `VERSION` constant. Methods get added, never changed or removed, so code
written against VERSION 1 keeps working. Read the constant if you want to be sure something newer is there:

```java
int v = MerchantShipApi.VERSION;   // 2 = ports, portAt, isPortMember, portSpot, onPortEvent
```

**Server settings win.** An admin can turn the contract board off or scale the payout (`/mship admin` →
Settings → *Harbour contracts*; the settings page has sub-tabs and a search box). Your jobs stay registered
either way; they simply are not posted while the board is off. Treat the reward as a suggestion, not a promise.

---

Questions, or a hook you are missing? Open an issue — this API grows with what people actually build on it.
